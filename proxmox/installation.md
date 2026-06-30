# Proxmox Installation auf dem Thin Client

## Ziel

Aufsetzen einer Proxmox-Hypervisor-Umgebung auf einem Thin Client als Basis für das gesamte Homelab (LXC-Container für AdGuard, Nextcloud und weitere Dienste).

---

## Hardware-Vorbereitung

Thin Client geöffnet, zusätzliches 8 GB RAM-SODIMM nachgerüstet, um genug Ressourcen für mehrere parallel laufende Container bereitzustellen.

---

## Installation

Proxmox-ISO per Balena Etcher auf einen USB-Stick geschrieben, davon gebootet und auf dem Thin Client installiert.

Nach der Installation ist die WebUI vom Main-PC aus erreichbar:

```
https://<deine-IP>:8006
```

### Netzwerk-Check

```bash
ip a
ping -c 5 google.com
```

Verbindung funktionierte direkt — der Thin Client erhielt problemlos eine IP per DHCP vom Router.

---

## Subscription-Nag entfernen

Proxmox zeigt ohne kostenpflichtiges Abo bei jedem WebUI-Login einen Hinweis-Dialog. Für den Privatgebrauch unnötig, daher entfernt:

```bash
sed -i.bak "s/NotFound/Active/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

---

## Paketquellen auf No-Subscription umstellen

Standardmäßig sind die Paketquellen auf das kostenpflichtige Enterprise-Repository konfiguriert, das ohne gültiges Abo keine Updates liefert. Umstellung auf das kostenlose No-Subscription-Repository:

```bash
# PVE-Version und Debian-Codename prüfen
pveversion
# pve-manager/9.x = trixie, pve-manager/8.x = bookworm

# Vorher prüfen welche Dateien existieren
ls /etc/apt/sources.list.d/

# Enterprise-Repo (braucht Abo) deaktivieren
echo "# deactivated" > /etc/apt/sources.list.d/pve-enterprise.list
echo "# deactivated" > /etc/apt/sources.list.d/pve-enterprise.sources

# Ceph-Enterprise deaktivieren
echo "# deactivated" > /etc/apt/sources.list.d/ceph.list
echo "# deactivated" > /etc/apt/sources.list.d/ceph.sources

# Kostenloses Repo aktivieren (Codename aus pveversion anpassen!)
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-sub.list
```

---

## System updaten

```bash
apt update && apt dist-upgrade -y
reboot
```

Nach dem Neustart Version prüfen:

```bash
pveversion
```

```
pve-manager/9.2.3/d0fde103346cf89a (running kernel: 7.0.6-2-pve)
```

---

## Virtualisierungs-Troubleshooting

### Problem: KVM nicht verfügbar

Symptom: Virtualisierung im BIOS deaktiviert, VM-Erstellung dadurch nicht möglich.

### Fix

Thin Client neu starten → `F1` → Security/Advanced → Intel Virtualization Technology → Enable → speichern → Proxmox neu starten.

Relevant für später geplante VM-Workloads (z. B. Kali Linux), bei denen im Gegensatz zu LXC-Containern ein eigener Kernel benötigt wird.

---

## Architektur-Entscheidung: LXC vs. VM

| | LXC | VM |
|---|---|---|
| Kernel | Teilt sich den Host-Kernel | Eigener Kernel |
| Ressourcenverbrauch | Gering | Höher |
| Einsatzzweck | Leichte Dienste ohne Kernel-Module | Tiefe Systemisolation, andere Betriebssysteme |
| Beispiele | AdGuard, Nextcloud, WireGuard, Uptime Kuma | Windows, Active Directory, OPNsense |

Grundsatz für dieses Homelab: rund 90 % der Dienste lassen sich als LXC betreiben — entsprechend ist LXC der Standardansatz, VM nur die Ausnahme bei spezifischem Bedarf.

---

## Geplante Firewall-Strategie

Hierarchie: **Default Drop** auf Datacenter-Ebene, darunter selektive Portfreigaben pro Container je nach tatsächlichem Bedarf (z. B. für Nextcloud nur SSH 22 und HTTPS 443). Regelanwendung in der Reihenfolge Datacenter → Node → CT/VM.

---

## Ergebnis

Proxmox lief stabil auf dem Thin Client, mit aktuellem No-Subscription-Repository und ohne Lizenzhinweise. Basis für die anschließende Erstellung der LXC-Container (AdGuard, Nextcloud) und künftiger VM-Workloads.
