---
title: "Famous last comments"
categories:
    - gis
tags:
    - gis
    - smallworld
    - magik
    - development
    - 
---

Some months ago an error `_unset does not understand get_dd_table_from_tid()` occured. When investigating it I found the most epic comment I've ever seen in a Magik method:

```magik
_pragma(classify_level=restricted, topic={database,geometry}, usage=internal)
_method gis_ds_view.init_rwo_table_from_def(o, _optional descriptor)
            ##
            ## Initialise rwo info in view and on exemplar for O which is
            ## a record from the sw_gis!rwo_definition table. If DESCRIPTOR
            ## is not supplied then one is created or retrieved from the
            ## cache. 
            ##
            
            rwo_code << o.rwo_code
            rwo_type << o.name.as_symbol()
            _if o.tid > 0
            _then
                        _if descriptor _is _unset 
                        _then descriptor << _self.get_dd_table_from_tid(o.tid,_unset )
                        _endif

                        #
                        # I'm really not sure how descriptor can ever be unset...
                        #
                        _if descriptor _is _unset
                        _then
                                    table_name << rwo_type
                        _else
                                    table_name << descriptor.name
                        _endif
```
(source: SW_PRODUCTS\core\sw_core\modules\sw_core\datastore_base\source\gis_ds_view.magik)

I still laugh my head off whenever I think about it and the backstory.

Somewhere between Smallworld GIS 4.x and 5.2.3 the optional parameter `descriptor` was added. The thing is that a developer had changed this method in the meantime. He added something like

```magik
# start mod
_local t

<do something with t>
# end mod
_if o.tid > 0
_then
            _if descriptor _is _unset 
            _then descriptor << _self.get_dd_table_from_tid(o.tid,_unset )
            _endif

            #
            # I'm really not sure how descriptor can ever be unset...
            #
```

For unknown reasons the same developer changed that to

```magik
# start mod
_local t, descriptor

<do something with t>
# end mod
```

during the upgrade to 5.2.3.

What happens when defining a variable this way? It is unset!

Since I'm a nerd I always think this comment must be printed on a tee.