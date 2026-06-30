# Remote-Zugriff auf das Homelab via Tailscale

## Motivation

Das Proxmox-Homelab läuft hinter einem Kabelanschluss ohne öffentliche IPv4-Adresse (DS-Lite), wodurch klassisches Port-Forwarding für VPN-Lösungen wie WireGuard nicht möglich ist. Tailscale baut auf WireGuard auf, übernimmt aber NAT-Traversal automatisch — kein Port-Forwarding nötig.

Um den Proxmox-Host selbst sauber zu halten, läuft Tailscale in einem eigenen LXC-Container als Subnet Router fürs gesamte Homelab-Netz.

---

## Proxmox-Host vorbereiten

```bash
apt update && apt upgrade -y
pveversion
pvesm list local --content vztmpl
```

---

## LXC Container erstellen

```bash
pct create 102 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname tailscale \
  --memory 512 \
  --cores 2 \
  --rootfs local-lvm:10 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --features nesting=1 \
  --password
```

512 MB RAM, 2 Cores, 10 GB Storage, unprivileged.

---

## TUN-Device-Zugriff konfigurieren

Tailscale benötigt Zugriff auf das TUN-Device, das im unprivilegierten Container standardmäßig nicht verfügbar ist.

```bash
nano /etc/pve/lxc/102.conf
```

Ergänzt:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Container danach neu gestartet:

```bash
pct stop 102
pct start 102
pct enter 102
```

---

## Tailscale installieren

```bash
apt update && apt upgrade -y
apt install curl -y
curl -fsSL https://tailscale.com/install.sh | sh
```

### Statische IP

Für Tailscale selbst nicht zwingend nötig, im lokalen Netz aber sinnvoll für konsistente Erreichbarkeit:

```bash
pct set 102 --net0 name=eth0,bridge=vmbr0,ip=192.168.x.x/24,gw=192.168.x.x
```

---

## Troubleshooting: TUN-Device fehlte zunächst

```bash
ls -la /dev/net/tun
```

Device war im Container nicht vorhanden — Ursache war die fehlende Host-Config aus dem vorherigen Schritt. Nach Ergänzen von `lxc.cgroup2.devices.allow` und `lxc.mount.entry` in `/etc/pve/lxc/102.conf` sowie Container-Neustart war das Device verfügbar.

```bash
systemctl start tailscaled
```

---

## Subnet Router aktivieren

```bash
tailscale up --advertise-routes=192.168.0.0/24
```

### Troubleshooting: IP-Forwarding deaktiviert

Warnung beim Ausführen des obigen Befehls wegen deaktiviertem IP-Forwarding — ohne das kann der Container keinen Traffic zwischen Tailscale-Netz und LAN weiterleiten.

Behoben, zunächst zur Laufzeit:

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1
```

Dauerhaft persistiert:

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

Nach erneutem Ausführen von `tailscale up --advertise-routes=192.168.0.0/24` liefert der Befehl einen Authentifizierungslink (`https://login.tailscale.com/a/xxxxxxxxxxxxxx`), über den der Container im Tailscale-Account freigeschaltet wird.

---

## Ergebnis

Der Tailscale-Container fungiert als Subnet Router für das gesamte Homelab-Netz (`192.168.0.0/24`). Damit ist das komplette LAN — inklusive AdGuard- und Nextcloud-Container — von außen ohne Port-Forwarding und ohne öffentliche IPv4-Adresse erreichbar, sobald die Route im Tailscale-Adminbereich freigegeben wurde.
