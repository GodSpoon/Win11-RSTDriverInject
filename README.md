# Windows 11 RST Driver Injector

This script automates injecting Intel Rapid Storage Technology (RST) drivers into a Windows 11 ISO file. This is useful for systems that require RST drivers during installation.

## Prerequisites

- Windows PowerShell (Run as Administrator)
- At least 10GB free disk space
- Original Windows 11 ISO file

## Usage

1. Save this script as `Inject-RSTDrivers.ps1`:

```powershell
# Create working directories
$workDir = Join-Path $env:TEMP "RSTInjector"
New-Item -ItemType Directory -Force -Path $workDir | Out-Null
Push-Location $workDir

Write-Host "Downloading required files..."

# Download tools and drivers
Invoke-WebRequest "https://github.com/GodSpoon/Win11-RSTDriverInject/raw/refs/heads/main/oscdimg.exe" -OutFile "oscdimg.exe"
Invoke-WebRequest "https://github.com/GodSpoon/Win11-RSTDriverInject/raw/refs/heads/main/SetupRST_extracted.zip" -OutFile "SetupRST_extracted.zip"
Expand-Archive "SetupRST_extracted.zip" -DestinationPath "drivers" -Force

# Prompt for ISO file
Add-Type -AssemblyName System.Windows.Forms
$FileBrowser = New-Object System.Windows.Forms.OpenFileDialog
$FileBrowser.Filter = "ISO files (*.iso)|*.iso"
$FileBrowser.Title = "Select Windows 11 ISO"
if ($FileBrowser.ShowDialog() -ne "OK") { 
    Write-Host "No ISO selected. Exiting."
    exit 
}
$isoPath = $FileBrowser.FileName
$outputPath = $isoPath.Replace(".iso", "_RST.iso")

Write-Host "Creating mount points..."
New-Item -ItemType Directory -Force -Path "$workDir\mount-iso" | Out-Null
New-Item -ItemType Directory -Force -Path "$workDir\mount-wim" | Out-Null

Write-Host "Mounting ISO..."
$isoMount = Mount-DiskImage -ImagePath $isoPath -PassThru
$isoDrive = ($isoMount | Get-Volume).DriveLetter

Write-Host "Copying and preparing install.wim..."
Copy-Item "${isoDrive}:\sources\install.wim" "$workDir\mount-wim\install.wim"
attrib -r "$workDir\mount-wim\install.wim"
takeown /f "$workDir\mount-wim\install.wim" | Out-Null
icacls "$workDir\mount-wim\install.wim" /grant administrators:F | Out-Null

Write-Host "Preparing mount directory..."
takeown /f "$workDir\mount-iso" | Out-Null
icacls "$workDir\mount-iso" /grant administrators:F | Out-Null

Write-Host "Mounting WIM file..."
dism /mount-image /imagefile:"$workDir\mount-wim\install.wim" /index:1 /mountdir:"$workDir\mount-iso"

Write-Host "Injecting RST drivers..."
dism /image:"$workDir\mount-iso" /add-driver /driver:"$workDir\drivers\production\Windows10-x64\15063\Drivers" /recurse

Write-Host "Committing changes and unmounting..."
dism /unmount-image /mountdir:"$workDir\mount-iso" /commit

Write-Host "Creating new ISO..."
& "$workDir\oscdimg.exe" -m -o -u2 -udfver102 -bootdata:2#p0,e,b${isoDrive}:\boot\etfsboot.com#pEF,e,b${isoDrive}:\efi\microsoft\boot\efisys.bin ${isoDrive}:\ $outputPath

Write-Host "Cleaning up..."
Dismount-DiskImage -ImagePath $isoPath
Pop-Location
Remove-Item -Recurse -Force $workDir

Write-Host "Done! Modified ISO saved as: $outputPath"
```

## How it Works

The script creates a temporary working directory
Downloads required tools (oscdimg) and RST drivers
Prompts you to select your Windows 11 ISO file
Mounts the ISO and extracts the install.wim
Injects the RST drivers into the Windows image
Creates a new ISO file with the injected drivers
Cleans up temporary files

## Output
The script will create a new ISO file with "_RST" added to the original filename. For example:

Input: Win11_24H2_English_x64.iso
Output: Win11_24H2_English_x64_RST.iso

## Notes

The script must be run as Administrator
The process requires approximately 10GB of free space for temporary files
The output ISO will be saved in the same directory as the input ISO
All temporary files are automatically cleaned up after completion
