# List all the computers in the DevOps OU

```powershell
(Get-DomainOU -Identity DevOps).distinguishedname | %{Get-DomainComputer -SearchBase $_} | select name

# % is an alias of ForEach-Object
```

# Enumerate GPO applied on the DevOps OU

```powershell
(Get-DomainOU -Identity DevOps).gplink

Get-DomainGPO -Identity '{blah blah blah}'
```

# Enumerate ACLs for the Applocker and DevOps GPOs

> [!note]
> User **BloodHound** for this
