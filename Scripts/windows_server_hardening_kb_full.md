# KB - Windows Server Post-Installation Hardening and Configuration

Author: Andrea Balconi  
Version: 2.0  
Last Update: 2026-03-12  
Audience: System Administrators / Infrastructure Engineers  
Document Type: Operational Hardening KB  
Status Taxonomy: PASS / WARNING / FAIL  
Scope: Procedura manuale di hardening e configurazione baseline per Windows Server post-installazione.

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

Questa knowledge base definisce una sequenza standard di configurazioni post-installazione per:
- uniformare la baseline di sicurezza
- ridurre la superficie di attacco
- abilitare governance centralizzata degli aggiornamenti

Ogni sezione include:
- comando copiabile
- obiettivo tecnico
- verifica rapida

---

## 2) Prerequisites

Eseguire PowerShell come Administrator.

~~~powershell
whoami
hostname
(Get-CimInstance Win32_OperatingSystem).Caption
~~~

Obiettivo:
- confermare privilegi e contesto macchina prima delle modifiche

---

## 3) Rename Server

~~~powershell
$NewComputerName = "SERVER01"
Rename-Computer -NewName $NewComputerName -Force
~~~

Verifica:

~~~powershell
hostname
Get-ComputerInfo | Select-Object CsName
~~~

Risultato atteso:
- nome host coerente con naming convention

Nota:
- reboot richiesto per applicazione completa

---

## 4) Power Plan

~~~powershell
$plan = Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object { $_.ElementName -eq "High performance" }

$plan.Activate()
~~~

Verifica:

~~~powershell
Get-WmiObject -Namespace root\cimv2\power -Class Win32_PowerPlan |
Where-Object { $_.IsActive -eq $true }
~~~

Risultato atteso:
- piano attivo impostato su High performance

---

## 5) Install .NET Framework 3.5

~~~powershell
Install-WindowsFeature NET-Framework-Core
~~~

Opzione offline:

~~~powershell
Install-WindowsFeature NET-Framework-Core -Source D:\sources\sxs
~~~

Verifica:

~~~powershell
Get-WindowsFeature NET-Framework-Core
~~~

Risultato atteso:
- feature installata correttamente

---

## 6) Install Telnet Client

~~~powershell
Install-WindowsFeature Telnet-Client
~~~

Verifica:

~~~powershell
Get-WindowsFeature Telnet-Client
~~~

Risultato atteso:
- Telnet Client presente e installato

---

## 7) Configure NTP

~~~powershell
$ntp = "192.168.1.1"

w32tm /config /manualpeerlist:$ntp /syncfromflags:manual /reliable:yes /update
w32tm /resync
~~~

Verifica:

~~~powershell
w32tm /query /configuration
w32tm /query /status
~~~

Risultato atteso:
- source NTP corretta e sincronizzazione attiva

---

## 8) Windows Activation (KMS)

~~~powershell
$productkey = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$kms = "kms01.arubacloud.com:12688"

cscript C:\Windows\System32\slmgr.vbs /ipk $productkey
cscript C:\Windows\System32\slmgr.vbs /skms $kms
cscript C:\Windows\System32\slmgr.vbs /ato
~~~

Verifica:

~~~powershell
slmgr /dlv
~~~

Risultato atteso:
- activation channel e KMS host valorizzati correttamente

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

Verifica:

~~~powershell
Get-NetAdapterPowerManagement
~~~

Risultato atteso:
- opzioni di power-saving non attive sugli adapter operativi

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

Configurazione layout tastiera:

~~~powershell
$lang = New-WinUserLanguageList it-IT
$lang[0].InputMethodTips.Clear()
$lang[0].InputMethodTips.Add("0410:00000410")
Set-WinUserLanguageList $lang -Force
~~~

Verifica:

~~~powershell
Get-Culture
Get-WinSystemLocale
Get-TimeZone
~~~

Risultato atteso:
- lingua, locale e timezone allineati alla baseline

Nota:
- alcune impostazioni convergono completamente dopo logoff o reboot

---

## 11) Disable IE Enhanced Security Configuration

~~~powershell
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" IsInstalled 0
~~~

Verifica:

~~~powershell
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
reg query "HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}" /v IsInstalled
~~~

Risultato atteso:
- entrambe le chiavi con valore 0

---

## 12) Disable Telemetry

~~~powershell
Stop-Service DiagTrack -Force
Set-Service DiagTrack -StartupType Disabled

Stop-Service dmwappushservice -Force
Set-Service dmwappushservice -StartupType Disabled
~~~

Configurazione registry:

~~~powershell
New-Item "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Force
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Force

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" AllowDiagnosticData 0
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" AllowDiagnosticData 0
~~~

Verifica:

~~~powershell
Get-Service DiagTrack, dmwappushservice | Select-Object Name, Status, StartType
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection /v AllowDiagnosticData
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection /v AllowDiagnosticData
~~~

Risultato atteso:
- servizi disabilitati
- AllowDiagnosticData impostato a 0 in entrambe le chiavi

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

Verifica:

~~~powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
Get-CimInstance Win32_Service -Filter "Name='wuauserv'" | Select-Object Name, StartMode
~~~

Risultato atteso:
- policy AU applicata e servizio Windows Update in Manual

---

## 14) Disable IPv6

~~~powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" -Force
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\TCPIP6\Parameters" DisabledComponents 255
~~~

Disabilitazione binding:

~~~powershell
Get-NetAdapter |
Where-Object { $_.Status -eq "Up" } |
Disable-NetAdapterBinding -ComponentID ms_tcpip6 -Confirm:$false
~~~

Verifica:

~~~powershell
reg query HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters /v DisabledComponents
Get-NetAdapterBinding -ComponentID ms_tcpip6 | Select-Object Name, Enabled
~~~

Risultato atteso:
- DisabledComponents a 255
- binding IPv6 disabilitato sugli adapter target

---

## 15) Disable Unnecessary Services

Esempio:

~~~powershell
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled

Stop-Service iphlpsvc -Force
Set-Service iphlpsvc -StartupType Disabled
~~~

Verifica:

~~~powershell
Get-Service | Where-Object { $_.StartType -eq "Disabled" }
~~~

Risultato atteso:
- servizi non necessari disabilitati in base alla policy aziendale

---

## 16) Disable UAC Remote Restrictions

~~~powershell
New-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Force
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" LocalAccountTokenFilterPolicy 1
~~~

Verifica:

~~~powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy
~~~

Risultato atteso:
- LocalAccountTokenFilterPolicy impostato a 1

---

## 17) Firewall Baseline Rules

~~~powershell
New-NetFirewallRule -DisplayName "Allow SMB-In" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow SMB-Out" -Direction Outbound -Protocol TCP -RemotePort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Direction Inbound -Protocol ICMPv4 -Action Allow
~~~

Verifica:

~~~powershell
Get-NetFirewallRule | Where-Object DisplayName -Match "SMB|ICMP"
Get-NetFirewallRule -DisplayName "Allow SMB-In","Allow SMB-Out","Allow ICMPv4-In" | Select-Object DisplayName, Enabled
~~~

Risultato atteso:
- regole create e abilitate

---

## 18) Disable NetBIOS

~~~powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object { $_.IPEnabled -eq $true } |
ForEach-Object {
    Invoke-CimMethod -InputObject $_ -MethodName SetTcpipNetbios -Arguments @{ TcpipNetbiosOptions = 2 }
}
~~~

Verifica:

~~~powershell
Get-CimInstance Win32_NetworkAdapterConfiguration |
Where-Object { $_.IPEnabled } |
Select-Object Description, TcpipNetbiosOptions
~~~

Risultato atteso:
- TcpipNetbiosOptions impostato a 2

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

Verifica:

~~~powershell
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" /v DisableMulticast
reg query "HKLM\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" /v DisableMulticast
Get-Service mdnsresponder -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType
~~~

Risultato atteso:
- multicast name resolution disabilitata da policy e global key

---

## 20) Enable Kerberos Logging

~~~powershell
New-Item "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" -Force

Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters" LogLevel 1
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" AuditKerberos 1
~~~

Abilitazione operational log:

~~~powershell
wevtutil set-log Microsoft-Windows-Kerberos/Operational /enabled:true
~~~

Verifica:

~~~powershell
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v LogLevel
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v AuditKerberos
wevtutil gl "Microsoft-Windows-Kerberos/Operational"
~~~

Risultato atteso:
- LogLevel e AuditKerberos impostati a 1
- canale Operational abilitato

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

Verifica:

~~~powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v AUOptions
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoUpdate
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate /v DisableWindowsUpdateAccess
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU /v NoAutoRebootWithLoggedOnUsers
~~~

Risultato atteso:
- chiavi WUServer e WUStatusServer valorizzate
- policy di aggiornamento allineata al modello manuale governato

---

## 22) Post-Change Verification

Checklist finale consigliata:
1. Riavviare il server se presenti modifiche pending.
2. Eseguire la KB di verifica configurazioni.
3. Salvare output e screenshot come evidenza di compliance.
4. Aprire ticket di remediation per eventuali controlli non conformi.

Classificazione esiti:
- PASS: allineato alla baseline
- WARNING: parzialmente allineato o in stato transitorio
- FAIL: non allineato o configurazione mancante

---

## 23) Evidence Collection Template

Campi minimi consigliati per audit/compliance:
- ServerName
- Data/Ora intervento
- Operatore
- Configurazione applicata
- Evidenza comando/output
- Esito (PASS/WARNING/FAIL)
- Note e change/ticket ID
