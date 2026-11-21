# Restoring a deleted project file on Windows 11 with Veeam Agent

I backed up a project folder on a Windows 11 development workstation, then deliberately deleted `heron.py` to test the restore. I recovered the 391-byte file to its original path from one Veeam restore point. Its SHA-256 hash matched the one I recorded before deletion.

## Lab environment

| Component | Configuration |
|---|---|
| Workstation | `SOFTPLUS-WS-01`, a Windows 11 Pro 64-bit guest in VMware Workstation with two logical processors and 8 GB of RAM |
| System storage | 100 GB GPT virtual NVMe disk with the NTFS system volume on `C:` |
| Backup storage | Separate 20 GB NTFS virtual disk mounted as `E:` and labeled `SoftPlus-WS-01-Respaldo` |
| Tools | Veeam Agent for Microsoft Windows FREE, Veeam Backup Browser, and Windows PowerShell 5.1 |

## Recovery media preparation

I used the Veeam recovery media wizard to create an ISO in `E:\RecoveryMedia` and selected the options to include the workstation's network settings and hardware drivers. The wizard reported **Recovery media has been created**, and I could see the disc image in the destination folder. I later restored the test file from the running Windows guest. [Media options](../evidence/recovery/recovery-media-options.png) · [Creation log](../evidence/recovery/recovery-media-created.png) · [ISO in File Explorer](../evidence/recovery/recovery-media-iso-location.png)

## Backup configuration and execution

I chose a file-level job for the project folder so I could browse the restore point for a specific file.

| Job setting | Recorded value |
|---|---|
| Name and mode | `SoftPlus-WS-01-Archivos`, file-level backup |
| Source | `C:\SoftPlus\Projects`, including its subfolders |
| Destination | `E:\VeeamBackup` |
| Schedule | Daily at 8:00 p.m. If the workstation was off, run the missed backup after power-on. |
| Retention | Seven days with backups. Days without a backup were excluded from the count. |
| Compression and encryption | Optimal compression. Backup encryption was disabled in this lab. |

[Destination and retention](../evidence/recovery/backup-destination-and-retention.png) · [Schedule](../evidence/recovery/backup-schedule.png) · [Job summary](../evidence/recovery/file-backup-job-summary.png)

I ran the job manually to obtain a restore point before deleting the test file. Veeam showed a green check for the last backup. The job folder on `E:` contained a `.vbm` metadata file and a `.vbk` full backup of the selected data. I then opened the restore point in Veeam Backup Browser and confirmed that `heron.py` appeared under `C:\SoftPlus\Projects\Archivos_programacion`. [Backup status](../evidence/recovery/backup-run-completed.png) · [Files on the backup disk](../evidence/recovery/backup-output-files.png) · [File in the restore point](../evidence/recovery/restore-point-file.png)

## Simulated file loss and restoration

### 1. Record the original file hash

Before deleting the live copy, I recorded its SHA-256 hash so I could check the restored contents independently of Veeam's job status. The PowerShell blocks show the commands I used in the lab.

```powershell
Get-FileHash "C:\SoftPlus\Projects\Archivos_programacion\heron.py" -Algorithm SHA256
```

[Original SHA-256 output](../evidence/recovery/original-sha256.png)

### 2. Confirm the file was missing

After deleting `heron.py` from the live project folder, I ran the hash command again and got `PathNotFound`. I then checked the exact path with `Test-Path`.

```powershell
Test-Path "C:\SoftPlus\Projects\Archivos_programacion\heron.py"
```

PowerShell returned `False` for the original path. [Missing-file check](../evidence/recovery/file-missing.png)

### 3. Restore to the original path

With the file absent at its original path, I opened the backed-up `Archivos_programacion` folder in Veeam Backup Browser. I selected `heron.py` and chose **Restore → Overwrite** to return it to its original path. [Restore menu](../evidence/recovery/restore-overwrite-selection.png)

Veeam reported **Restore completed successfully** with 1 success, 0 errors, and 391 bytes transferred. The dialog recorded `00:00:05` for the file restore itself. [Restore result](../evidence/recovery/restore-result.png)

### 4. Verify the recovered file in PowerShell

```powershell
Test-Path "C:\SoftPlus\Projects\Archivos_programacion\heron.py"
Get-FileHash "C:\SoftPlus\Projects\Archivos_programacion\heron.py" -Algorithm SHA256
```

`Test-Path` returned `True` at the original path. I compared the full hashes from before deletion and after recovery to check the file's contents.

| File checked | SHA-256 |
|---|---|
| Live file before deletion | `7DB7175BE556620BC68990268D424992AA022A430180EC0BB349E518155B62EB` |
| Restored file at the original path | `7DB7175BE556620BC68990268D424992AA022A430180EC0BB349E518155B62EB` |

The hashes matched. [Before deletion](../evidence/recovery/original-sha256.png) · [After restoration](../evidence/recovery/restored-sha256.png)

## Limits and operational recommendations

- The source and backup disks shared one physical VMware host. A host failure could make both unavailable. In production, encrypt backups, keep a recoverable copy on storage independent of the workstation and its host, and test a restore from that copy.
- I used one restore point from a manually started backup. Scheduled runs and retention expiry were outside the observation period. In regular operation, monitor completed and missed jobs, check restore point dates, and set the backup frequency based on how much work the team can afford to lose.
- I verified the contents of one small file. Permissions, user access, and the rest of the project tree were outside the test. For routine restore tests, recover representative files to a separate folder and verify the contents and access with the intended account. Keep the working copy until the recovered files have been checked.
- I created the ISO. Bootability and whole-workstation recovery remain untested. If the workstation needs to be recoverable from scratch, back up the system and boot volumes, then boot the recovery media and test the full recovery in an isolated environment.

## Evidence index

| Stage | Reviewed screenshots |
|---|---|
| Recovery media | [Wizard options](../evidence/recovery/recovery-media-options.png) · [Creation log](../evidence/recovery/recovery-media-created.png) · [ISO in File Explorer](../evidence/recovery/recovery-media-iso-location.png) |
| Backup configuration | [Destination and retention](../evidence/recovery/backup-destination-and-retention.png) · [Schedule](../evidence/recovery/backup-schedule.png) · [Job summary](../evidence/recovery/file-backup-job-summary.png) |
| Backup run | [Completion status](../evidence/recovery/backup-run-completed.png) · [Backup files on disk](../evidence/recovery/backup-output-files.png) |
| Before the restore | [File in the restore point](../evidence/recovery/restore-point-file.png) · [Original SHA-256](../evidence/recovery/original-sha256.png) · [Missing-file check](../evidence/recovery/file-missing.png) |
| Restore | [Overwrite selection](../evidence/recovery/restore-overwrite-selection.png) · [Transfer result](../evidence/recovery/restore-result.png) |
| After the restore | [Restored path and SHA-256](../evidence/recovery/restored-sha256.png) |
