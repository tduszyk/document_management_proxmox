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

```bash
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
