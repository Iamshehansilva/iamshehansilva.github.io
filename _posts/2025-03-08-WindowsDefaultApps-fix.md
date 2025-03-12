---
title: "Sysprep & Windows Updates Breaking Default Apps? Fix It in AVD with Nerdio"
date: "2025-03-08 00:00:00 +0100"
categories: [Azure, AVD, Windows Updates, Nerdio, Windows Default Apps, Custom Image, Windows 11]
toc: true
tags: [Azure, AVD, Windows Updates, Nerdio, Windows Default Apps, Custom Image, Windows 11]
image:
  path: /assets/img/MSdefaultAppsFix/MS-Default-App-Main.png
  src: /assets/img/MSdefaultAppsFix/MS-Default-App-Main.png
---

## 🕵️‍♂️ Situation

Recently, I ran into an issue where Windows default apps (like Snipping Tool, Calculator, and Paint) disappeared after Windows Updates and Sysprep on Windows 11. The default apps were present in the customer image, but once we converted it into a deployable image, they were removed for some reason.

Troubleshooting and findings.

1. Rolling back the latest Windows updates and skipping Sysprep and voilà! The default apps were back.
2. Manually registering app packages on the customer image and then running Sysprep, but the apps still disappeared in the final deployable image.
3. Registering app packages directly on production VMs and the missing apps returned.

Since this issue started with the January updates, we initially removed them, assuming February updates would fix it. Unfortunately, that wasn’t the case, so we needed a way to register the missing app packages before users logged in.

## 💡 Finding a Solution

Our first approach was to create a PowerShell script that would register the required app packages each night when VMs were recreated. However this failed because Appx package registration requires a user context. The Local System account lacks the necessary permissions to install or register UWP apps, and we didnt want to modify permissions on the WindowsApps folder as it could lead to corruption.

So, instead, we decided to:

- ✅ Use a scheduled task that runs at startup
- ✅ Write a PowerShell script to configure this scheduled task
- ✅ Deploy it using Nerdio Script Action

## 🎯 Why Not Use a GPO Logon Script?

Instead of a GPO logon script, we chose this approach to ensure Windows default apps are available before users log in. This prevents unnecessary logon delays and improves the user experience by handling app registration at startup rather than during login.

## ⚙️ Scripts

### 📜 AppX Registeration Script
🛠 **Script Functionality**
- Filters the WindowsApps Folder
- Check and Register the Latest Versions of Each Package

```powershell
# Define the path to the WindowsApps folder
$path = "C:\Program Files\WindowsApps"

# Get all directories in the WindowsApps folder
$packages = Get-ChildItem -Path $path -Recurse -Directory

# Filter for Microsoft.WindowsAppRuntime.1.5 packages (x86 and x64)
$windowsAppRuntime15Packages = $packages | Where-Object { $_.Name -like "Microsoft.WindowsAppRuntime.1.5*" }

# Separate x86 and x64 packages
$x86Runtime15Packages = $windowsAppRuntime15Packages | Where-Object { $_.Name -match "x86" }
$x64Runtime15Packages = $windowsAppRuntime15Packages | Where-Object { $_.Name -match "x64" }

# Get the latest x86 version by sorting based on version and selecting the highest version
$latestX86Runtime15Package = $x86Runtime15Packages | Sort-Object Name -Descending | Select-Object -First 1

# Get the latest x64 version by sorting based on version and selecting the highest version
$latestX64Runtime15Package = $x64Runtime15Packages | Sort-Object Name -Descending | Select-Object -First 1

# Register the latest x86 version of Microsoft.WindowsAppRuntime.1.5
if ($latestX86Runtime15Package) {
    $x86Runtime15PackageManifest = Join-Path -Path $latestX86Runtime15Package.FullName -ChildPath "AppxManifest.xml"
    Write-Host "Registering latest x86 Microsoft.WindowsAppRuntime.1.5 package: $x86Runtime15PackageManifest"
    Add-AppxPackage -Register $x86Runtime15PackageManifest -DisableDevelopmentMode
}

# Register the latest x64 version of Microsoft.WindowsAppRuntime.1.5
if ($latestX64Runtime15Package) {
    $x64Runtime15PackageManifest = Join-Path -Path $latestX64Runtime15Package.FullName -ChildPath "AppxManifest.xml"
    Write-Host "Registering latest x64 Microsoft.WindowsAppRuntime.1.5 package: $x64Runtime15PackageManifest"
    Add-AppxPackage -Register $x64Runtime15PackageManifest -DisableDevelopmentMode
}

# Define an array of package names you want to check
$packageNames = @(
    "Microsoft.VCLibs",
    "Microsoft.VCLibs.140.00.UWPDesktop",
    "Microsoft.NET.Native.Framework",
    "Microsoft.NET.Native.Runtime",
    "Microsoft.WindowsAppRuntime",
    "Microsoft.UI.Xaml",
    "Microsoft.Paint",
    "Microsoft.WindowsNotepad",
    "Microsoft.ScreenSketch",
    "Microsoft.WindowsCalculator",
    "Microsoft.HEIFImageExtension",
    "Microsoft.RawImageExtension",
    "Microsoft.VP9VideoExtensions",
    "Microsoft.WebMediaExtensions",
    "Microsoft.WebpImageExtension",
    "MicrosoftWindows.Client.WebExperience",
    "Microsoft.Windows.CloudExperienceHost",
    "Windows.CBSPreview",
    "Windows.PrintDialog",
    "MicrosoftWindows.CrossDevice"
)

# Loop through each package name
foreach ($packageName in $packageNames) {
    # Get all directories in the WindowsApps folder
    $packages = Get-ChildItem -Path $path -Recurse -Directory

    # Filter the directories for those that match the current package name
    $windowsAppRuntimePackages = $packages | Where-Object { $_.Name -like "$packageName*" }

    # Filter for both x86 and x64 architectures
    $x86Packages = $windowsAppRuntimePackages | Where-Object { $_.Name -match "x86" }
    $x64Packages = $windowsAppRuntimePackages | Where-Object { $_.Name -match "x64" }

    # Sort and get the latest version for x86
    $latestX86Package = $x86Packages | Sort-Object Name -Descending | Select-Object -First 1

    # Sort and get the latest version for x64
    $latestX64Package = $x64Packages | Sort-Object Name -Descending | Select-Object -First 1

    # Check and register the latest x86 package
    if ($latestX86Package) {
        $x86PackageManifest = Join-Path -Path $latestX86Package.FullName -ChildPath "AppxManifest.xml"
        Write-Host "Registering x86 package: $x86PackageManifest"
        Add-AppxPackage -Register $x86PackageManifest -DisableDevelopmentMode
    }

    # Check and register the latest x64 package
    if ($latestX64Package) {
        $x64PackageManifest = Join-Path -Path $latestX64Package.FullName -ChildPath "AppxManifest.xml"
        Write-Host "Registering x64 package: $x64PackageManifest"
        Add-AppxPackage -Register $x64PackageManifest -DisableDevelopmentMode
    }
}

```

### 📜 Scheduled Task Creation
🛠 **Script Functionality**
- This script creates a scheduled task that runs a PowerShell script at startup using a specific user account, ensuring apps or settings are configured automatically.

```Powershell
# Define the path to your PowerShell script
$scriptPath = "C:\###\###\StartupDefaultAppx.ps1"

$Username = $ADUsername
$ADPassword = $ADPassword

# Create the action to run the PowerShell script
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -NoProfile -NonInteractive -WindowStyle Hidden -File `"$scriptPath`""

# Create the trigger to run at startup
$trigger = New-ScheduledTaskTrigger -AtStartup

# Register the scheduled task
Register-ScheduledTask -TaskName "Default App Fix Script at Startup" -Action $action -Trigger $trigger -User $Username -Password $ADPassword

```

## 🔧 Configuration
1. Use Group Policy to copy the script across all VMs.
2. Add the Scheduled Task Creation Script to Nerdio Script Actions as a Windows Script.
3. Enable Script Action in Host Pool Properties and VM Deployment:
  - Configure the script to execute automatically when a new host VM is created.
  - Make sure to pass Active Directory (AD) credentials for the script to run correctly.

![Script Action](/assets/img/MSdefaultAppsFix/MS-Default-App-Nerdio.png)

## 🌟 Summary

If you’ve ever dealt with Windows updates or Sysprep breaking default apps, you know how frustrating it can be. Suddenly, Calculator, Notepad, or other essentials disappear, disrupting the user experience. Instead of manually fixing it each time, automation is the solution.

By leveraging Nerdio Script Actions, along with PowerShell scripts and scheduled tasks, we can ensure default apps are automatically restored at startup, minimizing disruptions and saving time. This way, default apps stay intact, and users never notice a thing.

⚠️ Always test in a non-production environment before deploying. With great power comes great responsibility!

Keep automating and innovating, cheers! 🚀
