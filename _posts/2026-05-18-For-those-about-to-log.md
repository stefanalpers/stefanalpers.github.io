---
title: "For those about to log"
categories:
    - gis
tags:
    - devops
    - gis
    - java
    - logging
    - magik
    - smallworld
---

- [Logging in Smallworld 5 — What I Got Wrong (and What I Finally Found)](#logging-in-smallworld-5--what-i-got-wrong-and-what-i-finally-found)
  - [What I Overlooked](#what-i-overlooked)
  - [The `simple_logger_plugin`](#the-simple_logger_plugin)
  - [The `msg_logger_plugin`](#the-msg_logger_plugin)
- [What's not available, yet](#whats-not-available-yet)
  - [Loading a configuration from a database](#loading-a-configuration-from-a-database)
  - [Hot-Reload of the configuration during runtime](#hot-reload-of-the-configuration-during-runtime)
  - [Centralised log aggregation over a network](#centralised-log-aggregation-over-a-network)
- [A Side Note on What Might Be Happening Under the Hood](#a-side-note-on-what-might-be-happening-under-the-hood)


# Logging in Smallworld 5 — What I Got Wrong (and What I Finally Found)
 
Back in 2021, in [sw-gis Smallworld Developers](https://groups.io/g/sw-gis) a question was raised about the logging framework in Smallworld 5 and about integrating log4sh in Smallword.
 
The Magik side of the logging framework is documented here (it is not public):
[AppDev - Logging Framework](https://smallworld-gnm.gevernova.com/documentation/sw52/en/swDocs5.htm#../Subsystems/AppDev/Content/Logging/LoggingFramework.htm)
 
I read it and gave an answer, concluding that it didn't seem very flexible. My main concern was that, when using the described classes, everything had to be defined in Magik — no XML configuration like you'd have with log4j or SLF4J. Maybe I got it totally wrong.
 
Since nobody objected or corrected me, and since I never had access to products that would have taught me otherwise, I basically wrote it off:
 
> *"Well, nice try — but why isn't it complete?"*
 
---
 
## What I Overlooked
 
What I definitely overlooked is the second sentence in that article:
 
> *"The implementation of Magik on Java allows access to standard Java logging mechanisms such as SLF4J or log4j. The Smallworld Core logging framework includes a consistent application API, based on log4j APIs, which enables the creation of different types of logger."*
 
And I also missed the existence of `simple_logger_plugin` and `msg_logger_plugin`, mentioned here:
[Core – Logging Configuration](https://smallworld-gnm.gevernova.com/documentation/sw52/en/swDocs5.htm#../Subsystems/Core/Content/Logging/LoggingConfig.htm)
 
---
 
## The `simple_logger_plugin`
 
The `simple_logger_plugin` provides exactly the flexibility I was missing. You add the plugin, its loggers, and their properties to your `plugin_framework`'s `config.xml` — and after activation, the plugin is available inside that framework. No further code needed beyond wiring up the `plugin_framework` itself.
 
```xml
<?xml version="1.0" standalone="yes"?>
<config>
    <plugins>
        <plugin name="my_logger" class_name="simple_logger_plugin">
            <property name="level" value="info"/>
            <loggers>
                <logger name="json"   class="simple_json_log"   level="${LOGGING_LEVEL}" output="!output!"/>
                <!-- for the sake of convenience I use !output! instead of a file -->
                <logger name="file"   class="simple_file_log"   level="${LOGGING_LEVEL}" output="!output!"/>
                <logger name="output" class="simple_output_log" level="${LOGGING_LEVEL}" output="!output!"/>
            </loggers>
        </plugin>
    </plugins>
</config>
```
 
The corresponding `plugin_framework` in Magik is minimal (well, you'll need a module):
 
```magik
#% text_encoding = iso8859_1
 
_package sw
 
def_slotted_exemplar(:my_plugin_framework,
    {},
    :plugin_framework)
$
 
_method my_plugin_framework.config_definition_module_name
    >> _self.module_name
_endmethod
$
 
_method my_plugin_framework.gui_definition_module_name
    >> _self.config_definition_module_name
_endmethod
$
 
_method my_plugin_framework.config_definition_file_name
    >> "my_config.xml"
_endmethod
$
```
 
Activate it and use it:
 
```
Magik> (mp << my_plugin_framework.new(:x)).activate()
a my_plugin_framework(x)
Magik> mp.plugin(:my_logger).log(:info, "WHO", "WHAT")
 
file  2026-05-18T18:11:54.199Z info WHOWHAT <- sad, no out of the box space between
      2026-05-18T18:11:54.199Z info WHOWHAT
{"sw_log":{"session":null,"class":"info","name":"WHO","module":"json"},"json":"WHAT","type":"SWLogData","@timestamp":"2026-05-18T18:11:54.199Z"}
```
 
Three loggers, one call — file, console output, and JSON, all at once. That's actually quite nice.

## The `msg_logger_plugin`

That's a subclass of simple_logger_plugin and it enables you to log in different languages. I haven't stepped further into it, yet.

# What's not available, yet

The underlying Java frameworks provide more. All of the following isn't necessary in a classic Smallworld session but maybe for Geospatial Server?

## Loading a configuration from a database

In a large ESB with many micro-services it is not practicable to host a logging configuration as a file on each server. Instead, appender definitions, log levels and output targets are centralised in a database table. Each service reads its configuration during startup from there. That enables a single point of control without direct server access — an administrator can change the log level for every single service to DEBUG without SSH access or any direct server interaction.

## Hot-Reload of the configuration during runtime

In a 24/7 production system — e.g. an e-commerce platform — a restart because of a configuration change is not acceptable. Logback monitors the configuration file in a background thread and reloads it automatically after a change. A provider can change the log level for a specific module from INFO to DEBUG to examine a live problem — and reset it afterwards — without interrupting a running system.

## Centralised log aggregation over a network

In a distributed environment with dozens of servers — for example in a cloud infrastructure — it is not practicable to write log files on each server individually. Appenders like SMTP (sending critical errors via e-mail to an on-call engineer) or a socket appender (direct forwarding to Elasticsearch/Kibana) collect all logs in a central location. Teams can use a single UI to search logs across all servers and correlate events.
 
---
 
# A Side Note on What Might Be Happening Under the Hood
 
While working on a Java-based logging bridge for Smallworld (turned out to be unnecessary :upside_down_face:), I stumbled across something worth mentioning — though I can't claim to have fully verified all of it.
 
Smallworld's libs folder contains `pax-logging-log4j`, `pax-logging-log`, and a configuration file named `org.ops4j.pax.logging.properties`. That strongly suggests Smallworld uses [PAX Logging](https://github.com/ops4j/org.ops4j.pax.logging) internally.
 
PAX Logging is an OSGi-specific logging bridge. The idea is that it intercepts calls from all common logging APIs — [SLF4J](https://slf4j.org/), [Log4j](https://logging.apache.org/log4j/2.x/index.html) — and routes them through a single unified backend. If that's actually what's happening here, the flow would look something like this:
 
```
Your code  →  logger.info(...)
                    ↓
              SLF4J API
                    ↓
              PAX Logging     ← registered as the SLF4J binding in the OSGi container
                    ↓
              Log4j backend   ← pax-logging-log4j
                    ↓
              Logfile / Console
```
 
Interestingly, SLF4J itself doesn't appear as a standalone JAR in the Smallworld libs. It's possibly embedded inside one of the core bundles — or the documentation's mention of *"access to standard Java logging mechanisms such as SLF4J"* simply means you're free to use it, not that Smallworld ships it. I'm honestly not sure which interpretation is correct.
 