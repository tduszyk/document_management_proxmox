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

                                  
