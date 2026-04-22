# Privilege Escalation - Local
## Services Issues using **PowerUp**

```powershell
# Get services with unquoted paths and a space in their name.
Get-ServiceUnquoted -Verbose

# Get services where the current user can write to its binary path or change arguments to the binary
Get-ModifiableServiceFile -Verbose

# Get the services whose configuration current user can modify.
Get-ModifiableService -Verbose
```

## Run all checks from :

```powershell
# PowerUp 
Invoke-AllChecks 

# Privesc
Invoke-PrivEscCheck 

# PEASS-ng
winPEASx64.exe
```

# Privilege Escalation - Feature Abuse

Like the ability to exploit older versions of **Jenkins** to get privileged access on the system.

# Privilege Escalation - Relaying

- In a relaying attack, the target credentials are not captured. Instead, they are forwarded to a local or remote service or an endpoint for authentication. 
- Two types based on authentication:
	 - NTLM relaying 
	 - Kerberos relaying 
- LDAP and AD CS are the two most abused services for relaying

# Privilege Escalation - GPO Abuse

## GPOddity

- GPOddity combines NTLM relaying and modification of Group Policy Container. 
- By relaying credentials of a user who has WriteDACL on GPO, we can modify the path (gPCFileSysPath) of the group policy template (default is SYSVOL). 
- This enables loading of a malicious template from a location that we control.

![[GPOddity.png]]

