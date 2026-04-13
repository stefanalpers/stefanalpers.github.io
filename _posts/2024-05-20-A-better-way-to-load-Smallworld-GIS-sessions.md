---
title: "A better way to load Smallworld GIS sessions"
categories:
    - gis
tags:
    - gis
    - smallworld
    - administration
    - magik
---

By now, anyone working with Smallworld 5 should know that using precompiled JAR files significantly improve startup performance. However, the starting point of a session is still the module in CUSTOMER_PRODUCTS\config\magik_sessions, which requires some Magik code to be loaded:

- startup and other procedures
- startup options
- sessions

Depending on the amount of sessions and other definitions we are using this can take some time. But is all this code needed in every session, e.g. in the session the main users are starting for their daily work? Most likely not..

With a structured design of folders and (in this example) gis_alias stanzas as well as some code this issue can be solved.

A structured folder design can be

```
\magik_sessions\source
					\base
						base_sessions.magik
						base_startup_options.magik
						base_startup_procs.magik
						load_list.txt
					\session_snafu
						<magik files>
						load_list.txt
					\session_foobar
						...
					\...
```

Now how do we achieve to load only the code needed? Let's try to create a simple sessions_loader. For example

```magik
_pragma(classify_level=basic, topic={sessions})
def_slotted_exemplar(:sessions_loader,
	{
		{:sessions_directories, _unset, :writable, :private}
	})
$
	
_pragma(classify_level=basic, topic={sessions})
_private _method sessions_loader.init()

	.sessions_directories << rope.new_with("base")

	>> _self
_endmethod
$
```

As you can see the sessions_loader has the slot .session_directories which initially holds a rope that contains "base". Yes, you're right, it represents the folder \magik_sessions\source\base and yes, this folder is always loaded because it contains base definitions.

```magik
_pragma(classify_level=basic, topic={sessions})
_private _method sessions_loader.load_base_sessions

	_self.load_sessions_from_directory("base")
_endmethod
$
```

But how do we get it done to load the code for session_snafu and/or session_foobar? Let's choose gis_alias stanzas because they are already used for starting sessions. We can set an environment_variable SESSIONS_DIRECTORIES with for example session_foobar:

```
session_foobar:
	title				 = FOOBAR session
	session				 = my_product:foobar
	SESSIONS_DIRECTORIES = session_foobar
```
	
During execution our sessions_loader has access to this variable and therefore can load the content of the directory with this method

```magik
_pragma(classify_level=basic, topic={sessions})
_private _method sessions_loader.load_sessions_specified_in_environment_variable
	## The environment variable's name is SESSIONS_DIRECTORIES.
	##
	## It should be defined in the gis_alias.

	_local l_environment_variable << system.getenv("SESSIONS_DIRECTORIES").
					 default("").
					 split_by(%;)

	_for i_directory_name _over l_environment_variable.fast_elements()
	_loop@directory_name
		_self.load_sessions_from_directory(i_directory_name)
	_endloop
	
_endmethod
$
```

If you provide more than one directory separated by ';' all of these directories are loaded.

Both methods packed together yield the one necessary public method 

```magik
_pragma(classify_level=basic, topic={sessions})
_method sessions_loader.load_sessions
	
	_handling warning _with _self.warning_handler
	
	_self.load_base_sessions
	_self.load_sessions_specified_in_environment_variable
_endmethod
$
```

With this simple magik exemplar the content of the above mentioned entry module's `load_list.txt` can be reduced to

```
load_sessions.magik
```

And that file only needs this content:

```magik
#% text_encoding = iso8859_1
_package sw
$

!global_auto_declare?! << _true
$

_block
	sessions_loader.new().load_sessions	
_endblock
$
```

This solution is easy to implement and if you establish and maintain a clear structural and naming pattern it is easy to remember and support. And it saves time — not only for your customers, but for everyone maintaining the environment.