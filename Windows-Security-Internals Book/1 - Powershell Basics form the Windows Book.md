```
file:///X:/MEDIA/BOOKS/WIndows/WindowsSecurityInternals-NEVER-SHARE.pdf
```

## Set up for the exercises on Security Internals

Set the script execution policy, which determines what types of scripts PowerShell can execute. For
Windows clients running PowerShell 5.1, the default is Restricted,
```ps
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
```
More permemnat version of `powershell -ep bypass`

Install a specific Module 
```ps
Install-Module NtObjectManager -Scope CurrentUser -Force
```
James Module for interogating security: Tokens , Descript handles and more 

The easiest approach is to use the PowerShell Gallery (https://www.powershellgallery.com), an online repository of modules.

Once it’s installed, you can load the module using the Import-Module
command:
```
Import-Module NtObjectManager
```


Verify the installation by running a command like :

```
New-NtSecurityDescriptor
```


### Common Operators Table

| Operator | Name                      | Examples                    | Results            |
| -------- | ------------------------- | --------------------------- | ------------------ |
| `+`      | Addition or concatenation | 1 + 2, `"Hello" + "World!"` | 3, `"HelloWorld!"` |
| -        | Subtraction               | 2 - 1                       | 1                  |
| *        | Multiplication            | 2 * 4                       | 8                  |
| /        | Division                  | 8 / 4                       | 2                  |
| %        | Modulus                   | 6 % 4                       | 2                  |
| []       | Index                     | `@(3, 2, 1, 0)[1]`          | 2                  |
| `-f`     | String formatter          | `"0x{0:x} {1}" -f 42, 123`  | `"0x2a 123"`       |
| `-band`  | Bitwise AND               | 0x1FF -band 0xFF            | 255                |
| `-bor`   | Bitwise OR                | 0x100 -bor 0x20             | 288                |
| `-bxor`  | Bitwise XOR               | 0xCC -bxor 0xDD             | 17                 |
| `-bnot`  | Bitwise NOT               | -bnot 0xEE                  | -239               |
| `-and`   | Boolean AND               | `$true -and $false`         | `$false`           |
| `-or`    | Boolean OR                | `$true -or $false`          | `$true`            |
| `-not`   | Boolean NOT               | `-not $true`                | `$false`           |
| `-eq`    | Equals                    | `"Hello" -eq "Hello"`       | `$true`            |
| `-ne`    | Not equals                | `"Hello" -ne "Hello"`       | `$false`           |
| `-lt`    | Less than                 | `4 -lt 10`                  | `$true`            |
| `-gt`    | Greater than              | `4 -gt 10`                  | `$false`           |


There are also some predefined variables we’ll use in the rest of this
book. These variables are:
- `$null`-  Represents the NULL value, which indicates the absence of a value in comparisons
- `$pwd`-  Contains the current working directory Setting Up a PowerShell Testing Environment 7
- `$pid` - Contains the process ID of the shell 
- `$env`-  Accesses the process environment (for example, `$env:WinDir` to get
the Windows directory)


Enumerate all variables using the `Get-Variable` command
```
Get-Variable
```

### Escape Chars 
Example 
```
$message = "Name`tAge`nJeff`t42`nLiz`t38"
Write-Host $message
```

### 🧵 PowerShell String Escape Characters

| Escape   | Meaning                | Example                  | Output                        |
| -------- | ---------------------- | ------------------------ | ----------------------------- |
| `` `0 `` | Null character         | `"Null`0Char"`           | Null in string                |
| `` `a `` | Alert (bell)           | `"Beep!`a"`              | Triggers a beep               |
| `` `b `` | Backspace              | `"abc`b"`                | "ab" (in some cases)          |
| `` `n `` | Newline (line feed)    | `"Hello`nWorld"`         | Hello ↵ World                 |
| `` `r `` | Carriage return        | `"Hello`rWorld"`         | World (overwrites line start) |
| `` `t `` | Horizontal tab         | `"Hello`tWorld"`         | Hello World                   |
| `` `v `` | Vertical tab   ????    | `"Hello`vWorld"`         | Hello World                   |
| ``       | Escape backtick itself | "This is a backtick: ``" | This is a backtick: `         |
| `` `" `` | Double quote inside "" | `"He said,` "Hi!`"`"     | He said, "Hi!"                |
| `` `' `` | Single quote inside '' | `'It`'s fine'`           | It's fine                     |


**String to an Array**
```
"Hello".ToCharArray()
```


### Construct almost any `.NET` type
Use PowerShell to construct almost any .NET type. The simplest
way to do this is to cast a value to that type by specifying the .NET type in
square brackets. When casting, PowerShell will try to find a suitable con-
structor for the type to invoke. Eg; the following command will
convert a string to a `System.Guid` object; PowerShell will find a constructor
that accepts a string and call it:

```
[System.Guid]"6c0a3a17-4459-4339-a3b6-1cdb1b3e8973
```

You can also call a constructor explicitly by calling the new method on
the type. The previous example can be rewritten as follows:

```
[System.Guid]::new("6c0a3a17-4459-4339-a3b6-1cdb1b3e8973")
```

This syntax can also be used to invoke static methods on the type. For
example, the following calls the NewGuid static method to create a new ran-
dom globally unique identifier (GUID):
```
[System.Guid]::NewGuid()
```

You can create new objects too, using the New-Object command:
```
New-Object -TypeName Guid -ArgumentList "6c0a3a17-4459-4339-a3b6-1cdb1b3e8973"
```

### Executing Commands

The  common pattern for most PS cmds: 
- **a verb and a noun, separated by a dash.** eg: `Get-Item`

You can capture the result of an entire command or pipeline into a
variable, then interact with the results. For example, the following captures
the result of the Get-Item command and queries for the FullName property:

```powershell
PS> $var = Get-Item -Path "C:\Windows"
PS> $var.FullName
C:\Windows
```

Or enclose the command in parentheses and directly access its properties and methods:
```
PS> (Get-Item -Path "C:\Windows").FullName
C:\Windows
```

*The length of a command line is effectively infinite. However, you’ll want to try to split up long lines to make the commands more readable. The shell will automatically split a line on the pipe character. If you need to split a long line with no pipes, you can use the backtick character, then start a new line. The backtick must be the last character on the line; otherwise, an error will occur when the script is parsed.*

### Discovering Commands and Getting Help

To help, you can use two built-in commands, `Get-Command` and `Get-Help`

#### Get-Command
In its simplest form, you can execute `Get-Command` without any parameters and it will print all commands from all modules.  It’s more useful to filter on a specific word you’re interested in. For example `Get-Command -Name *SecurityDescriptor*` will list only the commands with the word `SecurityDescriptor` in their names. 

**List the commands available in a module:**
For example, exported by the *NtObject Manager module* and begin with the verb "Start".
```powershell
PS> Get-Command -Module NtObjectManager -Name Start-*
PS> C:\> Get-Command -Module appx -Name Moun*
```
#### Get-Help
Once you’ve found a command that looks promising, you can use the `Get-Help` command to inspect its parameters and get some usage examples.
```
Get-Help Start-NtWait
```

This will show the Name , Short synopsis, syntax and deeper description of the module.

You can also list out detaisl on all parameters 
```powershell
Get-Help Start-NtWait -Parameter *
```

Or a specific one eg the `Object` Parameter: 
```powershell
Get-Help Start-NtWait -Parameter Object
```

If you want usage examples for a command, use the Examples parameter,

```
Get-Help Start-NtWait -Examples
```

You can also see the full help in a pop out window.

```
Get-Help Start-NtWait -ShowWindow
```

### Aliases

List all the current aliases whci hare set:
```
Get-Alias
```

Set a new alias
```
PS> New-Alias -Name swt -Value Start-NtWait
```


# Functions

While it’s not required to use the standard PowerShell command naming convention of a
verb followed by a noun, it pays to do so, as it makes it clear to the user what your function does.

```powershell
PS> function Get-NameValue {
param(
[string]$Name = "",
$Value
)
return "We've got $Name with value $Value"
}

PS> Get-NameValue -Name "Hello" -Value "World"
We've got Hello with value World

PS> Get-NameValue "Goodbye" 12345
We've got Goodbye with value 12345
```
You can omit the return keyword in this case, as PowerShell will return any values uncaptured in variables.


#### Script Block
If you want to run a small block of code without defining a function, you can create a script block. A script block is one or more statements enclosed in braces, `{}`. This block can be assigned to a variable and executed when needed using the `Invoke-Command` command or the `&` operator,
```powershell
PS> $script = {Write-Output "HEllo"}
PS> & $script
HEllo
```

## Displaying and Manipulating Objects
Automatically. the console will use a formatter to display the results, in either a table or a list. 
It’s also possible to specify custom formatters.


For example , the `Get-Process` will default to a table, however we can pipe to the `Select-Object` command and choose the columns we want only.
```
Get-Process | Select-Object Id, ProcessName
```

Some commands including the `Get-Process`
```
Get-Process | Format-List
Get-Process | Format-Table
```


**Find the names of the available properties**
You can use the `Get-Member` command on one of the objects that `Get-Process` returns. 
```
Get-Process | Get-Member -Type Property
```

**Show all the Properties**
Use * as a wildcard to show all properties
```
Get-Process | Format-Table *
```

See all the methods
```
Get-Process | Get-Member -Type Method
```

**Paging Output with `Out-Host`**
If the output from a command is too long to fit on the screen, you can page the output so that only the first part is displayed, and the console will wait for you to press a key before displaying more.
```
Get-Process | Out-Host -Paging
```

## GUI based Table View of things

```
Get-Process | Select-Object * | Out-GridView
```

You can also specify the `-PassThru` parameter to Out-GridView, which causes the command to wait for you to click the OK button in the GUI. Any rows in the view that are selected when you click OK will be written to the command pipeline.

```
Get-Process | Select-Object * | Out-GridView -PassThru
```

### Filtering, Ordering, and Grouping Objects
A traditional shell passes raw text between commands; PowerShell passes objects. Passing objects lets you access individual properties of the objects and trivially filter the pipeline. You can even order and group the objects easily.

**Where-Object**
```
Get-Process | Where-Object ProcessName -EQ "explorer"
Get-Process | Where ProcessName -EQ "explorer"   # "Where" is the alias for it
```


**Common Operators for Where-Object**

| Operator    | Example                      | Description                                   |     |     |     |
| ----------- | ---------------------------- | --------------------------------------------- | --- | --- | --- |
| `-EQ`       | ProcessName -EQ "explorer"   | Equal to the value                            |     |     |     |
| `-NE`       | ProcessName -NE "explorer"   | Not equal to the value                        |     |     |     |
| `-Match`    | ProcessName -Match "ex.*"    | Matches a string against a regular expression |     |     |     |
| `-NotMatch` | ProcessName -NotMatch "ex.*" | Inverse of the -Match operator                |     |     |     |
| `-Like`     | ProcessName -Like "ex*"      | Matches a string against a wildcard           |     |     |     |
| `-NotLike`  | ProcessName -NotLike "ex*"   | Inverse of the -Like operator                 |     |     |     |
| `-GT`       | ProcessName -GT "ex"         | Greater-than comparison                       |     |     |     |
| `-LT`       | ProcessName -LT "ex"         | Less-than comparison                          |     |     |     |



Sorting objects by Handles

```
Get-Process | Where-Object ProcessName -LT "explorer" | Sort-Object
```

```
Get-Process | Sort-Object Handles       # Sort by the Handles
```

```
Get-Process | Sort-Object Handles -Descending     # Sort by the Handles descending order
```

Get rid of Duplicates and fileter by a property name
```
Get-Process | Group-Object ProcessName
```

## Exploring Data

**Writing output to files**
Either of these:
```
Get-Process | Out-File processes.txt
Get-Process > processes.txt        
```

Writing to csv files
```
Get-Process | Select-Object Id, ProcessName | Export-Csv processes.csv -NoTypeInformation
```

We can also write to XML 
```
Get-Process | Select-Object Id, ProcessName | Export-CliXml processes.xml
Get-Content processes.xml
$ps = Import-CliXml processes.xml
$ps[0]
```
Recommended Book : # PowerShell for Sysadmins: Workflow Automation Made Easy Paperback – Illustrated, 6 Feb. 2020

by [Adam Bertram](https://www.amazon.co.uk/Adam-Bertram/e/B083G9JBLL/ref=dp_byline_cont_book_1) (Author)