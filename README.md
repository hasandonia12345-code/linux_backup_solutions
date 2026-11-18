# Enterprise Data Protection: Amanda & LTO-6 Tape Systems

Documentation, configuration examples, and operational notes for a centralized backup environment using **Amanda**, **GNU Tar**, and **LTO tape libraries**.

> **Public-repository notice:** This repository is intended for learning, reference, and portfolio use. All hostnames, IP addresses, usernames, paths, device names, tape labels, and other environment-specific values in examples are placeholders. Do not commit production configuration files, private keys, passwords, tokens, logs containing sensitive data, or real tape barcodes.

## Project Overview

Amanda (the Advanced Maryland Automatic Network Disk Archiver) coordinates backups from multiple client systems to a centralized backup server. In the documented database-backup use case, database backup scripts first create export, dump, or RMAN files on the client. Amanda then backs up the selected directory using **GNU Tar (`GNUTAR`)** and transfers the backup stream to the backup server.

The backup server can stage backup images on a **holding disk** before writing them to tape. An LTO tape drive and an autochanger/tape library provide removable-media storage. Amanda's configuration determines which DLEs are backed up, which backup program is used, how data is compressed, and how holding-disk images are flushed to tape.

The detailed administration guide is here:

- [`amanda-backup-guide.md`](amanda-backup-guide.md) — architecture, Amanda configuration, GNU Tar, compression, holding-disk policies, tape/changer management, administration commands, recovery, and troubleshooting.

## Architecture

A typical deployment contains these components:

| Component | Role |
|---|---|
| Amanda backup server | Schedules and coordinates backup runs, receives backup streams, manages holding-disk data, and writes to tape. |
| Amanda client | Runs `amandad` and provides access to the selected filesystem or directory. |
| Database backup scripts | Generate database backup files before Amanda reads the directory. |
| GNU Tar (`GNUTAR`) | Archives the files selected by a DLE into the backup stream. |
| `disklist` | Defines the client host, disk/directory (DLE), and dumptype for each backup. |
| `amanda.conf` | Defines backup policy, dumptypes, compression, holding disk, tape type, changer, and other settings. |
| Holding disk | Temporary staging space for backup images before tape writing. |
| LTO tape drive and autochanger | Store backup images on removable tape and load/unload cartridges. |

### Backup flow

1. A database backup job creates the required files on a client, for example under `/backup/database/export` or `/backup/database/rman`.
2. Amanda reads the configured DLEs from `disklist`.
3. Amanda contacts the client using the communication/authentication method configured for that DLE's dumptype.
4. The client-side Amanda daemon (`amandad`) coordinates the requested operation and GNU Tar reads the selected directory.
5. The backup stream is transferred to the Amanda server.
6. Amanda stages the dump on the holding disk when the selected holding-disk policy allows it, or sends it directly to the storage device.
7. The taper writes eligible dump images to tape. Amanda records run and volume metadata for reporting and recovery.

**Important:** Amanda is not merely an SCP wrapper. It coordinates the backup protocol, dump program, scheduling, staging, tape writing, metadata, and recovery workflow.

## Client Communication: Choose One Authentication Model

Amanda supports multiple communication/authentication methods. The `auth` setting in the dumptype selected by a DLE determines which method Amanda uses. The two examples below are alternatives. **Do not describe or configure one DLE as using SSH and xinetd/`bsdtcp` at the same time.**

### Option A — SSH authentication

Use this model when the Amanda dumptype specifies:

```text
auth "ssh"
```

With SSH authentication, Amanda uses SSH to start `amandad` on the client. SSH provides authentication and encrypted transport. Passwordless public-key authentication is normally configured for the dedicated Amanda service account.

A basic connectivity preparation sequence, run as the Amanda backup account on the server, may look like this:

```bash
# Run as the Amanda service account; adjust the home directory and key name.
ssh-keygen -t ed25519 -f ~/.ssh/amanda_client_ed25519

# Install the public key on the client account.
ssh-copy-id -i ~/.ssh/amanda_client_ed25519.pub \
  amandabackup@client-a.example.invalid

# Test non-interactive SSH authentication.
ssh -i ~/.ssh/amanda_client_ed25519 \
  -o BatchMode=yes \
  amandabackup@client-a.example.invalid true
```

These commands are a starting point, not a complete Amanda SSH configuration. The dumptype's SSH-related options, client username, daemon path, key location, and key restrictions must match the installed Amanda release and local package layout. For unattended operation, use a dedicated key and restrict it in `authorized_keys` (for example, with a forced Amanda command and disabled forwarding features) as described by the Amanda SSH-authentication documentation.

**Security note:** Do not blindly run `passwd -u amandabackup` as a standard setup step. Unlocking an account changes its password/account state and is not the same as enabling SSH public-key authentication. Follow the operating-system and SSH/PAM policy for service accounts. Do not enable password-based SSH login merely to make key authentication work.

**xinetd is not used to launch `amandad` for this SSH-authentication path.** SSH starts the daemon.

### Option B — `bsdtcp` authentication through xinetd

Use this model when the client-side xinetd service launches `amandad` with `-auth=bsdtcp`, and the selected Amanda dumptype uses:

```text
auth "bsdtcp"
```

Illustrative client file, commonly installed as `/etc/xinetd.d/amanda`:

```text
service amanda
{
    only_from   = 192.0.2.20
    socket_type = stream
    protocol    = tcp
    wait        = no
    user        = amandabackup
    group       = disk
    groups      = yes
    server      = /usr/lib64/amanda/amandad
    server_args = -auth=bsdtcp amdump
    disable     = no
}
```

Replace `192.0.2.20` with the actual, trusted Amanda server address and verify the `amandad` executable path on the target client. Some distributions install it under a different directory. The xinetd service name and port must also agree with the client's `/etc/services` entry and Amanda's `client-port` configuration (the common default service is `amanda`, TCP port `10080`).

The client-side example uses `amdump` only. `amindexd` and `amidxtaped` are recovery/index services normally configured on the Amanda **server** when required; they should not be added to a client service just because they appear in a server-side example.

For `bsdtcp`, configure the appropriate `.amandahosts` authorization and restrict network access to the backup server. A typical client authorization entry is conceptually:

```text
backup-server.example.invalid  amandabackup  amdump
```

Check the installed Amanda documentation and the actual service account's home directory before creating the file.

## GNU Tar and Database Backup Files

This project assumes that database-specific scripts may run before Amanda. Those scripts generate the source files; Amanda does not create the database-native export or RMAN backup itself.

Example source directories:

```text
/backup/database/export
/backup/database/rman
```

Example DLE (illustrative only):

```text
client-a.example.invalid /backup/database/rman db-archive
```

The `db-archive` dumptype must be defined in `amanda.conf` and must select the intended program and authentication model. For a GNU Tar backup, the program is commonly configured as `GNUTAR`.

## Configuration Concepts

- **`amanda.conf`:** Defines dumptypes, tapetypes, holding disks, storage/changer settings, compression, tape policy, and other Amanda behavior.
- **`disklist`:** Defines what to back up. A DLE normally includes a client hostname, diskname, and dumptype.
- **Dumptype:** Defines how a DLE is backed up, including the backup program, authentication, compression, and holding-disk policy.
- **Holding disk:** Temporarily stores dump images before tape writing. Its capacity must be evaluated against dump sizes and tape-flushing thresholds.
- **Tape type:** Defines Amanda's estimate of a tape volume's usable length and speed. Do not assume a manufacturer's advertised compressed capacity is guaranteed; actual compression depends on the source data.
- **Autochanger:** Controls cartridge movement between magazine slots and the tape drive. The changer device and tape-drive device are different device interfaces and must be identified independently.

For detailed holding-disk flushing scenarios and their capacity constraints, see Section 7.6 of [`amanda-backup-guide.md`](amanda-backup-guide.md).

## Repository Layout

The following is the intended public layout. Configuration files should be committed as sanitized examples (`*.example`), not as unredacted production files.

```text
.
├── README.md
├── amanda-backup-guide.md
├── configs/
│   ├── amanda.conf.example
│   ├── disklist.example
│   └── xinetd-amanda-client.example
├── scripts/
│   ├── setup_ssh_auth.sh
│   └── amanda_cron_job.sh
└── docs/
    └── troubleshooting.md
```

The supporting files can be added progressively. If a file is not present in the repository yet, treat it as a planned example rather than an available script or configuration.

## Operations

Common Amanda commands used in this project include:

```bash
# Check configuration and client readiness
amcheck BACKUP_SET

# Run a backup
amdump BACKUP_SET

# Display a run's status
amstatus BACKUP_SET

# Generate a report for a completed run
amreport BACKUP_SET

# Flush eligible holding-disk dumps to tape
amflush BACKUP_SET
```

Replace `BACKUP_SET` with the actual Amanda configuration name. Confirm the command options against the version installed on the backup server before scheduling jobs.

A cron job should run under the Amanda service account and should log its output. Do not put credentials, private keys, or production host inventories in a publicly committed cron script.

## Recovery and Troubleshooting

Recovery should be tested regularly, not inferred from a successful backup report. The detailed guide covers Amanda restore utilities, tape positioning, tape-device checks, SCSI generic devices, changer checks, holding-disk capacity, EOF messages, and tape block-size troubleshooting.

Useful diagnostics include:

```bash
lsscsi -g
mt -f /dev/nstX status
mtx -f /dev/sgX status
 df -h
 df -i
```

Replace device placeholders with the verified local device paths. Tape erase, relabel, and overwrite operations are destructive; verify the cartridge and retention policy before executing them.

## Scope and Disclaimer

This repository is an operational learning and documentation project, not a vendor-supported deployment package. Commands, configuration syntax, executable paths, authentication behavior, and tape/changer integration can differ by Amanda release, Linux distribution, and hardware. Validate examples in a test environment and consult the man pages installed with the target Amanda version before applying changes to production.

## References

- [Amanda authentication methods (`amanda-auth(7)`)](https://www.mankier.com/7/amanda-auth)
- [Amanda main configuration (`amanda.conf(5)`)](https://manpages.debian.org/unstable/amanda-common/amanda.conf.5.en.html)
- [Amanda DLE / disklist format (`disklist(5)`)](https://www.mankier.com/5/disklist)
- [Amanda community backup guide](https://www.zmanda.com/wp-content/uploads/2020/03/amanda-community-backup-guide.pdf)
