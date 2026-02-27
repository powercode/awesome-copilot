---
name: powershell-syntax-parser
description: 'Use the PowerShell language parser to troubleshoot script syntax errors. Provides structured access to tokens and parse errors via [System.Management.Automation.Language.Parser]. Use when a user needs to diagnose syntax problems, inspect token streams, or validate PowerShell scripts without executing them.'
---

# PowerShell Syntax Parser Skill

Use the built-in `[System.Management.Automation.Language.Parser]` class to parse PowerShell scripts and extract **tokens** and **parse errors** as structured objects — without executing any code.

## Core API

Two static methods cover all scenarios:

| Method | Use When |
|---|---|
| `[Parser]::ParseInput($code, [ref]$tokens, [ref]$errors)` | Parsing a string of code |
| `[Parser]::ParseFile($path, [ref]$tokens, [ref]$errors)` | Parsing a file on disk |

Both return a `ScriptBlockAst` (the root of the Abstract Syntax Tree) and populate the two `[ref]` output parameters.

## Parsing a String

```powershell
using namespace System.Management.Automation.Language

$code = @'
function Greet-World {
    param($Name)
    Write-Host "Hello, $Name
}
'@

$tokens = $null
$errors  = $null
$ast     = [Parser]::ParseInput($code, [ref]$tokens, [ref]$errors)
```

## Parsing a File

```powershell
using namespace System.Management.Automation.Language

$tokens = $null
$errors  = $null
$ast     = [Parser]::ParseFile('C:\Scripts\MyScript.ps1', [ref]$tokens, [ref]$errors)
```

## Working with Parse Errors

`$errors` is an array of `[ParseError]` objects. Each exposes:

| Property | Type | Description |
|---|---|---|
| `Message` | `string` | Human-readable error description |
| `Extent` | `IScriptExtent` | Source location (line, column, text) |
| `ErrorId` | `string` | Stable identifier for the error kind |
| `IncompleteInput` | `bool` | `$true` when the script is truncated mid-construct |

```powershell
foreach ($error in $errors) {
    [pscustomobject]@{
        Line      = $error.Extent.StartLineNumber
        Column    = $error.Extent.StartColumnNumber
        ErrorId   = $error.ErrorId
        Message   = $error.Message
        Text      = $error.Extent.Text
    }
}
```

### Checking for Errors

```powershell
if ($errors.Count -gt 0) {
    Write-Warning "$($errors.Count) parse error(s) found."
} else {
    Write-Host 'No syntax errors detected.'
}
```

## Working with Tokens

`$tokens` is an array of `[Token]` objects. Useful properties:

| Property | Type | Description |
|---|---|---|
| `Kind` | `TokenKind` | Enum value (e.g. `Command`, `StringLiteral`, `NewLine`) |
| `TokenFlags` | `TokenFlags` | Additional flags (e.g. `CommandName`, `Keyword`) |
| `Text` | `string` | The raw source text of the token |
| `Extent` | `IScriptExtent` | Location in source |

```powershell
# List all tokens as structured objects
$tokens | ForEach-Object {
    [pscustomobject]@{
        Line   = $_.Extent.StartLineNumber
        Column = $_.Extent.StartColumnNumber
        Kind   = $_.Kind
        Flags  = $_.TokenFlags
        Text   = $_.Text
    }
}
```

### Filtering by Token Kind

```powershell
# Find all command names
$tokens | Where-Object { $_.TokenFlags -band [TokenFlags]::CommandName }

# Find all string literals
$tokens | Where-Object { $_.Kind -in 'StringLiteral', 'StringExpandable' }

# Find all keywords
$tokens | Where-Object { $_.TokenFlags -band [TokenFlags]::Keyword }
```

## Reusable Helper Function

```powershell
function Get-ScriptSyntaxInfo {
    <#
    .SYNOPSIS
        Parses PowerShell source code and returns tokens and errors as structured output.
    .PARAMETER Code
        A string containing PowerShell source code to parse.
    .PARAMETER Path
        Path to a PowerShell script file to parse.
    .OUTPUTS
        PSCustomObject with properties: Ast, Tokens, Errors, HasErrors
    #>
    [CmdletBinding(DefaultParameterSetName = 'Code')]
    [OutputType([pscustomobject])]
    param(
        [Parameter(Mandatory, ParameterSetName = 'Code', ValueFromPipeline)]
        [string]$Code,

        [Parameter(Mandatory, ParameterSetName = 'Path')]
        [string]$Path
    )

    using namespace System.Management.Automation.Language

    $tokens = $null
    $errors  = $null

    $ast = switch ($PSCmdlet.ParameterSetName) {
        'Code' { [Parser]::ParseInput($Code, [ref]$tokens, [ref]$errors) }
        'Path' { [Parser]::ParseFile($Path,  [ref]$tokens, [ref]$errors) }
    }

    [pscustomobject]@{
        Ast       = $ast
        Tokens    = $tokens
        Errors    = $errors | ForEach-Object {
            [pscustomobject]@{
                Line            = $_.Extent.StartLineNumber
                Column          = $_.Extent.StartColumnNumber
                ErrorId         = $_.ErrorId
                Message         = $_.Message
                Text            = $_.Extent.Text
                IncompleteInput = $_.IncompleteInput
            }
        }
        HasErrors = $errors.Count -gt 0
    }
}
```

### Usage

```powershell
# Parse inline code
$result = Get-ScriptSyntaxInfo -Code 'function Foo { param($x } '
$result.HasErrors          # True
$result.Errors | Format-Table Line, Column, ErrorId, Message -AutoSize

# Parse a file
$result = Get-ScriptSyntaxInfo -Path '.\MyScript.ps1'
$result.Tokens | Where-Object Kind -eq 'Command' | Select-Object Text -Unique
```

## Common TokenKind Values

| `TokenKind` | Represents |
|---|---|
| `Command` | A command or function name invocation |
| `Parameter` | A parameter name (`-Verbose`) |
| `StringLiteral` | Single-quoted string |
| `StringExpandable` | Double-quoted string |
| `Variable` | A variable reference (`$x`) |
| `Number` | A numeric literal |
| `NewLine` | Line break |
| `Comment` | A `#` comment or `<# ... #>` block |
| `EndOfInput` | End of the script |
| `Keyword` | Language keyword (`if`, `foreach`, `function`, etc.) |

## Key Notes

- Parsing is **safe** — no code is executed.
- `$ast` is a full AST even when `$errors` is non-empty; partial trees are returned on error.
- `IncompleteInput = $true` means the script was cut off (unterminated string, missing closing brace, etc.) rather than containing an outright illegal construct.
- Use `$ast.FindAll({ param($node) $node -is [FunctionDefinitionAst] }, $true)` to walk the AST for specific node types after parsing.
