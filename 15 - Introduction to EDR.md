# MDE - Credential Extraction – LSASS Dump using Custom APIs

MiniDumpDotNet Blog - https://www.whiteoaksecurity.com/blog/minidumpdotnet-part-1/ NanoDump Github: https://github.com/helpsystems/nanodump 
PostDump Github: https://github.com/YOLOP0wn/POSTDump 
Custom MiniDumpWriteDump BOF Github: https://github.com/rookuu/BOFs/tree/main/MiniDumpWriteDump 
ReactOS Source code for MiniDumpWriteDump implementation: https://doxygen.reactos.org/d8/d5d/minidump_8c_source.html 
MiniDumpWriteDump Windows API function: https://learn.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump

## MDE - Credential Extraction – LSASS Dump using Custom APIs - Find LSASS PID

Here is a code snippet of a custom function called **`FindPID`** in C++ to dynamically enumerate the LSASS PID:

```CPP
// Find PID of a process by name 
{ int pid = 0; PROCESSENTRY32 proc = {}; proc.dwSize = sizeof(PROCESSENTRY32); HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0); bool bProc = Process32First(snapshot, &proc); while (bProc) { if (strcmp(procname, proc.szExeFile) == 0) { pid = proc.th32ProcessID; break; } bProc = Process32Next(snapshot, &proc); } return pid; }
```

> [!important]
> Checking for any detections by Windows using **DefenderCheck**

- Downloading tools over HTTP(S) can be risky as it does increase the risk score and chances of detection by the EDR. 
- However, if binaries that are intended for downloads such as Edge (msedge.exe) are available on the target we can perform HTTP(S) downloads without any detections. 
- Another opsec friendly way would be to share files over SMB. Execution can be directly performed from a readable share and is less risky than standard download and execute actions.

## MDE - Lateral Movement - ASR Rules

- MDE correlates detections heavily around Attack Surface Reduction (ASR) rules. 
- ASR rules are configurations that can be applied and customized to reduce the attack surface of a machine. These rules can be customized and referenced with their unique GUIDs. 
- ASR rules are written in **`.lua`** and can be reversed and extracted from a specific target Windows machine.

> [!resources]
> Microsoft ASR rules: https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/attack-surface-reduction?view=o365-worldwide 
> ****
> Gist of ASR rules: https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/attack-surface-reduction-rules-reference?view=o365-worldwide#attack-surface-reduction-rules-by-type

## MDE - Lateral Movement - ASR Rules Bypass

- ASR rules are easy to understand. For example, the **`GetMonitoredLocations`** function displays processes that are monitored and remote execution using them will result in a detection.  
- OS trusted methods like **WMI** and **`Psremoting`** or administrative tools like **`PSExec`** are detected by MDE. 
- To avoid detections based on a specific ASR rule such as the "Block process creations originating from **`PSExec`** and **WMI** commands" rule:
	- We can use alternatives such as **`winrm`** access (**`winrs`**) instead of **`PSExec`**/**WMI** execution (This is undetected by MDE but detected by MDI)
	- Use the **`GetCommandLineExclusions`** function which displays a list of command line exclusions (Ex: `".:\\windows\\ccm\\systemtemp\\.+“` ), if included in the command line will result in bypassing this rule and detection.

```PowerShell
C:\AD\Tools\WSManWinRM.exe eu-sql.eu.eurocorp.local "cmd /c notepad.exe C:\Windows\ccm\systemtemp\"
```

