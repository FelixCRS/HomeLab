# AdGuard Home Setup – LXC Container auf Proxmox

## Ziel

DNS-Adblocker für das gesamte Netzwerk — alle Geräte profitieren automatisch, keine Client-Installation nötig. Diente als Einstieg in LXC: unkomplizierte Installation, schnell sichtbares Ergebnis.

Vorteile:
- Einfache Web-UI
- Geringer Ressourcenverbrauch
- Netzwerkweiter Werbe-/Tracker-Schutz ohne Geräte-Konfiguration

---

## Warum LXC und nicht VM

AdGuard Home läuft als simpler Go-Binary, braucht nur Netzwerkzugriff und einen Port — keine Kernel-Module, kein privilegierter Zugriff nötig. Klassischer LXC-Fall.

Grundsatz: 90 % der Homelab-Dienste lassen sich als LXC betreiben (leichte Dienste ohne eigene Kernel-Module). VMs sind nur nötig wenn ein eigener Kernel oder tiefe Systemisolation gebraucht wird (z. B. Windows, Active Directory, OPNsense).

---

## LXC Container erstellen

```bash
# Verfügbare Templates aktualisieren
pveam update

# Debian-Templates anzeigen
pveam available | grep -i debian

# Template herunterladen
pveam download local debian-13-standard_13.0-1_amd64.tar.zst
```

```bash
pct create 100 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname adguard \
  --memory 512 \
  --cores 1 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --password
```

- `100` — Container-ID (frei wählbar, muss einmalig sein)
- `--memory 512` — 512 MB RAM reicht für AdGuard locker
- `--rootfs local-lvm:8` — 8 GB Festplattenspeicher
- `--net0 ... ip=dhcp` — Netzwerkkarte hängt am virtuellen Switch vmbr0, IP per DHCP
- `--unprivileged 1` — Sicherheitsmodus, Container läuft ohne Root-Rechte auf dem Host

```bash
pct start 100
pct enter 100
```

### Troubleshooting: Systemd 257

> **Warnung:** `Systemd 257 detected. You may need to enable nesting.`

Systemd (Prozessmanager in Debian 13) versucht im unprivilegierten Container standardmäßig blockierte Funktionen zu nutzen. Nesting erlaubt das.

```bash
pct set 100 --features nesting=1
```

Für AdGuard selbst zunächst unkritisch, aber als Lektion direkt bei der Nextcloud-Container-Erstellung präventiv gesetzt.

---

## System vorbereiten

```bash
apt update && apt upgrade -y
ip a
ping -c 3 google.com
apt install curl -y
```

`curl` wird benötigt um Daten von URLs herunterzuladen bzw. Anfragen an Webserver zu schicken — hier für das AdGuard-Installationsskript.

---

## AdGuard Home installieren

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

```bash
/opt/AdGuardHome/AdGuardHome -s start
```

Status prüfen:

```
2026/06/19 14:54:52.582138 [info] service: running
```

WebUI aufrufen: `http://<Container-IP>:3000`

### Erstkonfiguration

- **Admin-Weboberfläche:** Schnittstelle: Alle Interfaces (0.0.0.0), Port: 80 → danach erreichbar unter `http://<Container-IP>` ohne Portangabe
- **DNS-Server:** Schnittstelle: Alle Interfaces (0.0.0.0), Port: 53 (Standard-DNS-Port) → alle Geräte im Netzwerk können den Container als DNS nutzen
- Benutzername: `admin`, Passwort frei wählbar

---

## Troubleshooting: Login schlägt fehl

Symptom: Login mit korrektem Passwort scheitert.

Ursache: Leerzeichen hinter dem Passwort-Wert in der Config-Datei.

```bash
nano /opt/AdGuardHome/AdGuardHome.yaml
```

Leerzeichen hinter dem Passwort-Eintrag entfernen, Dienst neu starten.

---

## DNS netzwerkweit verteilen

### Problem: Vodafone Station erlaubt keine manuellen DNS-Einstellungen

Der ISP-Router lässt keine direkte Konfiguration eines alternativen DNS-Servers zu. Zwei Optionen:

1. **DNS manuell pro Gerät eintragen** (IP des AdGuard-Containers) — temporäre Lösung zum Testen, aktuell im Einsatz
2. **Zwischenrouter** (GL.iNet / TP-Link, OpenWrt-kompatibel) zwischen Vodafone Station und Netzwerk schalten, AdGuard-DNS dort netzwerkweit über DHCP verteilen — saubere, dauerhafte Lösung (geplant)

---

## Statische IP für den Container

### Problem

DHCP vergibt bei jedem Neustart potenziell eine neue IP-Adresse über den Router — der DNS-Eintrag am PC würde dann nicht mehr stimmen.

### Lösung

```bash
nano /etc/network/interfaces
```

```
auto eth0
iface eth0 inet static
    address 192.168.x.x
    netmask 255.255.255.0
    gateway 192.168.x.1
```

`iface eth0 inet static` — keine DHCP-Anfrage mehr, IP ist fest und ändert sich nicht mehr bei Neustarts.

---

## Firewall

Geplante Hierarchie für den gesamten Homelab: **Default Drop** auf Datacenter-Ebene, darunter nur die jeweils notwendigen Ports pro Container freigeben. Reihenfolge der Regelanwendung in Proxmox: Datacenter → Node → CT/VM.

---

## Ergebnis

AdGuard läuft stabil, DNS-Eintrag am Test-PC erfolgreich gesetzt und funktionsfähig (Werbeblockierung netzwerkweit aktiv für konfigurierte Geräte). Diente als Vorlage für die spätere Nextcloud-Container-Erstellung (gleiche Template-Basis, gleiche Sicherheits- und Netzwerklogik).
