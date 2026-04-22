# Find a file share where studentx has Write permissions

```powershell
# Get machines' names
Get-DomainComputer | select -ExpandProperty dnshostname

# Import PowerHuntShares
Import-Module C:\AD\Tools\PowerHuntShares.psm1

# Invoke the script
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

