---
name: powershell-invocation
description: 'Safely invoke pwsh/powershell from any shell (bash, sh, CI runners, or a parent pwsh). Covers variable expansion pitfalls with pwsh -Command, quoting strategies, and using -EncodedCommand as a reliable fallback. Use when a user or AI agent needs to call PowerShell from a shell script, Dockerfile, Makefile, CI pipeline command, parent PowerShell process, or tool-use context where an agent runs pwsh to gather system information.'
---

# Invoking pwsh Safely

> **TL;DR:** Never pass PowerShell code containing `$variables` inside double-quoted strings. The **calling shell** expands `$variables` before `pwsh` sees them, silently breaking your command.

---

## AI Agent Quick Start

**If you are an AI agent running commands in a PowerShell terminal** (the most common case in VS Code / tool-use), just run the commands directly. No wrappers, no child process, no special quoting needed.

### Rule Zero — Write a Script File for Anything Multi-Line

Terminal buffering and paste timing make interactive multi-line commands **unreliable** for AI agents. Lines can be split, reordered, or partially consumed. If your command is more than one or two lines:

1. Write a `.ps1` file
2. Run it with `pwsh -NoProfile -File script.ps1`

```powershell
# FRAGILE — multi-line interactive paste may break under programmatic terminal input
$errors = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile(
    'script.ps1', [ref]$null, [ref]$errors)
if ($errors) { $errors } else { Write-Host 'No syntax errors' }
```

```powershell
# RELIABLE — write to a file, then run it
@'
$errors = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile(
    'script.ps1', [ref]$null, [ref]$errors)
if ($errors) { $errors } else { Write-Host 'No syntax errors' }
'@ | Set-Content -Path check.ps1

pwsh -NoProfile -File check.ps1
```

Single-line commands are fine to run directly. When in doubt, use a file.

### Default — Just Run the Commands

You are already in PowerShell. Type the code:

```powershell
$info = Get-ComputerInfo | Select-Object OsName, OsVersion
$info
```

```powershell
$x = Get-Date
Write-Host "Today is $x"
```

### When to Use a Script Block `& { ... }`

Script blocks are anonymous functions. Use them when you want **scope isolation** — variables created inside the block do not leak into the session:

```powershell
# $tempData exists only inside the block
& {
    $tempData = Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 Name, CPU
    $tempData
}
# $tempData is $null out here
```

Pass data in with `param()` to keep things clean:

```powershell
& {
    param($Path)
    Get-ChildItem -Path $Path -Directory | Select-Object Name
} -Path 'C:\Users\Public'
```

If you don't care about variable leakage (typical for quick one-off commands), skip the script block entirely.

### Common Templates

**Get system information:**
```powershell
[PSCustomObject]@{
    OS      = [System.Runtime.InteropServices.RuntimeInformation]::OSDescription
    PSVer   = $PSVersionTable.PSVersion.ToString()
    DotNet  = [System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
    Host    = [System.Net.Dns]::GetHostName()
}
```

**Check file syntax (PowerShell parser):**
```powershell
$errors = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile('script.ps1', [ref]$null, [ref]$errors)
if ($errors) { $errors } else { Write-Host 'No syntax errors' }
```

**Capture output into a variable:**
```powershell
$result = Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 Name, CPU
$result
```

### What NOT to Do

```powershell
# WRONG — double-quoted string: parent expands $x to empty before child sees it
pwsh -Command "$x = 42; Write-Host $x"
#   → child receives: " = 42; Write-Host " → error or silent blank output

# WRONG — backslash is NOT the escape character in PowerShell (that's bash)
pwsh -Command "\$x = 42; Write-Host \$x"
#   → child receives literal "\$x" → error
```

---

## Common Mistakes by AI Agents

These are the most frequent errors. Scan this table before writing any pwsh invocation.

| Mistake | Why it fails | Fix |
|---|---|---|
| Multi-line interactive paste | Terminal buffering splits or reorders lines unpredictably | Write a `.ps1` file and run with `pwsh -File` |
| `"..."` with `$variables` | Calling shell expands `$var` to empty | Use single quotes or script block |
| `{ ... }` from bash | Bash passes it as a literal string, not a script block | Use single quotes: `pwsh -Command '...'` |
| `\$` in PowerShell | `\` is not the PowerShell escape character | Use `` `$ `` (backtick) or single quotes |
| Forgetting `-Command -` with heredoc | pwsh ignores stdin without it | Always include `-Command -` |
| `@'...'@` inside bash `'...'` | The `'@` closes the bash single quote | Use heredoc or `-EncodedCommand` |
| `$args` from bash with `-Command 'string'` | Extra args are concatenated, not placed in `$args` | Use `-File` with `param()` |
| `-o XML` without `2>&1` | Errors go to stderr as raw CLIXML text | Add `2>&1` to capture error objects |
| `-OutputFormat Text` to "prevent CLIXML" | Plain text is already the default | Remove the flag entirely |

---

## The One Rule

When a child `pwsh` process is launched from **any** shell — bash, sh, zsh, CI, or a parent PowerShell — the **calling shell** processes the command string **first**. If you use double quotes, every `$variable` is expanded by the caller before PowerShell sees it.

```
You type:    pwsh -Command "$x = 42; Write-Host $x"
Parent sees: "$x = 42; Write-Host $x"   ← expands $x (undefined → empty)
Child gets:  " = 42; Write-Host "        ← broken
```

The fix is always the same: **prevent the caller from seeing the `$` characters**. The mechanism differs by shell:

| Calling shell | How to protect `$` |
|---|---|
| **PowerShell** | Script block `& { ... }` or single quotes `'...'` |
| **bash / sh** | Single quotes `'...'` or heredoc `<< 'EOF'` |
| **Any shell** | `-File script.ps1` or `-EncodedCommand` (no string parsing) |

---

## Quick Reference — Safest Pattern Per Shell

| Calling shell | Simplest safe pattern | Example |
|---|---|---|
| **PowerShell** | Script block (no external data) | `& { $x = 42; Write-Host $x }` |
| **PowerShell** | Parameterized script block | `& { param($P) Write-Host $P } -P $value` |
| **bash / sh / zsh** | Single-quoted string | `pwsh -Command '$x = 42; Write-Host $x'` |
| **Any shell** | `-File` with `param()` | `pwsh -File script.ps1 -Name $value` |
| **CI / complex quoting** | `-EncodedCommand` | See [-EncodedCommand](#-encodedcommand--most-reliable-fallback) |

> **Script blocks `{ ... }` only work when the caller is PowerShell.** From bash, `{ ... }` is passed as a literal string — pwsh prints the text instead of executing it.

---

# From PowerShell — All Patterns

## Script Block (Recommended)

Use the **call operator** (`&`) with a **script block**. No quoting or escaping needed:

```powershell
& {
    $errors = $null
    $errors = 'hello'
    Write-Host "errors=$errors"
}
```

For process isolation (different PowerShell version, clean environment), pass the block to `pwsh -Command`:

```powershell
pwsh -NoProfile -Command { $errors = $null; $errors = 'hello'; Write-Host "errors=$errors" }
```

## Single-Quoted String

PowerShell single-quoted strings suppress `$` expansion:

```powershell
pwsh -NoProfile -Command '$errors = $null; $errors = "hello"; Write-Host "errors=$errors"'
```

Embed a literal single quote by doubling it:

```powershell
pwsh -NoProfile -Command '$x = ''it''''s a test''; Write-Host $x'
```

## Here-String Variable

For multi-line scripts, use a single-quoted here-string (`@'...'@`) stored in a variable:

```powershell
$script = @'
$errors = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile('script.ps1', [ref]$null, [ref]$errors)
$errors
'@
pwsh -NoProfile -Command $script
```

## Backtick Escape in Double Quotes

Inside double-quoted strings, escape `$` with backtick (`` ` ``):

```powershell
pwsh -NoProfile -Command "`$errors = `$null; `$errors = `"hello`"; Write-Host `"errors=`$errors`""
```

Becomes unreadable fast — prefer script blocks or single quotes.

## Parameterized Script Block — Same Process

Use `param()` with named arguments to pass data into the block:

```powershell
$serverUrl = 'https://example.com/api?query=hello world&limit=10'
$authToken = "it's a secret token"

& {
    param($Url, $AuthToken)
    Write-Host "Url=$Url"
    Write-Host "Token=$AuthToken"
} -Url $serverUrl -AuthToken $authToken
```

## Parameterized Script Block — Child Process

Use `-args` to pass values positionally to a child `pwsh`:

```powershell
pwsh -NoProfile -Command {
    param($Url, $AuthToken)
    Write-Host "Url=$Url"
    Write-Host "Token=$AuthToken"
} -args $serverUrl, $authToken
```

## `-File` (Works From Any Shell)

Put the code in a `.ps1` file with `param()` and call with `-File`:

```powershell
pwsh -NoProfile -File script.ps1 -Path 'deploy/script.ps1'
```

---

# From Bash / sh / zsh — All Patterns

## Single Quotes (Simplest)

Bash does **not** expand variables inside single-quoted strings:

```bash
pwsh -Command '
    $errors = $null
    $ast = [System.Management.Automation.Language.Parser]::ParseFile("script.ps1", [ref]$null, [ref]$errors)
    $errors
'
```

**Limitation:** You cannot embed a literal single quote inside a bash single-quoted string.

## Escape `$` with Backslash

Inside bash double quotes, prefix every PowerShell `$` with `\`:

```bash
pwsh -Command "\$errors = \$null; [System.Management.Automation.Language.Parser]::ParseFile('script.ps1', [ref]\$null, [ref]\$errors); \$errors"
```

Error-prone as the script grows — prefer single quotes or heredoc.

## Heredoc (Multi-line Scripts)

Use a quoted heredoc delimiter (`'EOF'`) so bash performs no expansion. The `-Command -` flag tells pwsh to read from stdin:

```bash
pwsh -NoProfile -Command - << 'EOF'
$errors = $null
$ast = [System.Management.Automation.Language.Parser]::ParseFile('script.ps1', [ref]$null, [ref]$errors)
$errors
EOF
```

Two rules are critical:
1. **`-Command -`** is required — without it, pwsh ignores stdin.
2. **The closing `EOF` must be at column 0** — no leading whitespace.

### Heredoc inside command substitution

```bash
output=$(pwsh -NoProfile -Command - << 'EOF'
$errors = $null
$ast = [System.Management.Automation.Language.Parser]::ParseFile('script.ps1', [ref]$null, [ref]$errors)
$errors
EOF
)
echo "$output"
```

### Piping to stdin

```bash
echo '$x = 99; Write-Host "x is: $x"' | pwsh -NoProfile -Command -
```

## `-File` with `param()` (Best for Passing Data)

Bash cannot pass script blocks, so use a `.ps1` file with `param()`:

```bash
pwsh -NoProfile -File deploy.ps1 -Url 'https://example.com/api?query=hello world' -AuthToken "it's a secret"
```

```powershell
# deploy.ps1
param($Url, $AuthToken)
Write-Host "Url=$Url"
Write-Host "Token=$AuthToken"
```

> **Why not `$args` from bash?** When `-Command` receives a **string** (the only option from bash), extra arguments are **concatenated** into the command string rather than being placed in `$args`. Use `-File` with `param()` instead.

---

# `-EncodedCommand` — Most Reliable Fallback

`-EncodedCommand` accepts a Base64-encoded UTF-16LE string. Because it is opaque to the outer shell, **no variable expansion or quoting issues can occur**.

## Generate from PowerShell

```powershell
$script = @'
$errors = $null
$null = [System.Management.Automation.Language.Parser]::ParseFile('deploy/script.ps1', [ref]$null, [ref]$errors)
$errors
'@
$encoded = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($script))
pwsh -NoProfile -EncodedCommand $encoded
```

## Generate and use from bash (single pipeline)

```bash
SCRIPT='$errors=$null; $null = [System.Management.Automation.Language.Parser]::ParseFile("deploy/script.ps1",[ref]$null,[ref]$errors); $errors'
ENCODED=$(printf '%s' "$SCRIPT" | iconv -t UTF-16LE | base64 -w0)
pwsh -NoProfile -EncodedCommand "$ENCODED"
```

> **Note:** macOS `base64` does not support `-w0`; use `base64 | tr -d '\n'` instead.

### Why you cannot build the encoded string inside a pwsh single-quoted bash command

```bash
# BROKEN — do not use
ENCODED=$(pwsh -NoProfile -Command '
    $s = "$errors = ..."
    [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($s))
')
```

This fails because:
- Inside a PowerShell **double-quoted** string, `$errors` is still a variable — it expands to empty.
- A **single-quoted here-string** (`@'...'@`) cannot be used because `'@` terminates the surrounding bash single-quoted string.

The `printf | iconv | base64` pattern avoids both problems.

---

# `-OutputFormat` — When to Use It

The default output is **plain text**. You do **not** need `-OutputFormat Text`.

| Flag | From bash / sh | From PowerShell |
|---|---|---|
| *(default)* | Plain text strings | Plain text strings |
| `-o Text` | Plain text strings | Plain text strings |
| `-o XML` | Raw CLIXML text (XML markup) | **Deserialized objects** |

Use `-o XML` when calling `pwsh` from a **parent PowerShell process** and you want typed objects back:

```powershell
# CORRECT — 2>&1 merges stderr so errors come back as ErrorRecord objects
$results = pwsh -NoProfile -o XML -c 'Get-Date; Write-Error "oops"' 2>&1
$results[0].GetType().Name  # DateTime
$results[1].GetType().Name  # ErrorRecord

# WRONG — without 2>&1, errors appear as raw CLIXML text on stderr
$date = pwsh -NoProfile -o XML -c 'Get-Date; Write-Error "oops"'
```

Do not use `-o XML` from bash — the output is raw CLIXML markup, not useful.

---

# Decision Guide

**Two questions** determine the right approach:

**1. What shell is calling pwsh?**

| Caller | Default pattern |
|---|---|
| PowerShell | `& { ... }` |
| bash / sh | `pwsh -Command '...'` (single-quoted) |
| Any shell | `pwsh -File script.ps1` |

**2. Are you passing external data (variables, paths, tokens) into the script?**

| Caller | With external data |
|---|---|
| PowerShell | `& { param($P) ... } -P $value` |
| bash / sh | `pwsh -File script.ps1 -Name $value` |
| Any shell (complex quoting) | `-EncodedCommand` |

---

## Key Notes

- **The problem is the same in bash and PowerShell:** double-quoted strings are interpolated by the **caller** before the child `pwsh` sees them.
- In bash, **single-quoted strings suppress all expansion** — no way to embed a literal `'` inside one.
- In PowerShell, **single-quoted strings suppress `$` expansion** — embed `'` by doubling it (`''`).
- PowerShell **script blocks** `{ ... }` are the safest approach when the caller is PowerShell. They do **not work** from bash.
- The escape character is `\` in bash but `` ` `` (backtick) in PowerShell.
- `-EncodedCommand` requires **UTF-16LE** encoding. UTF-8 base64 produces `ParserError` or garbled output.
- **Always capture the return value of `ParseFile` / `ParseInput`** into `$null`. The uncaptured AST floods the pipeline.
- **`-Command -`** is required when piping or using heredoc to feed stdin.
- **Heredoc closing delimiter must be at column 0** — any leading whitespace breaks it.
- **`pwsh` may not be on PATH** in WSL or Linux CI. Use `which pwsh` to verify.
- `-NoProfile` is recommended in CI to avoid user profile side effects.
- Test with `Write-Host` first to confirm PowerShell receives the string you expect.
