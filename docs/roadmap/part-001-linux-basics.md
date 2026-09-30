# Part 001: พื้นฐาน Linux & Server
## Road to 1,000,000 Users/Day — chuaikan.com

> **Level:** Beginner
> **Steps:** 1-10
> **เวลาโดยประมาณ:** 8 ชั่วโมง
> **Prerequisites:** ไม่มี (เริ่มต้นได้เลย)

---

## 🎯 สิ่งที่จะได้เรียนรู้ใน Part นี้

1. คำสั่ง Linux command line พื้นฐาน (ls, cd, cp, mv, rm, find, grep)
2. การตั้งค่า SSH และสร้าง key pair สำหรับ secure connection
3. การจัดการ file permissions อย่างละเอียด (chmod, chown, umask)
4. การใช้ apt สำหรับ package management บน Ubuntu 24.04
5. การตั้งค่า UFW firewall เพื่อป้องกัน server
6. การอ่าน log ด้วย journalctl
7. การจัดการ service ด้วย systemd
8. การตั้ง crontab สำหรับ scheduled tasks
9. การ monitor resource ของ server (CPU, RAM, Disk, I/O)
10. การเขียน backup scripts ด้วย tar และ rsync

---

## 📖 ทฤษฎีและแนวคิด

### Linux File System Hierarchy

Linux ใช้โครงสร้าง directory แบบ tree โดยมี root (`/`) เป็นจุดเริ่มต้น ทุก path บน Linux จะเริ่มจาก `/`

```
/
├── bin/        → คำสั่งพื้นฐาน (ls, cp, mv)
├── boot/       → kernel และ bootloader
├── dev/        → device files
├── etc/        → config files ของระบบ
├── home/       → home directories ของ users
│   └── ubuntu/ → home ของ user "ubuntu"
├── lib/        → shared libraries
├── opt/        → optional software (third-party apps)
├── proc/       → virtual filesystem สำหรับ process info
├── root/       → home ของ root user
├── srv/        → data สำหรับ services (web, ftp)
├── sys/        → virtual filesystem สำหรับ kernel
├── tmp/        → temporary files (ถูกลบเมื่อ reboot)
├── usr/        → user programs และ libraries
│   ├── bin/    → user commands
│   ├── local/  → locally installed software
│   └── share/  → shared data
└── var/        → variable data (logs, databases, mail)
    ├── log/    → system logs
    └── www/    → web files
```

### Unix Permission Model

```
-rwxr-xr--  1  ubuntu  www-data  4096  Sep 30 10:00  server.js
│└──┴──┴──     └────┘  └──────┘
│  │  │  │      owner    group
│  │  │  └── others permissions (r--)
│  │  └───── group permissions  (r-x)
│  └──────── owner permissions  (rwx)
└─────────── file type (- = file, d = directory, l = symlink)

Permission bits:
r = 4  (read)
w = 2  (write)
x = 1  (execute)

ตัวอย่าง:
755 = rwxr-xr-x  (owner: rwx, group: r-x, others: r-x)
644 = rw-r--r--  (owner: rw-, group: r--, others: r--)
600 = rw-------  (owner: rw-, ไม่มีสิทธิ์ group/others)
```

### systemd Service Architecture

```
systemd (PID 1)
     │
     ├── target: multi-user.target
     │        ├── nginx.service
     │        ├── postgresql.service
     │        ├── redis.service
     │        └── chuaikan.service   ← app ของเรา
     │
     └── target: network.target
              └── networkd.service
```

---

## ⚙️ Environment Setup

### ตรวจสอบ Ubuntu Version

```bash
# ตรวจสอบ OS version
lsb_release -a
# Expected output:
# Distributor ID: Ubuntu
# Description:    Ubuntu 24.04.1 LTS
# Release:        24.04
# Codename:       noble

# ตรวจสอบ kernel version
uname -r
# Expected: 6.8.0-xx-generic

# ตรวจสอบ architecture
uname -m
# Expected: x86_64

# ตรวจสอบ disk space
df -h /
# Expected: มี space อย่างน้อย 20GB

# ตรวจสอบ RAM
free -h
# Expected: มี RAM อย่างน้อย 2GB
```

---

## 🛠️ Step-by-Step Implementation

### Step 1: Linux Command Line Basics

```bash
# ─── Navigation ───────────────────────────────────────────
# แสดงไฟล์ใน directory ปัจจุบัน
ls
ls -la           # แสดงแบบละเอียดพร้อม hidden files
ls -lh           # แสดงขนาดไฟล์แบบ human-readable
ls -lt           # เรียงตามเวลาแก้ไขล่าสุด
ls /var/log      # แสดงไฟล์ใน /var/log

# เปลี่ยน directory
cd /var/log      # ไปที่ /var/log
cd ..            # ขึ้นไป 1 ระดับ
cd ~             # กลับ home directory
cd -             # กลับ directory ก่อนหน้า
pwd              # แสดง path ปัจจุบัน

# ─── File Operations ──────────────────────────────────────
# สร้าง directory
mkdir /opt/chuaikan
mkdir -p /opt/chuaikan/logs/nginx    # สร้างทั้ง nested path

# สร้างไฟล์
touch /opt/chuaikan/README.md
echo "Hello chuaikan" > /opt/chuaikan/README.md    # เขียนทับ
echo "Version 1.0" >> /opt/chuaikan/README.md      # ต่อท้าย

# คัดลอกไฟล์
cp /opt/chuaikan/README.md /tmp/README.backup.md
cp -r /opt/chuaikan /tmp/chuaikan-backup           # คัดลอกทั้ง directory

# ย้าย/เปลี่ยนชื่อไฟล์
mv /tmp/README.backup.md /tmp/README.md
mv /tmp/README.md /opt/README.md

# ลบไฟล์
rm /tmp/chuaikan-backup/README.backup.md
rm -rf /tmp/chuaikan-backup    # ลบทั้ง directory (ระวัง!)

# ─── Search & Filter ──────────────────────────────────────
# หาไฟล์ด้วย find
find /var/log -name "*.log" -type f          # หา .log files
find /var/log -name "*.log" -mtime -7        # แก้ไขใน 7 วันที่ผ่านมา
find /opt -name "*.js" -size +100k           # ไฟล์ .js ขนาดใหญ่กว่า 100KB
find /tmp -type f -mtime +30 -delete        # ลบไฟล์เก่ากว่า 30 วัน

# ค้นหา text ในไฟล์ด้วย grep
grep "ERROR" /var/log/syslog                  # หา ERROR ใน syslog
grep -r "database" /opt/chuaikan/            # ค้นหาใน directory
grep -n "PORT" /opt/chuaikan/.env            # แสดงเลขบรรทัด
grep -i "error" /var/log/nginx/access.log    # ไม่สนใจ case
grep -v "DEBUG" /var/log/app.log             # แสดงบรรทัดที่ไม่มี DEBUG
grep -c "200" /var/log/nginx/access.log     # นับจำนวนบรรทัด

# ดูเนื้อหาไฟล์
cat /etc/hostname
head -20 /var/log/syslog        # แสดง 20 บรรทัดแรก
tail -50 /var/log/nginx/error.log          # แสดง 50 บรรทัดสุดท้าย
tail -f /var/log/nginx/access.log          # ติดตาม log แบบ real-time
less /var/log/syslog             # scroll ดู (q เพื่อออก)
wc -l /var/log/syslog            # นับจำนวนบรรทัด
```

### Step 2: SSH Setup และ Key Generation

```bash
# ─── บน LOCAL MACHINE (computer ของคุณ) ──────────────────
# สร้าง SSH key pair
ssh-keygen -t ed25519 -C "deploy@chuaikan.com" -f ~/.ssh/chuaikan_ed25519
# -t ed25519: algorithm ใหม่ปลอดภัยกว่า RSA
# -C: comment สำหรับระบุ key
# -f: ชื่อไฟล์ key

# ดู public key
cat ~/.ssh/chuaikan_ed25519.pub
# Output: ssh-ed25519 AAAA...xxx deploy@chuaikan.com

# ─── บน SERVER ────────────────────────────────────────────
# สร้าง user สำหรับ deployment
sudo adduser deploy
sudo usermod -aG sudo deploy     # เพิ่มเข้า sudo group

# สร้าง .ssh directory สำหรับ deploy user
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo touch /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh

# ─── บน LOCAL MACHINE ─────────────────────────────────────
# คัดลอก public key ไป server
ssh-copy-id -i ~/.ssh/chuaikan_ed25519.pub deploy@YOUR_SERVER_IP
# หรือ manual:
cat ~/.ssh/chuaikan_ed25519.pub | ssh ubuntu@YOUR_SERVER_IP \
  "sudo tee -a /home/deploy/.ssh/authorized_keys"

# ทดสอบ connection
ssh -i ~/.ssh/chuaikan_ed25519 deploy@YOUR_SERVER_IP

# สร้าง SSH config สำหรับ shortcut
cat >> ~/.ssh/config << 'EOF'
Host chuaikan-prod
    HostName YOUR_SERVER_IP
    User deploy
    IdentityFile ~/.ssh/chuaikan_ed25519
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF

# ทดสอบด้วย alias
ssh chuaikan-prod
```

```bash
# ─── SSH Server Hardening (บน SERVER) ────────────────────
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

sudo tee /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
# ปิด root login
PermitRootLogin no

# ใช้ Key authentication เท่านั้น
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# ปิด unused authentication
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

# Port (เปลี่ยนจาก 22 เพื่อลด brute force)
# Port 2222

# อนุญาตเฉพาะ user ที่ระบุ
AllowUsers deploy ubuntu

# Timeout settings
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 60

# จำกัด authentication attempts
MaxAuthTries 3
MaxSessions 10

# ปิด X11 forwarding
X11Forwarding no

# ปิด TCP forwarding ถ้าไม่ต้องการ
# AllowTcpForwarding no

# Banner warning
Banner /etc/ssh/banner.txt
EOF

# สร้าง SSH banner
sudo tee /etc/ssh/banner.txt << 'EOF'
┌─────────────────────────────────────────────────────────┐
│  WARNING: Authorized Access Only                        │
│  chuaikan.com Production Server                         │
│  All connections are logged and monitored               │
└─────────────────────────────────────────────────────────┘
EOF

# ทดสอบ config ก่อน restart
sudo sshd -t
# ถ้าไม่มี error:
sudo systemctl restart sshd
sudo systemctl status sshd
```

### Step 3: File Permissions Deep Dive

```bash
# ─── chmod ────────────────────────────────────────────────
# Numeric (Octal) notation
chmod 644 /opt/chuaikan/.env          # rw-r--r-- (config files)
chmod 755 /opt/chuaikan/scripts/      # rwxr-xr-x (directories, executables)
chmod 600 /home/deploy/.ssh/id_rsa    # rw------- (private keys)
chmod 700 /home/deploy/.ssh/          # rwx------ (ssh directory)
chmod 400 /etc/ssl/private/cert.key   # r-------- (read-only cert)
chmod +x /opt/chuaikan/deploy.sh      # เพิ่ม execute ให้ทุกคน
chmod -R 755 /opt/chuaikan/public/    # -R = recursive

# Symbolic notation
chmod u+x script.sh       # เพิ่ม execute ให้ owner (u)
chmod g-w file.txt        # ลบ write ออกจาก group (g)
chmod o-r secret.conf     # ลบ read ออกจาก others (o)
chmod a+r README.md       # เพิ่ม read ให้ทุกคน (a = all)
chmod ug=rw data.csv      # ตั้งค่า user และ group ให้ rw

# ─── chown ────────────────────────────────────────────────
sudo chown deploy /opt/chuaikan/app.js            # เปลี่ยน owner
sudo chown deploy:www-data /opt/chuaikan/uploads/ # เปลี่ยน owner:group
sudo chown -R deploy:deploy /opt/chuaikan/        # recursive

# ─── umask ────────────────────────────────────────────────
# umask กำหนด default permissions เมื่อสร้างไฟล์/directory ใหม่
# Formula: permissions = base - umask
# Base ของไฟล์ = 666, Base ของ directory = 777

umask                 # ดู umask ปัจจุบัน (ปกติ 0022)
umask 0027           # ตั้งค่า: file=640, dir=750

# ตั้งค่า umask สำหรับ service user ใน /etc/profile.d/
sudo tee /etc/profile.d/umask.sh << 'EOF'
# ตั้ง umask สำหรับ deploy user
if [ "$(id -un)" = "deploy" ]; then
    umask 0027
fi
EOF

# ─── Special Permissions ──────────────────────────────────
# SUID (Set User ID) - ทำงานด้วย permission ของ owner
chmod u+s /usr/bin/passwd    # ตัวอย่าง: passwd ต้องการ root

# SGID (Set Group ID) - ไฟล์ใหม่ inherit group ของ directory
chmod g+s /opt/chuaikan/shared/
# ทดสอบ:
ls -la /opt/chuaikan/shared/
# ควรเห็น: drwxr-sr-x (s ใน position ของ group execute)

# Sticky bit - ลบได้เฉพาะ owner
chmod +t /tmp
ls -la / | grep tmp
# ควรเห็น: drwxrwxrwt (t แทน x ใน others)
```

### Step 4: Package Management ด้วย apt

```bash
# ─── apt Basics ───────────────────────────────────────────
# อัพเดต package list (สำคัญมาก ต้องทำก่อน install)
sudo apt update

# อัพเกรด packages ที่ติดตั้งแล้ว
sudo apt upgrade -y
sudo apt full-upgrade -y     # อัพเกรดพร้อม resolve dependencies

# ติดตั้ง package
sudo apt install -y curl wget git vim htop tree unzip

# ค้นหา package
apt search nginx
apt-cache search postgresql

# ดูรายละเอียด package
apt show nginx
apt-cache show postgresql-17

# ลบ package
sudo apt remove nginx              # ลบ package แต่เก็บ config
sudo apt purge nginx               # ลบ package และ config ด้วย
sudo apt autoremove                # ลบ packages ที่ไม่ได้ใช้
sudo apt autoclean                 # ลบ cache เฉพาะ old packages
sudo apt clean                     # ลบ cache ทั้งหมด

# ─── ติดตั้ง Software สำหรับ Production ──────────────────
# ติดตั้ง essential tools
sudo apt install -y \
  curl wget git vim nano \
  htop iotop nethogs \
  tree unzip zip \
  net-tools dnsutils \
  build-essential \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg lsb-release \
  fail2ban \
  logrotate

# ตรวจสอบ versions
git --version      # git version 2.43.x
curl --version     # curl 8.x.x
```

### Step 5: UFW Firewall Setup

```bash
# ─── UFW (Uncomplicated Firewall) ─────────────────────────
# ตรวจสอบ status
sudo ufw status verbose

# ก่อน enable ต้องอนุญาต SSH ก่อน! (สำคัญมาก ไม่งั้นจะ lock ตัวเอง)
sudo ufw allow 22/tcp comment 'SSH'

# Enable UFW
sudo ufw enable
# ตอบ y เมื่อถามยืนยัน

# ─── Allow web traffic ────────────────────────────────────
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Allow Node.js app port (เฉพาะ localhost)
sudo ufw allow from 127.0.0.1 to any port 3000 comment 'Next.js local'
sudo ufw allow from 127.0.0.1 to any port 3001 comment 'API local'

# Allow PostgreSQL เฉพาะ localhost
sudo ufw allow from 127.0.0.1 to any port 5432 comment 'PostgreSQL local'

# Allow Redis เฉพาะ localhost
sudo ufw allow from 127.0.0.1 to any port 6379 comment 'Redis local'

# ─── Block/Deny Rules ─────────────────────────────────────
# Block IP ที่น่าสงสัย
sudo ufw deny from 1.2.3.4 comment 'Blocked attacker'
sudo ufw deny from 192.168.100.0/24 comment 'Blocked subnet'

# ลบ rule
sudo ufw delete allow 8080/tcp
sudo ufw delete deny from 1.2.3.4
# หรือดู rule numbers แล้วลบ
sudo ufw status numbered
sudo ufw delete 5          # ลบ rule ที่ 5

# ─── Advanced UFW Rules ───────────────────────────────────
# Rate limiting สำหรับ SSH (ป้องกัน brute force)
sudo ufw limit 22/tcp comment 'Rate limit SSH'

# Allow จาก specific IP เท่านั้น (สำหรับ admin)
sudo ufw allow from 203.0.113.0/24 to any port 22 comment 'Admin office'

# ดู rules ทั้งหมด
sudo ufw status verbose
# Expected output:
# Status: active
# To                         Action      From
# --                         ------      ----
# 22/tcp                     LIMIT IN    Anywhere
# 80/tcp                     ALLOW IN    Anywhere
# 443/tcp                    ALLOW IN    Anywhere

# Reload UFW
sudo ufw reload
```

### Step 6: journalctl สำหรับ Log Viewing

```bash
# ─── journalctl Basics ────────────────────────────────────
# ดู logs ทั้งหมด
sudo journalctl

# ดู logs ของ service เฉพาะ
sudo journalctl -u nginx              # nginx logs
sudo journalctl -u postgresql         # PostgreSQL logs
sudo journalctl -u chuaikan           # app logs

# ติดตาม log แบบ real-time (-f = follow)
sudo journalctl -u nginx -f

# ─── Filter by Time ───────────────────────────────────────
sudo journalctl --since "2024-01-01"
sudo journalctl --since "2024-01-01 10:00:00"
sudo journalctl --until "2024-01-31"
sudo journalctl --since "2024-01-01" --until "2024-01-31"
sudo journalctl --since "1 hour ago"
sudo journalctl --since "today"
sudo journalctl --since yesterday

# ─── Filter by Priority ───────────────────────────────────
sudo journalctl -p err            # errors เท่านั้น
sudo journalctl -p warning        # warnings และสูงกว่า
sudo journalctl -p info           # info และสูงกว่า

# Priority levels: emerg(0), alert(1), crit(2), err(3),
#                 warning(4), notice(5), info(6), debug(7)

# ─── Format Options ───────────────────────────────────────
sudo journalctl -u nginx --output=json-pretty   # JSON format
sudo journalctl -u nginx -n 100                 # แสดง 100 บรรทัดล่าสุด
sudo journalctl -u nginx --no-pager             # ไม่ใช้ pager

# ─── Disk Usage ───────────────────────────────────────────
sudo journalctl --disk-usage    # ดูขนาด journal logs
sudo journalctl --vacuum-size=500M   # ลด journal ให้เหลือ 500MB
sudo journalctl --vacuum-time=30d    # ลบ logs เก่ากว่า 30 วัน

# ─── ตัวอย่าง Real-world Usage ────────────────────────────
# หา 500 errors ใน nginx ช่วง 24 ชั่วโมงที่ผ่านมา
sudo journalctl -u nginx --since "24 hours ago" | grep "500"

# ดู boot logs
sudo journalctl -b          # current boot
sudo journalctl -b -1       # previous boot
sudo journalctl --list-boots
```

### Step 7: systemd Service Management

```bash
# ─── systemctl Basics ─────────────────────────────────────
# Start/Stop/Restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx      # reload config โดยไม่ restart

# Enable/Disable (auto-start on boot)
sudo systemctl enable nginx
sudo systemctl disable nginx
sudo systemctl enable --now nginx   # enable และ start เลย

# Status และ Logs
sudo systemctl status nginx
sudo systemctl is-active nginx     # active/inactive
sudo systemctl is-enabled nginx    # enabled/disabled

# ─── สร้าง Custom Service สำหรับ chuaikan.com ────────────
sudo tee /etc/systemd/system/chuaikan.service << 'EOF'
[Unit]
Description=chuaikan.com Next.js Application
Documentation=https://github.com/chuaikan/chuaikan-app
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
# User และ Group
User=deploy
Group=deploy

# Working Directory
WorkingDirectory=/opt/chuaikan/app

# Environment Variables
Environment=NODE_ENV=production
Environment=PORT=3000
EnvironmentFile=/opt/chuaikan/.env.production

# Start Command
ExecStart=/usr/bin/node /opt/chuaikan/app/node_modules/.bin/next start
ExecReload=/bin/kill -HUP $MAINPID

# Restart Policy
Restart=always
RestartSec=5
StartLimitInterval=60
StartLimitBurst=3

# Security (Hardening)
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/opt/chuaikan/app/.next /var/log/chuaikan

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=chuaikan

# Resource Limits
LimitNOFILE=65536
MemoryMax=2G
CPUQuota=80%

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd หลังสร้างหรือแก้ไข service file
sudo systemctl daemon-reload

# Enable และ start service
sudo systemctl enable chuaikan
sudo systemctl start chuaikan
sudo systemctl status chuaikan
```

### Step 8: Crontab และ Scheduled Tasks

```bash
# ─── Crontab Syntax ───────────────────────────────────────
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 7) (0,7 = Sunday)
# │ │ │ │ │
# * * * * * command

# ตัวอย่าง Crontab
# 0 2 * * *     = ทุกวัน เวลา 02:00
# */5 * * * *   = ทุก 5 นาที
# 0 0 * * 0     = ทุกวันอาทิตย์ เที่ยงคืน
# 0 9-18 * * 1-5 = ทุกชั่วโมง จันทร์-ศุกร์ 9am-6pm

# แก้ไข crontab
crontab -e           # แก้ไข crontab ของ user ปัจจุบัน
sudo crontab -e -u deploy  # แก้ไข crontab ของ deploy user
crontab -l           # แสดง crontab ปัจจุบัน
crontab -r           # ลบ crontab (ระวัง!)

# ─── Production Crontab สำหรับ chuaikan.com ──────────────
sudo tee /etc/cron.d/chuaikan << 'EOF'
# chuaikan.com Scheduled Tasks
# Environment
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""

# Backup database ทุกวัน 02:00 AM
0 2 * * * deploy /opt/chuaikan/scripts/backup-db.sh >> /var/log/chuaikan/backup.log 2>&1

# Cleanup temporary files ทุกวัน 03:00 AM
0 3 * * * deploy find /tmp/chuaikan -type f -mtime +7 -delete

# Renew SSL certificate (Let's Encrypt) ทุกวันเวลา 12:00
0 12 * * * root certbot renew --quiet --post-hook "systemctl reload nginx"

# Sync static files ทุก 6 ชั่วโมง
0 */6 * * * deploy /opt/chuaikan/scripts/sync-static.sh

# Database VACUUM ทุกอาทิตย์ ตี 4
0 4 * * 0 deploy vacuumdb -U chuaikan_user -d chuaikan_db --analyze >> /var/log/chuaikan/vacuum.log 2>&1

# Health check ทุก 5 นาที
*/5 * * * * deploy /opt/chuaikan/scripts/health-check.sh
EOF
```

### Step 9: Resource Monitoring

```bash
# ─── ติดตั้ง Monitoring Tools ─────────────────────────────
sudo apt install -y htop iotop sysstat dstat ncdu

# ─── CPU & Memory ─────────────────────────────────────────
# htop: interactive process viewer
htop
# กด: F2 = setup, F3 = search, F4 = filter, F5 = tree, F9 = kill, q = quit

# top แบบ one-shot (ไม่ interactive)
top -bn1 | head -20

# CPU info
lscpu | grep -E "CPU\(s\)|Model name|MHz"

# Memory
free -h                    # RAM usage
free -m                    # RAM usage ใน MB
cat /proc/meminfo | head -10

# ─── Disk ─────────────────────────────────────────────────
df -h                      # disk usage ของทุก filesystem
df -h /                    # root filesystem
du -sh /opt/chuaikan/      # ขนาดของ directory
du -sh /var/log/*           # ขนาดของแต่ละ log directory
ncdu /opt/chuaikan/         # interactive disk usage

# ─── I/O Statistics ───────────────────────────────────────
# iostat
sudo iostat -x 1 5         # extended stats ทุก 1 วิ จำนวน 5 ครั้ง
# ดูที่: %util (ถ้าสูงกว่า 90% = disk bottleneck)

# vmstat
vmstat 1 5                 # ดู CPU, memory, I/O ทุก 1 วิ
# Columns: r=run queue, b=blocked, si/so=swap, bi/bo=block I/O

# ─── Network ──────────────────────────────────────────────
ss -tuln                   # ดู listening ports
ss -tulnp                  # พร้อม process name
netstat -tuln              # alternative (ต้องติดตั้ง net-tools)
iftop                      # real-time bandwidth (sudo iftop)

# ─── Process ──────────────────────────────────────────────
ps aux | grep node          # หา Node.js processes
ps aux --sort=-%cpu | head  # เรียงตาม CPU usage
ps aux --sort=-%mem | head  # เรียงตาม memory usage
pgrep -a node               # หา PID ของ node processes
```

### Step 10: Backup Scripts

```bash
# ─── สร้าง Directory Structure ────────────────────────────
sudo mkdir -p /opt/chuaikan/scripts
sudo mkdir -p /var/backups/chuaikan/{db,files,configs}
sudo chown -R deploy:deploy /opt/chuaikan/scripts
sudo chown -R deploy:deploy /var/backups/chuaikan
```

```bash
# ─── สร้าง Database Backup Script ────────────────────────
cat > /opt/chuaikan/scripts/backup-db.sh << 'SCRIPT'
#!/bin/bash
# backup-db.sh — PostgreSQL backup script สำหรับ chuaikan.com
set -euo pipefail

# ─── Configuration ────────────────────────────────────────
DB_NAME="chuaikan_db"
DB_USER="chuaikan_user"
BACKUP_DIR="/var/backups/chuaikan/db"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/chuaikan_${DATE}.sql.gz"
LOG_FILE="/var/log/chuaikan/backup.log"

# ─── Functions ────────────────────────────────────────────
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# ─── Main ─────────────────────────────────────────────────
log "Starting database backup..."

# สร้าง directory ถ้ายังไม่มี
mkdir -p "$BACKUP_DIR"
mkdir -p "$(dirname "$LOG_FILE")"

# Backup
if pg_dump -U "$DB_USER" "$DB_NAME" | gzip > "$BACKUP_FILE"; then
    BACKUP_SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
    log "Backup created: $BACKUP_FILE (${BACKUP_SIZE})"
else
    log "ERROR: Backup failed!"
    exit 1
fi

# ตรวจสอบ integrity
if gzip -t "$BACKUP_FILE"; then
    log "Backup integrity: OK"
else
    log "ERROR: Backup file is corrupted!"
    rm -f "$BACKUP_FILE"
    exit 1
fi

# ลบ backup เก่า
DELETED=$(find "$BACKUP_DIR" -name "chuaikan_*.sql.gz" \
    -mtime "+${RETENTION_DAYS}" -delete -print | wc -l)
log "Deleted $DELETED old backup(s)"

log "Backup completed successfully"
SCRIPT

chmod +x /opt/chuaikan/scripts/backup-db.sh
```

```bash
# ─── สร้าง File Backup Script (rsync) ────────────────────
cat > /opt/chuaikan/scripts/backup-files.sh << 'SCRIPT'
#!/bin/bash
# backup-files.sh — rsync backup สำหรับ chuaikan.com
set -euo pipefail

SOURCE_DIRS=(
    "/opt/chuaikan/app/public/uploads"
    "/opt/chuaikan/configs"
    "/etc/nginx/sites-available"
)
BACKUP_DIR="/var/backups/chuaikan/files"
REMOTE_HOST="backup@backup-server.chuaikan.com"
REMOTE_DIR="/backups/chuaikan"
LOG_FILE="/var/log/chuaikan/backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Local rsync backup
for DIR in "${SOURCE_DIRS[@]}"; do
    if [ -d "$DIR" ]; then
        DEST="${BACKUP_DIR}${DIR}"
        mkdir -p "$DEST"
        rsync -avz --delete --exclude='*.tmp' \
            "$DIR/" "$DEST/"
        log "Synced: $DIR → $DEST"
    else
        log "WARNING: Source dir not found: $DIR"
    fi
done

# Remote rsync backup (ถ้ามี backup server)
# rsync -avz --delete -e "ssh -i /home/deploy/.ssh/backup_key" \
#     "$BACKUP_DIR/" "${REMOTE_HOST}:${REMOTE_DIR}/"

log "File backup completed"
SCRIPT

chmod +x /opt/chuaikan/scripts/backup-files.sh

# ทดสอบ scripts
/opt/chuaikan/scripts/backup-db.sh
/opt/chuaikan/scripts/backup-files.sh
```

---

## 🔧 Configuration Files

### /etc/logrotate.d/chuaikan

```conf
/var/log/chuaikan/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 deploy deploy
    sharedscripts
    postrotate
        systemctl reload chuaikan > /dev/null 2>&1 || true
    endscript
}
```

### fail2ban สำหรับ SSH Protection

```bash
sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
backend = systemd

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
maxretry = 3
bantime = 86400
EOF

sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```

---

## 🧪 Testing

```bash
# ─── Test Linux Commands ──────────────────────────────────
# ทดสอบ file permissions
touch /tmp/test-perms.txt
chmod 644 /tmp/test-perms.txt
ls -la /tmp/test-perms.txt
# Expected: -rw-r--r-- 1 user group ...

# ทดสอบ find
find /var/log -name "*.log" -type f -mtime -1 | head -5

# ─── Test SSH ─────────────────────────────────────────────
# ทดสอบ SSH config syntax
sudo sshd -t && echo "SSH config is valid"

# ─── Test UFW ─────────────────────────────────────────────
sudo ufw status verbose
sudo ufw show added

# ─── Test systemd Service ─────────────────────────────────
sudo systemctl status chuaikan
sudo journalctl -u chuaikan --since "5 minutes ago"

# ─── Test Backup ──────────────────────────────────────────
ls -lh /var/backups/chuaikan/db/
gzip -t /var/backups/chuaikan/db/chuaikan_*.sql.gz && echo "Backup OK"

# ─── System Health Check Script ───────────────────────────
cat > /opt/chuaikan/scripts/health-check.sh << 'SCRIPT'
#!/bin/bash
# health-check.sh
set -euo pipefail

ENDPOINT="http://localhost:3000/api/health"
TIMEOUT=10
LOG="/var/log/chuaikan/health.log"

RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
    --max-time "$TIMEOUT" "$ENDPOINT" 2>/dev/null || echo "000")

if [ "$RESPONSE" = "200" ]; then
    echo "[$(date)] Health check: OK (HTTP $RESPONSE)"
else
    echo "[$(date)] Health check: FAIL (HTTP $RESPONSE)" | tee -a "$LOG"
    # ส่ง alert (ตัวอย่าง)
    # curl -X POST "https://hooks.slack.com/..." \
    #   -d "{\"text\":\"chuaikan.com is DOWN! HTTP $RESPONSE\"}"
fi
SCRIPT
chmod +x /opt/chuaikan/scripts/health-check.sh
```

---

## ❌ Common Errors & Solutions

### Error 1: Permission denied เมื่อ SSH

```
❌ Error:
ssh: connect to host server.com port 22: Connection refused
Permission denied (publickey)

✅ Solution:
# ตรวจสอบว่า SSH service กำลัง run
sudo systemctl status sshd

# ตรวจสอบว่า authorized_keys ถูกต้อง
cat ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh/

# Debug SSH connection
ssh -vvv -i ~/.ssh/chuaikan_ed25519 deploy@server.com
# ดูที่ output: "Authentication succeeded (publickey)"
```

### Error 2: UFW ล็อค SSH ตัวเอง

```
❌ Error: ล็อค SSH access หลัง enable UFW

✅ Prevention:
# เสมอ allow SSH ก่อน enable UFW
sudo ufw allow 22/tcp
sudo ufw enable

# ถ้า locked แล้ว → ใช้ console/VNC เพื่อ disable
sudo ufw disable
sudo ufw allow 22/tcp
sudo ufw enable
```

### Error 3: systemd Service ไม่ Start

```
❌ Error:
● chuaikan.service - chuaikan.com Next.js Application
   Loaded: loaded (/etc/systemd/system/chuaikan.service; enabled)
   Active: failed (Result: exit-code)

✅ Solution:
# ดู logs ละเอียด
sudo journalctl -u chuaikan -n 50 --no-pager

# ตรวจสอบ:
# 1. ExecStart path ถูกต้อง?
which node    # /usr/bin/node หรือ /usr/local/bin/node?
# 2. Working directory มีอยู่?
ls -la /opt/chuaikan/app/
# 3. User มีสิทธิ์?
sudo -u deploy ls /opt/chuaikan/app/
# 4. EnvironmentFile มีอยู่?
ls -la /opt/chuaikan/.env.production
```

### Error 4: Cron Job ไม่ทำงาน

```
❌ Error: Cron job ไม่ execute ตามกำหนด

✅ Solution:
# ตรวจสอบ cron service
sudo systemctl status cron

# ดู cron logs
sudo grep CRON /var/log/syslog | tail -20

# Debug: test run script manually ด้วย user เดียวกัน
sudo -u deploy /opt/chuaikan/scripts/backup-db.sh

# ตรวจสอบ PATH ใน cron script
# Cron มี limited PATH ต้อง set explicitly
export PATH=/usr/local/bin:/usr/bin:/bin
```

---

## 📊 Performance Benchmark

| ตัวชี้วัด | Before Optimization | After Optimization |
|---------|---------------------|-------------------|
| SSH login time | ~3 seconds | < 1 second |
| Log rotation frequency | None | Daily |
| Backup size | Uncompressed | 70% smaller (gzip) |
| Failed login attempts | >100/day | <5/day (fail2ban) |
| Open file descriptors limit | 1024 | 65536 |

---

## ✅ Checklist

- [ ] ติดตั้ง Ubuntu 24.04 LTS
- [ ] อัพเดต packages (`sudo apt update && sudo apt upgrade -y`)
- [ ] สร้าง user `deploy` ที่ไม่ใช่ root
- [ ] ตั้งค่า SSH key-based authentication
- [ ] ปิด SSH password authentication
- [ ] ปิด SSH root login
- [ ] Enable UFW firewall
- [ ] Allow port 22, 80, 443 ใน UFW
- [ ] ติดตั้ง fail2ban
- [ ] ตั้งค่า file permissions ให้ถูกต้อง (644/755)
- [ ] สร้าง systemd service สำหรับ app
- [ ] ตั้ง crontab สำหรับ backup และ maintenance
- [ ] ติดตั้ง htop, iostat สำหรับ monitoring
- [ ] ตั้งค่า logrotate
- [ ] ทดสอบ backup script และ verify integrity
- [ ] ทดสอบ health-check script

---

## 🔗 References

- [Ubuntu 24.04 LTS Documentation](https://ubuntu.com/server/docs)
- [Ubuntu UFW Guide](https://help.ubuntu.com/community/UFW)
- [systemd Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
- [SSH Hardening Guide](https://www.ssh.com/academy/ssh/hardening)
- [Crontab Guru](https://crontab.guru)
- [fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)

---
*Part 001 | Road to 1,000,000 Users/Day | chuaikan.com*
