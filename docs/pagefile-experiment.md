# Testing a fixed paging file on Windows 11

As part of the slowness investigation, I compared five performance counters with Windows managing the paging file and with a fixed 4,096 MB file. The test ran on `SOFTPLUS-WS-01`, a Windows 11 Pro guest in VMware Workstation. I kept the guest at 8 GB of RAM for both runs.

## Baseline under automatic management

Task Manager showed 4.5 GB of physical memory in use (56%), including 456 MB of compressed memory, and 3.5 GB available. The **Committed** field showed 5.4 GB of committed virtual memory against a 9.9 GB system commit limit. [Task Manager memory baseline](../evidence/memory/memory-baseline-task-manager.png)

A separate Windows PowerShell 5.1 check returned `AutomaticManagedPagefile=True`. For `C:\pagefile.sys`, `Win32_PageFileUsage` reported 1,984 MB allocated, with current and peak usage both at 16 MB. These were point-in-time readings. [Automatic policy and initial allocation](../evidence/memory/pagefile-automatic-baseline.png)

## Set the fixed size and verify it after restart

In the **Virtual Memory** dialog, I cleared **Automatically manage paging file size for all drives**, selected `C:`, and set both the initial and maximum size to **4,096 MB**. The dialog listed the new range while **Currently allocated** still showed **1,984 MB**. [Fixed-size settings before restart](../evidence/memory/pagefile-fixed-settings.png)

I restarted the guest to apply the change, then checked the settings and the active allocation. `Win32_ComputerSystem` returned `AutomaticManagedPagefile=False`, and `Win32_PageFileSetting` returned 4,096 MB for both `InitialSize` and `MaximumSize`. `Win32_PageFileUsage` reported an `AllocatedBaseSize` of **4,096 MB**, so I knew the new size was in use before the second measurement. [Recorded PowerShell checks](../evidence/memory/pagefile-policy-checks.txt)

## Five performance counter averages

I ran the same `Get-Counter` command under each configuration. Each run requested six samples of all five counters at two-second intervals. I grouped the samples by counter path and averaged each counter’s `CookedValue`, rounding to two decimal places. [Before the change](../evidence/memory/pagefile-counters-before.png) · [After the restart](../evidence/memory/pagefile-counters-after.png)

| Windows performance counter | Unit | Automatic · 1,984 MB | Fixed · 4,096 MB |
|---|---|---:|---:|
| `\Memory\Available MBytes` | MB | 3,926.17 | 3,815.67 |
| `\Memory\% Committed Bytes In Use` | % | 54.59 | 41.82 |
| `\Paging File(_Total)\% Usage` | % | 5.33 | 0.38 |
| `\Memory\Pages/sec` | pages/s | 0 | 18 |
| `\Memory\Page Reads/sec` | disk read operations/s | 0 | 3.33 |

## What the changes mean

The commit percentage fell from 54.59% to 41.82%. The larger paging file raised the system commit limit while physical RAM stayed at 8 GB. Windows divides committed bytes by that limit, so the percentage can fall even if demand stays the same. The percentage alone does not tell me how much application demand changed. [Microsoft’s explanation of system commit](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/introduction-to-the-page-file)

The paging file usage counter went from 5.33% to 0.38%. At the recorded sizes, the bytes occupying 5.33% of a 1,984 MB file would occupy about 2.58% of a 4,096 MB file. The observed 0.38% was lower still. The counter shows how full the file was and gives no timing information for disk reads or application response.

Available RAM averaged 110.50 MB less in the second window, which also showed 18 pages per second and 3.33 disk read operations per second. `Pages/sec` counts pages read from or written to disk during paging. `Page Reads/sec` counts disk reads that resolve hard page faults. A single read can fetch multiple pages from an executable, a memory-mapped file, or the paging file. [Microsoft’s guide to paging counters](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/how-to-determine-the-appropriate-page-file-size-for-64-bit-versions-of-windows)

I had one short run with each configuration and a restart in between. I did not repeat an identical workload or time an application task. Running processes, background activity, and cache state could vary between runs. I would need repeated measurements of the same task under comparable conditions to tell whether the fixed size made the workstation faster or slower.

## Restore automatic management

I switched automatic paging file management back on because I had no measured application benefit from the fixed setting. The dialog still showed **4,096 MB currently allocated** when I selected the automatic option. The final PowerShell query returned **`AutomaticManagedPagefile=True`**, which confirmed the policy change. I did not record the resulting allocation. [Rollback in Virtual Memory](../evidence/memory/pagefile-automatic-restored.png) · [Final PowerShell check](../evidence/memory/pagefile-policy-checks.txt)
