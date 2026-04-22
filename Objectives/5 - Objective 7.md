**Identify a machine in the target domain where a Domain Admin session is availabele**

```powershell title:Invoke-SessionHunter
Invoke-SessionHunter -NoPortScan -Raw-Results C:\AD\Tools\servers.txt | select Hostname,UserSession,Access
```