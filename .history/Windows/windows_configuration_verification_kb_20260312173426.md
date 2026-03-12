# KB - Windows Configuration Verification Checklist

Author: Andrea Balconi  
Version: 2.0  
Last Update: 2026-03-12  
Audience: System Administrators / Infrastructure Engineers  
Scope: Verifica manuale della configurazione di hardening Windows Server post-deploy.

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

Questa knowledge base documenta i controlli di verifica manuale da eseguire su server Windows configurati tramite baseline di hardening.

Ogni sezione include:
- comando copiabile
- risultato atteso
- note operative

---

## 2) Prerequisites

Eseguire PowerShell come Administrator.

```powershell
whoami
hostname
(Get-CimInstance Win32_OperatingSystem).Caption
```

Risultato atteso:
- account con privilegi amministrativi
- hostname coerente con naming convention
- sistema operativo previsto

---

## 3) Naming and Host Identity

```powershell
hostname
Get-ComputerInfo | Select-Object CsName
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName' | Select-Object ComputerName
```

Risultato atteso:
- `hostname` e `CsName` coerenti
- eventuale valore differente in registry indica rename pending fino al reboot

---

## 4) Power Plan

```powershell
powercfg /L
powercfg /GETACTIVESCHEME
```

Risultato atteso:
- piano attivo: High performance

---

## 5) Feature Validation (.NET 3.5, Telnet)

```powershell
Get-WindowsOptionalFeature -Online -FeatureName NetFx3
dism /online /Get-FeatureInfo /FeatureName:TelnetClient
```

Risultato atteso:
- NetFx3: `State = Enabled`
- Telnet Client: `State : Enabled`

---

## 6) Time Sync (NTP)

```powershell
w32tm /query /configuration
w32tm /query /status
```

Risultato atteso:
- `Type = NTP`
- `NtpServer` valorizzato con peer atteso (gateway o server custom)

---

## 7) Activation and KMS

```powershell
slmgr /dlv
```

Risultato atteso:
- canale di attivazione coerente
- KMS host e stato attivazione corretti

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

Risultato atteso:
- su cmdlet: `AllowComputerToTurnOffDevice = Disabled`
- fallback registry: `PnPCapabilities = 24`

---

## 9) Locale and Timezone

```powershell
Get-Culture
Get-WinSystemLocale
Get-TimeZone
```

Risultato atteso:
- cultura, system locale e timezone allineati alla baseline

Nota:
- dopo modifica possono servire sign-out/reboot per convergenza completa

---

## 10) IE Enhanced Security Configuration

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
```

Risultato atteso:
- entrambe le chiavi: `IsInstalled REG_DWORD 0x0`

---

## 11) Telemetry

```powershell
Get-Service DiagTrack, dmwappushservice | Select-Object Name, Status, StartType
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection /v AllowDiagnosticData
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection /v AllowDiagnosticData
```

Risultato atteso:
- servizi: `StartType = Disabled`
- entrambe le chiavi: `AllowDiagnosticData REG_DWORD 0x0`

---

## 12) Windows Update Policy

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
Get-CimInstance Win32_Service -Filter "Name='wuauserv'" | Select-Object Name, StartMode
```

Risultato atteso:
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

Risultato atteso:
- `DisabledComponents = 0xFF`
- sugli adapter attivi: `Enabled = False`

Nota:
- dopo la modifica alcuni adapter possono restare temporaneamente `Enabled=True` fino al reboot

---

## 14) Disabled Services Review

```powershell
Get-Service | Where-Object {$_.StartType -eq "Disabled"}
```

Risultato atteso:
- elenco coerente con policy aziendale

---

## 15) LocalAccountTokenFilterPolicy

```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy
```

Risultato atteso:
- valore `1` (UAC remote restrictions disabilitate)

---

## 16) Firewall Rules (SMB/ICMP)

```powershell
Get-NetFirewallRule | Where-Object {$_.DisplayName -match "SMB|ICMP"}
Get-NetFirewallRule -DisplayName "Allow SMB-In","Allow SMB-Out","Allow ICMPv4-In" | Select-Object DisplayName, Enabled
```

Risultato atteso:
- regole previste presenti
- regole target abilitate in linea con baseline

---

## 17) NetBIOS over TCP/IP

```powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object {$_.IPEnabled} |
Select-Object Description, TcpipNetbiosOptions
```

Risultato atteso:
- `TcpipNetbiosOptions = 2` (NetBIOS disabilitato)

---

## 18) Multicast Name Resolution

```powershell
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" /v DisableMulticast
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" /v DisableMulticast
Get-Service mdnsresponder -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType
```

Risultato atteso:
- policy key: `DisableMulticast REG_DWORD 0x1`
- global key: `DisableMulticast REG_DWORD 0x1`
- `mdnsresponder` disabilitato se presente

---

## 19) Kerberos Logging

```powershell
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v LogLevel
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v AuditKerberos
wevtutil gl "Microsoft-Windows-Kerberos/Operational"
```

Risultato atteso:
- `LogLevel REG_DWORD 0x1`
- `AuditKerberos REG_DWORD 0x1`
- canale Operational abilitato

Nota:
- in alcuni casi serve refresh o reboot per vedere piena coerenza

---

## 20) WSUS Configuration

```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoRebootWithLoggedOnUsers
```

Chiavi importanti da verificare:
- `WUServer`
- `WUStatusServer`

Risultato atteso:
- server WSUS valorizzati correttamente
- policy AU coerente con baseline

---