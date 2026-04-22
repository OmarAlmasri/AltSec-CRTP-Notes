Unlike Linux (Bash), PowerShell handles **Objects** (data structures with properties), not just text. This makes it incredibly powerful but slightly confusing at first.

### 1. The Basics: Variables & Execution

| **Symbol** | **Name**        | **Function**                                | **Example**                                                                  |
| ---------- | --------------- | ------------------------------------------- | ---------------------------------------------------------------------------- |
| **$**      | Variable Prefix | Everything starting with `$` is a variable. | `$user = "student467"`                                                       |
| **`        | `**             | Pipe                                        | Passes the **Output** of the left command as **Input** to the right command. |
| **;**      | Separator       | Runs multiple commands on one line.         | `ipconfig; whoami`                                                           |
| **#**      | Comment         | Text ignored by the script.                 | `# This checks connection`                                                   |

---

### 2. The "Magic" Characters (Shortcuts)

PowerShell has aliases (nicknames) for common commands to make typing faster.

| **Character** | **Real Name**    | **What it does**                                                          | **Example**                                 |
| ------------- | ---------------- | ------------------------------------------------------------------------- | ------------------------------------------- |
| **?**         | `Where-Object`   | **Filters** data. "Keep only the rows where..."                           | `Get-Process                                |
| **%**         | `ForEach-Object` | **Loops**. "Do this for every single item."                               | `Get-Service                                |
| **$_**        | Current Object   | Refers to "The item I am looking at _right now_." Only works inside `{}`. | `...                                        |
| **{}**        | Script Block     | A container for code, used in loops or conditions.                        | `if ($true) { Write-Host "Yes" }`           |
| **()**        | Order of Ops     | Forces commands inside to run first.                                      | `Stop-Process -Id (Get-Process notepad).Id` |

---

### 3. Filtering & Comparison Operators

PowerShell **does not** use `==`, `!=`, or `>` for comparison. It uses hyphens.

| **Operator**     | **Meaning**         | **Example**                                               |
| ---------------- | ------------------- | --------------------------------------------------------- |
| **-eq**          | Equal               | `if ($user -eq "Admin")`                                  |
| **-ne**          | Not Equal           | `if ($status -ne "Running")`                              |
| **-gt / -lt**    | Greater / Less Than | `if ($count -gt 5)`                                       |
| **-like**        | Wildcard Match      | `if ($name -like "*svc*")` (Matches `websvc`, `svc_sql`)  |
| **-match**       | Regex Match         | `if ($output -match "\d{3}")` (Advanced pattern matching) |
| **-not** / **!** | Inverse             | `if (-not $isAdmin)`                                      |

**Real-World Pentest Example:**

```PowerShell
# Get all domain users, FILTER for those with "admin" in the name
Get-DomainUser | ? { $_.samaccountname -match "admin" }
```

---

### 4. Selecting Data (`Select-Object`)

Sometimes a command gives you too much info. Use `select` to pick specific columns.

- **Syntax:** `Command | select Property1, Property2`

- **Example:** You dumped all users but only want their names and last logon times.

```PowerShell
Get-DomainUser | select samaccountname, lastlogon
```

- **Expand:** To extract the raw value (without the header):

```PowerShell
# Returns just the text "S-1-5-21..." instead of a table
Get-DomainUser -Identity student467 | select -ExpandProperty objectsid
```

---

### 5. Redirection (Saving Output)

Similar to Linux, but with a few twists.

| **Symbol** | **Function**        | **Example**                                                          |
| ---------- | ------------------- | -------------------------------------------------------------------- |
| **>**      | Write (Overwrite)   | `Get-Process > processes.txt`                                        |
| **>>**     | Append (Add to end) | `Get-Date >> log.txt`                                                |
| **2>**     | Error Output        | `Get-Item C:\Restricted 2> errors.txt` (Save only errors)            |
| **2>&1**   | Merge Output        | `Script.ps1 > output.txt 2>&1` (Save errors AND output to same file) |
| **$null**  | The Void            | `Command > $null` (Run command but hide all output)                  |

---

### 6. The "Current Object" (`$_`) Explained

This is the most confusing part for beginners.

Imagine the Pipeline `|` is a conveyor belt.

- **`Get-DomainUser`** puts 1,000 users on the belt.

- **`? { ... }`** is a robotic arm that picks up **one user at a time**.

- While the robot is holding that specific user, it calls it **`$_`**.

```PowerShell
Get-ChildItem C:\Users | ? { $_.LastWriteTime -gt (Get-Date).AddDays(-1) }
# "Check the folder. Look at each file ONE BY ONE ($_) and check if ITS ($_) time is recent."
```

### 7. Common Syntax "Gotchas"

1. **Quotes:**
    - `'Single Quotes'`: Literal string. `$var` is **not** replaced.
    - `"Double Quotes"`: Dynamic string. `$var` **is** replaced with its value.

2. **Parenthesis `()` vs Subexpression `$()`**:
    
    - Use `$()` when you need to do math or look up a property _inside_ a string.
    - _Bad:_ `"My ID is $user.Id"` -> Output: "My ID is userobject.Id"
    - _Good:_ `"My ID is $($user.Id)"` -> Output: "My ID is 500"

### Summary One-Liner

```PowerShell
Get-Command | ? { $_.Property -eq "Value" } | select Name, Id | Export-Csv output.csv
# (Get Data) -> (Filter it) -> (Pick columns) -> (Save it)
```

# Useful Commands

```PowerShell title="Active IP Ranges Sweep"
1..254 | % { $subnet = "172.16.$_" $p1 = Test-Connection -ComputerName "$subnet.1" -Count 1 -Quiet $p2 = Test-Connection -ComputerName "$subnet.254" -Count 1 -Quiet if ($p1 -or $p2) { Write-Host "[+] ACTIVE SUBNET FOUND: $subnet.0/24" -ForegroundColor Green } }
```

```PowerShell title="IP Range Ping Scan"
$Octet = 3
1..254 | % {
    $ip = "172.16.$Octet.$_"
    if (Test-Connection -ComputerName $ip -Count 1 -Quiet) {
        Write-Host "$ip is UP" -ForegroundColor Green
    }
}
```

```PowerShell title="All-in-one Script"
# Scans 172.16.X.X entirely
1..254 | % {
    $3rd = $_
    Write-Host "Scanning Subnet 172.16.$3rd.0/24 ..."
    # Generate list of IPs for this subnet
    $ipList = 1..254 | % { "172.16.$3rd.$_" }

    # Launch a background job for the whole subnet
    # ThrottleLimit 32 keeps CPU usage normal
    Test-Connection -ComputerName $ipList -Count 1 -AsJob -ThrottleLimit 32 | Receive-Job -Wait | ? { $_.ResponseTime -ne $null } | select Address, ResponseTime
}
```

