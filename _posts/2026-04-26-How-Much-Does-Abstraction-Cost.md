---
title: "value_object Benchmark – How Much Does Abstraction Cost"
categories:
    - gis
tags:
    - gis
    - magik
    - performance
    - smallworld
---

# value_object Benchmark – How Much Does Abstraction Cost?

In [this post](https://markhing.com/smallworld/beeble-object-a-javascript-like-prototypal-object/) 
Mark Hing introduced `beeble_object` (I changed it to `value_object`) - a JavaScript-inspired prototypal 
object for Smallworld GIS Magik. It opens up interesting possibilities: 
cleaner code, better testability, and easier integration with web services.

But how does it perform compared to the native `rwo_record`?

I ran a benchmark across 203,832 records comparing three data types
- `rwo_record`
- `value_object`
- `property_list`

The first results were sobering — but digging deeper revealed more than a simple 'it's slow' verdict.

You can find the code and the benchmark [here](https://github.com/stefanalpers/SmallworldValueObject).