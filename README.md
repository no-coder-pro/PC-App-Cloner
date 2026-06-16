# How to Clone UWP/Store Apps on Windows

This guide outlines the step-by-step process to clone any Windows Store (UWP / MSIX) desktop application so that you can run multiple instances side-by-side with separate accounts, settings, and data folders.

---

## Prerequisites
1. **Developer Mode Enabled:** Windows must allow sideloading of unsigned packages.
2. **Access to `C:\Program Files\WindowsApps`:** This folder is protected, but you can read/copy files from it as a standard user or administrator.

---

## Step-by-Step Cloning Process

### Step 1: Enable Developer Mode
Open **PowerShell as Administrator** and run the following commands to enable Developer Mode in the registry:

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" -Name "AllowDevelopmentWithoutDevLicense" -Value 1 -Type DWord -ErrorAction SilentlyContinue
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock" -Name "AllowAllTrustedApps" -Value 1 -Type DWord -ErrorAction SilentlyContinue
```

### Step 2: Find the Target App Location
Find the installation folder of the app you want to clone by running:

```powershell
Get-AppxPackage -Name *AppName*
```
Look for the `InstallLocation` line in the output. For example:
`C:\Program Files\WindowsApps\5319275A.WhatsAppDesktop_2.2620.102.0_x64__cv1g1gvanyjgm`

### Step 3: Copy App Files
Copy the entire application folder to a user-writable directory (e.g. under AppData):

```powershell
# Example: Copying WhatsApp to a custom folder
robocopy "C:\Program Files\WindowsApps\Target_App_Folder" "$env:LOCALAPPDATA\AppClone_App" /E /R:1 /W:1
```

### Step 4: Remove Signatures and Metadata
Since you will modify the application layout, the original digital signature is no longer valid. Delete the signature files so Windows can register it as an unsigned developer layout:

```powershell
$clonePath = "$env:LOCALAPPDATA\AppClone_App"

# Delete signature and blockmap files
Remove-Item -Path "$clonePath\AppxSignature.p7x" -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$clonePath\AppxBlockMap.xml" -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$clonePath\AppxMetadata" -Recurse -Force -ErrorAction SilentlyContinue
```

### Step 5: Modify the Manifest (`AppxManifest.xml`)
Open `AppxManifest.xml` located in the root of the clone folder. Modify the XML file as follows:

1. **Change Package Identity:**
   Find the `<Identity>` node near the top:
   ```xml
   <Identity Name="OriginalAppName" Publisher="..." Version="..." ... />
   ```
   Change the `Name` attribute to a unique value (e.g. `OriginalAppNameClone` or add `.Clone` to the end).
   
2. **Change Display Name:**
   Change the `<DisplayName>` elements in the `<Properties>` block and under the `<Application>` block to something recognizable (e.g. `App Name Clone`).
   
3. **Change Protocols (Optional but Recommended):**
   If the app registers a protocol handler (e.g. `whatsapp:`), change the protocol name in the manifest (e.g. to `whatsapp-clone:`) to prevent it from hijacking launch requests intended for the primary app.

#### PowerShell XML Automation Example:
You can automate this modification using PowerShell's XML parser:

```powershell
$manifestPath = "$env:LOCALAPPDATA\AppClone_App\AppxManifest.xml"
[xml]$xml = Get-Content -Path $manifestPath

# 1. Update Identity
$xml.Package.Identity.Name = "OriginalAppNameClone"

# 2. Update Display Names
$xml.Package.Properties.DisplayName = "App Name Clone"
$xml.Package.Applications.Application.VisualElements.DisplayName = "App Name Clone"

# 3. Save Manifest
$xml.Save($manifestPath)
```

### Step 6: Register the Cloned Package
Register the app manifest under your Windows user profile:

```powershell
Add-AppxPackage -Register "$env:LOCALAPPDATA\AppClone_App\AppxManifest.xml"
```

### Step 7: Create a Launch Shortcut
Find the `PackageFamilyName` of the cloned app:

```powershell
(Get-AppxPackage -Name *OriginalAppNameClone*).PackageFamilyName
```

Create a desktop shortcut with the target set to:
`explorer.exe shell:AppsFolder\PackageFamilyName!App` (where `App` is the `<Application Id="xxx">` specified in the manifest, usually `App`).

---

## Can other apps be cloned like this?

**Yes!** Most store apps can be cloned using this exact method, but there are some exceptions and limitations:

### 🟢 What CAN be cloned easily:
* **Win32 apps packaged as MSIX:** Many desktop apps (like WhatsApp, Telegram, Spotify, or Discord Desktop) are standard Win32 programs packaged inside MSIX. These clone easily because they behave like standard applications.
* **Simple UWP games and utilities:** Small tools, calculators, and independent offline apps.

### 🔴 What CANNOT be cloned easily:
* **Apps with DRM / License Checks:** Apps that strictly check with the Microsoft Store licensing API on launch (like paid games: Minecraft, Forza, etc.) will detect that the package identity is changed and fail to launch.
* **System Integrations:** Apps that install background system drivers or bind to hardcoded Windows services (like antivirus tools or deep integration components) may fail when their package name is altered.
* **Dependency Issues:** Some apps require specific system dependencies that are strictly bound to their original Package Family Name.
