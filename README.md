# High-Performance Proxmox Storage & Document Architecture: Native ZFS, Alpine Samba, and Paperless-ngx LXC

This guide details migrating legacy file storage and document management from a monolithic virtual machine (e.g., FreeBSD / TrueNAS) to a decoupled, high-performance architecture on Proxmox VE. 

---

## Architecture Overview

                     +---------------------------------------+
                     |       Proxmox VE Host (sulaco)        |
                     |  - Native ZFS Storage Pool (xxxT)    |
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
