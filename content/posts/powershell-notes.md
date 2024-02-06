+++
title = "Notes on Powershell"
publishDate = "2024-02-12T08:00:00Z"
tags = ["System Administration", "Powershell", "Windows", "Today I Learned"]
showFullContent = false
readingTime = false
hideComments = false
+++

Some silly little powershell notes from a silly little man.

## Mapping network printers
This one is super duper straightforward. Can probably use it remotely through the command line option on a UEM (Unified Endpoint Management) tool.

```powershell
Add-Printer -ConnectionName \\printer-server\PRINTERNAME
```

## Fetching an installer from a URL and install it
This one we create a temporary directory, set the URL where the installer can be located, download, then install it. If there's any switches you want applied to the installer, place them after the `/i <installer>` portion.

```powershell
New-Item -Path 'C:\temp' -ItemType Directory -Force
$installerURL = "https://pkgs.tailscale.com/stable/tailscale-setup-1.58.2-amd64.msi"
Invoke-WebRequest $installerURL -OutFile C:\temp\tailscale-setup-amd64.msi
Start-Process msiexec.exe -Wait -ArgumentList '/i C:\temp\tailscale-setup-amd64.msi'
```

## Uninstall a package
Fairly straight forward, we need to find the installed Win32 package, then we target it with a uninstall function.

First we find our target and the proper details for filtering. Note this will only work on Win32_Product installed items. A quick google tells me "The Win32_Product WMI class represents products as they are installed by Windows Installer." as for what a Windows Installer is and isn't I'm not fully sure. I see several things that are absent from this list. But we proceed!

```powershell
Get-WmiObject -Class Win32_Product 
```

You'll receive a list of items similiar to the following.

```
IdentifyingNumber : {23170F69-40C1-2701-2201-000001000000}
Name              : 7-Zip 22.01
Vendor            : Igor Pavlov
Version           : 22.01.00.0
Caption           : 7-Zip 22.01
```

Then we run the following. In this case using the IdentifyingNumber is preferable as this can help unambiguously point at a specific product. But using Name is also possible. 

```powershell
$application1 = Get-WmiObject -Class Win32_Product -Filter "IdentifyingNumber = '{23170F69-40C1-2701-2201-000001000000}'";
$application1.Uninstall()
```