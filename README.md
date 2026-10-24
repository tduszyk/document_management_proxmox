# High-Performance Proxmox Storage & Document Architecture: Native ZFS, Alpine Samba, and Paperless-ngx LXC

This guide details migrating legacy file storage and document management from a monolithic virtual machine (e.g., FreeBSD / TrueNAS) to a decoupled, high-performance architecture on Proxmox VE. 

---

## Architecture Overview

                     +---------------------------------------+
                     |       Proxmox VE Host (sulaco)        |
                     |  - Native ZFS Storage Pool (xxxT)     |
                     |  - Linux VFS Bind Mounts              |
                     +-------------------+-------------------+
                                         |
                    +--------------------+--------------------+
                    |                                         |
    +---------------+---------------+         +---------------+---------------+
    |       Alpine Linux LXC        |         |        Debian 12 LXC          |
    |            (105)              |         |            (106)              |
    |-------------------------------|         |-------------------------------|
    | - Samba File Server           |         | - Paperless-ngx (Bare Metal)  |
    | - vfs_fruit macOS Tuning      |         | - Granian ASGI Webserver      |
    | - Drop-folder Ingestion Share |         | - PostgreSQL (UTF-8) & Redis  |
    +---------------+---------------+         | - Celery Ingestion Workers    |
                    |                         +---------------+---------------+
                    |                                         |
      smb://192.168.1.108/storage               [http://192.168.1.109:8000]
      smb://192.168.1.108/consume
                    |                                         |
                    +--------------------+--------------------+
                                         |
                                  +------+------+
                                  | Mac Clients |
                                  +-------------+

                                  
### Core Design Principles
* **Hypervisor Cleanliness:** The Proxmox host manages only the hardware, native ZFS zpool, and container lifecycles. Zero third-party application daemons listen directly on host management ports.
* **Service Isolation:** 
  * Samba runs inside a minimal 512 MB Alpine container (`105`).
  * Paperless-ngx runs inside a Debian 12 container (`106`) with dependencies installed in an isolated Python virtual environment.
* **Storage Hierarchy:** Fast metadata, queues, and PostgreSQL transactions live on host SSD/NVMe root storage, while bulk media archives and ingestion folders live on spinning rust (`storage` pool) via unprivileged LXC bind mounts.
* **Preserved Client Identities:** Existing IP addresses, MAC addresses, SMB credentials, and fruit attributes are replicated 1:1, making the cutover invisible to macOS clients.

---

## 1. Host Storage Pool Tuning & Dataset Layout

Import the storage pool natively onto the Proxmox host and create dedicated datasets for Paperless ingestion, media storage, and disaster recovery.

### Host Storage Setup
Execute on the **Proxmox host (`sulaco`)**:

```
# 1. Import pool and set stable mountpoint
zpool import -f storage
zfs set mountpoint=/bays/storage storage

# 2. Disable automated NFS shares inherited from legacy configs
zfs set sharenfs=off storage

# 3. Apply ZFS optimizations for spinning disks & macOS extended attributes
zfs set atime=off storage
zfs set compression=zstd storage
zfs set xattr=sa storage
zfs set recordsize=1M storage

# 4. Create Paperless datasets
zfs create storage/paperless
zfs create storage/paperless/consume
zfs create storage/paperless/media
zfs create storage/paperless/export

# 5. Align unprivileged container permissions
# Container root (UID 0) maps to host 100000; container group 'backup' (GID 1002) maps to host 101002
chown -R 100000:101002 /bays/storage
chmod -R 2775 /bays/storage
```

## 2. Alpine Samba LXC (Container 105)
Deploy a lightweight Alpine Linux container to serve SMB shares with macOS Finder optimizations (vfs_fruit).

Container Provisioning
Execute on the Proxmox host:

```
# 1. Download the latest Alpine template
pveam update
pveam download local alpine-3.24-default_20260714_amd64.tar.xz

# 2. Create unprivileged LXC with identical IP and MAC identity
pct create 105 local:vztmpl/alpine-3.24-default_20260714_amd64.tar.xz \
  --hostname storage \
  --memory 512 \
  --swap 512 \
  --cores 1 \
  --ostype alpine \
  --unprivileged 1 \
  --features nesting=1 \
  --onboot 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.108/24,gw=192.168.1.1,hwaddr=6a:31:63:16:9a:1f \
  --rootfs local-lvm:2

# 3. Attach ZFS datasets via bind mount
pct set 105 -mp0 /bays/storage,mp=/bays/storage
pct set 105 -mp1 /bays/storage/paperless/consume,mp=/bays/storage/consume

# 4. Start container
pct start 105
pct enter 105
```

Samba Service Configuration
Execute inside Alpine LXC 105 (storage:~#):

```
# 1. Install Samba
apk update && apk add samba samba-common-tools

# 2. Create group and user matching existing permissions
addgroup -g 1002 backup
adduser -D -G backup t

# Set Samba password (use original SMB credentials)
smbpasswd -a t
smbpasswd -e t

# 3. Write /etc/samba/smb.conf
cat << 'EOF' > /etc/samba/smb.conf
[global]
    workgroup = WORKGROUP
    server string = storage
    netbios name = storage
    security = user
    passdb backend = tdbsam

    # Performance & Protocol
    server min protocol = SMB2_10
    server max protocol = SMB3
    use sendfile = yes
    aio read size = 1
    aio write size = 1

    # macOS Finder Optimizations (vfs_fruit)
    vfs objects = fruit streams_xattr
    fruit:metadata = stream
    fruit:model = Macmini
    fruit:posix_rename = yes
    fruit:veto_appledouble = no
    fruit:wipe_intentionally_left_blank_rfork = yes
    fruit:delete_empty_adfiles = yes
    fruit:encoding = native

[storage]
    comment = ZFS Shared Storage
    path = /bays/storage
    valid users = @backup
    write list = @backup
    read only = no
    browseable = yes
    guest ok = no
    force group = backup
    create mask = 0664
    force create mode = 0664
    directory mask = 0775
    force directory mode = 0775
    inherit permissions = yes

[consume]
    comment = Paperless Ingestion Drop
    path = /bays/storage/consume
    valid users = @backup
    write list = @backup
    read only = no
    browseable = yes
    guest ok = no
    force group = backup
    create mask = 0664
    force create mode = 0664
    directory mask = 0775
    force directory mode = 0775
    inherit permissions = yes
EOF

# 4. Enable and start Samba
rc-update add samba default
rc-service samba start
```

## 3. Paperless-ngx Debian LXC (Container 106)
Deploy a dedicated Debian 12 (Bookworm) container to host the OCR engine, background processing workers, and web interface.

Container Provisioning
Execute on the Proxmox host:

```
# 1. Download Debian 12 template
pveam download local debian-12-standard_12.7-1_amd64.tar.zst

# 2. Create unprivileged LXC
pct create 106 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname paperless \
  --memory 4096 \
  --swap 2048 \
  --cores 4 \
  --ostype debian \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --onboot 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.109/24,gw=192.168.1.1 \
  --rootfs local-lvm:16

# 3. Bind mount ZFS datasets to internal paths
pct set 106 -mp0 /bays/storage/paperless/consume,mp=/mnt/consume
pct set 106 -mp1 /bays/storage/paperless/media,mp=/mnt/media
pct set 106 -mp2 /bays/storage/paperless/export,mp=/mnt/export

# 4. Start container
pct start 106
pct enter 106
```

System Locale & Package Setup
Execute inside Debian LXC 106 (paperless:~#):

```
# 1. Configure UTF-8 system locales (Critical for PostgreSQL and Python OCR)
apt update && apt install -y locales
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 2. Install runtime and build dependencies
apt install -y \
  curl gnupg sudo git python3 python3-pip python3-venv \
  libpq-dev build-essential python3-dev pkg-config default-libmysqlclient-dev \
  imagemagick tesseract-ocr tesseract-ocr-eng ghostscript icc-profiles-free \
  qpdf liblept5 libxml2 libxslt1.1 antiword unpaper poppler-utils \
  redis-server postgresql postgresql-contrib

# 3. Permit ImageMagick PDF rendering in security policy
sed -i 's/<policy domain="coder" rights="none" pattern="PDF" \/>/<policy domain="coder" rights="read|write" pattern="PDF" \/>/' /etc/ImageMagick-6/policy.xml
```

PostgreSQL & Redis Initialization
Initialize a clean, UTF-8 encoded database cluster using template0:

```
cd /tmp
sudo -u postgres psql -c "CREATE USER paperless WITH PASSWORD 'paperless_secure_pwd';"
sudo -u postgres psql -c "CREATE DATABASE paperless WITH TEMPLATE template0 ENCODING 'UTF8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8' OWNER = paperless;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE paperless TO paperless;"

# Start and enable Redis
systemctl enable --now redis-server
```

Paperless-ngx Core Installation

```
# 1. Create service account and directory structure
groupadd -g 1002 backup 2>/dev/null || true
useradd --system --home-dir /opt/paperless --shell /bin/bash -G backup paperless

mkdir -p /opt/paperless /opt/paperless/data
chown -R paperless:backup /opt/paperless
chown -R paperless:backup /mnt/consume /mnt/media /mnt/export
chmod -R 2775 /mnt/consume /mnt/media /mnt/export

# 2. Virtual environment and release asset download
cd /opt/paperless
sudo -u paperless python3 -m venv venv
sudo -u paperless ./venv/bin/pip install --upgrade pip

PAPERLESS_VER=$(curl -s [https://api.github.com/repos/paperless-ngx/paperless-ngx/releases/latest](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases/latest) | grep tag_name | cut -d '"' -f 4)
curl -L -o paperless.tar.xz "[https://github.com/paperless-ngx/paperless-ngx/releases/download/$](https://github.com/paperless-ngx/paperless-ngx/releases/download/$){PAPERLESS_VER}/paperless-ngx-${PAPERLESS_VER}.tar.xz"
tar -xf paperless.tar.xz -C /opt/paperless --strip-components=1
rm paperless.tar.xz

# 3. Install Python requirements (includes Granian ASGI server)
sudo -u paperless ./venv/bin/pip install -r requirements.txt

# 4. Generate configuration file
cat << 'EOF' > /opt/paperless/paperless.conf
# Paths
PAPERLESS_DATA_DIR=/opt/paperless/data
PAPERLESS_MEDIA_ROOT=/mnt/media
PAPERLESS_CONSUMPTION_DIR=/mnt/consume
PAPERLESS_EXPORT_DIR=/mnt/export

# Database
PAPERLESS_DBENGINE=postgresql
PAPERLESS_DBHOST=localhost
PAPERLESS_DBPORT=5432
PAPERLESS_DBNAME=paperless
PAPERLESS_DBUSER=paperless
PAPERLESS_DBPASS=paperless_secure_pwd

# Redis
PAPERLESS_REDIS=redis://localhost:6379

# Storage Performance Tuning
PAPERLESS_CONSUMER_POLLING=30
PAPERLESS_TASK_WORKERS=2
PAPERLESS_THREADS_PER_WORKER=2

# Network & Host Binding
PAPERLESS_URL=[http://192.168.1.109:8000](http://192.168.1.109:8000)
PAPERLESS_ALLOWED_HOSTS=192.168.1.109,localhost,paperless.local
PAPERLESS_OCR_LANGUAGE=eng

# Generate a unique secret key
PAPERLESS_SECRET_KEY=change-this-to-a-random-secret-key-32-chars
EOF

chown paperless:backup /opt/paperless/paperless.conf
chmod 640 /opt/paperless/paperless.conf

# 5. Run database migrations & initialize administrator
cd /opt/paperless/src
sudo -u paperless ../venv/bin/python3 manage.py migrate
sudo -u paperless ../venv/bin/python3 manage.py createsuperuser
```

## 4. Systemd Service Architecture
Paperless operates through four distinct systemd service units inside the Debian container:

1. Web Application Server (paperless-webserver.service)
Powered by the modern Rust-based ASGI server Granian:

```
[Unit]
Description=Paperless Webserver (Granian)
After=network.target postgresql.service redis-server.service

[Service]
Type=simple
User=paperless
Group=backup
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/granian --interface asgi --host 0.0.0.0 --port 8000 --workers 2 paperless.asgi:application
EnvironmentFile=/opt/paperless/paperless.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

2. Task Queue Worker (paperless-worker.service)
Handles OCR, thumbnailing, and indexing operations:

```
[Unit]
Description=Paperless Celery Worker
After=network.target postgresql.service redis-server.service paperless-webserver.service

[Service]
Type=simple
User=paperless
Group=backup
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/celery --app paperless worker --loglevel INFO -c 2
EnvironmentFile=/opt/paperless/paperless.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

3. File Consumer (paperless-consumer.service)
Monitors the /mnt/consume dataset for incoming documents:

```
[Unit]
Description=Paperless Document Consumer
After=network.target postgresql.service redis-server.service paperless-webserver.service

[Service]
Type=simple
User=paperless
Group=backup
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/python3 manage.py document_consumer
EnvironmentFile=/opt/paperless/paperless.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

4. Celery Beat Scheduler (paperless-scheduler.service)
Manages periodic tasks and maintenance jobs:

```
[Unit]
Description=Paperless Celery Beat Scheduler
After=network.target postgresql.service redis-server.service paperless-webserver.service

[Service]
Type=simple
User=paperless
Group=backup
WorkingDirectory=/opt/paperless/src
ExecStart=/opt/paperless/venv/bin/celery --app paperless beat --loglevel INFO
EnvironmentFile=/opt/paperless/paperless.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and Start Stack

```
systemctl daemon-reload
systemctl enable --now paperless-webserver paperless-worker paperless-consumer paperless-scheduler
```

## 5. Automated Disaster Recovery
Paperless-ngx includes a native export utility that dumps all metadata, correspondents, document tags, and raw files into an immutable, self-contained archive.

Configure a nightly export on Debian LXC 106:

```
# Add cron job scheduled for 02:00 nightly
(crontab -l 2>/dev/null; echo "0 2 * * * cd /opt/paperless/src && sudo -u paperless ../venv/bin/python3 manage.py document_exporter /mnt/export -c > /dev/null 2>&1") | crontab -
```

## 6. Verification and Daily Workflow
1. Mount Storage: Open Finder on macOS -> Cmd + K -> smb://192.168.1.108/storage.
2. Mount Drop Folder: Connect to smb://192.168.1.108/consume.
3. Ingest Document: Drop any PDF, scan, or TIFF into consume.
4. Processing Pipeline:
   a. Samba writes the file with GID 1002.
   b. paperless-consumer picks up the file within 30 seconds and queues a Celery task.
   c. paperless-worker extracts text using Tesseract OCR, generates a thumbnail, files the asset into /mnt/media, and clears the drop folder.
5. Access Dashboard: Review, search, and tag documents via the Web UI at http://192.168.1.109:8000.
