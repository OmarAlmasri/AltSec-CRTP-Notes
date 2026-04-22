
```powershell 
# Find all machines on the current domain where the current user has local admin access 
Find-LocalAdminAccess -Verbos

Find-WMILocalAdminAccess.ps1 
# and 
Find-PSRemotingLocalAdminAccess.ps1


# Find computers where a domain admin (or specified user/group) has sessions: 
Find-DomainUserLocation -Verbose 
Find-DomainUserLocation -UserGroupIdentity "RDPUsers" 
# This function queries the DC of the current or provided domain for members of the given group (Domain Admins by default) using 
Get-DomainGroupMember 
# gets a list of computers 
(Get-DomainComputer) 
# and list sessions and logged on users 
(Get-NetSession/Get-NetLoggedon) 
# from each machine.

# Find computers where a domain admin session is available and current user has admin access (uses `Test-AdminAccess`). 

Find-DomainUserLocation -CheckAccess


# Find computers (File Servers and Distributed File servers) where a domain admin session is available.

Find-DomainUserLocation -Stealth
```

List sessions on remote machines ( https://github.com/Leo4j/Invoke-SessionHunter ) 

```powershell
Invoke-SessionHunter -FailSafe

# Above command doesn’t need admin access on remote machines. Uses Remote Registry and queries HKEY_USERS hive.
```

An opsec friendly command would be (avoid connecting to all the target machines by specifying targets)

```powershell
Invoke-SessionHunter -NoPortScan -Targets C:\AD\Tools\servers.txt
```

