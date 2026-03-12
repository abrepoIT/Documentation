# KB - Windows Configuration Verification Checklist
**Author:** Andrea Balconi  
**Version:** 2.0  
**Last Update:** 2026-03-12  
**Audience:** System Administrators / Infrastructure Engineers  
**Scope:** Manual verification of post-deployment Windows Server hardening configuration.

---

## Table of Contents

1. Purpose
2. Prerequisites
3. Naming and Host Identity
4. Power Plan
5. Feature Validation (.NET 3.5, Telnet)
6. Time Sync (NTP)
7. Activation and KMS
8. Network Adapter Power Management
9. Locale and Timezone
10. IE Enhanced Security Configuration
11. Telemetry
12. Windows Update Policy
13. IPv6
14. Disabled Services Review
15. LocalAccountTokenFilterPolicy
16. Firewall Rules (SMB/ICMP)
17. NetBIOS over TCP/IP
18. Multicast Name Resolution
19. Kerberos Logging
20. WSUS Configuration

---

## 1) Purpose

This knowledge base documents the manual verification checks to run on Windows servers configured using a hardening baseline.

Each section includes:
- copyable command
- expected result
- operational notes

---

## 2) Prerequisites

Run PowerShell as Administrator.

```powershell
whoami
hostname
(Get-CimInstance Win32_OperatingSystem).Caption
```

Expected result:
- account with administrative privileges
- hostname aligned with naming convention
- expected operating system

---

## 3) Naming and Host Identity

```powershell
hostname
Get-ComputerInfo | Select-Object CsName
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName' | Select-Object ComputerName
```

Expected result:
- `hostname` and `CsName` are consistent
- any different value in the registry indicates a pending rename until reboot

---

## 4) Power Plan

```powershell
powercfg /L
powercfg /GETACTIVESCHEME
```

Expected result:
- active plan: High performance

---

## 5) Feature Validation (.NET 3.5, Telnet)

```powershell
Get-WindowsOptionalFeature -Online -FeatureName NetFx3
dism /online /Get-FeatureInfo /FeatureName:TelnetClient
```

Expected result:
- NetFx3: `State = Enabled`
- Telnet Client: `State : Enabled`

---

## 6) Time Sync (NTP)

```powershell
w32tm /query /configuration
w32tm /query /status
```

Expected result:
- `Type = NTP`
- `NtpServer` set with the expected peer (gateway or custom server)

---

## 7) Activation and KMS

```powershell
slmgr /dlv
```

Expected result:
- activation channel is consistent
- KMS host and activation status are correct

---

## 8) Network Adapter Power Management

```powershell
Get-NetAdapterPowerManagement

Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002BE10318}\[0-9][0-9][0-9][0-9]' -ErrorAction SilentlyContinue |
ForEach-Object {
    $p = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        KeyName          = $_.PSChildName
        NetCfgInstanceId = $p.NetCfgInstanceId
        PnPCapabilities  = $p.PnPCapabilities
    }
} | Where-Object { $_.NetCfgInstanceId } | Select-Object KeyName, NetCfgInstanceId, PnPCapabilities
```

Expected result:
- in cmdlet output: `AllowComputerToTurnOffDevice = Disabled`
- registry fallback: `PnPCapabilities = 24`

---

## 9) Locale and Timezone

```powershell
Get-Culture
Get-WinSystemLocale
Get-TimeZone
```

Expected result:
- culture, system locale, and timezone aligned with the baseline

Note:
- after changes, sign-out/reboot may be required for full convergence

---

## 10) IE Enhanced Security Configuration

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
```

Expected result:
- both keys: `IsInstalled REG_DWORD 0x0`

---

## 11) Telemetry

```powershell
Get-Service DiagTrack, dmwappushservice | Select-Object Name, Status, StartType
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection /v AllowDiagnosticData
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection /v AllowDiagnosticData
```

Expected result:
- services: `StartType = Disabled`
- both keys: `AllowDiagnosticData REG_DWORD 0x0`

---

## 12) Windows Update Policy

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
Get-CimInstance Win32_Service -Filter "Name='wuauserv'" | Select-Object Name, StartMode
```

Expected result:
- `AUOptions = 2`
- `NoAutoUpdate = 1`
- `DisableWindowsUpdateAccess = 1`
- `wuauserv StartMode = Manual`

---

## 13) IPv6

```powershell
reg query HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters /v DisabledComponents
Get-NetAdapterBinding -ComponentID ms_tcpip6 | Select-Object Name, Enabled
```

Expected result:
- `DisabledComponents = 0xFF`
- on active adapters: `Enabled = False`

Note:
- after the change, some adapters may temporarily remain `Enabled=True` until reboot

---

## 14) Disabled Services Review

```powershell
Get-Service | Where-Object {$_.StartType -eq "Disabled"}
```

Expected result:
- list aligned with company policy

---

## 15) LocalAccountTokenFilterPolicy

```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy
```

Expected result:
- value `1` (UAC remote restrictions disabled)

---

## 16) Firewall Rules (SMB/ICMP)

```powershell
Get-NetFirewallRule | Where-Object {$_.DisplayName -match "SMB|ICMP"}
Get-NetFirewallRule -DisplayName "Allow SMB-In","Allow SMB-Out","Allow ICMPv4-In" | Select-Object DisplayName, Enabled
```

Expected result:
- expected rules are present
- target rules are enabled in line with the baseline

---

## 17) NetBIOS over TCP/IP

```powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object {$_.IPEnabled} |
Select-Object Description, TcpipNetbiosOptions
```

Expected result:
- `TcpipNetbiosOptions = 2` (NetBIOS disabled)

---

## 18) Multicast Name Resolution

```powershell
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" /v DisableMulticast
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" /v DisableMulticast
Get-Service mdnsresponder -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType
```

Expected result:
- policy key: `DisableMulticast REG_DWORD 0x1`
- global key: `DisableMulticast REG_DWORD 0x1`
- `mdnsresponder` disabled if present

---

## 19) Kerberos Logging

```powershell
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v LogLevel
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v AuditKerberos
wevtutil gl "Microsoft-Windows-Kerberos/Operational"
```

Expected result:
- `LogLevel REG_DWORD 0x1`
- `AuditKerberos REG_DWORD 0x1`
- Operational channel enabled

Note:
- in some cases, a refresh or reboot is needed to observe full consistency

---

## 20) WSUS Configuration

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoRebootWithLoggedOnUsers
```

Important keys to verify:
- `WUServer`
- `WUStatusServer`

Expected result:
- WSUS servers configured correctly
- AU policy aligned with the baseline

---