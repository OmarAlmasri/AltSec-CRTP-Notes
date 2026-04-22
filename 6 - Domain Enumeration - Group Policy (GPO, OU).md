Group Policy provides the ability to manage configuration and changes easily and centrally in AD.

**A policy setting may only affect a computer or a user:**
-  **For Computers** - Security settings, startup and shutdown scripts, assigned applications and more.  
- **For Users** - Security settings, logon and logoff scripts, assigned applications and more.

# GPOs

```powershell
# Get list of GPO in current domain. 
Get-DomainGPO 
Get-DomainGPO -ComputerIdentity dcorp-student1


# Get GPO(s) which use Restricted Groups or groups.xml for interesting users 
Get-DomainGPOLocalGroup


# Get users which are in a local group of a machine using GPO 
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1


# Get machines where the given user is member of a specific group 
Get-DomainGPOUserLocalGroupMapping -Identity student1 Verbose
```

# OUs

```powershell
# Get OUs in a domain 
Get-DomainOU 
Get-ADOrganizationalUnit -Filter * -Properties *  


# Get GPO applied on an OU. Read GPOname from gplink attribute from Get-NetOU 
Get-DomainGPO -Identity "{0D1CC23D-1F20-4EEE-AF64 D99597AE2A6E}"
```