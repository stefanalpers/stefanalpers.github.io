---
title: "The Magik code compilation process explained"
categories:
    - gis
tags:
    - concepts
    - devops
    - gis
    - magik
    - performance
    - powershell
    - smallworld
---
- [Why should Magik code be compiled into JAR files?](#why-should-magik-code-be-compiled-into-jar-files)
- [What do I need?](#what-do-i-need)
- [The magic of Magik code compilation](#the-magic-of-magik-code-compilation)
  - [smallworld\_product.reinitialise()](#smallworld_productreinitialise)
  - [smallworld\_product.compile\_all\_modules()](#smallworld_productcompile_all_modules)
  - [smallworld\_product.compile\_messages(), smallworld\_product.compile\_modules\_messages()](#smallworld_productcompile_messages-smallworld_productcompile_modules_messages)
  - [smallworld\_product.save\_serialised\_module\_definitions()](#smallworld_productsave_serialised_module_definitions)
- [How do I start the compilation?](#how-do-i-start-the-compilation)
- [A real-world script](#a-real-world-script)

## Why should Magik code be compiled into JAR files?

Well, Java is the running engine of Smallworld GIS and Magik code must therefore be translated into Java byte code.

This can be done during the startup phase of a Smallworld session. But that really slows it down and the result isn't persisted. Every user starts their own compilation.

The better way is to persist the result into JAR files. This must be done only once when Magik code has changed and the result is available for everyone.

During startup Smallworld somehow determines if there are JAR files for the session's modules and loads them directly. If a JAR file is missing, the Magik code is loaded instead.

## What do I need?

You simply register a `magik_session` like this and make sure that is loaded into your session:

```magik
magik_session.register_new(
    "magik2java",
    # Use a session that has most of the Smallworld products that you need already included.
    :parent_session, "nrmb:nrmb_swaf_closed",
    # Add all products that are not included in the parent_session but are needed during the compilation.
    # For example if any module in your CUSTOMER_PRODUCTS changes record exemplars defined in the product :nrm.
    :add_products, { :nrm, :soms, :sw_dxf, _scatter addon_products},
    # this procedure is invoked after the session has been started.
    :post_build_proc, magik2java_proc )
```

Please notice that this session doesn't need any more parameters like `:startup_proc` or `:load_modules`. All the magic is done within the post_build_proc.

The other thing you need is a stanza in your gis_alias file.

## The magic of Magik code compilation

Compilation is by standard only available for a Smallworld product and it is executed with this method `smallworld_product.compile_all_modules()`.

Therefore inside the post_build_proc we need a handle to our smallworld products and execute the method for them.

The following code is an example and includes more method calls that I'll explain after the code block. Every product you want to compile must be known by the Smallworld environment, either using smallworld_registry or environment variables like CUSTOMER_PRODUCTS_DIR.

```magik
magik2java<<
    _proc@magik2java()
    
        _local l_product << _unset
        _local l_products2compile << {:customer_product, ...}
        _local l_customisation_products2compile << {:customisation_product, ...}
        
        _protect
            _handling error _with
            _proc@build_error_handler(p_error)
                write("Compilation of modules failed at ",date_time.now()," with message: ")
                p_error.report_on(!output!)
                !traceback!(!output!)
                write("QUIT session")
            _endproc
            
            _for i_product_name _over l_products2compile.fast_elements()
            _loop
                show(i_product_name)
                l_product << smallworld_product.product(i_product_name)
                l_product.reinitialise()
                l_product.compile_all_modules()
                l_product.compile_messages()
                l_product.compile_module_messages()
                l_product.save_serialised_module_definitions()
            _endloop

            _for i_product_name _over l_customisation_products2compile.fast_elements()
            _loop
                show(i_product_name)
                l_product << smallworld_product.product(i_product_name)
                l_product.save_serialised_module_definitions()
            _endloop
            
        _protection
            write("QUIT session")
        _endprotect
    _endproc
	
_endblock
$
```

### smallworld_product.reinitialise()

If the product is still known to the session this call makes sure that its actual content is known, too.

### smallworld_product.compile_all_modules()

Creates the JAR files in the folder `PRODUCT`\libs. Without further action this can lead to the ["JAR locking problem"](http://sw-gis.wikidot.com/swv-builds#toc9).

### smallworld_product.compile_messages(), smallworld_product.compile_modules_messages()

At least one of them compiles MSG files into MSGC files.

### smallworld_product.save_serialised_module_definitions()

Creates the `product.ser` which contains serialised information about the modules included in the product. This file is really needed if you want a quick start.

For the same reason it is also needed for customisation products that only include MSG, TRN, XML and other resources.

<div class="notice--info" markdown="1">
An important thing to know about these file is that it isn't overwritten when a product is compiled. That can lead to unexpected errors when a new module is added and is required by an already existing module. 

The requirements are part of the product.ser and until it is recreated, new requirements defined in the module.def are not known.
</div>


Of course you can delete the product.ser inside the procedure prior this method call.

## How do I start the compilation?

Just open a command prompt and execute something like this

```batch
runalias.exe -e environment.bat -a gis_aliases magik2java 
```

This is fine for a quick test.

In practice a script is needed that can be executed whenever you want and which contains more actions like solving the JAR locking problem or deleting product.ser.

## A real-world script

```powershell
#region initialise environment
. $PSScriptRoot\..\environment_local_session.ps1
#endregion initialise environment

#region variables
$CustomerProductsDir = "$env:GIS_LOCAL_CLIENT_PATH\CUSTOMER_PRODUCTS"
$AddonProductsDir = "$CustomerProductsDir\..\ADDON_PRODUCTS"
$SWProductsDir = "$CustomerProductsDir\..\SW_PRODUCTS"
$LogDir = "$CustomerProductsDir\..\X_LOGS\COMPILE_PRODUCT"

$CustomerProductsLibsLinkDirectory = "$CustomerProductsDir\libs"
$CustomerProductsLibsTargetDirectory = "$CustomerProductsDir\libs_$env:TIMESTAMP"
#endregion variables

Write-Host "START remove files."
#region Delete JAR and SER in additional products
Remove-Item -Path $AddonProductsDir\example_product_* -Force -Recurse -Include *.jar, product.ser -ErrorAction SilentlyContinue
# IMPORTANT NOTICE FOR READERS:
# Only delete JAR files if you can recreate them by your own.
# Never delete JAR files from partner products or even core products!
#endregion Delete JAR in additional products

#region Delete SER in customer products
Remove-Item -Path $CustomerProductsDir -Force -Recurse -Include product.ser -ErrorAction SilentlyContinue
Remove-Item -Path $CustomerProductsDir\..\CUSTOMISATION_PRODUCTS -Force -Recurse -Include product.ser -ErrorAction SilentlyContinue
#endregion Delete JAR in customer products
Write-Host "END remove files."

#region Create junction
Write-Host "START create junction."
New-Item -Type Directory -Path $CustomerProductsLibsTargetDirectory -Force

If ((Test-Path $CustomerProductsLibsLinkDirectory) -eq $true)
{
    Get-ChildItem $CustomerProductsDir -Attributes ReparsePoint | ForEach-Object { $_.Delete() }
}

New-Item -Path $CustomerProductsLibsLinkDirectory -ItemType Junction -Value $CustomerProductsLibsTargetDirectory
Write-Host "END create junction."
# REMARK FOR READERS:
# This only creates a junction for CUSTOMER_PRODUCTS.
# I haven't yet tried this for products in ADDITIONAL_PRODUCTS.
#endregion Create junction

#region Delete database context
Write-Host "START Delete database context."
# Deleting the database context is never an error. 
# Maybe a database has changed during the deployment of a Smallworld module.
Get-ChildItem -Directory -Path $CustomerProductsDir\.. -Filter SW_DB_CONTEXT_DIR* | ForEach-Object {
	$directoryFullName = $_.FullName
    Remove-Item -Path "$directoryFullName\*" -Exclude .gitignore
}
Write-Host "END Delete database context."
#endregion Delete database context

#region compile code
Write-Host "START magik2java."
Start-Process -FilePath $SWProductsDir\core\bin\x86\runalias.exe -ArgumentList "-e $env:SW_GIS_ENVIRONMENT_FILE -a $CustomerProductsDir\config\gis_aliases magik2java" -Wait -RedirectStandardOutput "$LogDir\1.compile_code.log"
Start-Process notepad++.exe -ArgumentList "$LogDir\1.compile_code.log"
Write-Host "END magik2java."
#endregion compile code

#region build new database context
Write-Host "START build new database context."
Start-Process -FilePath $SWProductsDir\core\bin\x86\runalias.exe -ArgumentList "-e $env:SW_GIS_ENVIRONMENT_FILE -a $CustomerProductsDir\config\gis_aliases build_database_context" -Wait -RedirectStandardOutput "$LogDir\2.build_database_context.log"
Start-Process notepad++.exe -ArgumentList "$LogDir\2.build_database_context.log"
Write-Host "END build new database context."
#endregion build new database context

#region Delete old libs
$MaxFolderAgeInDays = 30
Write-Host $CustomerProductsDir
Get-ChildItem -Path $CustomerProductsDir -Filter libs_* -Directory | ForEach-Object {
    Write-Host $_.Name
    try {
        $FolderDate = [DateTime]::ParseExact($_.Name.Replace("libs_", ""), 'yyyy-MM-dd-HHmm', $null)
        Write-Host $FolderDate
        $FolderAgeInDays = (New-TimeSpan -End $env:TIMESTAMP -Start $FolderDate).Days
        
        If ($FolderAgeInDays -gt $MaxFolderAgeInDays) {
            Remove-Item $_.FullName -Force -Recurse
        }
    }
    catch {
        # no action needed
    }
}
#endregion Delete old libs
```

There is room for improvements. The log files must only be displayed in case of an error. And if an error occurs during compilation, the database context must not be created, yet. And a time stamp can be added to the log files.

Feel free to use it and customise it to your own needs.