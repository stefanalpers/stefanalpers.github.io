---
title: "No more batch files"
categories:
    - gis
tags:
    - gis
    - smallworld
    - powershell
    - concepts
---
## Introduction into Smallworld's use of batch files

Smallworld is designed to use environment variables available from the operating system. Historically they are defined in BAT files, which are called when starting the GIS.

The environment.bat from the folder SW_PRODUCTS\core\config contains several environment variables that are generally used by Smallworld. Normally this file remains untouched and is called by the customer's environment.bat from the folder CUSTOMER_PRODUCTS\core\config.

At least the variable SMALLWORLD_GIS must be set since many core environment variables depend on it:

```batch
set SW_MESSAGE_DB_DIR=%SMALLWORLD_GIS%\..\smallworld_registry
set SW_GIS_PATTERN_DIR=%SMALLWORLD_GIS%\data\xview_patterns
set SW_FONT_CONFIG=%SMALLWORLD_GIS%\config\font
set SW_CODE_TABLES=%SMALLWORLD_GIS%\data\code_tables
set SW_GIS_TEMPLATE_DIR=%SMALLWORLD_GIS%\data\template
set SW_ACP_PATH=%SMALLWORLD_GIS%\etc\x86
set PATH=%SMALLWORLD_GIS%\bin\x86;%PATH%
```

This is a tpyical call of the core's environment.bat in a customer's environment.bat

```batch
set SMALLWORLD_GIS=<PATH_TO_SW_PRODUCTS>\core
call %SMALLWORLD_GIS%\config\environment.bat
```

And a typical start script for Smallworld may contain something like this:

```batch
echo OFF

%SMALLWORLD_X86%\runalias -e %SW_GIS_ENVIRONMENT_FILE% -a %CONFIG_DIR%\gis_aliases %GIS_SESSION%
```

## The same in powershell

Not long ago, I took the time to transform this to Powershell code and this is the result:

```powershell
Start-Process -FilePath "$env:SMALLWORLD_X86\runalias.exe" -ArgumentList "-e $env:SW_GIS_ENVIRONMENT_FILE -a $env:CONFIG_DIR\gis_aliases $env:GIS_SESSION"
```

In Powershell, environment variables are called by $env:$lt;VARIABLE$gt; and set with $env:$lt;VARIABLE$gt;=$lt;VALUE$gt;

The commandlet Start-Process starts an executable handed over with the parameter -FilePath and with the arguments passed by the paramater -ArgumentList.

## Benefits of using powershell

Maybe you ask yourself:"Fine, that looks <insert your impression here>! But why should I change my scripts?"

At least in my reality the scripts aren't always as simple as the provided examples.

Maybe you want to
- check if certain files or path exists
- use different environment.bat or gis_aliases files.
- map a network drive
- generate a timestamp

Of course all of this can be done using DOS commands. But I never got deeper into them because implementing complex processes with DOS gave me headaches.

I prefer Powershell because it is easier understandable by me. Your mileage may vary, but isn't

```powershell
$env:TIMESTAMP=Get-Date -Format 'yyyyMMdd_HHmmss'
```

easier then

```batch
for /F "usebackq tokens=1,2 delims==" %%i in (`%windir%\system32\wbem\wmic os get LocalDateTime /VALUE 2^>NUL`) do if '.%%i.'=='.LocalDateTime.' set X=%%j
set TIMESTAMP=%X:~2,6%_%X:~8,6%
```

to get the current date and time in a nice format?

The following DOS commands checks if a network drive contains a folder CUSTOMER_PRODUCTS and sets its drive letter as an environment variable.

```batch
for %%a in (z y x w v u t s r q p o n m l k j i h g f e d c) do if exist %%a:\CUSTOMER_PRODUCTS (
	set GIS=%%a:
	goto :network_share_already_mapped
)
```

The same in Powershell:

```powershell
$env:GIS_SERVER_SHARE="\\server\share"

Get-PSDrive -PSProvider filesystem | 
Where-Object -Match -Property DisplayRoot -Value "\\\\*" | # the backslash must be escaped
ForEach-Object {
    If ($_.DisplayRoot -eq $env:GIS_SERVER_SHARE) {
        $env:GIS=$_.Name + ":"
    }
}
```

Ok, mapping a network drive looks easier using DOS:

```batch
FOR /f "tokens=2" %%a IN ('NET USE * "\\%GIS_SHARE%" ^|find "connected" ') do set "GIS=%%a"
```

These lines are needed in Powershell:

```powershell
# map PSDriveRoot to the first free drive letter
If ($env:GIS -eq $false) {
    $PSDriveList=(Get-PSDrive -PSProvider filesystem).Name
    Foreach ($PSDriveLetter in "DEFGHIJKLMNOPQRSTUVWXYZ".ToCharArray()) {
        If ($PSDriveList -notcontains $PSDriveLetter) {
            New-PSDrive -PSProvider filesystem -Name $PSDriveLetter -Root $env:GIS_SERVER_SHARE -Persist -Scope Global
            $env:GIS=$PSDriveLetter + ":"
            break
        }
    }
}
```

But with Powershell you have full and easier access to other features you might need prior or after starting the GIS. Or even after the GIS process is finished!

If you use Start-Process with the parameter -Wait any code afterwards is only executed when the GIS process is finished. I leave to your imaginations why you might need this. Or have a look at this example.

```powershell
$env:GIS_SERVER='production_server'
$env:GIS_SHARE='GIS'
$env:GIS_LOCAL_CLIENT_PATH='C:\GIS_CLIENT'

$env:GIS_SESSION='session'

$env:GIS_SERVER_SHARE="\\$env:GIS_SERVER\$env:GIS_SHARE"
$env:CONFIG_DIR="$env:GIS_SERVER_SHARE\CUSTOMER_PRODUCTS\config"
$env:SW_GIS_ENVIRONMENT_FILE="$env:CONFIG_DIR\environment.switch.bat"
$env:TIMESTAMP=Get-Date -Format 'yyyyMMdd_HHmmss'

& $PSScriptRoot\start_session.pre_processes.ps1

# ---
$env:SMALLWORLD_X86="$env:SW_GIS\SW_PRODUCTS\core\bin\x86"

Start-Process -Wait -FilePath "$env:SMALLWORLD_X86\runalias.exe" -ArgumentList "-e $env:SW_GIS_ENVIRONMENT_FILE -a $env:CONFIG_DIR\gis_aliases $env:GIS_SESSION" #-RedirectStandardOutput C:\temp\GIS.log

& $PSScriptRoot\start_session.post_processes.ps1
```

Thinking further, this approach can lead to a state where only the core environment.bat needs to be passed to the gis.exe or runalias.exe. Or an empty file ;).