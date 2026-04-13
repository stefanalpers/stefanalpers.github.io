---
title: "Automatic setting of a Smallworld module's version"
categories:
    - gis
tags:
    - gis
    - smallworld
    - git
    - development
---

## What is a Smallworld module?

A Smallworld module is a collection of Magik code and resources like XML, messages, png. It can be loaded into a Smallworld GIS session.

The minimum a Smallworld module needs is a file called module.def. This defines the module's name and its requirements (other modules) and can contain a description.

```
some_gui

description
    A gui for something
end

requires
    some_logic
end
```

And of course it can contain the version of a module. In this case, no version is given explicitly and therefore it is 1.

## Module versions in the past

Prior to Smallworld GIS 5 setting or even updating a module's version had no real benefit or impact. And honestly, nobody — not even Smallworld itself — cared.

Even as I write this, I can't imagine a use case or realistic scenario for two coexisting module versions in a Smallworld GIS environment.

## What has changed?

Smallworld GIS 5 introduced the possibility, sorry, necessity to compile Magik code into Java byte code stored in JAR files. The naming convention for these files is

product_name.module_name.VERSION(!!!).JAR

## Consequences for deployments

Prior to Smallworld GIS 5, Magik code was compiled into RAM and then a so-called image, a memory dump, was saved. During startup the image unfolded into RAM and had no connection to Magik files.

To introduce new functionality, you modified the Magik code. Until you saved a new image, this new functionalities weren't available.

Have you ever tried to delete or overwrite a JAR file deployed to production? Chances are good that it is linked to a Smallworld session and therefore can't be deleted.

Have you ever tried to merge a git test branch into production branch during working hours? Chances are good that the production environment is critically hurt afterwards. Try changing the module's version the next time.

Do you like to merge changes from test to production after working hours or at the weekend? Try changing the module's version the next time.

If the version is different (or better: higher than before), the name of the JAR file is different and you have no problems. More than one JAR file for a module can coexist in given directory. It is only necessary that the module.def contains a version, too. Then the corresponding JAR is loaded.

(Of course you should delete old versions as soon as possible, since too many JARs can have an impact on startup performance).

## How to set a version number?

Just check which module you made changes in, open its module.def and place an arbitrary number after the module's name. It's that simple.

And it's annoying. I'm not aware of any feature in Emacs, MDT or Visual Studio Code that offers a more compelling approach and supports developers in this — in my view — increasingly important step.

## A Solution using git a pre-commit hook

## Git hooks

> **What are they?**
> Scripts Git executes automatically at specific points in your workflow — before or after committing, pushing, or merging. They live in `.git/hooks` and can be written in any scripting language.
>
> **Categories:** `pre-commit`, `pre-push`, `post-merge` (client-side) · `pre-receive`, `post-receive` (server-side)
>
> ### `pre-commit`
> Runs before Git asks for a commit message. Exiting non-zero aborts the commit. Bypass with `--no-verify`.
> Common uses: run linters, run tests, check trailing whitespace, inspect staged snapshot.
>
> **Further reading:** [Git hooks guide](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) · [githooks reference](https://git-scm.com/docs/githooks)