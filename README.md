# Enterprise Data Protection: AMANDA & LTO-6 Tape Systems

## 📌 Project Overview
Documentation and configuration for a mission-critical backup environment using **AMANDA** and **LTO-6 Tape Autoloaders**. 

## 🏗 System Architecture
AMANDA pulls data from clients using `gnutar` via SSH. The backup server utilizes `xinetd` on the client-side to manage the `amandad` process.

### Automated SSH Authentication
To enable passwordless backups, the `amandabackup` user is unlocked and keys are exchanged:
1. `passwd -u amandabackup`
2. `su - amandabackup -c "ssh-keygen"`
3. `ssh-copy-id amandabackup@[Client_IP]`

### Client xinetd Config #communt 
```text
service amanda {
    socket_type = stream
    protocol    = tcp
    user        = amandabackup
    group       = disk
    server      = /usr/lib64/amanda/amandad
    server_args = -auth=bsdtcp amdump amindexd amidxtaped
}

.
├── configs/
│   ├── amanda.conf            # The "overridden" options you mentioned
│   ├── disklist               # Sample DLE (Disk List Entries)
│   └── xinetd_amandaclient    # Your xinetd service configuration
├── scripts/
│   ├── setup_ssh_auth.sh      # Script to automate the ssh-keygen/copy-id process
│   └── amanda_cron_job.sh     # Example of the amdump/amreport crontab
├── docs/
│   └── troubleshooting.md     # Your notes on SCSI errors and LTO tape sizes
└── README.md                  # The main portfolio page
