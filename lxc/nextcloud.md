# Nextcloud Setup – LXC Container auf Proxmox

## Ziel

Netzwerkseitige Backup-Lösung für Ableton-Projektdateien (~200+ Dateien, ca. 3 GB), die bisher ausschließlich auf der SSD des Haupt-PCs lagen.

Am Thin Client befindet sich eine externe HDD (kein Speicherverlust bei Stromausfall, im Gegensatz zu RAM/SSD-Cache-Szenarien).

Nextcloud übernimmt sowohl die LAN-Sicherung als auch später den Cloud-Upload — ein zentraler Punkt, kein separater Sync-Job auf dem PC nötig.

Cloud-Anbindung folgt später (Kandidaten: Backblaze B2, Hetzner Storage Box — in jedem Fall EU-Anbieter, DSGVO-konform).

---

## Warum LXC und nicht VM

Nextcloud braucht keine eigenen Kernel-Module, läuft als klassischer Webserver-Stack (nginx + PHP + MariaDB) — klassischer LXC-Fall. Gleiche Entscheidungslogik wie beim AdGuard-Container.

- Debian 13 LXC (selbes Template wie AdGuard)
- nginx als Webserver
- PHP 8.4
- MariaDB als Datenbank
- Nextcloud selbst per Direct-Download (nicht über Paketmanager — dort oft veraltete Versionen)

---

## LXC Container erstellen

Ressourcen großzügiger als beim AdGuard-Container, da Nextcloud mehr braucht:

- RAM: 1024 MB (Nextcloud + PHP + MariaDB zusammen)
- CPU: 2 Cores
- Disk: 10 GB für das System selbst — Datenspeicher kommt separat als Mountpoint (externe HDD)
- Netzwerk: vmbr0, erst DHCP, danach statische IP
- unprivileged: 1
- nesting=1 direkt gesetzt (Lektion aus dem AdGuard-Setup wegen Systemd 257)

Alle Befehle per SSH vom Main-PC ausgeführt, kein Umweg über die Proxmox-WebUI nötig (SSH-Key-Auth, geringere Angriffsfläche, funktioniert auch wenn die WebUI mal hängt).

```bash
pct create 101 local:vztmpl/debian-13-standard_13.0-1_amd64.tar.zst \
  --hostname nextcloud \
  --memory 1024 \
  --cores 2 \
  --rootfs local-lvm:10 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 1 \
  --features nesting=1 \
  --password

pct start 101
pct enter 101
```

### Statische IP festlegen

```bash
nano /etc/network/interfaces
```

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.x.x
    netmask 255.255.255.0
    gateway 192.168.x.1
```

### DNS-Eintrag in AdGuard Home

AdGuard läuft bereits und ist als DNS am PC eingetragen — lokaler Hostname kann vergeben werden:

`AdGuard WebUI → Filters → DNS Rewrites → Add DNS Rewrite`

`nextcloud.home` → IP des Nextcloud-Containers, damit ist Nextcloud im LAN über `http://nextcloud.home` statt über die IP erreichbar.

---

## Stack installieren

```bash
apt update && apt upgrade -y
apt install curl -y
apt install nginx mariadb-server php php-fpm php-mysql php-xml php-mbstring php-curl php-zip php-gd php-intl php-bcmath php-gmp -y
```

- **nginx** — Webserver, nimmt HTTP-Anfragen entgegen und gibt Nextcloud zurück
- **mariadb-server** — Datenbank, speichert Metadaten (Nutzer, Dateistruktur, Settings) — nicht die Dateien selbst
- **php + php-fpm** — Nextcloud ist in PHP geschrieben, php-fpm führt den Code aus

### MariaDB absichern

```bash
mariadb-secure-installation
```

> **Problem:** `mysql_secure_installation` nicht gefunden — ab MariaDB 11.x wurde das Skript zu `mariadb-secure-installation` umbenannt. Inhaltlich identisch, betrifft alle neueren Debian-Systeme mit MariaDB 11.x aus den Paketquellen.

### Datenbank und User anlegen

```bash
mariadb -u root -p
```

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'passwort';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

- `utf8mb4` unterstützt alle Zeichen inkl. Emojis — Nextcloud-Anforderung
- `@'localhost'` — User darf sich nur lokal verbinden, nicht von außen (minimale Rechte, minimale Angriffsfläche)
- `GRANT ALL ... ON nextcloud.*` — volle Rechte, aber nur auf die nextcloud-Datenbank

### Nextcloud herunterladen

```bash
cd /var/www
curl -O https://download.nextcloud.com/server/releases/latest.zip
apt install unzip -y
unzip latest.zip
chown -R www-data:www-data nextcloud
```

`chown` setzt den Besitzer auf `www-data` — der User unter dem nginx und php-fpm laufen. Ohne das hätte der Webserver keine Schreibrechte und Nextcloud könnte keine Dateien anlegen.

---

## nginx konfigurieren

```bash
nano /etc/nginx/sites-available/nextcloud
```

Funktionierende Config (nach Troubleshooting, siehe unten):

```nginx
upstream php-handler {
    server unix:/var/run/php/php8.4-fpm.sock;
}

server {
    listen 80 default_server;
    server_name nextcloud.home 192.168.0.76;
    root /var/www/nextcloud;

    index index.php index.html;

    location = /robots.txt { allow all; log_not_found off; access_log off; }
    location = /.well-known/carddav { return 301 $scheme://$host/remote.php/dav; }
    location = /.well-known/caldav  { return 301 $scheme://$host/remote.php/dav; }

    location / {
        rewrite ^ /index.php;
    }

    location ~ ^\/(?:index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+)\.php(?:$|\/) {
        fastcgi_split_path_info ^(.+?\.php)(\/.*|)$;
        fastcgi_pass php-handler;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ \.php$ { return 404; }

    location ~* \.(?:css|js|svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;
        access_log off;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

- `upstream php-handler` — definiert wohin nginx PHP-Anfragen weiterleitet, über einen Unix-Socket statt Netzwerk-Port (schneller, sicherer, von außen nicht erreichbar)
- `listen 80 default_server` — nginx hört auf Port 80 und beantwortet Anfragen auch wenn sie nicht exakt auf `server_name` matchen
- `root` — zeigt nginx wo die Nextcloud-Dateien liegen
- die `location`-Blöcke leiten `.php`-Anfragen über den Socket an php-fpm weiter, alles andere wird direkt aus dem Dateisystem ausgeliefert
- `nginx -t && systemctl reload nginx` — Config auf Syntax prüfen, dann ohne Verbindungsabbruch neu laden

**Request-Flow:** Browser → nginx → php-fpm führt den Nextcloud-PHP-Code aus → gibt HTML zurück → nginx → Browser. nginx selbst kann kein PHP ausführen, das übernimmt php-fpm.

### Troubleshooting: Redirect-Schleife

Die ursprüngliche einfache Config (`try_files $uri $uri/ /index.php$request_uri;`) führte zu einer internen Redirect-Schleife (`rewrite or internal redirection cycle ... /index.php/index.php/login`). Grund: die `try_files`-Direktive war für die komplexere Nextcloud-Routing-Logik nicht ausreichend. Gelöst durch die vollständige, offizielle Nextcloud-nginx-Config (siehe oben) mit dedizierten `location`-Blöcken für PHP-Routen, statische Assets und WebDAV-Redirects.

### Troubleshooting: Default-Page statt Nextcloud

Symptom: Aufruf der Server-IP zeigte die nginx-Default-Page statt Nextcloud, obwohl die Nextcloud-Config korrekt war.

Ursache: Zwei `server`-Blöcke lauschten beide auf Port 80 — der generische Default-Block (`server_name _;`, `root /var/www/html`) griff bei Aufruf per IP, weil die Nextcloud-Config ursprünglich nur auf `server_name nextcloud.home` reagierte. Gelöst durch `default_server` in der Nextcloud-Config und mehrere `server_name`-Einträge (Hostname und IP).

---

## trusted_domains konfigurieren

Nach Erstaufruf meldet Nextcloud "Zugriff über eine nicht vertrauenswürdige Domain", wenn der Server per IP statt per konfiguriertem Hostnamen aufgerufen wird.

```bash
nano /var/www/nextcloud/config/config.php
```

```php
'trusted_domains' =>
array (
  0 => 'nextcloud.home',
  1 => '192.168.0.76',
),
```

### Syntax-Validierung

PHP-Configs lassen sich vor dem Neuladen auf Syntaxfehler prüfen, ohne sie auszuführen:

```bash
php -l /var/www/nextcloud/config/config.php
```

`-l` steht für *lint* — reiner Syntax-Check. Analoge Tools für andere Dienste: `nginx -t`, `apache2ctl -t`, `sshd -t`, `bash -n script.sh`.

---

## Upload-Limit erhöhen

Standard-Limit von Nextcloud/PHP liegt bei 512 MB — einige Ableton-Projektdateien (`.als`) überschreiten das.

```bash
nano /etc/php/8.4/fpm/php.ini
```

```ini
upload_max_filesize = 10G
post_max_size = 10G
memory_limit = 512M
```

```bash
systemctl restart php8.4-fpm
```

---

## Backup-Lösung für Ableton-Dateien

Ableton speichert Projekte als `.als`-Dateien, die eigentlichen Audiodaten liegen als WAV-Dateien im Projektordner unter `Samples`. Gesamtordner: 200+ Dateien, ca. 3 GB.

### 3-2-1-Regel

Ziel ist grundsätzlich die 3-2-1-Backup-Regel. Mit diesem Setup ist aktuell 2-2-0 erreicht: zwei Kopien auf zwei verschiedenen Speichermedien (PC-SSD und externe HDD am Thin Client). Eine spätere Cloud-Anbindung (Backblaze B2 oder Hetzner Storage Box) vervollständigt die Regel auf 3-2-1.

### Nextcloud Desktop Client einrichten

Download: Nextcloud Files für Windows 10+ (64 bit) auf dem Main-PC.

Nach der Installation:

1. Anmelden → Server-Adresse `http://192.168.0.76` eingeben
2. Lokalen Ordner auf den Ableton-Projektordner zeigen (`C:\Users\felix\Desktop\00_Ableton`)
3. "Alle Daten vom Server synchronisieren" wählen
4. "Lokale Daten behalten" — bestehende Dateien werden nicht gelöscht, sondern hochgeladen
5. Synchronisierungsgrenze auf 3000 MB setzen (Projektordner ist ~2,7 GB groß)

Der Client läuft danach im Hintergrund und überwacht den Ordner kontinuierlich. Jede neue oder geänderte Datei wird automatisch synchronisiert. Erfolgreich synchronisierte Dateien erhalten ein grünes Häkchen-Symbol im Windows Explorer. Die Dateien bleiben dabei vollständig lokal erhalten — der Client hält beide Seiten konsistent, es findet kein Verschieben statt.

---

## Erweiterbarkeit

Nextcloud ist durch ein modulares App-System erweiterbar — Kalender, Kontakte, kollaborative Dokumentenbearbeitung oder ein internes Chat-System lassen sich per App Store nachinstallieren. Für dieses Projekt liegt der Fokus auf der Dateisynchronisation, das System wäre aber ohne zusätzlichen Infrastrukturaufwand zu einer vollständigen Groupware-Lösung erweiterbar.

---

## Offene Punkte / nächste Schritte

- Externe HDD als Mountpoint in den LXC durchreichen, Nextcloud-Datenpfad dorthin umziehen
- Cloud-Anbieter für externes Backup festlegen (Backblaze B2 vs. Hetzner Storage Box)
- External Storage in Nextcloud für den gewählten Cloud-Anbieter einrichten (S3-kompatibel bzw. WebDAV)
- Firewall-Regeln auf Proxmox-Ebene für den Nextcloud-Container setzen (nur benötigte Ports freigeben, z. B. SSH 22, HTTP/HTTPS)
