# Windows Server Post‑Installation Hardening & Configuration Guide

Author: Andrea Balconi\
Version: 1.0\
Audience: System Administrators / Infrastructure Engineers\
Scope: Windows Server manual post‑installation configuration and
baseline hardening.

------------------------------------------------------------------------

# Table of Contents

1.  Prerequisites
2.  Naming and System Configuration
3.  Power Configuration
4.  Feature Installation
5.  Time Synchronization
6.  Windows Activation
7.  Network Adapter Configuration
8.  Regional and Language Settings
9.  Security Hardening
10. Windows Update Configuration
11. Network Protocol Hardening
12. Service Hardening
13. Firewall Baseline Rules
14. Logging and Diagnostics
15. WSUS Configuration

------------------------------------------------------------------------

# 1. Prerequisites

Run PowerShell **as Administrator**.

Recommended checks:

``` powershell
whoami
hostname
(Get-CimInstance Win32_OperatingSystem).Caption
```

------------------------------------------------------------------------

# 2. Rename Server

``` powershell
$NewComputerName = "SERVER01"
Rename-Computer -NewName $NewComputerName -Force
```

Verification

``` powershell
hostname
```

⚠ Reboot required.

------------------------------------------------------------------------

# 3. Set Power Plan to High Performance

``` powershell
$plan = Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object {$_.ElementName -eq "High performance"}

$plan.Activate()
```

Verification

``` powershell
Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object {$_.IsActive -eq $true}
```

------------------------------------------------------------------------

# 4. Install .NET Framework 3.5

``` powershell
Install-WindowsFeature NET-Framework-Core
```

Offline installation

``` powershell
Install-WindowsFeature NET-Framework-Core -Source D:\sources\sxs
```

Verification

``` powershell
Get-WindowsFeature NET-Framework-Core
```

------------------------------------------------------------------------

# 5. Install Telnet Client

``` powershell
Install-WindowsFeature Telnet-Client
```

Verification

``` powershell
Get-WindowsFeature Telnet-Client
```

------------------------------------------------------------------------

# 6. Configure NTP

``` powershell
$ntp = "192.168.1.1"

w32tm /config /manualpeerlist:$ntp /syncfromflags:manual /reliable:yes /update
w32tm /resync
```

Verification

``` powershell
w32tm /query /status
```

------------------------------------------------------------------------

# 7. Windows Activation (KMS)

``` powershell
$productkey = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$kms = "kms01.arubacloud.com:12688"

cscript C:\Windows\System32\slmgr.vbs /ipk $productkey
cscript C:\Windows\System32\slmgr.vbs /skms $kms
cscript C:\Windows\System32\slmgr.vbs /ato
```

Verification

``` powershell
slmgr /dlv
```

------------------------------------------------------------------------

# 8. Disable NIC Power Saving

``` powershell
Get-NetAdapter |
Where-Object {$_.Status -eq "Up"} |
ForEach-Object {

Get-NetAdapterPowerManagement -Name $_.Name |
Set-NetAdapterPowerManagement -AllProperties $false -Confirm:$false

}
```

------------------------------------------------------------------------

# 9. Configure Language and Regional Settings

``` powershell
Set-WinUILanguageOverride en-US
Set-WinUserLanguageList it-IT -Force
Set-Culture it-IT
Set-WinSystemLocale it-IT
Set-WinHomeLocation -GeoId 118
Set-TimeZone "W. Europe Standard Time"
```

Keyboard layout

``` powershell
$lang = New-WinUserLanguageList it-IT
$lang[0].InputMethodTips.Clear()
$lang[0].InputMethodTips.Add("0410:00000410")
Set-WinUserLanguageList $lang -Force
```

------------------------------------------------------------------------

# 10. Disable IE Enhanced Security Configuration

``` powershell
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
```

------------------------------------------------------------------------

# 11. Disable Telemetry

``` powershell
Stop-Service DiagTrack -Force
Set-Service DiagTrack -StartupType Disabled

Stop-Service dmwappushservice -Force
Set-Service dmwappushservice -StartupType Disabled
```

Registry

``` powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Force
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Force

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" AllowDiagnosticData 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" AllowDiagnosticData 0
```

------------------------------------------------------------------------

# 12. Configure Windows Update Manual Mode

``` powershell
Set-Service wuauserv -StartupType Manual
```

``` powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" AUOptions 2
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" DisableWindowsUpdateAccess 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoUpdate 1
```

------------------------------------------------------------------------

# 13. Disable IPv6

``` powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" -Force
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" DisabledComponents 255
```

Disable binding

``` powershell
Get-NetAdapter |
Where-Object {$_.Status -eq "Up"} |
Disable-NetAdapterBinding -ComponentID ms_tcpip6 -Confirm:$false
```

------------------------------------------------------------------------

# 14. Disable Unnecessary Services

Example

``` powershell
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled

Stop-Service iphlpsvc -Force
Set-Service iphlpsvc -StartupType Disabled
```

------------------------------------------------------------------------

# 15. Disable UAC Remote Restrictions

``` powershell
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Force
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" LocalAccountTokenFilterPolicy 1
```

------------------------------------------------------------------------

# 16. Firewall Baseline Rules

``` powershell
New-NetFirewallRule -DisplayName "Allow SMB-In" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow SMB-Out" -Direction Outbound -Protocol TCP -RemotePort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Direction Inbound -Protocol ICMPv4 -Action Allow
```

Verification

``` powershell
Get-NetFirewallRule | Where-Object DisplayName -Match "SMB|ICMP"
```

------------------------------------------------------------------------

# 17. Disable NetBIOS

``` powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object {$_.IPEnabled -eq $true} |
ForEach-Object {

Invoke-CimMethod -InputObject $_ -MethodName SetTcpipNetbios -Arguments @{TcpipNetbiosOptions=2}

}
```

------------------------------------------------------------------------

# 18. Disable mDNS

``` powershell
Stop-Service mdnsresponder -Force
Set-Service mdnsresponder -StartupType Disabled
```

``` powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Force

Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" DisableMulticast 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" DisableMulticast 1
```

------------------------------------------------------------------------

# 19. Enable Kerberos Logging

``` powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" -Force

Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" LogLevel 1
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" AuditKerberos 1
```

Enable operational log

``` powershell
wevtutil set-log Microsoft-Windows-Kerberos/Operational /enabled:true
```

------------------------------------------------------------------------

# 20. Configure WSUS Client

``` powershell
$wsus = "wsus.example.com:8530"
```

``` powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force
```

``` powershell
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" WUServer $wsus
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" WUStatusServer $wsus
```

``` powershell
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" AUOptions 2
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoUpdate 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoRebootWithLoggedOnUsers 1
```

``` powershell
Set-Service wuauserv -StartupType Manual
```

------------------------------------------------------------------------
