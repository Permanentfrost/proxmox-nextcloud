# Nextcloud on Proxmox (Ubuntu Server, Docker Compose) — LAN-only Starter Guide

This Guide walks you from a clean Ubuntu Server VM to a tidy, LAN-only Nextcloud. It includes a ~50 GB data cap option, performance tweaks, logging, backups, and a quick primer on **Users/Groups/Access control** for your production rollout.

Excellent for a relatively quick test if you like it. Then decide to upgrade system or add more ressources in Proxmox :)

> **Replace these placeholders when you copy-paste:**  
> - `192.168.1.150` → your VM’s LAN IP  
> - `ChangeMe_Root!`, `ChangeMe_DB!`, `ChangeMe_Admin!` → strong secrets you choose

---

## At-a-glance

- **Platform:** Proxmox VM → Ubuntu Server 24.04 (headless, no GUI)  
- **Spec (test):** 2 vCPU · 3–4 GB RAM · ~60 GB disk  
- **Stack:** `nextcloud:apache` + `mariadb:lts` + `redis:alpine` via **Docker Compose**  
- **Access (LAN):** `http://<VM_IP>:8080` (no internet exposure unless you forward ports)  

---

## 0) VM & OS (recap)

Install **Ubuntu Server 24.04** (no desktop), enable **OpenSSH**.  
Network: bridge the VM NIC to your LAN (so it gets a `192.168.1.x` address).

---

## 1) Install Docker CE + Compose (official apt repo)

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Optional: run docker without sudo (re-login or use newgrp)
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker run hello-world
```

---

## 2) (Optional) Hard-cap data at ~50 GB

If you want to **enforce** a 50 GB ceiling for Nextcloud’s data directory:

```bash
sudo mkdir -p /srv
sudo fallocate -l 50G /srv/nextcloud-data.img
sudo mkfs.ext4 -L nextclouddata /srv/nextcloud-data.img
sudo mkdir -p /srv/nextcloud-data
echo '/srv/nextcloud-data.img /srv/nextcloud-data ext4 loop,defaults 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Make writable for web user (UID 33: www-data)
sudo chown -R 33:33 /srv/nextcloud-data
```

> Skip this section if you don’t need a hard cap; Docker volumes are fine for tests.

---

## 3) Compose the stack

Create the project and compose file:

```bash
sudo mkdir -p /opt/nextcloud
sudo chown -R $USER:$USER /opt/nextcloud
cd /opt/nextcloud
nano docker-compose.yml
```

Paste (edit passwords/IPs as needed):

```yaml
services:
  db:
    image: mariadb:lts
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    environment:
      MARIADB_ROOT_PASSWORD: ChangeMe_Root!
      MARIADB_DATABASE: nextcloud
      MARIADB_USER: nextcloud
      MARIADB_PASSWORD: ChangeMe_DB!
    volumes:
      - db:/var/lib/mysql

  redis:
    image: redis:alpine
    restart: unless-stopped

  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    depends_on: [db, redis]
    # LAN-only bind to the VM's LAN IP (replace 192.168.1.150 with your VM IP):
    ports:
      - "192.168.1.150:8080:80"
    environment:
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ChangeMe_DB!
      MYSQL_HOST: db
      NEXTCLOUD_ADMIN_USER: admin
      NEXTCLOUD_ADMIN_PASSWORD: ChangeMe_Admin!
      NEXTCLOUD_TRUSTED_DOMAINS: "192.168.1.150"
      REDIS_HOST: redis
      PHP_MEMORY_LIMIT: 512M
      PHP_UPLOAD_LIMIT: 2G
    volumes:
      - nextcloud:/var/www/html
      - apps:/var/www/html/custom_apps
      - config:/var/www/html/config
      # Choose ONE data option:
      # - data:/var/www/html/data               # A) Simple (no hard cap)
      - /srv/nextcloud-data:/var/www/html/data # B) Hard-capped 50G (Section 2)

volumes:
  db:
  nextcloud:
  apps:
  config:
  # data:
```

Bring it up:

```bash
docker compose up -d
docker compose logs -f app
```

Login at **http://192.168.1.150:8080** with `admin / ChangeMe_Admin!`.

> If you see “permission denied to Docker socket”, either run `sudo docker …` or add yourself to the `docker` group (`sudo usermod -aG docker $USER && newgrp docker`).

---

## 4) LAN-only & firewall

- You **are not exposed to the internet** unless your router forwards ports to this VM.  
- Binding the port to the VM IP (`192.168.1.150:8080:80`) avoids accidental binds.  
- Optional UFW rules:
  ```bash
  sudo ufw allow OpenSSH
  sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp
  sudo ufw enable
  ```
- You can also enforce LAN in **Proxmox VM Firewall** (enable Firewall on the VM, allow TCP 8080 from `192.168.1.0/24`, drop the rest).

---

## 5) Switch background jobs to **Cron** (recommended)

**Why:** AJAX runs only when users visit ⇒ delayed housekeeping & previews. **Cron** runs every 5 min ⇒ smoother UX & timely tasks.

1) In Nextcloud: **Administration → Basic settings → Background jobs → Cron**  
2) On the host:
```bash
( crontab -l 2>/dev/null; \
  echo "*/5 * * * * docker exec -u www-data nextcloud-app php -f /var/www/html/cron.php >/dev/null 2>&1" ) | crontab -
```

---

## 6) Caching & locking (performance & data integrity)

- **APCu** = local PHP cache → fewer DB hits  
- **Redis** = file locking & cache → prevents race conditions on concurrent edits

Apply once:
```bash
docker compose exec -u www-data app php occ config:system:set memcache.local --value='\OC\Memcache\APCu'
docker compose exec -u www-data app php occ config:system:set memcache.locking --value='\OC\Memcache\Redis'
docker compose exec -u www-data app php occ config:system:set redis host --value=redis
docker compose exec -u www-data app php occ config:system:set redis port --value=6379 --type=integer
```

---

## 7) Logging & audit (visibility)

Send NC logs to syslog (great for centralizing later):
```bash
docker compose exec -u www-data app php occ config:system:set log_type --value=syslog
docker compose exec -u www-data app php occ config:system:set syslog_tag --value=Nextcloud
docker compose exec -u www-data app php occ app:enable admin_audit
```
Later, forward host syslog to your log server (rsyslog/Graylog/ELK).

---

## 8) Basic hardening (quick wins)

- **Disable user self-registration** (Admin → Security).  
- **Enforce 2FA** (TOTP) for admin (Profile → Security).  
- **Strong admin password**; keep apps minimal.  
- **Sharing policy:** If you don’t need public links, disable them (Admin → Sharing).  
- **Set default quotas** for new users (Admin → Users → Default quota).

---

## 9) User & access control (for “production”)

### A) Users & Groups (built-in)
- Create users: **Admin → Users → “+ New user”** (set username, display name, email, quota).  
- Create **Groups** (e.g., `Family`, `Finance`, `Media`) and assign users.  
- Quotas: define per user or set a **Default quota** (e.g., 5 GB) for new users.

### B) Group folders (team/household shares)
Use the **Group Folders** app to create shared libraries with fine-grained permissions (Read, Write, Create, Delete, Share).

1) Apps → enable **“Group folders”**.  
2) Admin → **Group folders**:
   - Create folder (e.g., `Family Shared`, `Finance Vault`).  
   - Assign groups and set permissions.  
   - Optional: set **quota** and **retention** for the folder.

> Group folders avoid per-user ownership hassles for shared data and enforce permissions at the group level.

### C) External storage (optional)
If you later add a disk/NAS, enable **External storage support** app and mount it for certain users/groups (CIFS/SMB, SFTP, etc.).  
Keep in mind: external storage should be reliable and backed up; performance depends on the backend.

### D) Roles / App access
- Nextcloud doesn’t have full RBAC, but you can **limit app availability** per group (Apps → gear icon → “Restrict to groups”).  
- For admin-only features, keep admin accounts separate from daily users.

### E) Policies to set on day one
- **Disable public link sharing** unless needed.  
- **Minimum password length/strength** (Password policy app).  
- **Default quotas** to stop one user from filling the disk.  
- **Enforce 2FA** for admins (and later, all users).

---

## 10) Backups & updates

### A) What to back up
- **Database** (`nextcloud` DB in MariaDB)  
- **Data directory** (`/srv/nextcloud-data` if you used the hard cap; otherwise the `data` Docker volume)  
- **Config** (`/var/www/html/config` volume)

### B) Simple backup examples

**DB dump:**
```bash
# adjust passwords/paths as needed
docker exec db sh -c 'exec mysqldump -unextcloud -pChangeMe_DB! nextcloud' \
  | gzip > /opt/nextcloud/db-$(date +%F).sql.gz
```

**Config & data (bind mount case):**
```bash
tar -czf /opt/nextcloud/config-$(date +%F).tgz -C /var/lib/docker/volumes .
tar -czf /opt/nextcloud/data-$(date +%F).tgz -C /srv nextcloud-data
```

Store on your backup disk/Proxmox backup storage. Test restores periodically.

### C) Update the stack (careful & repeatable)
```bash
# Optionally put Nextcloud in maintenance mode first:
docker compose exec -u www-data app php occ maintenance:mode --on

docker compose pull
docker compose up -d

# Run any DB migrations:
docker compose exec -u www-data app php occ upgrade

# Turn maintenance mode off:
docker compose exec -u www-data app php occ maintenance:mode --off
```

Create a **Proxmox VM snapshot** before major upgrades for easy rollback.

---

## 11) HTTPS (later)

Even on LAN, HTTPS protects creds on Wi-Fi and avoids browser quirks.

### Option A) Reverse proxy on the same VM (Caddy)
- Run a small Caddy container on ports 80/443, proxy to `nextcloud-app:80`.
- For pure LAN, use Caddy’s **internal CA** (`tls internal`) and import Caddy’s root cert on your devices.  
- Then set in Nextcloud:
  ```bash
  docker compose exec -u www-data app php occ config:system:set overwriteprotocol --value=https
  docker compose exec -u www-data app php occ config:system:set overwritehost --value=cloud.lan
  docker compose exec -u www-data app php occ config:system:set trusted_proxies 0 --value=172.18.0.0/16
  ```
  (Adjust `trusted_proxies` to your Docker network or proxy IP.)

### Option B) Nginx/Traefik or your edge router
Terminate TLS at your proxy, forward to the app, set the same `overwrite*` and `trusted_proxies` keys.

---

## 12) Sanity checks & quick tests

- **Admin → Overview:** security & setup warnings should be green after Cron runs (give it ~10 min).  
- Upload a **1–2 GB file** (verifies `PHP_UPLOAD_LIMIT` & storage).  
- **nmap** from another LAN host: `nmap -p 8080 <VM_IP>` (open only on LAN).  
- From mobile on cellular (not Wi-Fi): `http://<your_public_ip>:8080` should **not** connect.  
- Router: confirm **no port forward/UPnP** to this VM.

---

## 13) Troubleshooting (common)

- **Permission denied on `/var/run/docker.sock`:**  
  Use `sudo docker …` or `sudo usermod -aG docker $USER && newgrp docker`.
- **Can’t write data dir (bind mount):**  
  `sudo chown -R 33:33 /srv/nextcloud-data`
- **Trusted domains error:**  
  ```bash
  docker compose exec -u www-data app php occ config:system:get trusted_domains
  docker compose exec -u www-data app php occ config:system:set trusted_domains 1 --value=192.168.1.150
  ```
- **Slow UI / “file is locked” errors:**  
  Ensure Redis locking is configured (Section 6).

---

## 14) Appendix: Handy commands & afterthoughs

```bash
# Start/stop/recreate
cd /opt/nextcloud
docker compose up -d
docker compose down

# Logs
docker compose logs -f app
docker compose logs -f db

# Upgrade Nextcloud (image-based)
docker compose pull && docker compose up -d
docker compose exec -u www-data app php occ upgrade

# Check versions & config
docker compose exec -u www-data app php occ -V
docker compose exec -u www-data app php occ config:system:get

# Free space (hard-capped data)
df -h /srv/nextcloud-data

# Proxmox snapshot before upgrades (from PVE host CLI)
# qm snapshot <VMID> pre-nc-upgrade -description "Nextcloud before upgrade"
```

---

Also do not forget that you’re good to close that SSH session you set up initially: 
```
docker compose up -d started the containers detached (in the background).
docker compose logs -f app is just a live tail. Hitting Ctrl+C stops the log stream but does not stop the containers.

Exiting SSH won’t stop anything; Docker keeps running.
Day-to-day basics

# See stack status
cd /opt/nextcloud
docker compose ps

# Follow logs when you want, then Ctrl+C to exit
docker compose logs -f app
docker compose logs -f db
docker compose logs --since 10m -f

# Start / stop / restart the stack
docker compose stop
docker compose start
docker compose restart

# Fully stop & remove containers (volumes persist)
docker compose down

```
Potential questions is --> After reboot — will it auto-start?
Answer here is Yes! because the compose has restart: unless-stopped. As long as the Docker service starts at boot (it does by default), the containers will come back up automatically unless you previously stopped them. You can verify/ensure:

```
# Docker enabled on boot?
systemctl is-enabled docker
# enable if needed
sudo systemctl enable docker
If you manually stop the containers (docker compose stop), unless-stopped remembers that state and won’t auto-start them on reboot. That’s intentional.
```



DONE :) Enjoy
