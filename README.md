<div align="center">

# Windows Server 2025 · Heimdomäne

**Projektdokumentation** · Active-Directory-Domäne auf Windows Server 2025 als VM unter Proxmox VE, gesichert mit Proxmox Backup Server, aufgebaut nach gängiger Unternehmenspraxis

![Status](https://img.shields.io/badge/Status-in_Umsetzung-f0a020?style=flat-square)
![Windows Server](https://img.shields.io/badge/Windows_Server-2025_Standard-333333?style=flat-square)
![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-9.2-333333?style=flat-square)
![PBS](https://img.shields.io/badge/Backup-PBS-333333?style=flat-square)

</div>

| | |
|---|---|
| **Projekt** | Eigene AD-Domäne für das Heimnetz: zentrale Anmeldung an allen Windows-Clients, getrennte Benutzer- und Admin-Konten, GPOs, Windows LAPS |
| **Zeitraum** | Planung ab 21.10.2026, Umsetzung ab 10.11.2026 |
| **Umsetzung** | Matija Tasic |
| **Stand** | 12.11.2026, Windows Server (Evaluation) im VLAN 10 erreichbar, Grundkonfiguration läuft |
| **Plattform** | Windows Server 2025 Standard (Desktop Experience), Proxmox VE 9.2 auf Dell Wyse 5070, OPNsense, Proxmox Backup Server |

> [!TIP]
> Öffentliche Fassung: Domäne, interne Adressen, Gerätenamen und Kontonamen sind durch Platzhalter ersetzt (z. B. `example.com`, `10.0.x.x`). Zugangsdaten, Keys und Kennungen sind nicht enthalten.


## Inhalt

| | | |
|---|---|---|
| 01 · [Zusammenfassung](#1-zusammenfassung) | 06 · [Umsetzung Schritt für Schritt](#6-umsetzung-schritt-für-schritt) | 11 · [Offene Punkte](#11-offene-punkte) |
| 02 · [Ausgangslage und Ziel](#2-ausgangslage-und-ziel) | 07 · [Tests](#7-tests) | 12 · [Journal](#12-journal) |
| 03 · [Projektumfang](#3-projektumfang) | 08 · [Backup und Restore](#8-backup-und-restore) | |
| 04 · [Aufbau: Netz, VM, Konten](#4-aufbau) | 09 · [Betrieb](#9-betrieb) | |
| 05 · [Entscheidungen](#5-entscheidungen) | 10 · [Risiken und Rollback](#10-risiken-und-rollback) | |

---

## 1. Zusammenfassung

Die Windows-Clients im Heimnetz bekommen eine zentrale Anmeldung über eine eigene Active-Directory-Domäne. Der Domänencontroller (DC) läuft als VM auf dem Proxmox-Host, in einem eigenen Server-VLAN hinter der OPNsense, und wird jede Nacht vom bestehenden Proxmox Backup Server mitgesichert.

| Baustein | Umsetzung |
|---|---|
| ⚫ **Server** | Windows Server 2025 Standard, Desktop Experience, VM `dc01` auf Proxmox VE |
| 🟢 **Domäne** | `home.example.com`, NetBIOS `HOME` (Vorschlag), ein DC mit AD DS und DNS |
| 🟡 **Netz** | DC im Server-VLAN 10, Clients im VLAN 20, Firewall-Regeln nur für AD-Ports |
| 🔵 **Konten** | Alltagskonto, Client-Admin, Domänen-Admin getrennt; eingebauter Administrator nur als Notfallkonto |
| 🔴 **Absicherung** | Windows LAPS, AD-Papierkorb, Anmeldeverbot für Domänen-Admins an Clients, Backup über PBS |

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 2. Ausgangslage und Ziel

**Ausgangslage**

- Vier Windows-11-Pro-Clients mit lokalen Konten bzw. Microsoft-Konten, jedes Gerät einzeln verwaltet.
- Proxmox-Host `pve01` (Dell Wyse 5070, 24 GB RAM, 256 GB SSD, Gemini-Lake-CPU ohne AVX2) mit der OPNsense als VM.
- Netz bereits in VLANs segmentiert (Projekt OPNsense-Segmentierung).
- Proxmox Backup Server `pbs01` seit 22.10.2026 in Betrieb, Restore getestet. Damit existiert ein echtes Rollback.

**Ziel**

- Anmeldung als Benutzer oder als Admin an allen vier Geräten mit denselben Domänenkonten.
- Aufbau wie in Unternehmensumgebungen üblich, soweit es im Homelab einen echten Mehrwert hat.
- Jeder Schritt mit Rollback, jede Änderung dokumentiert.
- Doku in zwei Fassungen: intern mit echten Daten, öffentlich zensiert.

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 3. Projektumfang

**Im Umfang**

- DC-VM, AD DS und DNS, Zeitsynchronisation
- Server-VLAN 10, Firewall-Regeln, DNS-Umstellung VLAN 20
- OU-Struktur, Benutzer- und Admin-Konzept, Gruppen
- GPOs: Kennwortrichtlinie, Client-Admins, Anmeldebeschränkung, Windows LAPS, BitLocker-Schlüssel im AD, PIN-Anmeldung
- Domänenbeitritt der vier Clients mit Profilübernahme
- Backup, isolierter Restore-Test, Betriebsroutine

**Außerhalb des Umfangs**

- VPN auf der OPNsense (eigenes Projekt, später für das Notebook außer Haus)
- Zweiter DC (bewusst nicht, siehe Risiken)
- Nextcloud, Dateiserver, Zertifizierungsstelle (AD CS)

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 4. Aufbau

### 4.1 Netz- und Adressplan

| Netz | VLAN | Gateway | DHCP | Zweck |
|---|---|---|---|---|
| 10.0.10.0/24 | 10 „SERVER“ | 10.0.10.1 (OPNsense) | kein DHCP, nur feste IPs | Domänencontroller |
| 10.0.20.0/24 | 20 „PRIVAT“ | 10.0.20.1 (OPNsense) | .100–.200 | Clients, DNS = DC |

VLAN 10 existiert nur in Proxmox (`vmbr1`, VLAN-aware) und auf der OPNsense (`vtnet1`). DC und OPNsense hängen beide an `vmbr1`, Switches und AP bleiben unverändert.

### 4.2 Geräte

| Gerät | Hostname | IP | Rolle |
|---|---|---|---|
| Domänencontroller (VM) | `DC01` (Vorschlag) | 10.0.10.10 (Vorschlag) | AD DS, DNS, Zeitquelle der Domäne |
| Client 1 | offen | 10.0.20.10 | Client, Pilot |
| Client 2 | offen | 10.0.20.20 | Client |
| Client 3 | offen | 10.0.20.30 | Client |
| Notebook | offen | 10.0.20.40 | Client, auch außer Haus (zwischengespeicherte Anmeldung) |
| Proxmox-Host | `pve01` | – | Hypervisor |
| Backup-Server | `pbs01` | 10.0.1.6 | Proxmox Backup Server, Storage `pbs01-prod` |

Hostnamen: maximal 15 Zeichen (NetBIOS-Grenze), nur Buchstaben, Ziffern, Bindestrich.

### 4.3 Proxmox-VM `dc01`

| Einstellung | Wert | Begründung |
|---|---|---|
| Maschine / BIOS | q35, OVMF (UEFI) mit EFI-Disk | aktueller Standard für Windows |
| CPU | `x86-64-v2-AES`, 2 Kerne | Gemini Lake ohne AVX2 → kein `x86-64-v3`; `host` wegen VBS-Problemen nicht empfohlen. Server 2025 braucht SSE4.2 und POPCNT, beides enthalten |
| RAM | 4 GB fest, kein Ballooning | bei Trägheit auf 6 GB erhöhen |
| Disk | VirtIO SCSI single, 60 GB, Cache Default, Discard, IO thread, SSD-Emulation | |
| Netzwerk | VirtIO an `vmbr1`, VLAN-Tag 10 | |
| Storage-Pool | `local-lvm` (lvmthin, 131 GiB frei) | Snapshots belegen im Thin-Pool zusätzlich Platz, daher nicht dauerhaft liegen lassen |
| VM-ID | 101 | |
| Sonstiges | vmgenid aktiv, QEMU Guest Agent, virtio-win (stable) | vmgenid schützt AD bei Restore/Snapshot-Rücksprung |
| Start | Start at boot, Order 2, Startup delay 60 s (OPNsense = Order 1) | DC startet erst, wenn das Netz steht |
| Maschinentyp | gepinnt auf `pc-q35-11.0+pve2` (automatisch bei Windows) | stabile virtuelle Hardware über Proxmox-Updates hinweg |

> [!WARNING]
> virtio-win 0.1.285 hatte unter Server 2025 einen vioscsi-Fehler bei hoher Last. Vor dem Download die Known Issues im Proxmox-Wiki „Windows VirtIO Drivers“ prüfen.

### 4.4 OU-Struktur

```
home.example.com
└── OU=Heim
    ├── OU=Benutzer      Alltagskonten
    ├── OU=Admins        Client-Admin- und Domänen-Admin-Konten
    ├── OU=Clients       Computerkonten (Ziel von redircmp)
    └── OU=Gruppen       Sicherheitsgruppen
```

### 4.5 Konten- und Rechtekonzept (Tiering light)

| Konto | Beispielname | Gruppe | Darf sich anmelden an | Zweck |
|---|---|---|---|---|
| Alltagskonto | offen (z. B. `user`) | Domänen-Benutzer | allen Clients | E-Mail, Browser, Alltag. **Kein** Admin |
| Client-Admin | offen (z. B. `adm-user`) | `GG-Client-Admins` → per GPO lokaler Admin | allen Clients | Software installieren, „Als Administrator ausführen“ |
| Domänen-Admin | offen (z. B. `da-user`) | Domänen-Admins, Protected Users | nur DC01 | Verwaltung von AD, DNS, GPOs |
| Eingebauter Administrator | `Administrator` | Domänen-Admins | nur DC01, nur im Notfall | Notfallkonto (Break-Glass), Kennwort nur im Passwortmanager |
| Lokaler Admin je Client | `Administrator` (LAPS) | lokal | jeweiliger Client | Notzugang ohne DC, Kennwort rotiert per LAPS |

**Warum so:** Wird das Alltagskonto kompromittiert (Phishing, Browser), hat der Angreifer keine Admin-Rechte. Ein Domänen-Admin, der sich nie an Clients anmeldet, hinterlässt dort keine Anmeldedaten, die man abgreifen könnte. Das ist die Kernidee des Tier-Modells von Microsoft, auf zwei Ebenen reduziert.

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 5. Entscheidungen

| Datum | Entscheidung | Begründung / Hinweis |
|---|---|---|
| 21.10.2026 | Domäne `home.example.com` | Subdomain der eigenen Domain, nicht `.local` und nicht die Hauptdomain (zeigt auf GitHub Pages → Split-DNS-Probleme). Nach dem Hochstufen praktisch nicht mehr änderbar |
| 21.10.2026 | DNS Option A: DC ist DNS-Server für VLAN 20 | Microsoft-Empfehlung. Externe Anfragen leitet der DC an die OPNsense (Unbound) weiter |
| 21.10.2026 | DC im eigenen Server-VLAN 10 | Trennung von Clients und Server. Preis: OPNsense liegt im Pfad jeder Anmeldung (siehe Risiken) |
| 21.10.2026 | Rollback über PBS | `pbs01`, Storage `pbs01-prod`, Job „All“ täglich 03:00 sichert DC01 automatisch mit |
| 21.10.2026 | VPN als eigenes Projekt | |
| 10.11.2026 | Windows Server 2025 Standard, Desktop Experience, ein DC | Key für 2025 vorhanden. Desktop Experience ↔ Core ist nach der Installation nicht mehr umstellbar |
| 10.11.2026 | Doku in zwei Fassungen auf GitHub (bestätigt 11.11.2026) | intern (privat) mit echten Daten, öffentlich zensiert, erzeugt per Skript mit Leak-Check |
| 12.11.2026 | Betrieb vorerst als **Evaluation** (180 Tage, 1 Rearm) | Der vorhandene Key ließ sich nicht aktivieren (0xC004C003). Umwandlung per Snapshot zurückgerollt. Folge: Ein Eval-DC lässt sich nicht in Standard umwandeln, mit gültigem Key wird später ein neuer DC aufgebaut und die Rollen werden migriert |
| offen | Funktionsebene **Windows Server 2016** (`WinThreshold`) | Vorschlag: Hochstufen geht jederzeit, zurück nicht. 2025 bringt in der Heimdomäne kaum Nutzen und schließt spätere DCs mit 2022 aus |
| offen | NetBIOS-Name `HOME` | Vorschlag. Nach dem Hochstufen nicht mehr änderbar |
| offen | AD-Papierkorb aktivieren | Vorschlag. Einzelne gelöschte Objekte zurückholen, ohne die ganze VM zurückzuspielen |

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 6. Umsetzung Schritt für Schritt

Jede Phase beginnt mit einem Sicherungspunkt und endet mit einem Test. Befehle in PowerShell als Administrator, sofern nicht anders angegeben.

### Phase 0 · Vorbereitung

| Nr | Schritt | Status |
|---|---|---|
| 1 | Freien Speicher und Storage-Pool auf `pve01` prüfen (`pvesm status`) | erledigt: `local-lvm` für die VM, `local` für ISOs |
| 2 | CPU-Flags dokumentieren: `lscpu \| grep -ow -e aes -e avx2` | offen |
| 3 | ISOs: Windows Server 2025 Build 26100.32230, Evaluation, de-de (Umwandlung in Phase 3); virtio-win 0.1.302 stable. Microsoft veröffentlicht für die Eval-ISO keine Prüfsumme, SHA256 `64bd88c4cc99068ffc9727cc08f2e56ddc5707d175323f5aa0a8ab1d63d0815f` notiert | erledigt |
| 4 | Bestandsaufnahme Clients: Windows-Version, Konto-Typ, IP fest/DHCP, BitLocker | teilweise (alle Win 11 Pro) |
| 5 | Datensicherung aller Benutzerprofile inkl. BitLocker-Schlüssel | offen |
| 6 | Lokales Fallback-Admin-Konto auf jedem Client prüfen | offen |

### Phase 1 · Design festlegen

Offen: Hostnamen (Aufgabe 8), NetBIOS-Name (9), Kontenkonzept bestätigen (10), Firewall-Regeln final (11), Funktionsebene, Papierkorb.

### Phase 2 · OPNsense und Netz

Sicherung vorher: OPNsense-Konfig exportieren, VM-Snapshot OPNsense, `/etc/network/interfaces` auf `pve01` sichern.

1. VLAN-Gerät auf `vtnet1`, Tag 10, Interface `SERVER` 10.0.10.1/24, kein DHCP.
2. Aliase und Regeln (umgesetzt 12.11.2026). Der bestehende Alias `ADMIN_PCS` aus der OPNsense-Segmentierung (Zugriff FRITZ!Box und Management) bleibt unverändert; für den DC gibt es eigene Aliase, damit sich beide Zugriffe unabhängig voneinander einschränken lassen.

| Alias | Typ | Inhalt |
|---|---|---|
| `DC01` | Host | 10.0.10.10 |
| `DC_ADMIN_PCS` | Host | 10.0.20.10, .20.20, .20.30, .20.40 |
| `AD_TCP` | Port | 53, 88, 135, 389, 445, 464, 636, 3268:3269, 49152:65535 |
| `AD_UDP` | Port | 53, 88, 123, 389, 464 |
| `DC_ADMIN_PORTS` | Port | 3389 (RDP), 9389 (ADWS für RSAT/PowerShell) |
| `WEB_PORTS` | Port | 80, 443 |
| `PRIVATE_NETS` | Netz | bestehend: alle privaten Adressbereiche |

`AD_PORTS` aus der Planung ist in `AD_TCP` und `AD_UDP` aufgeteilt, damit jede Regel nur das Protokoll erlaubt, das AD tatsächlich nutzt.

**Interface PRIVAT** – die neuen Regeln stehen vor der bestehenden Regel „PRIVAT – Block – interne Netze“:

| Regel | Protokoll | Quelle → Ziel : Port |
|---|---|---|
| PRIVAT – AD TCP – DC01 | TCP | PRIVAT net → `DC01` : `AD_TCP` |
| PRIVAT – AD UDP – DC01 | UDP | PRIVAT net → `DC01` : `AD_UDP` |
| PRIVAT – ICMP – DC01 | ICMP | PRIVAT net → `DC01` |
| PRIVAT – Admin – DC01 | TCP | `DC_ADMIN_PCS` → `DC01` : `DC_ADMIN_PORTS` |

**Interface SERVER** – in genau dieser Reihenfolge:

| Regel | Aktion | Protokoll | Quelle → Ziel : Port |
|---|---|---|---|
| SERVER – DNS – Firewall | Pass | TCP/UDP | SERVER net → This Firewall : 53 |
| SERVER – NTP – Firewall | Pass | UDP | SERVER net → This Firewall : 123 |
| SERVER – ICMP – Firewall | Pass | ICMP | SERVER net → This Firewall |
| SERVER – Block – interne Netze | Block (Log) | any | SERVER net → `PRIVATE_NETS` |
| SERVER – Internet – Web | Pass | TCP | SERVER net → any : `WEB_PORTS` |

IoT, Gäste, Lab und Management erreichen SERVER nicht (bestehende Block-Regeln), WAN → SERVER ist geblockt.

> [!NOTE]
> In der neuen Regelansicht der OPNsense landen neue Regeln am Ende ihres Interface-Blocks. Verschieben: Regel markieren, in der Zielzeile auf „←“ klicken (markierte Regeln davor einfügen). Dabei darauf achten, dass nur die gewünschte Regel markiert ist. Beim Kopieren einer Regel vor dem Speichern Interface, Quelle und Ziel prüfen.

3. DHCP VLAN 20: Domänenname `home.example.com` als Suchdomäne mitgeben (DNS-Server erst in Phase 6 umstellen).

**Rollback:** Konfig-Export zurückspielen oder Snapshot `vor_vlan10` der OPNsense.

Proxmox: `vmbr1` ist bereits VLAN-aware (`bridge-vids 2-4094`), keine Änderung nötig. VLAN-10-Broadcasts gehen dadurch getaggt auch über `nic0` zum physischen Switch.

### Phase 3 · VM und Installation

1. VM nach [4.3](#43-proxmox-vm-dc01) anlegen.
2. Setup: Edition **Standard (Desktop Experience)**. Kein Datenträger sichtbar → „Treiber laden“, virtio-win-ISO, `vioscsi\2k25\amd64`.
3. Nach dem ersten Start: `virtio-win-guest-tools.exe` installieren (Netzwerk, Balloon, Guest Agent).
4. Grundkonfiguration:

```powershell
Rename-Computer -NewName "DC01" -Restart

# nach dem Neustart
$if = (Get-NetAdapter | Where-Object Status -eq 'Up').Name
New-NetIPAddress -InterfaceAlias $if -IPAddress 10.0.10.10 -PrefixLength 24 -DefaultGateway 10.0.10.1
Set-DnsClientServerAddress -InterfaceAlias $if -ServerAddresses 10.0.10.1
Set-TimeZone -Id "W. Europe Standard Time"
```

5. Alle Windows-Updates installieren, bis keine mehr angeboten werden (behebt bekannte Server-2025-DC-Fehler: Netzwerkprofil „Öffentlich“ nach Neustart, doppelte Schema-Einträge).
6. **Evaluation-Version umwandeln und aktivieren** (Quelle des ISOs ist das Microsoft Evaluation Center). Muss **vor** dem Hochstufen passieren, ein DC mit Eval-Version lässt sich nicht mehr umwandeln. Key nur aus dem Passwortmanager, nicht in die Doku.

```powershell
DISM /online /Get-TargetEditions          # ServerStandard muss aufgeführt sein
DISM /online /Set-Edition:ServerStandard /ProductKey:<KEY> /AcceptEula
# Neustart, danach:
slmgr /dlv                                # darf kein "Eval" mehr enthalten
slmgr /ato                                # Aktivierung, Status muss "Lizenziert" sein
```
   Schlägt die Aktivierung fehl (z. B. 0xC004C003, Key gesperrt): Snapshot `vor_edition` zurückrollen und als Evaluation weiterarbeiten. Prüfen mit `slmgr /dlv`: Lizenzstatus, Restlaufzeit, Rearm-Anzahl. PowerShell speichert eingegebene Befehle (auch Keys) im Verlauf, danach löschen: `Remove-Item (Get-PSReadLineOption).HistorySavePath`.
7. **Snapshot „Basis-Installation“** in Proxmox.

**Beobachtungen nach der Installation:**
- Proxmox zeigt die RAM-Belegung mit 100 % an. Ohne Ballooning-Gerät sieht Proxmox nicht in die VM hinein; den echten Verbrauch zeigt der Task-Manager in der VM.
- In der ersten Stunde nach der Installation liegt die CPU-Last nahe 100 % (Defender, .NET-Optimierung, Indexierung). Auf der Gemini-Lake-CPU ist das erwartbar.
- Guest Agent prüfen: `qm agent 101 ping` auf dem Proxmox-Host.

**Rollback:** VM löschen und neu anlegen; nach Schritt 7: Snapshot zurück.

### Phase 4 · AD DS und DNS

Voraussetzung: Phase 3 komplett, Hostname, IP, NetBIOS-Name und Funktionsebene entschieden.

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
  -DomainName "home.example.com" `
  -DomainNetbiosName "HOME" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM-Kennwort")
```

DSRM-Kennwort sofort in den Passwortmanager.

Nach dem Neustart:

```powershell
# DNS des DC: eigene IP zuerst, dann Loopback
Set-DnsClientServerAddress -InterfaceAlias $if -ServerAddresses 10.0.10.10,127.0.0.1

# Weiterleitung an Unbound
Set-DnsServerForwarder -IPAddress 10.0.10.1

# Reverse-Zonen
Add-DnsServerPrimaryZone -NetworkId "10.0.10.0/24" -ReplicationScope Forest
Add-DnsServerPrimaryZone -NetworkId "10.0.20.0/24" -ReplicationScope Forest

# Aufräumen alter Einträge
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval 7.00:00:00 -ApplyOnAllZones

# Zeit: PDC-Emulator holt die Zeit von der OPNsense
w32tm /config /manualpeerlist:"10.0.10.1,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync

# AD-Papierkorb (nicht rückgängig zu machen, aber ohne Nebenwirkungen)
Enable-ADOptionalFeature -Identity "Recycle Bin Feature" `
  -Scope ForestOrConfigurationSet -Target "home.example.com"
```

Danach: Unbound-Domain-Override `home.example.com` → 10.0.10.10 auf der OPNsense. **Vorher prüfen:** Ist `example.com` beim Provider per DNSSEC signiert, kann Unbound die Antworten verwerfen. Dann `domain-insecure`/`private-domain` für die Zone setzen.

**Test:** siehe [Kapitel 7](#7-tests), danach **Backup „AD frisch“** (manuelles PBS-Backup + Snapshot).

**Rollback:** Snapshot „Basis-Installation“.

### Phase 5 · Benutzer, Gruppen, GPOs

```powershell
$base = "DC=home,DC=example,DC=com"
New-ADOrganizationalUnit -Name "Heim" -Path $base
"Benutzer","Admins","Clients","Gruppen" | ForEach-Object {
  New-ADOrganizationalUnit -Name $_ -Path "OU=Heim,$base"
}
redircmp "OU=Clients,OU=Heim,$base"

New-ADGroup -Name "GG-Client-Admins" -GroupScope Global -GroupCategory Security `
  -Path "OU=Gruppen,OU=Heim,$base"

# Kennwortrichtlinie (Default Domain Policy)
Set-ADDefaultDomainPasswordPolicy -Identity "home.example.com" `
  -MinPasswordLength 14 -ComplexityEnabled $true -MaxPasswordAge 0 `
  -LockoutThreshold 10 -LockoutDuration 00:10:00 -LockoutObservationWindow 00:10:00
```

Konten anlegen (Namen offen), Domänen-Admin zusätzlich in `Protected Users`.

| GPO | Verknüpft mit | Inhalt |
|---|---|---|
| Default Domain Policy | Domäne | Kennwort- und Sperrrichtlinie (oben) |
| `C-Client-Admins` | OU Clients | Eingeschränkte Gruppen / Lokale Benutzer und Gruppen: `GG-Client-Admins` → Administratoren |
| `C-Anmeldung-Verboten-DA` | OU Clients | „Lokal anmelden verweigern“ und „Anmelden über RDP verweigern“ für Domänen-Admins, Organisations-Admins |
| `C-LAPS` | OU Clients | Windows LAPS: Sicherung im AD, Länge 20, Rotation 30 Tage |
| `C-BitLocker-AD` | OU Clients | Wiederherstellungsschlüssel im AD sichern (nur falls BitLocker aktiv) |
| `C-PIN-Anmeldung` | OU Clients | „Anmeldung mit PIN aktivieren“, sonst nur Kennwort für Domänenkonten |

Windows LAPS vorbereiten:

```powershell
Update-LapsADSchema
Set-LapsADComputerSelfPermission -Identity "OU=Clients,OU=Heim,$base"
```

Vorgehen vor der Umsetzung gegen die aktuelle Microsoft-Doku prüfen.

**Rollback:** GPO-Verknüpfung deaktivieren (nicht löschen), `Backup-GPO -All -Path <Ordner>` vor jeder Änderung.

### Phase 6 · DNS-Umstellung und Clients

Reihenfolge: erst nur der Pilot (Client 1), dann einzeln die übrigen.

1. Profil sichern, lokales Fallback-Admin-Konto prüfen.
2. DNS des Pilot-Clients manuell auf 10.0.10.10 setzen. Internet und `nslookup home.example.com` testen.
3. Domänenbeitritt:

```powershell
Add-Computer -DomainName "home.example.com" -Restart
```

4. Testen: Anmeldung Alltagskonto, Client-Admin per „Als Administrator ausführen“, `gpresult /r`, LAPS-Kennwort abrufbar (`Get-LapsADPassword`), Offline-Anmeldung.
5. Profil übernehmen (Werkzeug offen, Aufgabe 30).
6. Erst danach: DHCP VLAN 20 → DNS-Server 10.0.10.10, feste IPs anpassen, übrige Clients einzeln beitreten lassen.

**Rollback:** DNS zurück auf 10.0.20.1; Client mit lokalem Fallback-Konto zurück in die Arbeitsgruppe (`Remove-Computer -WorkgroupName WORKGROUP -Restart`).

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 7. Tests

| Nr | Test | Befehl / Vorgehen | Erwartung | Ergebnis |
|---|---|---|---|---|
| T1 | DC-Gesundheit | `dcdiag /v` | keine Fehler | offen |
| T2 | SRV-Einträge | `Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.home.example.com` | DC01 | offen |
| T3 | DC-Suche vom Client | `nltest /dsgetdc:home.example.com` | DC01 | offen |
| T4 | Ports vom Client | `Test-NetConnection 10.0.10.10 -Port 389` (88, 445, 53, 135) | `TcpTestSucceeded: True` | offen |
| T5 | Netzwerkprofil nach Neustart | `Get-NetConnectionProfile` | `DomainAuthenticated` | offen |
| T6 | Zeit | `w32tm /query /status` | Quelle 10.0.10.1 | offen |
| T7 | Externe Auflösung über DC | `Resolve-DnsName microsoft.com` vom Client | Antwort | offen |
| T8 | GPOs | `gpresult /r` am Client | Client-GPOs angewendet | offen |
| T9 | Anmeldeverbot | Anmeldung als Domänen-Admin am Client | verweigert | offen |
| T10 | LAPS | `Get-LapsADPassword -Identity <Client>` | Kennwort vorhanden | offen |
| T11 | Offline-Anmeldung | Notebook ohne Netz | Anmeldung per Cache | offen |
| T12 | Papierkorb | Test-Benutzer löschen, `Restore-ADObject` | Benutzer zurück | offen |
| T13 | Secure Channel | `Test-ComputerSecureChannel` am Client | `True` | offen |
| T14 | Restore | DC01 aus PBS als neue VM-ID **ohne Netzwerk** zurückspielen | startet, `dcdiag` lokal ok | offen |

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 8. Backup und Restore

- Kein eigener Job nötig: Der PVE-Backup-Job (Selection „All“, täglich 03:00, Snapshot-Modus, `pbs01-prod`) nimmt DC01 automatisch mit. Nach dem Anlegen der VM im nächsten Lauf prüfen.
- Aufbewahrung über den PBS-Prune-Job: keep-last 3, daily 7, weekly 4, monthly 6. Das liegt unter der Tombstone-Lebensdauer von 180 Tagen, alle Backups sind also nutzbar.
- Der QEMU Guest Agent friert das Dateisystem über VSS ein, die Sicherung ist damit konsistent.
- **Restore nur als ganze VM.** Nie parallel zum laufenden DC im selben Netz starten.
- **Achtung:** Ein Restore auf einen älteren Stand passt nicht mehr zu den Computerkonto-Kennwörtern der Clients. Reparatur am Client: `Test-ComputerSecureChannel -Repair -Credential HOME\<Domänen-Admin>`.
- Einzelne gelöschte Objekte zurückholen: über den AD-Papierkorb statt über einen VM-Restore.

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 9. Betrieb

| Wann | Was |
|---|---|
| monatlich (Wartungsfenster) | Snapshot, Windows-Updates DC01, Neustart, danach T1, T5, T6 |
| monatlich | PBS-Backup-Status von DC01 prüfen, Ereignisprotokolle (Directory Service, DNS Server, System) |
| vor jeder GPO-Änderung | `Backup-GPO -All` |
| halbjährlich | Restore-Test T14 |

**Hinweis:** Während der DC neu startet, haben die Clients in VLAN 20 keine Namensauflösung und damit kein Internet. Updates deshalb abends oder nachts.

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 10. Risiken und Rollback

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| Ein DC auf einem Host | Host oder OPNsense aus → keine Anmeldung außer per Cache, keine GPOs | bewusst akzeptiert; PBS-Restore; Fallback-Konten |
| DNS hängt am DC | DC aus → Clients ohne Internet | Wartungsfenster; Rollback: DNS VLAN 20 → 10.0.20.1 |
| OPNsense im Anmeldepfad (VLAN 10) | Regel-Fehler wirken wie AD-Fehler | Tests T3/T4; Firewall-Log prüfen |
| Profilverlust beim Beitritt | Daten/Einstellungen fehlen | Sicherung vorher, Pilot zuerst |
| Zeitabweichung > 5 Min. | Kerberos schlägt fehl | NTP-Kette OPNsense → DC → Clients, T6 |
| DC-Rollback nach Beitritten | Vertrauensstellung defekt | `Test-ComputerSecureChannel -Repair`; Papierkorb statt Restore |
| Speicher 256-GB-SSD | VM, Snapshots, lokale Backups voll | Aufgabe 1; Snapshots nicht dauerhaft liegen lassen |
| Thin-Pool überbucht (Summe aller Thin-Disks > Pool, kein Autoextend) | Pool real voll → VMs frieren ein, Datenverlust möglich | Füllstand `lvs pve/data` (Data%) im Blick, Grenze ca. 80 %; Snapshots nach Erfolg löschen; Backups liegen auf PBS, nicht im Pool |
| Domänen-/NetBIOS-Name | nach Hochstufen praktisch fix | vor Phase 4 final entscheiden |
| Edition | Desktop Experience ↔ Core nicht umstellbar | bei Installation entschieden |
| Funktionsebene 2025 | nur noch DCs mit 2025 möglich | Vorschlag 2016 |
| Lizenzierung | AD-Zugriff erfordert laut Microsoft CALs | Lizenzumfang des Keys prüfen |
| Evaluation läuft ab (180 Tage + 1 Rearm, ab 12.11.2026) | Server fährt nach Ablauf regelmäßig herunter, kein Weg zu Standard bei einem DC | Ablaufdatum im Kalender; rechtzeitig gültigen Key beschaffen, neuen DC aufbauen, Rollen migrieren, alten DC herabstufen |

**Rollback-Grundsatz:** Vor jeder Phase Konfig-Export bzw. Snapshot, Stand im Journal notieren. Immer zuerst ein Pilot, dann einzeln.

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 11. Offene Punkte

- [x] Storage-Pool `local-lvm` und VM-ID 101 (Aufgaben 1, 16)
- [ ] CPU-Flags dokumentieren (2)
- [ ] Bestandsaufnahme Clients vervollständigen (4)
- [ ] Hostnamen DC und Clients (8)
- [ ] NetBIOS-Name (9)
- [ ] Kontenkonzept und Kontonamen bestätigen (10)
- [ ] Firewall-Regeln final inkl. 9389 und Alias `ADMIN_PCS` (11)
- [ ] Funktionsebene 2016 oder 2025
- [ ] Werkzeug für Profilübernahme (30)
- [ ] CAL-Lizenzierung prüfen
- [ ] DNSSEC der Hauptdomain prüfen (für Unbound-Override)

<p align="right"><a href="#inhalt">↑ nach oben</a></p>

---

## 12. Journal

| Datum | Eintrag |
|---|---|
| 21.10.2026 | Planung, Scope festgelegt, Projekt pausiert bis PBS fertig |
| 22.10.2026 | PBS fertig, Restore-Test bestanden, Pause aufgehoben |
| 10.11.2026 | Entscheidung Windows Server 2025 statt 2022. Planung ergänzt: Papierkorb, Funktionsebene 2016, Port 9389, Anmeldeverbot DA, BitLocker/PIN-GPO, Lizenzierung ergänzt |
| 10.11.2026 | Umsetzung gestartet, Download Windows Server 2025 Standard. Doku-Struktur intern/öffentlich angelegt |
| 11.11.2026 | Festgelegt: Name in der öffentlichen Fassung, Kontenkonzept mit drei Konten plus Notfallkonto, Repo-Aufteilung intern/öffentlich |
| 11.11.2026 | Speicher geprüft: VM auf `local-lvm`, ISOs auf `local`. VM-ID 101 festgelegt |
| 11.11.2026 | Installationsmedium: Evaluation-ISO, Umwandlung in Standard mit vorhandenem Key vor dem Hochstufen |
| 11.11.2026 | ISOs auf `local` hochgeladen: Server 2025 (de-de) und virtio-win 0.1.302 |
| 11.11.2026 | VM 101 `dc01` angelegt und geprüft (`qm config`): vmgenid gesetzt, Start-Reihenfolge OPNsense → DC01 |
| 12.11.2026 | Windows Server 2025 Standard Evaluation (Desktopdarstellung) installiert, vioscsi-Treiber im Setup geladen, virtio-win-guest-tools installiert, Guest Agent erreichbar. Noch ohne Netz (VLAN 10 fehlt, nur APIPA-Adresse) |
| 12.11.2026 | Phase 2: Sicherung (Konfig-Export, Snapshot `vor_vlan10`, `interfaces`-Backup). VLAN-Gerät `vlan0.10`, Interface SERVER 10.0.10.1/24. Aliase und Regeln für PRIVAT und SERVER angelegt, Internet in VLAN 20 nach dem Übernehmen geprüft |
| 12.11.2026 | DC-IP 10.0.10.10/24 gesetzt, Tests ping Gateway, DNS-Auflösung und HTTPS (`tnc … -port 443`) bestanden. RDP aktiviert und getestet |
| 12.11.2026 | Snapshot `vor_edition`. Umwandlung Eval → ServerStandard per DISM erfolgreich, Aktivierung mit 0xC004C003 abgelehnt. Rollback auf `vor_edition`, Status: ServerStandardEval, lizenziert, 180 Tage, 1 Rearm. Thin-Pool-Belegung geprüft (ca. 16 %) |

