---
layout: post
title: "A winget upgrade Broke My Entire Dev Environment"
date: 2026-03-27
categories: [infrastructure, devops]
tags: [wsl, windows, winget, dism, backup, sysadmin, developer-experience]
author: Aaron Lamb
description: "A routine winget upgrade silently updated WSL to a broken version, killing my development environment for two hours. Here's the dead ends, the fix, and the playbook so it doesn't happen to you."
---

# A `winget upgrade` Broke My Entire Dev Environment

Thursday morning. I run `winget upgrade --all` like I have dozens of times before. Routine maintenance. Keep things current. Good practice.

Ten minutes later, I try to open my Ubuntu terminal.

```
Failed to attach disk 'C:\Program Files\WSL\system.vhd' to WSL2:
The system cannot find the file specified.
Error code: Wsl/Service/CreateInstance/CreateVm/MountDisk/HCS/ERROR_FILE_NOT_FOUND
```

My entire development environment is inside that WSL instance. Every project, every tool, every config. Docker containers, databases, SSH keys, git repos. Two years of careful setup.

All of it, unreachable.

## What Happened

The `winget upgrade` run silently updated WSL from a working version to 2.6.3. This version has a known bug: it fails to generate `system.vhd`, a virtual hard disk required for WSL2's VM initialization. Without this file, no distro can start. Period.

The file isn't bundled in the installer. It's supposed to be generated on first launch after the update. In WSL 2.6.3, that generation silently fails. No error during install. No warning. Just a broken system the next time you try to use it.

This isn't a rare edge case. There are multiple open issues on the [microsoft/WSL GitHub repo](https://github.com/microsoft/WSL/issues) documenting this exact failure across different Windows builds.

## The Dead Ends

What followed was two hours of increasingly creative troubleshooting, most of which accomplished nothing.

**Attempt 1: The obvious stuff.**

```powershell
wsl --shutdown
wsl --update
wsl --install --no-distribution
```

Reboot. Still broken. `system.vhd` not generated.

**Attempt 2: Rip and reinstall.**

```powershell
winget install --id Microsoft.WSL --force
```

Clean install. Still no `system.vhd`. The reinstaller has the same bug as the updater.

**Attempt 3: Check the plumbing.**

Verified that both `Microsoft-Windows-Subsystem-Linux` and `VirtualMachinePlatform` were enabled in Windows Features. They were. Not the cause.

**Attempt 4: Roll back the version.**

Downloaded the WSL 2.4.13 MSI directly and installed it. It worked, briefly. Then Windows automatically re-pushed 2.6.3 on the next reboot, overriding the rollback.

This is a particularly frustrating behavior. Windows treats WSL as a managed component and will re-upgrade it even after you explicitly downgrade.

**Attempt 5: System Restore.**

This is where things got worse.

System Restore successfully rolled WSL back to version 2.5.7. But it introduced a new error:

```
Wsl/Service/CreateInstance/CreateVm/E_UNEXPECTED
```

Digging into the HCS event log revealed error code `0xC0370103` - `ERROR_VID_PARTITION_DOES_NOT_EXIST`. The Hyper-V VID kernel drivers on my system were built for Windows 10.0.26200.8037, but System Restore had rolled back the WSL user-space components to an earlier version. Kernel and userspace were now mismatched.

I'd taken a bad situation and made it worse.

## The Fix

After exhausting every other option, I ran DISM:

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
```

Thirty minutes of scanning. Then a reboot.

WSL started. `system.vhd` was generated. Everything was back.

What DISM did was resync the Windows component store, including the Hyper-V kernel drivers, against the current build. Once the kernel components were healthy and matched the installed WSL version, everything worked correctly. WSL 2.6.3 was still installed (Windows re-applied it), but with a healthy kernel underneath it, the `system.vhd` generation succeeded.

Two commands. That's all it took. After two hours of dead ends.

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
# reboot
wsl -d Ubuntu-24.04
```

## The Data Was Never at Risk

One thing worth noting: the distro data was safe the entire time. WSL updates never touch the distro `ext4.vhdx` files. My 53 GB Ubuntu disk sat untouched in `AppData\Local\wsl\` through every failed fix attempt.

But I didn't know that with certainty when the error first appeared. "Failed to attach disk" is a terrifying message when your entire workflow lives on that disk.

I copied the `.vhdx` file to my Desktop before attempting any repairs. That took a few minutes but bought complete peace of mind. If everything had gone sideways, I could have reimported from that backup on a fresh WSL install.

## The Playbook

If you use WSL daily, here's the mitigation strategy I put in place afterward.

### 1. Pin WSL in winget

Never let `winget upgrade --all` touch WSL without your explicit approval:

```powershell
winget pin add --id Microsoft.WSL
```

Or exclude it from bulk upgrades:

```powershell
winget upgrade --all --exclude Microsoft.WSL
```

This is the single most important step. If I'd had this pin in place, none of this would have happened.

### 2. Export your distro regularly

```powershell
wsl --shutdown
wsl --export Ubuntu-24.04 "D:\Backups\WSL\Ubuntu-24.04-2026-03-27.tar"
```

This produces a portable tar file (mine was 49 GB) that can be reimported on any machine:

```powershell
wsl --import Ubuntu-24.04 C:\WSL\Ubuntu-24.04 "D:\Backups\WSL\Ubuntu-24.04-2026-03-27.tar"
```

Monthly, or before any upgrade cycle. Store it off the system drive.

### 3. If WSL breaks, skip the reinstall loop

The instinct is to uninstall and reinstall. Don't. The reinstaller often has the same bug as the updater. Go straight to DISM:

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
# reboot
wsl -d Ubuntu-24.04
```

If DISM alone doesn't resolve it:

```powershell
sfc /scannow
# reboot
```

### 4. Never use System Restore for WSL issues

System Restore rolls back user-space components but leaves the Windows kernel on its current build. For WSL, which depends on Hyper-V kernel drivers, this creates a version mismatch that's harder to fix than the original problem.

Exhaust DISM and reinstall options before reaching for System Restore.

## The Bigger Problem

This incident highlights a real tension in the WSL ecosystem. Microsoft ships WSL updates through `winget` and Windows Update, treating it like any other package. But WSL isn't just another package. It's a hypervisor-backed subsystem that depends on kernel-level components. When those get out of sync, the failure mode isn't a broken app. It's a completely inaccessible development environment.

There's no staged rollout. No canary. No "this update will restart your VM infrastructure, proceed?" prompt. It just updates, and if the new version has a bug, you find out the next time you open your terminal.

For developers and sysadmins who've built their entire workflow on WSL, that's a fragile foundation. The workaround is defensive: pin the version, export regularly, know the DISM recovery path.

WSL is an incredible tool. I run my entire stack on it. But after losing two hours to a silent upgrade, I'm treating it less like a system utility and more like production infrastructure. Pin it. Back it up. Update it deliberately.

---

**Recovery Checklist (save this somewhere):**

1. Don't panic. Your distro `.vhdx` data is almost certainly intact.
2. Copy the `.vhdx` file from `AppData\Local\wsl\` as a precaution.
3. Run `DISM /Online /Cleanup-Image /RestoreHealth` and reboot.
4. If still broken, run `sfc /scannow` and reboot.
5. If still broken, export your `.vhdx` backup, uninstall WSL completely, reinstall, and reimport.

**Prevention:**

```powershell
winget pin add --id Microsoft.WSL
```

That one command would have saved me two hours.

---

**About the Author:** Aaron Lamb is a founder of Hexaxia Technologies, a consultancy specializing in cybersecurity, infrastructure engineering, and AI product development. He's been building and breaking things in this industry for 30 years.
