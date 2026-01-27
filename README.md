# [>] GitLab CI/CD + DNS + HTTPS - Complete Setup Guide
> คู่มือฉบับสมบูรณ์: ตั้งแต่ติดตั้ง Debian, GitLab CI/CD, DNS Server (Technitium), ไปจนถึง HTTPS (Caddy)

## [#] Table of Contents
- [Overview](#-overview)
- [Prerequisites](#-prerequisites)
- [Part 1: VM1 Setup (GitLab Server)](#part-1-vm1-setup-gitlab-server)
- [Part 2: VM2 Setup (Production Server)](#part-2-vm2-setup-production-server)
- [Part 3: SSH Keys Configuration](#part-3-ssh-keys-configuration)
- [Part 4: GitLab Runner Registration](#part-4-gitlab-runner-registration)
- [Part 5: DNS Server Setup (Technitium)](#part-5-dns-server-setup-technitium)
- [Part 6: HTTPS Setup with Caddy](#part-6-https-setup-with-caddy)
- [Part 7: Complete CI/CD Pipeline](#part-7-complete-cicd-pipeline)
- [Testing & Verification](#-testing--verification)
- [Troubleshooting](#-troubleshooting)

---

## [*] Overview

โปรเจกต์นี้สร้างระบบ DevOps แบบสมบูรณ์ประกอบด้วย:

### Infrastructure Components
- **VM1 (192.168.10.20)**: GitLab Server + GitLab Runner + Container Registry
- **VM2 (192.168.10.30)**: Production Server + DNS Server (Technitium) + Reverse Proxy (Caddy)

### Key Features
- [x] GitLab CE สำหรับ Source Control
- [x] GitLab Runner สำหรับ CI/CD
- [x] Container Registry สำหรับเก็บ Docker Images
- [x] Technitium DNS Server สำหรับจัดการ Domain ภายใน
- [x] Caddy Reverse Proxy พร้อม HTTPS (Internal CA)
- [x] Automated Deployment Pipeline

### Architecture Diagram
```
                    ┌────────────────────────────┐
                    │   Developer Machine        │
                    │ • Git push via SSH         │
                    │ • Access via HTTPS/DNS     │
                    └──────────┬─────────────────┘
                               │
                               ▼
        ┌──────────────────────────────────────────────────┐
        │           VM2 (192.168.10.30)                    │
        │  ┌───────────────────────────────────────────┐  │
        │  │ Technitium DNS Server                     │  │
        │  │ • dns.group1.vec → 192.168.10.30         │  │
        │  │ • app.group1.vec → 192.168.10.30         │  │
        │  │ • gitlab.group1.vec → 192.168.10.20      │  │
        │  └───────────────────────────────────────────┘  │
        │  ┌───────────────────────────────────────────┐  │
        │  │ Caddy Reverse Proxy (HTTPS)               │  │
        │  │ • Internal CA for self-signed certs      │  │
        │  │ • Auto HTTPS for all services            │  │
        │  └───────────────────────────────────────────┘  │
        │  ┌───────────────────────────────────────────┐  │
        │  │ Production Apps                           │  │
        │  │ • Pull images from VM1 Registry          │  │
        │  └───────────────────────────────────────────┘  │
        └──────────────┬───────────────────────────────────┘
                       │
                       │ CI/CD Pipeline
                       ▼
        ┌──────────────────────────────────────────────────┐
        │           VM1 (192.168.10.20)                    │
        │  ┌───────────────────────────────────────────┐  │
        │  │ GitLab Server                             │  │
        │  │ • Web UI: gitlab.group1.vec               │  │
        │  │ • Registry: 192.168.10.20:5005            │  │
        │  │ • SSH: port 2222                          │  │
        │  └───────────────────────────────────────────┘  │
        │  ┌───────────────────────────────────────────┐  │
        │  │ GitLab Runner                             │  │
        │  │ • Build → Test → Deploy                   │  │
        │  │ • Push to Registry → Deploy to VM2        │  │
        │  └───────────────────────────────────────────┘  │
        └──────────────────────────────────────────────────┘
```

---

## [+] Prerequisites

### System Requirements
| Component | Requirement |
|-----------|------------|
| **OS** | Debian 13 (Bookworm) |
| **RAM** | VM1: 4GB+ (แนะนำ 8GB), VM2: 2GB+ |
| **Storage** | VM1: 30GB+, VM2: 20GB+ |
| **Network** | 2 VMs ในเครือข่าย VLAN เดียวกัน |

### IP Address Plan
| Server | IP Address | Hostname | Services |
|--------|------------|----------|----------|
| VM1 | 192.168.10.20 | gitlab | GitLab, Runner, Registry |
| VM2 | 192.168.10.30 | production | Apps, DNS, Caddy |
| Gateway | 192.168.10.1 | - | Default GW |

### Domain Plan (DNS Internal)
| Domain | IP | Service |
|--------|-----|---------|
| gitlab.group1.vec | 192.168.10.20 | GitLab Web UI |
| dns.group1.vec | 192.168.10.30 | Technitium Web UI |
| app.group1.vec | 192.168.10.30 | Production Apps |

---

## Part 1: VM1 Setup (GitLab Server)

### Step 1.1: ติดตั้ง Debian 13 และ Update System

```bash
# Login as root
su -

# Update package list และ upgrade
apt update && apt -y upgrade
```

### Step 1.2: (Optional) ตั้งค่า APT Mirror สำหรับความเร็วในการดาวน์โหลด

```bash
# แก้ไข sources.list
nano /etc/apt/sources.list
```

เปลี่ยนเป็น mirror ใกล้บ้าน (ตัวอย่าง: KKU Mirror):

```ini
deb http://mirror.kku.ac.th/debian bookworm main contrib non-free non-free-firmware
deb http://mirror.kku.ac.th/debian bookworm-updates main contrib non-free non-free-firmware
deb http://mirror.kku.ac.th/debian bookworm-backports main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

จากนั้น:

```bash
apt update
```

### Step 1.3: ติดตั้ง Required Packages

```bash
apt -y install nodejs npm docker.io docker-compose \
               openssh-server openssh-client ufw \
               git curl wget nano micro
```

### Step 1.4: ติดตั้ง Node.js LTS (Optional - สำหรับ Applications)

```bash
# ดึง script ติดตั้ง Node.js LTS
curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -

# ติดตั้ง Node.js
apt install -y nodejs

# ตรวจสอบเวอร์ชัน
node -v
npm -v
```

### Step 1.5: Clone Repository หรือสร้าง Working Directory

```bash
# Option 1: Clone จาก GitHub (ถ้ามี)
git clone https://github.com/vactuzx-dot/meox
cd meox

# Option 2: สร้างโฟลเดอร์ใหม่
mkdir -p /opt/gitlab-setup
cd /opt/gitlab-setup
```

### Step 1.6: ตั้งค่า Network (Static IP)

> [!] **หมายเหตุ**: รอให้ VLAN พร้อมใช้งานก่อน

```bash
# แก้ไขไฟล์ network interfaces
nano /etc/network/interfaces
```

เพิ่มเนื้อหา:

```ini
auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
    address 192.168.10.20/24
    gateway 192.168.10.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

รีสตาร์ท network:

```bash
systemctl restart networking

# ตรวจสอบ IP
ip addr show ens18
```

### Step 1.7: ตั้งค่า Firewall (UFW)

```bash
# เปิดใช้งาน UFW
ufw enable

# เปิด ports ที่จำเป็น
ufw allow OpenSSH              # SSH (port 22)
ufw allow 80/tcp               # HTTP
ufw allow 443/tcp              # HTTPS
ufw allow 443/udp              # HTTPS/QUIC
ufw allow 2222/tcp             # GitLab SSH
ufw allow 5000/tcp             # Custom app
ufw allow 5005/tcp             # GitLab Registry
ufw allow 5005/udp             # GitLab Registry
ufw allow 8080/tcp             # Alternative HTTP
ufw allow 9443/tcp             # Portainer (optional)

# ตรวจสอบสถานะ
ufw status numbered
```

### Step 1.8: ตั้งค่า Docker Daemon (Insecure Registry)

> [!!] **สำคัญมาก!** ต้องตั้งค่าให้ Docker ยอมรับ HTTP Registry

```bash
# สร้าง/แก้ไขไฟล์ daemon.json
nano /etc/docker/daemon.json
```

เพิ่มเนื้อหา:

```json
{
    "insecure-registries": ["192.168.10.20:5005"]
}
```

รีสตาร์ท Docker:

```bash
systemctl restart docker

# ตรวจสอบสถานะ
systemctl status docker
```

### Step 1.9: สร้าง User Account (Recommended)

```bash
# สร้าง user ใหม่
adduser admin

# เพิ่มสิทธิ์ sudo
usermod -aG sudo admin

# เพิ่มสิทธิ์ใช้ docker
usermod -aG docker admin

# (Optional) ตั้งค่าให้ใช้ sudo ไม่ต้องใส่รหัสผ่าน
visudo
```

เพิ่มบรรทัด:

```ini
admin ALL=(ALL) NOPASSWD: ALL
```

### Step 1.10: สร้างไฟล์ docker-compose.yml สำหรับ GitLab

```bash
# สร้างไฟล์ docker-compose.yml
nano docker-compose.yml
```

วางเนื้อหาจากไฟล์ `docker-compose.yml` ที่มีอยู่แล้ว (ตามที่แสดงด้านล่าง):

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: "192.168.10.20"
    environment:

      GITLAB_ROOT_PASSWORD: 'itcmtc1234'
      GITLAB_ROOT_EMAIL: 'taweesaknumma@gmail.com'

      GITLAB_OMNIBUS_CONFIG: |
        # URL สำหรับเข้าหน้าเว็บ
        external_url 'http://192.168.10.20'

        # ตั้งค่า Container Registry ผ่าน HTTP
        registry_external_url 'http://192.168.10.20:5005'
        gitlab_rails['registry_enabled'] = true
        registry['enable'] = true
        registry_nginx['listen_port'] = 5005
        registry_nginx['listen_https'] = false

        # ปิดการ redirect ไป HTTPS
        nginx['listen_https'] = false
        registry_nginx['proxy_set_headers'] = { "X-Forwarded-Proto" => "http" }

        # ลดภาระเครื่อง (Optional: ปิด service ที่ไม่จำเป็น)
        prometheus_monitoring['enable'] = false
    ports:
      - "80:80"
      - "443:443"
      - "2222:22" # SSH port (เลี่ยงการชนกับ port 22 ของเครื่อง)
      - "5005:5005" # Registry port
    volumes:
      - "./config:/etc/gitlab"
      - "./logs:/var/log/gitlab"
      - "./data:/var/opt/gitlab"
    shm_size: "256m"
    networks:
      - gitlab-net

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: always
    depends_on:
      - gitlab
    volumes:
      - "./runner-config:/etc/gitlab-runner"
      - "/var/run/docker.sock:/var/run/docker.sock"
    networks:
      - gitlab-net

networks:
  gitlab-net:
    driver: bridge
```

### Step 1.11: รัน GitLab และ GitLab Runner

```bash
# รัน containers
docker compose up -d

# ตรวจสอบ logs
docker logs -f gitlab

# รอประมาณ 3-5 นาที จนกว่า GitLab จะพร้อม
```

ตรวจสอบสถานะ:

```bash
docker ps
```

### Step 1.12: เข้าใช้งาน GitLab

1. เปิดเบราว์เซอร์ไปที่: `http://192.168.10.20`
2. Login ด้วย:
   - **Username**: `root`
   - **Password**: `itcmtc1234`

---

## Part 2: VM2 Setup (Production Server)

### Step 2.1 - 2.8: ทำตาม VM1 Setup (Step 1.1 - 1.8)

ทำขั้นตอนเดียวกับ VM1 **ยกเว้น**:

**เปลี่ยน IP Address เป็น:**

```ini
auto ens18
iface ens18 inet static
    address 192.168.10.30/24
    gateway 192.168.10.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

**เปลี่ยน Docker daemon.json เป็น:**

```json
{
    "insecure-registries": ["192.168.10.20:5005"]
}
```

> [!] **สำคัญ**: VM2 ต้องตั้งค่า insecure registry เหมือน VM1 เพื่อ pull images

### Step 2.9: สร้าง Network สำหรับ Services

```bash
# สร้าง Docker network สำหรับ services ทั้งหมด
docker network create app_net
```

### Step 2.10: ปิด systemd-resolved (สำหรับ DNS Server)

> [!] **สำคัญ**: ต้องปิด DNS service เดิมเพื่อให้ Technitium ใช้ Port 53 ได้

```bash
# ปิด systemd-resolved
systemctl stop systemd-resolved
systemctl disable systemd-resolved

# ตั้งค่า DNS ชั่วคราว
rm /etc/resolv.conf
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
```

---

## Part 3: SSH Keys Configuration

### 3.1: SSH Key สำหรับ Developer → GitLab (ED25519)

เพื่อให้ developers สามารถ push code ไปยัง GitLab ได้

**ทำบนเครื่อง Developer:**

```bash
# Generate SSH key แบบ ED25519
ssh-keygen -t ed25519 -C "your-email@example.com"

# แสดง public key
cat ~/.ssh/id_ed25519.pub
```

**เพิ่ม SSH key ใน GitLab:**

1. Copy ข้อความทั้งหมดจาก `id_ed25519.pub`
2. ใน GitLab: ไปที่ **User Settings → SSH Keys**
3. Paste public key และกด **Add key**

### 3.2: SSH Key สำหรับ VM1 → VM2 (RSA)

เพื่อให้ GitLab Runner สามารถ deploy ไป VM2 ได้

**ทำบน VM1:**

```bash
# Generate RSA key (4096 bits)
ssh-keygen -t rsa -b 4096 -C "gitlab-runner@vm1"

# แสดง private key (เก็บใน GitLab CI/CD Variables)
cat ~/.ssh/id_rsa

# แสดง public key (นำไปใส่ใน VM2)
cat ~/.ssh/id_rsa.pub
```

**ตั้งค่า GitLab CI/CD Variables:**

1. ใน GitLab: ไปที่ **Project → Settings → CI/CD → Variables**
2. กด **Add variable**
   - **Key**: `SSH_PRIVATE_KEY`
   - **Value**: วาง private key ทั้งหมดจาก `id_rsa`
   - **Type**: File
   - **Protected**: [✓] (แนะนำ)
   - **Masked**: [✗] (ห้าม mask เพราะเป็น multiline)

**เพิ่ม Public Key ใน VM2:**

**[วิธีที่ 1] ใช้ ssh-copy-id (แนะนำ)**

```bash
# ทำบน VM1
ssh-copy-id admin@192.168.10.30
```

**[วิธีที่ 2] Manual**

```bash
# ทำบน VM2
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
# วาง public key จาก VM1 (id_rsa.pub) ทั้งหมด
# Save และ Exit

# ตั้งค่าสิทธิ์
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

**ทดสอบการเชื่อมต่อ:**

```bash
# ทำบน VM1
ssh admin@192.168.10.30
# ควรเข้าได้โดยไม่ต้องใส่รหัสผ่าน
```

---

## Part 4: GitLab Runner Registration

### Step 4.1: สร้าง Runner Token

1. ใน GitLab: ไปที่ **Admin Area → CI/CD → Runners**
2. กด **New instance runner**
3. Copy **registration token**

### Step 4.2: Register Runner

```bash
# ทำบน VM1
docker exec -it gitlab-runner gitlab-runner register
```

ตอบคำถามดังนี้:

```
[?] Enter the GitLab instance URL:
[>] http://192.168.10.20

[?] Enter the registration token:
[>] <paste token ที่ copy มา>

[?] Enter a description for the runner:
[>] docker

[?] Enter tags for the runner:
[>] (กด Enter เว้นว่างไว้)

[?] Enter an executor:
[>] docker

[?] Enter the default Docker image:
[>] docker:24.0.5
```

### Step 4.3: แก้ไข Runner Configuration

```bash
# เข้าไปแก้ไขไฟล์ config.toml
docker exec -it gitlab-runner nano /etc/gitlab-runner/config.toml
```

แก้ไข **4 จุดสำคัญ**:

```toml
concurrent = 1
check_interval = 0

[[runners]]
  name = "docker"
  url = "http://192.168.10.20"
  token = "glrt-xxxxxxxxxxxxx"
  executor = "docker"

  # [✓] [จุดที่ 1] บอก Runner ให้ clone ผ่าน network
  clone_url = "http://192.168.10.20"

  [runners.docker]
    tls_verify = false
    image = "docker:24.0.5"

    # [✓] [จุดที่ 2] เปิด privileged mode
    privileged = true

    # [✓] [จุดที่ 3] Mount Docker socket
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]

    # [✓] [จุดที่ 4] ใส่ชื่อ Docker network
    # หาชื่อจาก: docker network ls (เช่น meox_gitlab-net)
    network_mode = "meox_gitlab-net"
```

> [!] **หมายเหตุ**: `network_mode` ให้ดูจาก `docker network ls` ในเครื่อง VM1

บันทึกและออก (Ctrl+O, Enter, Ctrl+X)

### Step 4.4: Restart GitLab Runner

```bash
docker restart gitlab-runner

# ตรวจสอบ logs
docker logs gitlab-runner
```

### Step 4.5: ตรวจสอบ Runner Status

ใน GitLab: ไปที่ **Admin Area → CI/CD → Runners**

ควรเห็น runner สีเขียว (online)

---

## Part 5: DNS Server Setup (Technitium)

### Step 5.1: สร้าง Volume และ Directory

```bash
# ทำบน VM2
docker volume create dns_config

# สร้างโฟลเดอร์สำหรับ compose file
mkdir -p /opt/technitium
cd /opt/technitium
```

### Step 5.2: สร้างไฟล์ docker-compose.yml

```bash
nano docker-compose.yml
```

วางเนื้อหา:

```yaml
networks:
  app_net:
    external: true

services:
  technitium-dns:
    image: technitium/dns-server:latest
    container_name: technitium-dns
    hostname: itcmtc.com
    restart: unless-stopped
    environment:
      - DNS_SERVER_DOMAIN=itcmtc.com
    ports:
      - "53:53/udp"
      - "53:53/tcp"
      - "5380:5380/tcp"
    volumes:
      - dns_config:/etc/dns
    networks:
      - app_net
    cap_add:
      - NET_ADMIN
```

### Step 5.3: รัน Technitium DNS

```bash
docker compose up -d

# ตรวจสอบ logs
docker logs -f technitium-dns
```

### Step 5.4: เข้าใช้งาน Technitium Web UI

1. เปิดเบราว์เซอร์: `http://192.168.10.30:5380`
2. ตั้งค่า admin password ครั้งแรก
3. Login

### Step 5.5: สร้าง DNS Zone และ Records

**สร้าง Zone:**

1. ไปที่ **Zones → Create Zone**
2. Zone Name: `group1.vec`
3. Type: `Primary Zone`

**เพิ่ม DNS Records:**

| Record Type | Name | Value |
|-------------|------|-------|
| A | gitlab.group1.vec | 192.168.10.20 |
| A | dns.group1.vec | 192.168.10.30 |
| A | app.group1.vec | 192.168.10.30 |

### Step 5.6: ตั้งค่า Client ให้ใช้ DNS Server

**บน Windows (Developer Machine):**

1. **Settings → Network & Internet → Ethernet/WiFi → Properties**
2. **DNS server assignment → Edit**
3. เปลี่ยนเป็น **Manual**
4. **Preferred DNS**: `192.168.10.30`
5. **Alternate DNS**: `8.8.8.8`

**บน Linux:**

```bash
# แก้ไข resolv.conf
sudo nano /etc/resolv.conf
```

เพิ่ม:

```
nameserver 192.168.10.30
nameserver 8.8.8.8
```

**ทดสอบ DNS:**

```bash
# Windows
nslookup gitlab.group1.vec

# Linux
dig gitlab.group1.vec
```

---

## Part 6: HTTPS Setup with Caddy

### Step 6.1: สร้าง Directories และ Volumes

```bash
# ทำบน VM2
sudo mkdir -p /opt/caddy /var/log/caddy

# ตั้งค่าสิทธิ์
sudo chown -R 100:101 /var/log/caddy

# สร้าง volumes
docker volume create caddy_data
docker volume create caddy_config
```

### Step 6.2: สร้าง Caddyfile

```bash
sudo nano /opt/caddy/Caddyfile
```

วางเนื้อหา:

```caddyfile
{
    # ใช้ Internal CA สำหรับ self-signed certificates
    cert_issuer internal
}

# Technitium DNS Web Admin
dns.group1.vec {
    tls internal
    encode zstd gzip
    log {
        output file /var/log/caddy/technitium_access.log
    }
    reverse_proxy https://192.168.10.30:5380 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}

# Production App
app.group1.vec {
    tls internal
    encode zstd gzip
    log {
        output file /var/log/caddy/app_access.log
    }
    reverse_proxy http://192.168.10.30:3000
}

# GitLab
gitlab.group1.vec {
    tls internal
    encode zstd gzip
    log {
        output file /var/log/caddy/gitlab_access.log
    }
    reverse_proxy http://192.168.10.20:80 {
        header_up X-Forwarded-Proto {scheme}
    }
}
```

### Step 6.3: สร้างไฟล์ docker-compose.yml สำหรับ Caddy

```bash
mkdir -p /opt/caddy-setup
cd /opt/caddy-setup
nano docker-compose.yml
```

วางเนื้อหา:

```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    networks:
      - app_net
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - /opt/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
      - /var/log/caddy:/var/log/caddy
    environment:
      - TZ=Asia/Bangkok

networks:
  app_net:
    external: true

volumes:
  caddy_data:
    external: true
  caddy_config:
    external: true
```

### Step 6.4: รัน Caddy

```bash
docker compose up -d

# ตรวจสอบ logs
docker logs -f caddy
```

### Step 6.5: ดึง Root CA Certificate

```bash
# รอ Caddy สร้าง certificate (ประมาณ 10 วินาที)
sleep 10

# ดึง Root CA ออกมา
docker cp caddy:/data/caddy/pki/authorities/local/root.crt /opt/caddy/caddy-root.crt
sudo chmod 644 /opt/caddy/caddy-root.crt
```

### Step 6.6: แจกจ่าย Root CA ให้ Client

**วิธีที่ 1: ใช้ HTTP Server ชั่วคราว**

```bash
# ทำบน VM2
cd /opt/caddy
python3 -m http.server 8000
```

บน Windows Client:

```powershell
# Download Root CA
Invoke-WebRequest http://192.168.10.30:8000/caddy-root.crt -OutFile C:\Users\Public\caddy-root.crt
```

**วิธีที่ 2: ใช้ SCP/USB**

คัดลอกไฟล์ `/opt/caddy/caddy-root.crt` ไปยังเครื่อง Client

### Step 6.7: ติดตั้ง Root CA บน Windows

เปิด **PowerShell as Administrator**:

```powershell
# Import Root CA
Import-Certificate -FilePath "C:\Users\Public\caddy-root.crt" -CertStoreLocation Cert:\LocalMachine\Root

# ตรวจสอบ
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like "*Caddy Local Authority*" | Select-Object Subject, Thumbprint, NotAfter
```

### Step 6.8: ทดสอบ HTTPS

เปิดเบราว์เซอร์:

- `https://gitlab.group1.vec`
- `https://dns.group1.vec`
- `https://app.group1.vec`

ควรเห็น **🔒 Secure** ไม่มี warning

---

## Part 7: Complete CI/CD Pipeline

### Step 7.1: สร้างตัวอย่าง Project

สร้าง project ใน GitLab และเพิ่มไฟล์ต่อไปนี้:

**ไฟล์: `.gitlab-ci.yml`**

```yaml
stages:
  - build
  - test
  - deploy

variables:
  REGISTRY: "192.168.10.20:5005"
  IMAGE_NAME: "$REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME"
  IMAGE_TAG: "$CI_COMMIT_SHORT_SHA"

# Build Stage
build-image:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  only:
    - main

# Test Stage
test-app:
  stage: test
  image: $IMAGE_NAME:$IMAGE_TAG
  script:
    - echo "Running tests..."
    - npm test || echo "No tests defined"
  only:
    - main

# Deploy Stage
deploy-production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan -H 192.168.10.30 >> ~/.ssh/known_hosts
  script:
    - |
      ssh admin@192.168.10.30 << EOF
        docker pull $IMAGE_NAME:latest
        docker stop myapp || true
        docker rm myapp || true
        docker run -d --name myapp \
          --network app_net \
          -p 3000:3000 \
          --restart unless-stopped \
          $IMAGE_NAME:latest
      EOF
  only:
    - main
  environment:
    name: production
    url: https://app.group1.vec
```

**ไฟล์: `Dockerfile`**

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

**ไฟล์: `package.json`**

```json
{
  "name": "sample-app",
  "version": "1.0.0",
  "description": "Sample Node.js app",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"No tests\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**ไฟล์: `index.js`**

```javascript
const express = require('express');
const app = express();
const PORT = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from CI/CD Pipeline!',
    version: process.env.npm_package_version,
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Step 7.2: Push Code และรอ Pipeline

```bash
git add .
git commit -m "Initial commit with CI/CD"
git push origin main
```

ตรวจสอบ Pipeline ใน GitLab:
- **CI/CD → Pipelines**

### Step 7.3: ตรวจสอบผลลัพธ์

เปิดเบราว์เซอร์: `https://app.group1.vec`

ควรเห็น:

```json
{
  "message": "Hello from CI/CD Pipeline!",
  "version": "1.0.0",
  "timestamp": "2026-01-27T13:43:06.000Z"
}
```

---

## [✓] Testing & Verification

### 1. ทดสอบ Container Registry

```bash
# ทำบน VM2
docker pull 192.168.10.20:5005/root/sample-app:latest
```

ถ้า pull ได้แปลว่าตั้งค่าถูกต้อง [✓]

### 2. ทดสอบ SSH Connection

```bash
# ทำบน VM1
ssh admin@192.168.10.30
```

ควรเข้าได้โดยไม่ต้องใส่รหัสผ่าน [✓]

### 3. ทดสอบ DNS Resolution

```bash
# Windows
nslookup gitlab.group1.vec
nslookup app.group1.vec

# Linux
dig gitlab.group1.vec
```

ควรได้ IP ที่ถูกต้อง [✓]

### 4. ทดสอบ HTTPS

เปิดเบราว์เซอร์:
- `https://gitlab.group1.vec` [✓]
- `https://dns.group1.vec` [✓]
- `https://app.group1.vec` [✓]

ต้องไม่มี certificate warning

### 5. ทดสอบ CI/CD Pipeline

1. แก้ไขไฟล์ใน project
2. Push to GitLab
3. ดู Pipeline status
4. ตรวจสอบ deployment บน VM2

---

## [!] Troubleshooting

### ปัญหา: GitLab ใช้งานไม่ได้

**อาการ:** ไม่สามารถเข้า Web UI ได้

**วิธีแก้:**

```bash
# ตรวจสอบ logs
docker logs gitlab

# ตรวจสอบว่า container ทำงานหรือไม่
docker ps

# Restart GitLab
docker restart gitlab

# ดู resource usage
docker stats gitlab
```

### ปัญหา: Runner ไม่เชื่อมต่อกับ GitLab

**วิธีแก้:**

```bash
# ตรวจสอบ config
docker exec -it gitlab-runner cat /etc/gitlab-runner/config.toml

# ตรวจสอบ network
docker network inspect meox_gitlab-net

# Restart runner
docker restart gitlab-runner
```

### ปัญหา: ไม่สามารถ pull image จาก registry

**วิธีแก้:**

```bash
# ตรวจสอบ daemon.json
cat /etc/docker/daemon.json

# Restart Docker
systemctl restart docker

# ทดสอบ registry
curl http://192.168.10.20:5005/v2/

# ควรได้ผลลัพธ์: {}
```

### ปัญหา: DNS ไม่ทำงาน

**วิธีแก้:**

```bash
# ตรวจสอบ Port 53
sudo netstat -tulpn | grep :53

# ตรวจสอบ systemd-resolved
systemctl status systemd-resolved
# ต้องเป็น disabled

# Restart Technitium
docker restart technitium-dns

# ดู logs
docker logs technitium-dns
```

### ปัญหา: HTTPS ใช้งานไม่ได้

**วิธีแก้:**

```bash
# ตรวจสอบ Caddy logs
docker logs caddy

# ตรวจสอบ Caddyfile syntax
docker exec caddy caddy validate --config /etc/caddy/Caddyfile

# ตรวจสอบว่ามี Root CA หรือไม่
docker exec caddy ls -la /data/caddy/pki/authorities/local/

# ดึง Root CA ใหม่
docker cp caddy:/data/caddy/pki/authorities/local/root.crt /tmp/caddy-root.crt
```

### ปัญหา: Pipeline ล้มเหลว (SSH Connection Failed)

**วิธีแก้:**

```bash
# ตรวจสอบ SSH key ใน GitLab CI/CD Variables
# ต้องมี SSH_PRIVATE_KEY

# ทดสอบ SSH จาก VM1
ssh admin@192.168.10.30

# ตรวจสอบ authorized_keys บน VM2
cat ~/.ssh/authorized_keys

# ตรวจสอบสิทธิ์
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### ปัญหา: UFW บล็อก connections

**วิธีแก้:**

```bash
# ตรวจสอบ rules
ufw status numbered

# เพิ่ม rule
ufw allow from 192.168.10.0/24

# ลบ rule ที่ไม่ต้องการ
ufw delete <number>

# Reload UFW
ufw reload
```

---

## [i] Additional Resources

### Useful Commands

```bash
# Docker Management
docker ps                           # ดู containers ที่ทำงาน
docker ps -a                        # ดู containers ทั้งหมด
docker logs -f <container>          # ดู logs แบบ real-time
docker exec -it <container> bash    # เข้า shell ใน container
docker network ls                   # ดู networks
docker volume ls                    # ดู volumes
docker stats                        # ดู resource usage

# System Monitoring
htop                                # ดู CPU/RAM usage
df -h                               # ดู disk usage
free -h                             # ดู memory usage
netstat -tulpn                      # ดู ports ที่เปิดอยู่

# GitLab Specific
docker exec -it gitlab gitlab-rake gitlab:check
docker exec -it gitlab gitlab-ctl status
docker exec -it gitlab gitlab-ctl tail

# DNS Testing
nslookup <domain>                   # Windows
dig <domain>                        # Linux
host <domain>                       # Both
```

### Important Files & Locations

| File/Directory | Description |
|----------------|-------------|
| `/etc/docker/daemon.json` | Docker daemon configuration |
| `/etc/gitlab-runner/config.toml` | GitLab Runner configuration |
| `/opt/caddy/Caddyfile` | Caddy reverse proxy config |
| `/etc/resolv.conf` | DNS resolver config |
| `/etc/network/interfaces` | Network configuration |
| `./config/` | GitLab configuration files |
| `./data/` | GitLab data (repos, uploads) |
| `./logs/` | GitLab logs |
| `/var/log/caddy/` | Caddy access logs |

### Port Reference

| Port | Service | Protocol |
|------|---------|----------|
| 22 | SSH | TCP |
| 53 | DNS | TCP/UDP |
| 80 | HTTP | TCP |
| 443 | HTTPS | TCP/UDP |
| 2222 | GitLab SSH | TCP |
| 3000 | Sample App | TCP |
| 5005 | GitLab Registry | TCP |
| 5380 | Technitium Web UI | TCP |
| 9443 | Portainer | TCP |

---

## [^] Learning Objectives

หลังจากทำตามคู่มือนี้แล้ว คุณควรจะสามารถ:

- [✓] ติดตั้งและตั้งค่า GitLab Server บน Linux
- [✓] ตั้งค่า GitLab Runner สำหรับ CI/CD
- [✓] ใช้งาน Container Registry
- [✓] ตั้งค่า SSH keys สำหรับ authentication
- [✓] Deploy applications ผ่าน CI/CD pipeline
- [✓] ตั้งค่า DNS Server ภายในเครือข่าย (Technitium)
- [✓] ตั้งค่า Reverse Proxy พร้อม HTTPS (Caddy)
- [✓] จัดการ Internal CA และ SSL Certificates
- [✓] Troubleshoot ปัญหาพื้นฐาน

---

## [-] License

คู่มือนี้สร้างขึ้นเพื่อการศึกษาในรายวิชา DevOps

## [~] Credits

- Created by: Taweesaknumma
- Email: taweesaknumma@gmail.com
- Repository: https://github.com/vactuzx-dot/meox

---

**Happy Learning! [>]**
