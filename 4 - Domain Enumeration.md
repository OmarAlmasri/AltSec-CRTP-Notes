For enumeration we can use the following tools:
The ActiveDirectory PowerShell module (MS signed and works even in PowerShell CLM)
https://learn.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2022-ps
https://github.com/samratashok/ADModule 

```powershell
Import-Module C:\AD\Tools\ADModule-master\Microsoft.ActiveDirectory.Management.dll Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1 
```


BloodHound (C# and PowerShell Collectors) https://github.com/BloodHoundAD/BloodHound 
PowerView (PowerShell) https://github.com/ZeroDayLab/PowerSploit/blob/master/Recon/PowerView.ps1 
C:\AD\Tools\PowerView.ps1 

SharpView (C#) - Doesn't support filtering using Pipeline https://github.com/tevora-threat/SharpView/

```powershell
# Get current domain
Get-Domain ## (PowerView) 
Get-ADDomain ## (ActiveDirectory Module) 


# Get object of another domain 
Get-Domain -Domain moneycorp.local ## (PowerView)
Get-ADDomain -Identity moneycorp.local ## (ActiveDirectory Module)


# Get domain SID for the current domain 
Get-DomainSID ## (PowerView)
(Get-ADDomain).DomainSID ## (ActiveDirectory Module)


# Get domain policy for the current domain 
Get-DomainPolicyData ## Please Check the Keberos Policy data when forging Tickets manually (not using Rubeus)
(Get-DomainPolicyData).systemaccess 


# Get domain policy for another domain 
(Get-DomainPolicyData -domain moneycorp.local).systemaccess


# Get domain controllers for the current domain 
Get-DomainController ## (PowerView)

Get-ADDomainController ## (ActiveDirectory Module)


# Get domain controllers for another domain 
Get-DomainController -Domain moneycorp.local ## (PowerView)

Get-ADDomainController -DomainName moneycorp.local Discover ## (ActiveDirectory Module)


# Get a list of users in the current domain 
Get-DomainUser ## (PowerView)
Get-DomainUser -Identity student1 ## (PowerView)

Get-ADUser -Filter * -Properties * ## (ActiveDirectory Module)
Get-ADUser -Identity student1 -Properties * ## (ActiveDirectory Module)


# Get list of all properties for users in the current domain
Get-DomainUser -Identity student1 -Properties *  ## (PowerView)
Get-DomainUser -Properties samaccountname,logonCount ## (PowerView)

Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member MemberType *Property | select Name ## (ActiveDirectory Module)

Get-ADUser -Filter * -Properties * | select name,logoncount,@{expression={[datetime]::fromFileTime($_.pwdlastset )}} ## (ActiveDirectory Module)


# Search for a particular string in a user's attributes: 
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description 
## (PowerView)

Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description ## (ActiveDirectory Module)

# Get a list of computers in the current domain 
## (PowerView)
Get-DomainComputer | select Name 
Get-DomainComputer -OperatingSystem "*Server 2022*" 
Get-DomainComputer -Ping 

## (ActiveDirectory Module)
Get-ADComputer -Filter * | select Name 
Get-ADComputer -Filter * -Properties * 
Get-ADComputer -Filter 'OperatingSystem -like "*Server 2022*"' Properties OperatingSystem | select Name,OperatingSystem 
Get-ADComputer -Filter * -Properties DNSHostName | %{Test Connection -Count 1 -ComputerName $_.DNSHostName}

# Get all the groups in the current domain
## (PowerView)
Get-DomainGroup | select Name 
Get-DomainGroup -Domain 
 
## (ActiveDirectory Module)
Get-ADGroup -Filter * | select Name 
Get-ADGroup -Filter * -Properties *
 
# Get all groups containing the word "admin" in group name 
Get-DomainGroup *admin*  ## (PowerView)
Get-ADGroup -Filter 'Name -like "*admin*"' | select Name ## (ActiveDirectory Module)


# Get all the members of the Domain Admins group 
Get-DomainGroupMember -Identity "Domain Admins" -Recurse  ## (PowerView)
Get-ADGroupMember -Identity "Domain Admins" -Recursive ## (ActiveDirectory Module)


# Get the group membership for a user: 
Get-DomainGroup -UserName "student1"  ## (PowerView) 
Get-ADPrincipalGroupMembership -Identity student1 ## (ActiveDirectory Module)


# List all the local groups on a machine (needs administrator privs on non-dc machines) : 
Get-NetLocalGroup -ComputerName dcorp-dc ## (PowerView)


# Get members of the local group "Administrators" on a machine (needs administrator privs on non-dc machines) : 
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators ## (PowerView)


# Get actively logged users on a computer (needs local admin rights on the target) 
Get-NetLoggedon -ComputerName dcorp-adminsrv ## (PowerView)


# Get locally logged users on a computer (needs remote registry on the target - started by-default on server OS) 
Get-LoggedonLocal -ComputerName dcorp-adminsrv ## (PowerView)


# Get the last logged user on a computer (needs administrative rights and remote registry on the target) 
Get-LastLoggedOn -ComputerName dcorp-adminsrv ## (PowerView)


# Find shares on hosts in current domain. 
Invoke-ShareFinder -Verbose 


# Find sensitive files on computers in the domain 
Invoke-FileFinder -Verbose 


# Get all fileservers of the domain 
Get-NetFileServer
```


> [!tip]
> Always check the `LogonCount` for users and machines to check if it's a real user/machine or a decoy


**Domain Admins vs Enterprise Admins:**

| Group                 | Scope  | Privilege Level             | Typical Role                         |
| --------------------- | ------ | --------------------------- | ------------------------------------ |
| **Domain Admins**     | Domain | Full control of one domain  | Manage that domain                   |
| **Enterprise Admins** | Forest | Full control of all domains | Manage entire forest, trusts, schema |

# Domain Enumeration - Shares

For enumerating shares, we can also use **PowerHuntShares** (https://github.com/NetSPI/PowerHuntShares ).

It can discover shares, sensitive files, ACLs for shares, networks, computers, identities etc. and generates a nice HTML report.

```powershell
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

> [!note]
> - The 'servers.txt' in the above command does not include the domain controller for better OPSEC. 
>  - The report also includes 'ShareGraph' that can be used to explore share relationships on your host machine (not on the student VM).


# Domain Enumeration - BloodHound

To make BloodHound collection stealthy, remove noisy collection methods like RDP, DCOM, PSRemote and LocalAdmin. 

Use the `-ExcludeDCs` to avoid detection by MDI: 

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SharpHound\SharpHound.exe args --collectionmethods Group,GPOLocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTarg ets,CertServices --excludedcs 
```

Remember to remove the 'CertServices' collection method when using BloodHound legacy collector.

# Domain Enumeration - SOAPHound

Use SOAPHound for even more stealth.

It talks to Active Driectory Web Services (ADWS - Port 9389) in place of sending LDAP queries - just like the AD Module. 
– Almost no network-based detection (like MDI).
– It retrieves information about all objects `(objectGuid=*)` and then process them. 
It means limited LDAP queries - less chance of endpoint detection

Build a cache that includes basic info about domain objects. 

```powershell
SOAPHound.exe --buildcache -c C:\AD\Tools\cache.txt
```

Collect BloodHound compatible data 

```powershell
SOAPHound.exe -c C:\AD\Tools\cache.txt --bhdump -o C:\AD\Tools\bloodhound-output --nolaps
```

