# Windows 11 Troubleshooting and Recovery Lab

I used a Windows 11 Pro VM to simulate a development workstation with a report of intermittent slowness. I recorded the workstation baseline, tested changes I could roll back, and ran a controlled file recovery.

## Lab walkthroughs

- I ran a file-level backup, deliberately deleted the 391-byte `heron.py`, and restored it to its original path. Its SHA-256 hash matched the one I recorded before deletion. [File backup and recovery with Veeam Agent](docs/backup-and-file-recovery.md)
- I checked resource use in Task Manager and PowerShell and mapped Windows services to their host processes. I lowered and restored the priority of one Visual Studio Code process. In a separate test, I stopped a responsive Notepad instance and confirmed it had exited. [Workstation diagnostics and process management](docs/workstation-diagnostics.md)
- I switched from automatic paging file management to a fixed 4,096 MB file and verified the allocated size after restarting. I compared five memory counters and used a final PowerShell query to confirm the return to automatic management. [Paging file experiment and rollback](docs/pagefile-experiment.md)

## Lab environment

| Component | Configuration |
|---|---|
| Workstation | Windows 11 Pro (64-bit) in VMware Workstation |
| Compute | Two logical processors and 8 GB of RAM |
| Storage | 100 GB virtual NVMe system disk and a separate 20 GB virtual backup disk |
| Tools | Task Manager, Windows PowerShell 5.1, Veeam Agent for Microsoft Windows FREE, and Veeam Backup Browser |

## Scope and limitations

- The short diagnostic captures left the cause of the reported slowness unresolved.
- The paging file comparison used two short runs with a restart in between. The commit and paging file usage percentages used larger totals after the change. I did not time application response.
- I tested one file restore using a restore point from a backup I started manually. I created a recovery ISO, and its bootability and full workstation recovery remain untested. Both virtual disks shared the same physical host, and backup encryption was disabled in the lab.
