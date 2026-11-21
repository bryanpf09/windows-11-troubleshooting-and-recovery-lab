# Investigating intermittent slowness on a Windows 11 workstation

I used Task Manager and Windows PowerShell 5.1 to investigate a report of intermittent slowness on `SOFTPLUS-WS-01`. The short captures showed which applications were using resources and which processes hosted Windows services. I did not capture a confirmed slowdown during that window, so I couldn't use those readings to isolate the cause. I tested process priority and termination separately.

## System baseline

I recorded the resources visible to the Windows guest before changing any process settings.

| Component | Recorded configuration |
|---|---|
| Workstation | `SOFTPLUS-WS-01`, a VMware Workstation guest running Windows 11 Pro, 64-bit, version `10.0.26300` (build `26300`) |
| CPU and memory | Windows reported `AMD Ryzen 9 270 w/ Radeon 780M Graphics`, two cores, two logical processors, and 8 GB of RAM |
| System disk | Disk 0, a 100 GB VMware Virtual NVMe Disk using GPT. Windows reported `Healthy` and `Online` for the virtual disk. |
| Partitions, in disk order | 200 MB EFI system, 16 MB Microsoft reserved, 98.92 GB NTFS Windows volume (`C:`), and 892 MB recovery |

`Get-Volume` showed 54.93 GB free on `C:` at the time of inventory. The `D:` entry was the mounted installation disc. The disk health reading describes the guest's view of its virtual disk. [Windows, CPU and RAM](../evidence/diagnostics/system-cpu-and-memory.png) · [Disk, partitions and volumes](../evidence/diagnostics/system-disk-and-partitions.png)

## Task Manager observations

| Processes view | Whole workstation | Leading entries in that capture |
|---|---|---|
| [Sorted by memory](../evidence/diagnostics/task-manager-memory.png) | 68% memory, 6% CPU | Visual Studio Code showed 927.1 MB across 10 processes. Google Chrome showed 665.8 MB across 17. |
| [Sorted by CPU](../evidence/diagnostics/task-manager-cpu.png) | 8% CPU, 67% memory | Google Chrome showed 3.1% CPU. Task Manager and System each showed 1.5%. |

To compare individual processes, I sorted them by `WorkingSet64` in PowerShell. The largest working set in that capture was `chrome` PID 10908 at 570.27 MiB, followed by `Code` PID 10020 at 389.76 MiB. Task Manager grouped several processes under each application and refreshed at a different time. I report the PowerShell values in MiB because the `RAM_MB` calculation divides bytes by 1,048,576. [Individual process working sets](../evidence/diagnostics/process-working-set.png)

## CPU use over a ten-second interval

I recorded cumulative `CPU` seconds from `Get-Process` by PID, timed a ten-second sleep with a stopwatch, and queried the processes again. To get each process's average share of the guest's total CPU capacity, I divided the increase in CPU seconds by the measured wait and the guest's two logical processors, then multiplied by 100. [Recorded sampling commands](../evidence/diagnostics/process-cpu-sampling-method.png) · [Microsoft Get-Process reference](https://learn.microsoft.com/en-us/powershell/module/Microsoft.PowerShell.Management/get-process?view=powershell-5.1)

| Process | PID during the sample | Average CPU | Working set at the second reading |
|---|---:|---:|---:|
| `chrome` | 10908 | 1.17% | 583.93 MiB |
| `Taskmgr` | 6796 | 0.93% | 99.25 MiB |
| `powershell` | 2488 | 0.70% | 109.41 MiB |

The table shows the three highest readings. Chrome's 1.17% came from one PID over the sample. Task Manager's 3.1% covered the Chrome group at a single refresh. I matched the PowerShell readings by PID alone and skipped unavailable values without logging them. That can miss short-lived processes or pair readings from two different processes if Windows reuses a PID. [Recorded CPU results](../evidence/diagnostics/process-cpu-sample.png)

## Running services and their host processes

I filtered `Win32_Service` to running services with a positive `ProcessId`, grouped them by PID, and looked up each host with `Get-Process`. For example, `WinDefend` ran in `MsMpEng` (PID 3796), and `DiagTrack` ran in `svchost` (PID 3344).

The `svchost` instance at PID 956 hosted `BrokerInfrastructure`, `DcomLaunch`, `PlugPlay`, `Power`, and `SystemEventsBroker`. The 32.89 MiB working set was for the shared process. That reading gave me no way to assign memory use to an individual service. [Running services grouped by host PID](../evidence/diagnostics/services-by-process.png)

## Controlled process-management tests

The PIDs below identify the instances observed during this lab session.

### Change and restore Visual Studio Code priority

I selected one existing `Code` process, PID 5192, checked its process name, and saved its original priority class before making the change. I queried the same PID after both assignments.

| Step | Command excerpt from the session | Verified priority class |
|---|---|---|
| Save original value | `$prioridadOriginal = $proceso.PriorityClass` | [Normal](../evidence/diagnostics/code-priority-original.png) |
| Lower priority | `$proceso.PriorityClass = "BelowNormal"` | [BelowNormal](../evidence/diagnostics/code-priority-below-normal.png) |
| Restore saved value | `$proceso.PriorityClass = $prioridadOriginal` | [Normal](../evidence/diagnostics/code-priority-restored.png) |

Priority class affects how Windows schedules the process's threads when they compete for CPU time. The readbacks verified the change and rollback for this one instance. Application response time was outside the test. [Windows scheduling priorities](https://learn.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities)

### Terminate a responsive Notepad process

I queried Notepad by name and checked `Responding` before acting. PowerShell returned `Notepad`, PID `2772`, with `Responding=True`. I then ran these commands in the lab session.

```powershell
Stop-Process -Id 2772
Get-Process -Id 2772 -ErrorAction SilentlyContinue
```

PowerShell returned to the prompt without a row for PID `2772`, which confirmed the controlled stop. [Notepad before and after termination](../evidence/diagnostics/notepad-controlled-termination.png)

## If the slowdown recurs

I would note the time, which application stalled, and what the user was doing, then capture a performance trace through the delay and a period of normal response. Looking at total and per-process CPU, available memory, and disk latency on the same timeline would help me see if any of them tracked the symptom. I would record process start times alongside PIDs so I could tell when an application restarted during the comparison.
