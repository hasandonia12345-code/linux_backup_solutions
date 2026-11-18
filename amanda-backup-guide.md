# Amanda Backup System
## Architecture, Configuration, Operations, Tape Management, and Recovery

**Technical Administration Guide – Public Edition**

> This guide uses generic hostnames, paths, tape-device placeholders, and backup-set names. Replace placeholders only in private deployment documentation. Do not publish real infrastructure identifiers, credentials, tape barcodes, logs, or internal network details.

---

## Table of Contents

1. [Introduction to Amanda](#1-introduction-to-amanda)
2. [GNU Tar and Its Role in Amanda](#2-gnu-tar-and-its-role-in-amanda)
3. [Amanda Configuration Files](#3-amanda-configuration-files)
4. [Compression in Amanda](#4-compression-in-amanda)
5. [Tape Devices and SCSI Configuration](#5-tape-devices-and-scsi-configuration)
6. [Tape Management and the Autochanger](#6-tape-management-and-the-autochanger)
7. [Holding Disk Management](#7-holding-disk-management)
8. [Managing Multiple Dumps on One Tape](#8-managing-multiple-dumps-on-one-tape)
9. [Amanda Administration Commands](#9-amanda-administration-commands)
10. [Recovery and Restoration](#10-recovery-and-restoration)
11. [Tape Management Using `mt`](#11-tape-management-using-mt)
12. [Troubleshooting and Common Errors](#12-troubleshooting-and-common-errors)
13. [Operational Recommendations](#13-operational-recommendations)
14. [Conclusion](#14-conclusion)
15. [Public Repository Security and Anonymization](#15-public-repository-security-and-anonymization)
16. [References](#16-references)

---

## 1. Introduction to Amanda

Amanda (Advanced Maryland Automatic Network Disk Archiver) is an open-source network backup and archiving solution. It allows administrators to manage backups from multiple client servers through a centralized backup server.

Amanda supports different backup methods, storage devices, compression mechanisms, and tape libraries.

In a typical deployment, Amanda collects backup data from remote client servers and transfers it to a centralized backup server, where the data is staged and eventually written to backup storage.

### 1.1 Main Components

| Component | Responsibility |
|---|---|
| Amanda Server | Coordinates backup operations and manages backup storage and tape devices |
| Amanda Client | Provides access to the files and directories being backed up |
| GNU Tar | Archives files and directories into a backup stream when the `GNUTAR` dumptype is selected |
| `disklist` | Defines the hosts and Disk List Entries (DLEs) that Amanda must back up |
| `amanda.conf` | Defines backup behavior, compression, holding disks, tape settings, and other options |
| Holding Disk | Temporary storage for backup images before they are written to tape |
| Tape Drive | Writes backup images to magnetic tape |
| Autochanger | Moves cartridges between storage slots and the tape drive |

### 1.2 How Amanda Works

A typical Amanda backup workflow is:

1. Amanda runs on the centralized backup server.
2. Amanda reads the configured DLEs from the `disklist` file.
3. It identifies the client server and the disk or directory that must be backed up.
4. It establishes communication with the client using the configured authentication method.
5. The selected backup mechanism, such as GNU Tar, reads the required files.
6. Amanda transfers the resulting backup data to the backup server.
7. Depending on the holding-disk and tape configuration, Amanda stages the data on the holding disk or sends it directly to storage.
8. The backup is recorded in Amanda's logs and databases, allowing administrators to monitor and restore it later.

**Important:** Amanda is not simply an SSH/SCP file-copy utility. SSH can be used as an authentication or transport-related option depending on the Amanda release and configuration. Amanda coordinates the backup process, client communication, data transfer, storage, and cataloging. Verify the authentication method supported by the installed release rather than assuming every Amanda deployment uses SSH.

---

## 2. GNU Tar and Its Role in Amanda

GNU Tar is a command-line utility used to create, extract, and manage archive files.

The name TAR comes from *Tape ARchiver*, because the utility was originally designed to store files on magnetic tape. Despite its name, GNU Tar can also create archives on local disks and other storage destinations.

### 2.1 Main Features

1. **Creating archives:** Combines multiple files and directories into one archive.
2. **Extracting archives:** Restores files from an existing archive.
3. **Listing archive contents:** Displays files and directories stored inside an archive without extracting them.
4. **Updating archives:** Can append files to an uncompressed archive, subject to format and storage limitations.

### 2.2 Basic GNU Tar Commands

**Create an archive:**

```bash
tar -cvf archive.tar file1 file2
```

- `c`: Create an archive.
- `v`: Display the files being processed.
- `f`: Specify the archive filename.

**Extract an archive:**

```bash
tar -xvf archive.tar
```

**List archive contents:**

```bash
tar -tvf archive.tar
```

**Append a file to an archive:**

```bash
tar -rvf archive.tar newfile
```

**Note:** The append operation does not work on compressed archives. It also has limitations with tape devices and other non-random-access storage. It should not be treated as a general method for modifying Amanda backups.

### 2.3 GNU Tar in a Database Backup Environment

In some environments, database administrators already maintain scripts responsible for generating database backup and archive files.

These scripts may create files inside predefined directories, for example:

```text
/backup/database/export
/backup/database/rman
```

Generated files may use extensions such as:

```text
*.dump
```

Amanda can be configured to use GNU Tar through a `GNUTAR` dumptype.

In this scenario:

- The database backup script generates the backup files.
- GNU Tar reads and archives the selected files or directories.
- Amanda coordinates the backup process and transfers the resulting archive stream to the backup server.

For example, when Amanda backs up `/backup/database/rman`, GNU Tar reads the files inside that directory and generates an archive stream. Amanda does not need to understand the internal database format of a `.dump` file; it treats it as a file to be archived.

---

## 3. Amanda Configuration Files

Amanda primarily uses two configuration files to determine what to back up and how to perform the backup.

### 3.1 `amanda.conf`

The `amanda.conf` file defines how Amanda operates. It contains settings such as:

- Client authentication and communication options.
- Backup program selection.
- Compression.
- Backup scheduling.
- Holding-disk configuration.
- Tape-device and changer configuration.
- Backup parallelism.
- Tape utilization and flushing policies.

Amanda also supports configuration overrides, allowing specific settings to be changed for individual backup operations or DLEs.

### 3.2 `disklist`

The `disklist` file defines what Amanda must back up. It contains hosts and Disk List Entries (DLEs).

A DLE identifies the host and the disk or directory that Amanda should back up. A traditional DLE has a structure similar to:

```text
hostname diskname [device] dumptype [spindle] [interface]
```

Example using a documentation-only hostname:

```text
client-a.example.invalid /backup/database/rman db-archive
```

| Field | Example | Description |
|---|---|---|
| Hostname | `client-a.example.invalid` | Client server |
| Diskname | `/backup/database/rman` | Backup directory identifier |
| Dumptype | `db-archive` | Backup configuration defined in `amanda.conf` |

The exact DLE grammar can vary with Amanda version and optional fields. Consult the installed `disklist(5)` manual page before using an example as a production configuration.

### 3.3 Hostname Resolution

Amanda can identify a client using a hostname or an address, depending on the configuration and client setup.

When using a hostname, the server must be able to resolve it through DNS or `/etc/hosts`.

Example using an address reserved for documentation:

```text
192.0.2.10 client-a.example.invalid
```

The client hostname, authentication configuration, host-key validation where applicable, and Amanda authorization must also be consistent. Using an IP address does not eliminate the need for proper authentication and authorization.

### 3.4 Relationship Between `amanda.conf` and `disklist`

The relationship is straightforward:

- **`disklist` answers: WHAT should Amanda back up?**
- **`amanda.conf` answers: HOW should Amanda perform the backup?**

For example, a DLE can select the `db-archive` dumptype, while that dumptype specifies GNU Tar, compression, and other backup options.

---

## 4. Compression in Amanda

Compression can reduce the amount of data transferred across the network and the amount of storage required on the holding disk and backup media. Its usefulness depends on the source data and where compression is performed.

### 4.1 Why Use Compression?

1. **Reduce network traffic:** Client-side compression can reduce the number of bytes transferred over the network.
2. **Reduce holding-disk utilization:** Compressed backup images require less temporary storage.
3. **Improve tape utilization:** Compression can reduce the physical tape capacity required to store a given amount of logical data.
4. **Reduce media requirements:** When data compresses well, fewer cartridges may be needed for a given retention period.

### 4.2 Compression Locations

| Method | Where compression takes place | Main consideration |
|---|---|---|
| Client compression | On the client server | Can reduce network traffic, but consumes client CPU |
| Server compression | On the backup server | Uses server CPU and compresses incoming data |
| No compression | Neither side compresses the backup stream | Appropriate when data is already compressed or CPU savings matter more |

The available Amanda compression modes and exact syntax depend on the installed release and dumptype configuration.

### 4.3 Compression Algorithms and LZ4

LZ4 is a fast compression algorithm designed to provide high compression and decompression speed with relatively low CPU overhead. It may be considered when the objective is to reduce backup size without introducing excessive CPU overhead.

A custom compression configuration may look conceptually like this:

```text
define dumptype db-archive {
    program "GNUTAR"
    auth "ssh"
    compress server custom
    server-custom-compress "amanda-lz4"
}
```

This is an **illustrative example, not a drop-in configuration**. `amanda-lz4` represents a custom wrapper program that must be implemented, installed, and tested. The wrapper must follow Amanda's compression-program calling convention, including the decompression mode required for recovery. Confirm the supported syntax and option names against the installed Amanda version.

Do not assume that entering the word `lz4` in `amanda.conf` automatically creates a working compression configuration.

### 4.4 Important Compression Consideration

Database backup files may already be compressed by the database backup process. Compressing already-compressed files generally provides limited additional benefit and may waste CPU resources.

Evaluate compression by measuring:

- Original backup size.
- Compressed backup size.
- CPU utilization.
- Backup duration.
- Network throughput.
- Actual tape utilization.

The objective is to find the best balance between performance and storage efficiency.

---

## 5. Tape Devices and SCSI Configuration

Amanda can write backup data to different storage devices, including tape drives and tape libraries. A typical tape-library environment contains multiple tape cartridges and one or more operational tape drives.

Understanding the difference between the tape drive and the changer device is essential.

### 5.1 Identifying SCSI Devices

To display attached SCSI devices, use:

```bash
lsscsi -g
```

The `-g` option displays associated SCSI generic device paths. It can help identify tape drives, medium-changer devices, and their device nodes.

### 5.2 Understanding Device Names

| Device | Purpose |
|---|---|
| `/dev/nstX` | Non-rewinding tape device |
| `/dev/stX` | Rewinding tape device |
| `/dev/sgX` | Generic SCSI device interface |

A tape drive may have both a tape device and a generic SCSI device interface. The autochanger also has its own generic SCSI interface. Therefore, the correct device must be identified from the actual SCSI device listing; do not assume that a particular `/dev/sgX` number always represents the changer.

### 5.3 Amanda Tape and Changer Configuration

Amanda has separate configuration concepts for the storage device and the changer. Depending on the Amanda version and changer implementation, configuration can involve directives such as:

```text
tapedev
tpchanger
changerdev
```

The exact placement and relationship of these options depend on the configured changer mechanism and Amanda release. Modern Amanda configurations can use device and changer sections; older configurations may use legacy global directives.

A common configuration error is assigning the tape drive's generic SCSI path to the changer configuration. This can prevent Amanda from communicating correctly with the autochanger.

### 5.4 Troubleshooting Example: Incorrect Changer Device

A tape-library problem can occur when the changer configuration points to the wrong generic SCSI device.

A safe troubleshooting process is:

1. Identify tape and changer devices using `lsscsi -g`.
2. Confirm which generic SCSI device represents the tape drive.
3. Confirm which generic SCSI device represents the changer.
4. Compare the detected devices against the Amanda configuration.
5. Correct the changer configuration if it points to the wrong device.
6. Test changer communication.
7. Run a controlled backup and verify successful tape writing.

Use placeholders in public documentation:

```text
TAPE_DEVICE=/dev/nstX
CHANGER_DEVICE=/dev/sgX
```

These are documentation placeholders, not commands to copy directly into a production configuration.

**Lesson learned:** Verify SCSI device mapping before configuring the tape drive and changer. Device numbering can change, so stable device identification is preferable to relying blindly on `/dev/sgX` numbers.

---

## 6. Tape Management and the Autochanger

An autoloader or tape library contains cartridge slots and one or more operational tape drives. The changer mechanism moves cartridges between storage slots and the drive.

Amanda communicates with the changer mechanism to request tape changes when required.

The general workflow is:

1. Amanda requests a tape volume.
2. The changer identifies an available cartridge.
3. The autoloader moves the cartridge from its storage slot to the tape drive.
4. Amanda writes backup data.
5. When another volume is required, Amanda requests a tape change.
6. The changer unloads the previous cartridge and loads another one.

The changer operation and tape-drive operation should not be confused. `mt` manages tape-drive operations, while `mtx` is commonly used to communicate with an SCSI medium changer.

To inspect an autoloader:

```bash
mtx -f CHANGER_DEVICE status
```

Replace `CHANGER_DEVICE` with the verified changer device path. The actual changer utility may differ depending on the hardware and Amanda changer script.

---

## 7. Holding Disk Management

The holding disk is temporary storage used by Amanda to store backup images before transferring them to tape. It helps separate client backup activity from physical tape-writing activity.

### 7.1 Why Use a Holding Disk?

The holding disk provides several advantages:

- Allows client backups to run without continuously waiting for tape operations.
- Separates network backup operations from physical tape writing.
- Provides temporary storage when the tape drive is unavailable.
- Allows Amanda to stage and organize backup data before writing it to tape.
- Can improve the overall workflow when sized and configured appropriately.

### 7.2 Holding-Disk Configuration

The holding disk is configured in `amanda.conf`.

| Parameter | Description |
|---|---|
| `directory` | Directory where Amanda stores temporary backup images |
| `use` | Maximum space Amanda may use in this holding-disk area |
| `chunksize` | Maximum size of an individual holding-disk chunk |

The default holding-disk directory is commonly:

```text
/dumps/amanda
```

Example configuration:

```text
define holdingdisk hd1 {
    directory "/path/to/holding"
    use 0          # Use all available space on this filesystem; set an explicit limit if required.
    chunksize 1 GB
}
```

The capacity and directory above are examples only. Choose values based on the available filesystem, expected backup volume, and Amanda version.

### 7.3 Understanding `chunksize`

The `chunksize` parameter defines the maximum size of an individual holding-disk file.

For example:

```text
chunksize 1 GB
```

If Amanda produces a backup image larger than 1 GB, it can store that image in multiple holding-disk chunks. These chunks are concatenated when Amanda writes the dump image to tape. Splitting a backup into holding-disk chunks does not necessarily mean that the backup will appear as multiple independent dumps on tape.

**Important:** Holding-disk `chunksize` is not the same as the tape device's record or block size. Changing `chunksize` may affect temporary-file handling and I/O behavior, but it does not directly change tape record size or guarantee better tape throughput.

### 7.4 Direct-to-Tape vs. Holding-Disk Backup

Amanda supports different holding-disk policies. Depending on the release, the `holdingdisk` dumptype option can use values such as:

| Value | General behavior |
|---|---|
| `auto` | Amanda decides whether to use holding disk |
| `yes` | Enables holding-disk usage |
| `required` | Requires holding disk for the dump |
| `never` | Avoids holding disk and sends the backup directly to storage |

Check the installed `amanda.conf(5)` manual page for the exact accepted values and behavior in the installed release.

### 7.5 Flushing from Holding Disk to Tape

Amanda provides four configuration parameters to control when backup images are transferred from the holding disk to tape:

- `flush-threshold-dumped`
- `flush-threshold-scheduled`
- `taperflush`
- `autoflush`

The main objective is to balance tape utilization, holding-disk capacity, and backup completion time.

### 7.6 Holding Disk and Tape Utilization Scenarios

The following scenarios are documented in Amanda's example configuration. They are valid example policies, but they are not universally optimal. Their results depend on the configured tape capacity, holding-disk capacity, backup volume, available media, and `runtapes` setting.

#### Understanding the Four Parameters

**`flush-threshold-dumped`**

Defines the minimum amount of actual backup data that must exist on the holding disk before Amanda starts writing to a new tape volume. The value is a percentage of the configured capacity of one tape volume.

**`flush-threshold-scheduled`**

Defines the minimum combined amount of data on the holding disk plus the estimated amount of data still scheduled to be dumped during the current run before Amanda starts writing to a new tape volume. This criterion is evaluated together with `flush-threshold-dumped`.

**`taperflush`**

At the end of a run, Amanda starts another tape to flush remaining holding-disk data if the remaining amount exceeds the configured percentage of one tape volume. A value of `0` means that any remaining eligible data can trigger a tape flush. If `taperflush` is greater than zero, `autoflush` must be set to `yes`.

**`autoflush`**

Controls whether an `amdump` run flushes dumps from the holding disk to tape:

- `no`: Do not automatically flush holding-disk dumps as an `amdump` autoflush operation.
- `yes`: Flush holding-disk dumps matching the command-line backup-set selection.
- `all`: Flush all holding-disk dumps.

The `all` value is documented in current Amanda 3.5.4 documentation. Confirm that the installed release supports it before using it. The default is `no`.

#### How the Thresholds Are Compared

Let:

- `T` = configured capacity of one tape volume (`tapetype length`).
- `H` = amount of data currently stored on the holding disk.
- `S` = estimated amount of backup data still scheduled for the current run.
- `D` = `flush-threshold-dumped` percentage.
- `Q` = `flush-threshold-scheduled` percentage.
- `F` = `taperflush` percentage.

The two volume-start conditions are conceptually:

```text
H >= T * D / 100
H + S >= T * Q / 100
```

Both threshold criteria must be satisfied before Amanda delays the start of a new tape volume for these thresholds. The man page describes these as minimum criteria; holding-disk constraints can cause Amanda to disregard a threshold for a dump that cannot be stored on the holding disk.

At the end of a run, another tape is started for remaining holding-disk data when:

```text
H_remaining > T * F / 100
```

The threshold values are percentages of **one configured tape volume**, not percentages of holding-disk capacity.

#### Scenario 1: Fill Tapes as Much as Possible and Allow Some Data to Remain on Holding Disk

**Objective:** Accumulate a substantial amount of backup data before starting a tape, reducing the chance of starting a new tape for a relatively small amount of data at the end of a run.

Configuration:

```text
flush-threshold-dumped    100
flush-threshold-scheduled 100
taperflush                100
autoflush                 yes
```

**Expected behavior:**

1. Amanda waits for the actual holding-disk data threshold and the combined holding-disk plus scheduled-data threshold to be satisfied before starting a new tape volume, when the holding-disk constraints permit.
2. At the end of the run, Amanda starts another tape to flush the remaining data only when that remaining amount is greater than 100% of one tape's configured capacity.
3. A smaller remainder can stay on the holding disk for a later run.
4. With `autoflush yes`, Amanda can flush held dumps matching the backup-set selection on the `amdump` command line.

**Result:** This policy favors tape utilization and can leave some eligible backup images on the holding disk. It is the appropriate example when the administrator does not mind retaining some dumps on disk between runs.

#### Scenario 2: Wait for a Full Tape's Worth of Data, Then Flush the Remaining Dumps

**Objective:** Wait until sufficient data has accumulated before starting tape writing, while avoiding deliberately leaving a small remainder on the holding disk at the end of the run.

Configuration:

```text
flush-threshold-dumped    100
flush-threshold-scheduled 100
taperflush                0
```

`autoflush` is not specified in Amanda's example for this scenario; its default is `no`.

**Expected behavior:**

1. Amanda waits for the two 100% threshold criteria to be satisfied before starting a new tape volume, when holding-disk constraints permit.
2. At the end of the run, `taperflush 0` means that any remaining eligible holding-disk data can cause Amanda to start another tape.
3. This may result in a partially filled tape being started to write the remainder.

**Result:** This policy waits for a full tape's worth of data before beginning tape writing, but then does not retain a small remainder solely to improve fill percentage.

**Important:** Amanda's example describes this scenario as flushing all dumps and leaving none on the holding disk. In an operational deployment, the result is still subject to available media, the `runtapes` upper limit, holding-disk constraints, and dump eligibility. Because `autoflush` is not enabled here, do not interpret this setting as a request to flush every old holding-disk dump from every backup set. If older held dumps must also be flushed, review whether `autoflush yes` or `autoflush all` is appropriate for the installed release and desired scope.

#### Scenario 3: Keep the Most Recent Dumps on Holding Disk for Faster Recovery

**Objective:** Retain more recent backup images on the holding disk for faster recovery while older dumps are rotated to tape during backup runs.

Configuration:

```text
flush-threshold-dumped    300
flush-threshold-scheduled 300
taperflush                300
autoflush                 yes
```

**Expected behavior:**

1. Amanda waits for the actual holding-disk data threshold and the combined holding-disk plus scheduled-data threshold to reach 300% of one tape's configured capacity, when holding-disk constraints permit.
2. At the end of the run, Amanda starts another tape only if the remaining holding-disk data exceeds 300% of one tape's configured capacity.
3. As a result, a larger amount of data can remain on the holding disk, potentially making recent dumps available for faster recovery.
4. With `autoflush yes`, held dumps matching the backup-set selection can be considered for flushing during `amdump`.

For an oldest-first selection strategy, Amanda's `taperalgo` supports:

```text
taperalgo first
```

`first` means first-in, first-out (FIFO), so the oldest eligible dump images are selected first. Confirm the behavior and suitability against the installed Amanda version and storage configuration.

**Result:** This policy attempts to preserve newer backup images on the holding disk while moving older images to tape. It requires a holding disk large enough to retain the intended amount of data.

#### Default Configuration

```text
flush-threshold-dumped    0
flush-threshold-scheduled 0
taperflush                0
autoflush                 no
```

The default has no threshold-based waiting restriction. Amanda can begin writing to tape without waiting for a minimum percentage of tape capacity, and it does not automatically flush older holding-disk dumps through `autoflush`.

#### Configuration Constraints and Practical Considerations

1. `flush-threshold-dumped` must not exceed `flush-threshold-scheduled`.
2. `taperflush` must not exceed `flush-threshold-scheduled`.
3. If `taperflush` is greater than zero, `autoflush` must be set to `yes`.
4. The holding disk must be large enough for the selected thresholds to be achievable. If a dump cannot be stored on the holding disk, Amanda can disregard the holding-disk threshold and start a new tape volume anyway.
5. The thresholds are calculated against `tapetype length`, not the holding-disk capacity and not automatically against the manufacturer's advertised compressed tape capacity.
6. `tapetype length` is used by Amanda for planning and scheduling. The actual device may continue writing until it reports an end-of-medium or another write error; configure the value conservatively based on measured, usable capacity.
7. `runtapes` is an upper limit on the number of volumes Amanda may use in one run. If it is too low, Amanda may be unable to flush all data in one run even when the flushing thresholds would otherwise permit it.
8. `autoflush yes` flushes holding-disk dumps matching the command-line backup-set selection. `autoflush all` flushes all holding-disk dumps in releases that support this value. Use the scope deliberately.
9. Test each policy with representative backup sizes and confirm the results using Amanda logs, reports, holding-disk usage, tape labels, and restore tests.

**Practical recommendation:** Select a flushing policy according to the ratio between holding-disk capacity and configured tape-volume capacity. A holding disk much smaller than one tape volume cannot reliably enforce 100% or 300% retention thresholds. For smaller holding disks, begin with the default zero thresholds, monitor the results, and adjust only after measuring real backup volumes and available tape capacity.

---

## 8. Managing Multiple Dumps on One Tape

Amanda can write multiple backup images from different DLEs onto the same tape volume.

For example, one tape may contain backups from:

```text
CLIENT_A /backup/database/export
CLIENT_A /backup/database/rman
CLIENT_B /backup/database/export
CLIENT_B /backup/database/rman
```

Each backup image remains identifiable through Amanda's metadata and tape headers. This does not mean Amanda combines all DLEs into one GNU Tar archive.

There is a distinction between:

- Multiple dump images stored sequentially on one tape.
- One large dump image split into multiple parts across volumes.
- Multiple source files archived together by GNU Tar within a single DLE.

Amanda's dump-splitting configuration determines how large backup images can be divided across storage volumes. Options such as `allow-split` and the relevant `tapetype` part-size and caching parameters must be checked for the installed release.

---

## 9. Amanda Administration Commands

The following are commonly used Amanda administration commands. Confirm command options with the installed release's manual pages.

### 9.1 `amdump`

The main Amanda backup command. It reads the configured DLEs and performs the scheduled backup operations.

```bash
amdump BACKUP_SET
```

It is commonly executed through `cron`.

Example command followed by report generation:

```bash
amdump BACKUP_SET && amreport BACKUP_SET
```

Replace `BACKUP_SET` with the actual Amanda configuration name in private deployment documentation.

### 9.2 `amflush`

The standard command is `amflush`, not `amflushdump`.

It transfers eligible backup images from the holding disk to tape. It does not initiate new client backup operations.

```bash
amflush BACKUP_SET
```

This is useful when a previous backup created holding-disk images but could not write them to tape because of a tape or changer problem. After correcting the original issue, the administrator can run `amflush`.

### 9.3 `amcleanup`

Used after an interrupted or incomplete Amanda run.

```bash
amcleanup BACKUP_SET
```

It performs cleanup operations after an unsuccessful or interrupted run. It is not the normal command for stopping an active `amdump` process. Options that kill associated processes must be used carefully and only after checking the installed manual page.

### 9.4 `amrestore`

A low-level utility used to extract backup images from Amanda volumes.

```bash
amrestore TAPE_DEVICE hostname diskname
```

Amanda searches the specified tape device for backup images belonging to the requested hostname and diskname. It can also be used for recovery when Amanda's catalog is unavailable.

### 9.5 `amstatus`

Displays the status of a running or completed Amanda backup.

```bash
amstatus BACKUP_SET
```

It provides information about backup progress and the status of individual DLEs.

### 9.6 `amreport`

Generates a report for a completed Amanda backup run.

```bash
amreport BACKUP_SET
```

Reports can include successful and failed DLEs, backup data sizes, original and compressed sizes, duration, tape usage, compression ratios, and error messages.

### 9.7 `amcheck`

Checks the Amanda configuration and performs pre-backup checks on clients and storage.

```bash
amcheck BACKUP_SET
```

It is useful before scheduled backup runs and after configuration changes.

---

## 10. Recovery and Restoration

Backup operations are only useful when data can be recovered successfully. Amanda provides several recovery utilities, including `amrestore`, `amfetchdump`, and `amrecover`.

### 10.1 Prepare the Restoration Directory

Create a directory for the recovered backup image:

```bash
mkdir -p /path/to/restore_directory
cd /path/to/restore_directory
```

Make sure the destination filesystem has enough free space.

### 10.2 Restore a Specific DLE

Example:

```bash
amrestore TAPE_DEVICE \
    client-a.example.invalid \
    /backup/database/rman
```

This instructs Amanda to search the specified tape device for backup images belonging to the requested hostname and diskname.

### 10.3 Using the `-b` Option

Amanda's `amrestore` provides a `-b` option to specify a block size when reading a volume. Confirm the exact syntax and units in the installed `amrestore(8)` manual page.

```bash
amrestore -b BLOCKSIZE TAPE_DEVICE hostname diskname
```

Do not assume an arbitrary block-size value is suitable. The correct value depends on the tape format, device configuration, Amanda version, and block size used when the backup was written.

### 10.4 Restoration Considerations

- Verify that the correct tape is loaded.
- Confirm the tape device path.
- Ensure the restoration directory has sufficient free space.
- Specify the correct hostname and diskname.
- Use the appropriate extraction tool for the backup program used by the DLE, such as GNU Tar for a `GNUTAR` archive.
- Confirm how decompression is handled by the Amanda recovery workflow.
- Verify the restored files before using them for database recovery.

For interactive or higher-level recovery, consider `amfetchdump` and `amrecover` as appropriate for the installed release and recovery objective.

---

## 11. Tape Management Using `mt`

Linux provides the `mt` command to manage tape-drive operations. It can inspect tape status and perform operations such as rewinding, unloading, and erasing.

**Display tape status:**

```bash
mt -f TAPE_DEVICE status
```

**Rewind the tape:**

```bash
mt -f TAPE_DEVICE rewind
```

**Unload or take the tape offline:**

```bash
mt -f TAPE_DEVICE offline
```

**Erase a tape:**

```bash
mt -f TAPE_DEVICE erase
```

The erase operation is destructive. Perform it only after verifying the correct tape and confirming that its contents are no longer required.

For an autoloader, use the appropriate changer utility for cartridge-slot operations rather than treating the tape drive and changer as the same device.

---

## 12. Troubleshooting and Common Errors

This section documents common problems in Amanda and tape-backup environments and provides a troubleshooting framework.

### 12.1 EOF Errors

EOF means End Of File. In tape environments, filemarks are used to separate tape files or records.

An EOF-related message can be normal in some tape operations, but an unexpected EOF may indicate:

- An incomplete backup image.
- Incorrect tape positioning.
- Incorrect block-size configuration.
- A tape read problem.
- An interrupted backup or restore operation.

Evaluate the error using Amanda logs, tape status, and the context in which it occurred. Do not treat every EOF indication as a failure without checking the command and operation that produced it.

### 12.2 Application Errors

Application errors may occur when the backup program fails to read or archive the requested data. When GNU Tar is used, possible causes include:

- Incorrect directory permissions.
- Missing files or directories.
- A source backup script that did not complete successfully.
- Insufficient disk space.
- Incorrect file ownership.
- GNU Tar errors.

Start by inspecting the Amanda report:

```bash
amreport BACKUP_SET
```

Then review the relevant error details and source backup-script logs.

### 12.3 SCSI Errors and Autochanger Problems

SCSI errors can prevent Amanda from controlling tape drives and changers.

Possible causes include:

- Incorrect SCSI generic device mapping.
- Incorrect changer configuration.
- Device permissions.
- A tape drive that is not ready.
- Communication problems between the server and autoloader.
- A cartridge-loading or unloading failure.

Recommended initial checks:

```bash
lsscsi -g
mt -f TAPE_DEVICE status
mtx -f CHANGER_DEVICE status
```

Replace placeholders with the actual paths detected on the backup server.

### 12.4 No Space Left on Device

The message does not always mean that the physical tape is full. It may originate from:

- Holding-disk exhaustion.
- Insufficient space in the backup server's filesystem.
- An exhausted tape volume.
- Insufficient temporary space.
- An index or logging directory that has become full.
- Incorrect Amanda tape-capacity configuration.

Check filesystem capacity:

```bash
df -h
```

Check inode utilization:

```bash
df -i
```

Inspect Amanda reports and logs to identify which stage failed.

**Tape-capacity consideration:** Tape manufacturers may advertise both native capacity and a larger compressed capacity based on an assumed compression ratio. The compressed figure is not guaranteed for every dataset. Already-compressed files may achieve little additional compression, and actual capacity can be much closer to native capacity.

For this reason, Amanda's configured `tapetype length` should reflect a realistic, tested usable capacity, not simply the manufacturer's maximum advertised compressed-capacity figure.

### 12.5 Tape Block-Size Problems

Incorrect tape block-size settings can cause problems when reading or restoring Amanda backups.

Distinguish between:

- Holding-disk `chunksize`.
- Tape record or block size.
- Amanda's restore `-b` parameter.

These are different settings. Changing holding-disk `chunksize` does not directly change tape record size. When a restore fails because of a block-size issue, verify the tape device configuration and the block size used when the backup was originally written.

---

## 13. Operational Recommendations

The following practices should be part of routine Amanda administration:

1. Verify client connectivity and authentication before scheduled backups.
2. Run configuration checks after modifying `amanda.conf` or `disklist`.
3. Monitor holding-disk utilization and filesystem capacity.
4. Monitor tape status, changer status, and media availability.
5. Review Amanda reports after every backup run.
6. Confirm that failed holding-disk backups are flushed after the original problem is resolved.
7. Keep a record of tape labels, backup dates, and retention policies in a protected inventory.
8. Test restoration periodically rather than relying only on successful backup reports.
9. Document actual tape block sizes and changer device mappings in private operational documentation.
10. Validate compression effectiveness using representative backup data.
11. Keep Amanda configuration backups and recovery procedures protected and access-controlled.
12. Test changes in a non-production configuration before applying them to production.

---

## 14. Conclusion

Amanda provides centralized backup management for database and application infrastructure.

In this architecture, source backup scripts generate the required files, GNU Tar archives the specified directories, and Amanda coordinates data transfer from client servers to the centralized backup server.

The holding disk provides temporary backup storage, while the tape drive and autoloader provide longer-term backup storage.

Correct configuration of authentication, DLEs, compression, holding-disk policies, SCSI devices, and tape parameters is essential for a reliable backup environment. Equally important is the ability to troubleshoot backup failures and perform successful restoration operations.

Amanda is therefore more than a tool for writing data to tape: it is the central component responsible for coordinating, organizing, monitoring, and managing the backup process.

---

## 15. Public Repository Security and Anonymization

This document is intended for public distribution. Real infrastructure identifiers and confidential operational information must not be included in public examples.

### 15.1 Information to Remove or Replace

| Information | Public documentation replacement |
|---|---|
| Real server hostnames | `client-a.example.invalid` |
| Internal DNS domains | `example.invalid` |
| Internal IP addresses | Documentation ranges such as `192.0.2.10` |
| Company and organization names | `Example Organization` |
| Employee names | Generic role names |
| Usernames | `backup_user` |
| Passwords and credentials | Never publish |
| SSH private keys | Never publish |
| Internal directory structures | Generic example directories |
| Tape serial numbers | `TAPE-001` |
| Tape barcodes | `TAPE-EXAMPLE-001` |
| Actual tape-device mapping | `TAPE_DEVICE` and `CHANGER_DEVICE` |
| Backup-set identifiers | `BACKUP_SET` |
| Email addresses | `admin@example.invalid` |
| Internal IP routes and network diagrams | Replace with documentation-only examples |
| Logs and screenshots | Redact hostnames, addresses, usernames, paths, serials, labels, and tokens |

### 15.2 Sanitized Amanda DLE Example

Do not publish real client hostnames or internal backup paths. Use a generic example:

```text
client-a.example.invalid /backup/database/rman db-archive
```

### 15.3 Sanitized Device Configuration

Avoid publishing actual device mappings from a production server.

Use placeholders:

```text
TAPE_DEVICE=/dev/nstX
CHANGER_DEVICE=/dev/sgX
```

These placeholders are for documentation only and must be replaced with verified device paths in a private deployment configuration.

### 15.4 Review Before Every Git Commit

Inspect the repository status and staged changes:

```bash
git status
git diff --cached
```

Search tracked files for possible identifiers or secrets:

```bash
git grep -n -i -E \
'password|passwd|secret|token|private.key|internal|production|@company'
```

This is a basic review aid, not a complete secret scanner. It may produce false positives and will not necessarily detect every secret format.

For a more thorough review, use an appropriate Git secret-scanning tool and inspect configuration files, logs, shell scripts, screenshots, attachments, and commit history.

### 15.5 Git History Considerations

Removing sensitive information from the latest document does not automatically remove it from previous Git commits.

If sensitive information has already been pushed:

1. Rotate any exposed credentials or keys immediately.
2. Remove the sensitive material from the repository.
3. Review and clean Git history where appropriate.
4. Coordinate with repository collaborators before rewriting shared history.
5. Check forks, clones, cached copies, CI artifacts, and other locations where the information may have been copied.

**Final rule:** Public documentation should explain architecture, commands, configuration concepts, and troubleshooting methodology without exposing the actual infrastructure in which they are used.

---

## 16. References

The following references were used to verify the Amanda flushing parameters and related configuration behavior. Always prefer the manual pages installed with the Amanda release used in the target environment.

1. [Amanda 3.5.4 example `amanda.conf`](https://sources.debian.org/src/amanda/1%3A3.5.4-2.1/example/amanda.conf.in) — includes the documented tape-utilization scenarios.
2. [Amanda 3.5.4 `amanda.conf(5)` manual page](https://manpages.debian.org/unstable/amanda-common/amanda.conf.5.en.html) — describes `autoflush`, `flush-threshold-dumped`, `flush-threshold-scheduled`, `taperflush`, `tapetype length`, `runtapes`, and `taperalgo`.
3. [Amanda quick configuration example](https://sources.debian.org/src/amanda/1%3A3.5.4-2/example/quick/amanda.conf) — provides a further configuration example.
4. [Amanda project website](https://www.amanda.org/) — project information and release resources.

---

**Version note:** Configuration examples in this document are intended as technical illustrations. Amanda syntax and available options can differ between older releases, Amanda Community versions, and newer releases. Validate examples with the installed release's manual pages and configuration-checking tools before applying them to a live system.
