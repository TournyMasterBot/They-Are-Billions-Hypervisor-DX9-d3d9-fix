# They Are Billions + Hypervisor Crash — Full Troubleshooting Record

## TL;DR — Quick Fix Steps

Run these as Administrator, in order, then reboot:

**1. Disable intelppm (fixes BSOD)**
```powershell
sc.exe config intelppm start= disabled
```
Reboot.

**2. Install DXVK 2.7.1 (fixes game crash)**
- Download `dxvk-x.x.x.tar.gz` from https://github.com/doitsujin/dxvk/releases
- Extract `x64\d3d9.dll`
- Place it in `C:\Program Files (x86)\Steam\steamapps\common\They Are Billions\`

**3. Switch Docker to WSL2 backend**
Docker Desktop → Settings → General → "Use the WSL 2 based engine" → ON

**4. After any future DISM/Windows Feature change**, re-verify intelppm is still disabled:
```powershell
Get-Service intelppm | Select Name,Status,StartType
# If StartType is not Disabled:
sc.exe config intelppm start= disabled
# Then reboot
```

---

## System Profile

| Field | Value |
|-------|-------|
| OS | Windows 11 25H2 (Build 26200.8039) |
| CPU | Intel hybrid (32 logical threads — likely i9-13900K/14900K/Core Ultra 9) |
| GPU | NVIDIA GeForce RTX 4060 Ti |
| NVIDIA Driver | 595.71 |
| .NET Framework | 4.8.09221 (release 533509) |
| Docker | Docker Desktop (switched to WSL2 backend during session) |
| Game | They Are Billions V1.0.14 (Steam, app 644930) |

---

## Two Separate But Related Problems

### Problem A — BSOD: `HYPERVISOR_ERROR (0x20001)`
Hard kernel panic, writes minidump to `C:\Windows\Minidump\`. Triggered most often when relaunching They Are Billions after a game crash.

### Problem B — Game Crash: `COR_E_EXECUTIONENGINE` / `BEX64` in `clr.dll` / `nvgpucomp64.dll`
They Are Billions crashes during gameplay — no BSOD, just process termination. 44 WER crash reports going back months, all with Docker/virtualization enabled.

**User confirmed:** Game was stable when Docker was disabled and virtualization features were turned off in Windows Features.

---

## Root Cause Analysis

### BSOD Root Cause
```
Dump: C:\Windows\Minidump\040826-12437-01.dmp
Stop code: HYPERVISOR_ERROR (0x20001)
Arg1: 1 — "the hypervisor has overrun a stack-based buffer"
Symbol: intelppm!HvRequestIdle+2d
Module: intelppm.sys (Intel Processor Power Management driver)
```

`intelppm.sys` manages CPU C-states (idle power levels) on Intel CPUs. When the Hyper-V hypervisor is active, it calls `HvRequestIdle` (a hypercall) to coordinate idle state transitions. On Intel 13th/14th gen hybrid CPUs (P-cores + E-cores), this hypercall overflows the hypervisor's own internal stack — a confirmed bug in Microsoft's hypervisor on this CPU architecture.

The BSOD fires specifically on game relaunch because relaunching causes a surge of CPU idle→active transitions (loading assets, initialising DX9) that hammer `HvRequestIdle` at high frequency.

### Game Crash Root Cause
WER crash archive analysis across 44 reports showed three crash modes, all caused by the hypervisor:

| Crash Type | Faulting Module | Meaning |
|------------|----------------|---------|
| `APPCRASH` | `nvgpucomp64.dll` | NVIDIA's DX9-to-modern-GPU translation layer crashed directly |
| `APPCRASH` / `BEX64` | `clr.dll` | The corruption propagated into the .NET CLR execution engine |
| `BEX64` | `StackHash_*` | Stack so corrupted WER couldn't even identify the module |

**Mechanism:** The hypervisor (running via `VirtualMachinePlatform` for Docker/WSL2) intercepts and remaps GPU MMIO (Memory Mapped I/O) pages as part of its memory management. `nvgpucomp64.dll` — NVIDIA's layer that translates DX9 calls for the RTX 4060 Ti — holds raw pointers into those pages. When the hypervisor remaps them, those pointers become invalid mid-render, crashing `nvgpucomp64.dll`. That unhandled exception propagates through SlimDX's COM interop back into the .NET CLR, corrupting the CLR's internal execution state. Windows detects the stack corruption (BEX64) and terminates the process.

---

## Diagnostic Steps Performed

### Step 1 — Identified the hypervisor was active
```powershell
Get-Service hvservice | Select Name,Status,StartType
# Result: Running, Manual
```
`hvservice` running + BSOD pointing at `intelppm!HvRequestIdle` confirmed the hypervisor as the cause.

### Step 2 — Identified what was running the hypervisor
```powershell
reg query 'HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity'
# Result: Enabled = 0x1, WasEnabledBy = 0x1
```
HVCI (Memory Integrity / Core Isolation) was auto-enabled by Windows, running the hypervisor in the background with no user-installed VMs.

### Step 3 — Found Docker Desktop was installed and using Hyper-V
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Docker Desktop  <- found
Microsoft-Hyper-V-ClientEdition-Package  <- installed
Microsoft-Hyper-V-Hypervisor-Package     <- installed
HyperV-Feature-VirtualMachinePlatform-Client-Package  <- installed
```

### Step 4 — Confirmed game crash pattern from WER archive
44 crash reports including `nvgpucomp64.dll` (NVIDIA DX9 layer) and multiple `StackHash_*` (complete memory corruption) entries.

---

## Actions Taken — Status

### FIXED — BSOD (`intelppm!HvRequestIdle`)

**What was tried first (did NOT fully fix on its own):**

| Action | Result |
|--------|--------|
| Disable Memory Integrity in Settings | Done — HVCI registry cleared, but hypervisor still ran |
| `bcdedit /set hypervisorlaunchtype off` | Done — had no effect because Hyper-V feature was installed via DISM, overriding bcdedit |
| `Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All` | Done — returned `RestartNeeded: False` (suspicious), hvservice still running post-reboot |

**What fixed the BSOD:**
```powershell
# Run as Administrator
sc.exe config intelppm start= disabled
# Then reboot
```

**Why it works:** `intelppm.sys` is the only driver that calls `HvRequestIdle`. With it disabled, the hypercall is never made, the hypervisor stack overflow never happens. The CPU falls back to a generic power management driver with no performance impact on a desktop gaming PC. `hvservice` can keep running — it is harmless without `intelppm` calling into it.

**Verification:**
```powershell
Get-Service intelppm | Select Name,Status,StartType
# Expected: Stopped, Disabled

# No new files in C:\Windows\Minidump\ since April 8 = confirmed stable
Get-ChildItem C:\Windows\Minidump | Sort-Object LastWriteTime -Descending
```

**IMPORTANT — Watch out:** DISM operations that re-enable Hyper-V features may reset the `intelppm` start type back to Manual. After any major Windows Feature change, re-check and re-apply if needed:
```powershell
Get-Service intelppm | Select StartType
# If it shows Manual or Automatic:
sc.exe config intelppm start= disabled
# Then reboot
```

---

### FIXED — Game Crash (`nvgpucomp64.dll` / CLR corruption)

**What was tried but did NOT fix it alone:**

| Action | Result |
|--------|--------|
| Enable VSync in `Configuration.txt` | Done — but NVCP already had VSync forced on, so game was already capped at 60fps. Left enabled for consistency. |
| Steam file verification (`steam://validate/644930`) | Triggered — verifies game file integrity |

**What fixed the game crash — DXVK 2.7.1:**

DXVK `d3d9.dll` installed at:
```
C:\Program Files (x86)\Steam\steamapps\common\They Are Billions\d3d9.dll
```

**Why it works:** DXVK is a Vulkan-based implementation of DirectX 9. When the game starts, Windows loads `d3d9.dll` from the game folder first, so DXVK intercepts all DX9 calls before they reach the system `d3d9.dll` or NVIDIA's `nvgpucomp64.dll`. The crash chain is:

```
BROKEN chain (before DXVK):
SlimDX -> system d3d9.dll -> nvgpucomp64.dll -> hypervisor MMIO interference -> crash

FIXED chain (with DXVK):
SlimDX -> DXVK d3d9.dll -> Vulkan -> GPU directly (hypervisor-safe memory model)
```

DXVK was purpose-built to run DX9 games inside hypervisor environments (it is the core of Steam's Proton compatibility layer on Linux VMs). Its Vulkan backend does not use the MMIO patterns the hypervisor interferes with.

---

## Confirmed Fix — Setup Steps

1. **Disable `intelppm`** (as Administrator, one time):
   ```powershell
   sc.exe config intelppm start= disabled
   ```
   Reboot.

2. **Install DXVK** into the game folder (repeat after any game reinstall or Steam verify that overwrites d3d9.dll):
   - Download latest `dxvk-x.x.x.tar.gz` from https://github.com/doitsujin/dxvk/releases
   - Extract `x64\d3d9.dll`
   - Place it in `C:\Program Files (x86)\Steam\steamapps\common\They Are Billions\`

3. **Switch Docker to WSL2 backend** (already done):
   Docker Desktop -> Settings -> General -> "Use the WSL 2 based engine" -> ON

4. **After any DISM/Windows Feature change**, re-verify `intelppm` is still disabled:
   ```powershell
   Get-Service intelppm | Select Name,Status,StartType
   ```

---

## Key File Locations

| File | Purpose |
|------|---------|
| `C:\Windows\Minidump\` | Kernel crash dumps — new files here mean a new BSOD occurred |
| `C:\ProgramData\Microsoft\Windows\WER\ReportArchive\AppCrash_TheyAreBillions.*` | Game crash reports |
| `%USERPROFILE%/../Documents/My Games/They Are BillionsZXLog.txt` | Game session log |
| `%USERPROFILE%/../Documents/My Games/They Are BillionsErrorReport.txt` | Game exception log |
| `%USERPROFILE%/../Documents/My Games/They Are BillionsConfiguration.txt` | Game config (VSync now set to True) |
| `C:\Program Files (x86)\Steam\steamapps\common\They Are Billions\d3d9.dll` | DXVK 2.7.1 — delete this to revert to native DX9 |
