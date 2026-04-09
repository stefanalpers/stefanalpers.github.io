---
layout: post
title: "Concept for a Smallworld GIS environment that is easy to deploy and maintain"
date: 2026-04-09
categories: gis, smallworld, administration
---

## Preface

A Smallworld GIS environment consists of at least a file server for the Smallworld products, a supported Java runtime and a directory for for the database context, and a database server. A crucial part is the file environment.bat. It contains environment variables for the path to the licence datastore message.ds or to executables like gis.executables

In the lifecycle of a Smallworld version there are specific dates for creating a productive and test environment. Plus environments for development or projects.

If the environment.bat contains absolute paths oder server names, in case of a copy to another location they must be controlled and changed manually. This is prone to errors and takes up unnecessary time. Follow-up operations like changing existing desktop links or moving of databases have to be considered, too.

The motivation of this concept was to facilitate the maintenance of test and productive environment and to create an easy way to deploy a raising number of project and development environments.

## How does the Smallworld GIS work?

Typically a Smallworld session is started via a desktop link, that calls a batch file that contains something like this

```
<PATH>\runalias.exe -e <PATH>\environment.bat -a <PATH>\gis_aliases <GIS-SESSION>
```

The environment.bat in SW_PRODUCTS\core\config contains some mandatory  variables. Normally this file remains untouched and is called in an environment.bat in CUSTOMER_PRODUCTS\config. In this at least the variable SMALLWORLD_GIS must be set since all others are dependent of it.

SW_PRODUCTS\core\config\environment.bat: 
```
set SW_MESSAGE_DB_DIR=%SMALLWORLD_GIS%\..\smallworld_registry
set SW_GIS_PATTERN_DIR=%SMALLWORLD_GIS%\data\xview_patterns
set SW_FONT_CONFIG=%SMALLWORLD_GIS%\config\font
set SW_CODE_TABLES=%SMALLWORLD_GIS%\data\code_tables
set SW_GIS_TEMPLATE_DIR=%SMALLWORLD_GIS%\data\template
set SW_ACP_PATH=%SMALLWORLD_GIS%\etc\x86
set PATH=%SMALLWORLD_GIS%\bin\x86;%PATH%
```

CUSTOMER_PRODUCTS\core\config\environment.bat:
```
set SMALLWORLD_GIS=<PATH>\core
call %SMALLWORLD_GIS%\config\environment.bat
...
```

If &lt;PATH&gt; or &lt;GIS-SESSION&gt; contain absolute paths a server changes becomes a nightmare. If database path isn't controlled by an environment variable, each connected must be checked and changed in the dataset controller.

## Requirements

Experience shows that the following requirements must be met in order to reduce maintenance and distribution costs.

1. Everythnig the GIS needs must be inside one directory and there must exist unified and generally understandable conventions ofr the structure and naming of directories.
2. Changes must be easily comprehensible.
3. If something must be changed, this should be necessary in as few places as possible. These places must be accessed at the least cost.
4. Rolled out desktop links must start only executable files without passing parameters.
5. Permissions for directories and file and network shares must be granted at group level.

## All in one and conventions

### File server
A Smallworld GIS environment contains these directories:

- ADDON_PRODUCTS
- CUSTOMER_PRODUCTS
- CUSTOMISATION_PRODUCTS
- SW_PRODUCTS
- SW_DB_CONteXT_DIR

They must be deployed in a directory on a partition of a server.

If there must be more than one environment on a server, they must be quick and cleary to find and to distinguish. This results in

Conventions I
- GIS enviroments are always on the same partition on all file servers.
- Productive enviroments are called GIS_P.&lt;VERSION&gt;, e. g. GIS_P.52106.
- Test enviroments are called GIS_T.&lt;VERSION&gt;, e. g. GIS_T.52106.
- Use similar conventions for project and development environments.

The need for providing another JRE for different GIS verions result in

Conventions II
- The used Java Runtime lies in the directory JRE that is parallel to the directory SW_PRODUCTS.

Concerning other directories you may use

Conventions III
- Other directories start with "X_" so that they do not slip between the standard folders.

### Database server
As with different environments on the file server there can exist datastores for different GIS versions or uses. These must also be quick and cleary to find and to distinguish, resulting in 

Conventions IV
- Datastores are always in the directory GIS_DB on the same partition on all database servers.
- Inside this directory are directories for the different GIS environments. They are named like the corresponding GIS environment (GIS environment GIS_P.&lt;VERSION&gt; → Datastore GIS_P.&lt;VERSION&gt;)

## Traceability of changes
All GIS environment are tracked by git. All changes must have a comment that is as meaningful as possible and must contain a ticket number if available.

Branches follow

Conventions V
- Branch names are similar to the GIS environment they contain.

In general, this direction applies to the transport of changes through the branches: &lt;DEVELOPMENT/PROJECT&gt; → GIS_T → GIS_P. Only in an emergency may this direction be reversed, and then changes may only be adopted using cherry pick. Since there are usually several changes in GIS_T that cannot be transferred to GIS_P at the same time, a cherry pick of the desired changes is always necessary here.

## Consistent and Meaningful Use of Environment Variables

The frequently mentioned control file environment.bat may only use environment variables that are based on relative specifications, and they must build upon each other. Only then will its potential be fully utilized and maintenance simplified.

There should be as few basic environment variables as possible, and these must be set externally when starting the environment. This encapsulates the GIS environment and makes it easy to transfer to other servers.

The more changes that need to be made, the higher the probability that something will be forgotten or done incorrectly. The points where changes are made should also be as easily accessible as possible, and not nested too deeply in directories.

## Control Over Desktop Shortcuts

The distribution of shortcuts must be controlled and centralized; otherwise, users' desktops will eventually become cluttered and there is a risk that outdated shortcuts will be used.

The shortcuts use UNC paths to call scripts that are located in a specific location and in which the environment variables for the GIS environments are set. The scripts should be built as generically as possible so that they are reusable and easy to maintain.

Conventions VI
- The startup scripts are named start_gis_&lt;p or t&gt;.&lt;version number&gt;.ps1.
- The shortcuts are named "GIS &lt;version number&gt; &lt;P or T&gt;"
- The icons for at least the P and T environments should differ in color and also have different tooltips.

## Implementation in Practice Using the Example of a Customer

All conventions from this concept apply to the GIS environments.

The desktop shortcuts are distributed with Baramundi jobs. No files are copied from A to B; instead, the shortcuts are created with PowerShell commands. Only for project or development environments are there files, which are also provided at a central location.

The scripts called by the shortcuts are located on a server in the GIS-Startskripte directory. Active Directory groups have "Read, Execute" access to this directory so that the scripts can be executed. The directory must also be shared on the network. To prevent unnecessary directories from appearing in users' Windows Explorer, the share name is "GIS-Startskripte$".

For using the GIS, these environment variables must be defined at startup:
- CUSTOMER_GIS_SERVER → File server
- CUSTOMER_GIS_SHARE → GIS environment share on CUSTOMER_GIS_SERVER
- CUSTOMER_GIS_LOCAL_CLIENT_PATH → Directory where the GIS is installed on the client
- CUSTOMER_SW_DB_SERVER → Database server
- CUSTOMER_SW_MESSAGE_DB_SERVER → License server
- CUSTOMER_SW_DB_ENVIRONMENT → Share of the databases to be used on CUSTOMER_SW_DB_SERVER
- GIS_SESSION → The GIS session to be started
- GIS_ALIASES_FILE → The gis_aliases to be used

All other environment variables in environment.bat build upon these.

For example, the shortcut "GIS 52106 P" has this target:
```
C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Unrestricted -command \\SERVER\GIS-Startskripte$\start_gis_p.52106.ps1
```

For example, the shortcut "GIS 52106 T" has this target:
```
C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Unrestricted -command \\SERVER\GIS-Startskripte$\start_gis_t.52106.ps1
```

In the environment-specific script start_gis_p.52106.ps1, the environment variables are set and a generic script is called that invokes all environment-specific scripts.

### Environment-Specific Script
```powershell
$env:CUSTOMER_GIS_SERVER='xxx'
$env:CUSTOMER_GIS_SHARE='GIS_P.52106'
$env:CUSTOMER_GIS_LOCAL_CLIENT_PATH='C:\GIS_CLIENT.52106'

$env:CUSTOMER_SW_DB_SERVER='yyy'
$env:CUSTOMER_SW_MESSAGE_DB_SERVER='zzz'
$env:CUSTOMER_SW_DB_ENVIRONMENT='P.52106'

$env:GIS_SESSION='CUSTOMER_open'
$env:GIS_ALIASES_FILE='gis_aliases'

& $PSScriptRoot\inform_user.ps1

& $PSScriptRoot\start_session.52106.ps1
```

The generic script start_session.52106.ps1 assembles environment variables, calls a file for pre-processes (e.g., connecting a share as a drive), starts the GIS, and after it terminates, calls a file for post-processes (e.g., disconnecting the share).

### Generic Script
```powershell
$env:CUSTOMER_GIS_SERVER_SHARE="\\$env:CUSTOMER_GIS_SERVER\$env:CUSTOMER_GIS_SHARE"
$env:CONFIG_DIR="$env:CUSTOMER_GIS_SERVER_SHARE\CUSTOMER_PRODUCTS\config"
$env:SW_GIS_ENVIRONMENT_FILE="$env:CONFIG_DIR\environment.bat"

$env:TIMESTAMP=Get-Date -Format 'yyyyMMdd_HHmmss'
$env:TODAY=Get-Date -Format 'yyyyMMdd'

& $PSScriptRoot\start_session.pre_processes.52106.ps1

## ---
$env:SMALLWORLD_X86="$env:SW_GIS\SW_PRODUCTS\core\bin\x86"

Start-Process -Wait -FilePath "$env:SMALLWORLD_X86\gis.exe" -ArgumentList "-e $env:SW_GIS_ENVIRONMENT_FILE -a $env:CONFIG_DIR\$env:GIS_ALIASES_FILE $env:GIS_SESSION"

& $PSScriptRoot\start_session.post_processes.52106.ps1
```

To provide users with a new GIS environment that can be started via network and desktop shortcuts, only these steps are necessary (example GIS version 6.6.6):
- Create GIS environment on file server in directory E:\GIS_P.666
- Store databases on database server in directory E:\GIS_DB\GIS_P.666
- Create environment-specific script and set environment variables
- Update and deploy Baramundi job for desktop shortcuts

For administrative purposes, GIS sessions can be started on the server (e.g., source code compilation, opening Emacs). Since drive letters can be used here, the configuration is simplified. In the GIS environment, there is an X_GIS_Sessions directory containing these files:
- environment_local_session.ps1 → Environment variables for the local environment
- start_local_session.ps1 → Calls environment_local_session.ps1 and starts the GIS session

As above, there are scripts in X_GIS-Sessions for specific purposes that call start_local_session.ps1.

### Environment-Specific Script for Emacs
```powershell
$env:GIS_ALIASES_FILE='gis_aliases'
$env:GIS_SESSION='emacs'
$env:SW_SESSION_TYPE='some_session'

& $PSScriptRoot\..\start_local_session.ps1
```

The script for compilation doesn't need to be changed at all, since it doesn't use start_local_session.ps1, but calls environment_local_session.ps1 at the beginning.

### Environment-Specific Script for Compilation
```powershell
#region initialise environment
. $PSScriptRoot\..\environment_local_session.ps1
#endregion initialise environment

#region variables
$CustomerProductsDir = "$env:CUSTOMER_GIS_LOCAL_CLIENT_PATH\CUSTOMER_PRODUCTS"
$AddonProductsDir = "$CustomerProductsDir\..\ADDON_PRODUCTS"
$SWProductsDir = "$CustomerProductsDir\..\SW_PRODUCTS"
$LogDir = "$CustomerProductsDir\..\X_LOGS\CUSTOMER_COMPILE_PRODUCT"

$CustomerProductsLibsLinkDirectory = "$CustomerProductsDir\libs"
$CustomerProductsLibsTargetDirectory = "$CustomerProductsDir\libs_$env:TIMESTAMP"
#endregion variables

Write-Host "START removing files."
##region Delete JAR and SER in additional products
Remove-Item -Path $AddonProductsDir\CUSTOMER_- -Force -Recurse -Include -.jar, product.ser -ErrorAction SilentlyContinue
Remove-Item -Path $AddonProductsDir\its_product\-_gw -Force -Recurse -Include -.jar, product.ser -ErrorAction SilentlyContinue
Remove-Item -Path $AddonProductsDir\ub_product\ub_- -Force -Recurse -Include -.jar, product.ser -ErrorAction SilentlyContinue
Remove-Item -Path $AddonProductsDir\sepm_product\x_translator_gw -Force -Recurse -Include -.jar, product.ser -ErrorAction SilentlyContinue
##endregion Delete JAR in additional products
...
```

### Epilogue

Using this concept, extensive activities would only be necessary in one case: if the location where the scripts for the desktop shortcuts are stored is changed (for example, during a server migration). Then the Baramundi jobs would have to be changed and reassigned. However, if the new server is given the name of the old one, then no follow-up work is necessary.

The concept works generally and must or can be adapted to the customer's circumstances. At one customer, for example, part of the Smallworld environment is currently installed on the clients (see CUSTOMER_GIS_LOCAL_CLIENT_PATH). This doesn't have to be the case with other customers. And this will also change in the near future, as the complete GIS environment except for the databases will then be located on the clients. Then the scripts will also have to be adapted to the new circumstances. The advantage is that there is already a generic script for pre-processes. This will then be used to check the currency of the client installation and update it when changes occur.

I am always amazed at how simple it has become.