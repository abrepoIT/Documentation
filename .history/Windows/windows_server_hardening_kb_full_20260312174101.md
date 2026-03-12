# KB - Windows Server Post-Installation Hardening and Configuration

Author: Andrea Balconi  
Version: 2.0  
Last Update: 2026-03-12  
Audience: System Administrators / Infrastructure Engineers  
Document Type: Operational Hardening KB  
Scope: Manual hardening and baseline configuration procedure for post-installation Windows Server.

---

## Table of Contents

1. Purpose
2. Prerequisites
3. Rename Server
4. Power Plan
5. Install .NET Framework 3.5
6. Install Telnet Client
7. Configure NTP
8. Windows Activation (KMS)
9. Disable NIC Power Saving
10. Configure Language and Regional Settings
11. Disable IE Enhanced Security Configuration
12. Disable Telemetry
13. Configure Windows Update Manual Mode
14. Disable IPv6
15. Disable Unnecessary Services
16. Disable UAC Remote Restrictions
17. Firewall Baseline Rules
18. Disable NetBIOS
19. Disable mDNS
20. Enable Kerberos Logging
21. Configure WSUS Client
22. Post-Change Verification
23. Evidence Collection Template

---

## 1) Purpose

This knowledge base defines a standard post-installation configuration sequence to:
- standardize the security baseline
- reduce the attack surface
- enable centralized update governance

Each section includes:
- copyable command
- technical objective
- quick verification

---

## 2) Prerequisites

Run PowerShell as Administrator.

~~~powershell
whoami
hostname
(Get-CimInstance Win32_OperatingSystem).Caption
~~~

Objective:
- confirm privileges and machine context before making changes

---

## 3) Rename Server

~~~powershell
$NewComputerName = "SERVER01"
Rename-Computer -NewName $NewComputerName -Force
~~~

Verification:

~~~powershell
hostname
Get-ComputerInfo | Select-Object CsName
~~~

Expected result:
- host name aligned with naming convention

Note:
- reboot required for full application

---

## 4) Power Plan

~~~powershell
$plan = Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object { $_.ElementName -eq "High performance" }

$plan.Activate()
~~~

Verification:

~~~powershell
Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object { $_.IsActive -eq $true }
~~~

Expected result:
- active plan set to High performance

---

## 5) Install .NET Framework 3.5

~~~powershell
Install-WindowsFeature NET-Framework-Core
~~~

Offline option:

~~~powershell
Install-WindowsFeature NET-Framework-Core -Source D:\sources\sxs
~~~

Verification:

~~~powershell
Get-WindowsFeature NET-Framework-Core
~~~

Expected result:
- feature installed correctly

---

## 6) Install Telnet Client

~~~powershell
Install-WindowsFeature Telnet-Client
~~~

Verification:

~~~powershell
Get-WindowsFeature Telnet-Client
~~~

Expected result:
- Telnet Client present and installed

---

## 7) Configure NTP

~~~powershell
$ntp = "192.168.1.1"

w32tm /config /manualpeerlist:$ntp /syncfromflags:manual /reliable:yes /update
w32tm /resync
~~~

Verification:

~~~powershell
w32tm /query /configuration
w32tm /query /status
~~~

Expected result:
- correct NTP source and active synchronization

---

## 8) Windows Activation (KMS)

~~~powershell
$productkey = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$kms = "kms01.arubacloud.com:12688"

cscript C:\Windows\System32\slmgr.vbs /ipk $productkey
cscript C:\Windows\System32\slmgr.vbs /skms $kms
cscript C:\Windows\System32\slmgr.vbs /ato
~~~

Verification:

~~~powershell
slmgr /dlv
~~~

Expected result:
- activation channel and KMS host configured correctly

---

## 9) Disable NIC Power Saving

~~~powershell
Get-NetAdapter |
Where-Object { $_.Status -eq "Up" } |
ForEach-Object {
    Get-NetAdapterPowerManagement -Name $_.Name |
    Set-NetAdapterPowerManagement -AllProperties $false -Confirm:$false
}
~~~

Verification:

~~~powershell
Get-NetAdapterPowerManagement
~~~

Expected result:
- power-saving options not active on operational adapters

---

## 10) Configure Language and Regional Settings

~~~powershell
Set-WinUILanguageOverride en-US
Set-WinUserLanguageList it-IT -Force
Set-Culture it-IT
Set-WinSystemLocale it-IT
Set-WinHomeLocation -GeoId 118
Set-TimeZone "W. Europe Standard Time"
~~~

Keyboard layout configuration:

~~~powershell
$lang = New-WinUserLanguageList it-IT
$lang[0].InputMethodTips.Clear()
$lang[0].InputMethodTips.Add("0410:00000410")
Set-WinUserLanguageList $lang -Force
~~~

Verification:

~~~powershell
Get-Culture
Get-WinSystemLocale
Get-TimeZone
~~~

Expected result:
- language, locale, and timezone aligned with the baseline

Note:
- some settings fully converge only after logoff or reboot

---

## 11) Disable IE Enhanced Security Configuration

~~~powershell
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
~~~

Verification:

~~~powershell
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
~~~

Expected result:
- both keys set to value 0

---

## 12) Disable Telemetry

~~~powershell
Stop-Service DiagTrack -Force
Set-Service DiagTrack -StartupType Disabled

Stop-Service dmwappushservice -Force
Set-Service dmwappushservice -StartupType Disabled
~~~

Registry configuration:

~~~powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Force
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Force

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" AllowDiagnosticData 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" AllowDiagnosticData 0
~~~

Verification:

~~~powershell
Get-Service DiagTrack, dmwappushservice | Select-Object Name, Status, StartType
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection /v AllowDiagnosticData
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection /v AllowDiagnosticData
~~~

Expected result:
- services disabled
- AllowDiagnosticData set to 0 in both keys

---

## 13) Configure Windows Update Manual Mode

~~~powershell
Set-Service wuauserv -StartupType Manual
~~~

~~~powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" AUOptions 2
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" DisableWindowsUpdateAccess 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoUpdate 1
~~~

Verification:

~~~powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
Get-CimInstance Win32_Service -Filter "Name='wuauserv'" | Select-Object Name, StartMode
~~~

Expected result:
- AU policy applied and Windows Update service set to Manual

---

## 14) Disable IPv6

~~~powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" -Force
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" DisabledComponents 255
~~~

Binding disable operation:

~~~powershell
Get-NetAdapter |
Where-Object { $_.Status -eq "Up" } |
Disable-NetAdapterBinding -ComponentID ms_tcpip6 -Confirm:$false
~~~

Verification:

~~~powershell
reg query HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters /v DisabledComponents
Get-NetAdapterBinding -ComponentID ms_tcpip6 | Select-Object Name, Enabled
~~~

Expected result:
- DisabledComponents set to 255
- IPv6 binding disabled on target adapters

---

## 15) Disable Unnecessary Services

Example:

~~~powershell
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled

Stop-Service iphlpsvc -Force
Set-Service iphlpsvc -StartupType Disabled
~~~

Verification:

~~~powershell
Get-Service | Where-Object { $_.StartType -eq "Disabled" }
~~~

Expected result:
- unnecessary services disabled according to company policy

---

## 16) Disable UAC Remote Restrictions

~~~powershell
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Force
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" LocalAccountTokenFilterPolicy 1
~~~

Verification:

~~~powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy
~~~

Expected result:
- LocalAccountTokenFilterPolicy set to 1

---

## 17) Firewall Baseline Rules

~~~powershell
New-NetFirewallRule -DisplayName "Allow SMB-In" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow SMB-Out" -Direction Outbound -Protocol TCP -RemotePort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Direction Inbound -Protocol ICMPv4 -Action Allow
~~~

Verification:

~~~powershell
Get-NetFirewallRule | Where-Object DisplayName -Match "SMB|ICMP"
Get-NetFirewallRule -DisplayName "Allow SMB-In","Allow SMB-Out","Allow ICMPv4-In" | Select-Object DisplayName, Enabled
~~~

Expected result:
- rules created and enabled

---

## 18) Disable NetBIOS

~~~powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object { $_.IPEnabled -eq $true } |
ForEach-Object {
    Invoke-CimMethod -InputObject $_ -MethodName SetTcpipNetbios -Arguments @{ TcpipNetbiosOptions = 2 }
}
~~~

Verification:

~~~powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object { $_.IPEnabled } |
Select-Object Description, TcpipNetbiosOptions
~~~

Expected result:
- TcpipNetbiosOptions set to 2

---

## 19) Disable mDNS

~~~powershell
Stop-Service mdnsresponder -Force
Set-Service mdnsresponder -StartupType Disabled
~~~

~~~powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Force

Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" DisableMulticast 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" DisableMulticast 1
~~~

Verification:

~~~powershell
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" /v DisableMulticast
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" /v DisableMulticast
Get-Service mdnsresponder -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType
~~~

Expected result:
- multicast name resolution disabled by policy and global key

---

## 20) Enable Kerberos Logging

~~~powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" -Force

Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" LogLevel 1
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" AuditKerberos 1
~~~

Operational log enablement:

~~~powershell
wevtutil set-log Microsoft-Windows-Kerberos/Operational /enabled:true
~~~

Verification:

~~~powershell
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v LogLevel
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v AuditKerberos
wevtutil gl "Microsoft-Windows-Kerberos/Operational"
~~~

Expected result:
- LogLevel and AuditKerberos set to 1
- Operational channel enabled

---

## 21) Configure WSUS Client

~~~powershell
$wsus = "wsus.example.com:8530"
~~~

~~~powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Force
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force
~~~

~~~powershell
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" WUServer $wsus
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" WUStatusServer $wsus
~~~

~~~powershell
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" AUOptions 2
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoUpdate 1
Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" NoAutoRebootWithLoggedOnUsers 1
~~~

~~~powershell
Set-Service wuauserv -StartupType Manual
~~~

Verification:

~~~powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoRebootWithLoggedOnUsers
~~~

Expected result:
- WUServer and WUStatusServer keys configured
- update policy aligned with the governed manual model

---

## 22) Post-Change Verification

Recommended final checklist:
1. Reboot the server if pending changes are present.
2. Run the configuration verification KB.
3. Save output and screenshots as compliance evidence.
4. Open remediation tickets for any non-compliant checks.

Result classification:
- PASS: aligned with baseline
- WARNING: partially aligned or in transitional state
- FAIL: not aligned or missing configuration
