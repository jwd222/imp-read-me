# 🧹 WSL Storage Cleanup and Shrink Guide

This guide explains how to clean up disk space used by WSL (Windows Subsystem for Linux) and shrink the virtual disk (`ext4.vhdx`) used by WSL2 distributions.

---

## 🔍 Check WSL Disk Usage

### Inside WSL
Use the following command to check disk usage:
```bash
df -h
du -h --max-depth=1 /
du -h --max-depth=1 /home
````

To find large files/folders:

```bash
sudo du -sh /home/* | sort -h
```

---

## 🧹 Clean Up Space in WSL

### 1. Clean APT Cache

```bash
sudo apt-get clean
sudo apt-get autoremove
```

### 2. Clean Conda (if used)

```bash
conda env list
conda remove --name <env_name> --all
conda clean --all
```

### 3. Remove Cached Files

```bash
rm -rf ~/.cache/*
rm -rf ~/Downloads/*
rm -rf ~/cmake-*/       # If no longer needed
```

### 4. Clean Docker (if installed inside WSL)

```bash
docker system prune -a
```

---

## 🧊 Shrinking WSL2 `.vhdx` File (Virtual Disk)

### ✅ Requirements

* WSL2 must be used.
* Hyper-V must be available/enabled.
* `Optimize-VHD` is part of the **Hyper-V PowerShell module**.

### 🔎 Check WSL Version

```powershell
wsl -l -v
```

> Only WSL2 distros have a `.vhdx` file.

### 📂 Find the `.vhdx` File

Typically located at:

```
C:\Users\<YourName>\AppData\Local\Packages\<DistroName>\LocalState\ext4.vhdx
```

Find it via PowerShell:

```powershell
Get-ChildItem -Path "$env:LOCALAPPDATA\Packages" -Recurse -Filter ext4.vhdx
```

---

## 🚫 If `Optimize-VHD` Fails

### ❌ Common Error

```
Optimize-VHD : The term 'Optimize-VHD' is not recognized...
```

### ✅ Solution: Enable Hyper-V

**Run in PowerShell as Administrator:**

```powershell
dism.exe /Online /Enable-Feature /FeatureName:Microsoft-Hyper-V-All /All /NoRestart
dism.exe /Online /Enable-Feature /FeatureName:Hyper-V-PowerShell /All /NoRestart
```

Then reboot your PC.

---

## ⚠️ If You’re on Windows Home Edition

Hyper-V is not officially supported. You have two choices:

### Option 1: Clean Files Inside WSL (Skip VHD Optimization)

* Perform all cleanup steps inside WSL.
* The `.vhdx` file won't shrink but won't grow if usage stays low.

### Option 2: Export → Re-Import (to Shrink)

```powershell
wsl --export Ubuntu "C:\Users\<You>\ubuntu_backup.tar"
wsl --unregister Ubuntu
wsl --import Ubuntu "C:\WSL\Ubuntu" "C:\Users\<You>\ubuntu_backup.tar"
```

This creates a fresh, compressed `.vhdx`.

---

## ✅ To Shrink `.vhdx` with Hyper-V

1. **Shutdown WSL**:

```powershell
wsl --shutdown
```

2. **Optimize the VHDX**:

```powershell
Optimize-VHD -Path "C:\Users\<You>\AppData\Local\Packages\<Distro>\LocalState\ext4.vhdx" -Mode Full
```

> Run this in PowerShell **as Administrator**

---

## 🧭 Summary of Options

| Option                         | Shrinks `.vhdx`? | Requires Hyper-V | Risk Level | Notes                              |
| ------------------------------ | ---------------- | ---------------- | ---------- | ---------------------------------- |
| Clean up files inside WSL      | ❌                | No               | 🟢 Safe    | Frees internal space               |
| Optimize-VHD                   | ✅                | Yes              | 🟢 Safe    | Official method to shrink `.vhdx`  |
| Enable Hyper-V on Windows Home | ✅                | Yes (unofficial) | 🔴 Risky   | Not supported by Microsoft         |
| Export → Import WSL            | ✅                | No               | 🟡 Medium  | Recommended for Home edition users |

---

## 📦 Notes

* Always back up important data before running `wsl --unregister`.
* You can automate cleanup and shrink steps in a script if used regularly.
```
