
![[ACL_Mindmap.png.png]]


```powershell
# Get the ACLs associated with the specified object 
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs ## (PowerView)


# Get the ACLs associated with the specified prefix to be used for search 
Get-DomainObjectAcl -SearchBase "LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs Verbose ## (PowerView)


# We can also enumerate ACLs using ActiveDirectory module but without resolving GUIDs 
(Get-Acl 'AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local') .Access ## (ActiveDirectory Module)


# Search for interesting ACEs
Find-InterestingDomainAcl -ResolveGUIDs  ## (PowerView)
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "studentx"}  


# Get the ACLs associated with the specified path 
Get-PathAcl -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol
```

