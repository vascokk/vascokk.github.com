---
layout: post
title: "Erlang-real-time-server-part-2-aggregation"
date: 2012-06-06 18:39
comments: true
categories: Erang Diameter Riak rebar
published: false
---

High level overview of the RTS
---------------------------------------------

In a schematic view, the RTS will look like this:



![](/images/rts-part-2/RTS1.png)


The idea here is: every Diameter request will be turned by the Diameter Server into a Riak command. The command then will be sent to the Aggregation, which as computational intensive module will run on a Riak cluster. 

Setting up a Riak cluster (plus a very lame introduction to Riak)
-------------------------------------------------------------------

Setting up a cluster could be a tedious job. Fortunately, there are always some giant's shoulders you can step on\*. In this case I will use the Ryan Zezeski's RiakCore rebar [templates](https://github.com/rzezeski/rebar_riak_core)

Once you clone the templates repository from GitHub, make a new erlang application directory in _apps_ and run _rebar_ from it:


``` bash
<project_root> $ ./mkdir apps/aggregation
<project_root> $ cd apps/aggregation 
<project_root>/apps/aggregation $ ../../rebar -f create template=riak_core_multinode appid=aggregation nodeid=aggregation
```

With this run correctly, you should have the following files in the application's _src_ directory:

``` bash
aggregation_app.erl
aggregation.app.src
aggregation_console.erl
aggregation.erl
aggregation.hrl
aggregation_node_event_handler.erl
aggregation_ring_event_handler.erl
aggregation_sup.erl
aggregation_vnode.erl
```

What you are looking at is an application, which will start the Riak's working horse - the vnode. The vnode ("virtual node") is the building block of the Riak cluster. Riak itself is heavily influenced by Amazon's Dynamo paper - ["Dynamo: Amazon’s Highly Available Key-value Store"](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). It is worth reading, but if you are not in an "academic mood" right now, here are some highlights:

- Riak is a distributed, highly scalable, key-value store
- Riak is usually deployed on a cluster (set of phisical hosts called "nodes")
- each node runs a certain number of "virtual nodes" (or "vnodes")
- the vnode is responsible for a partition of the Riak "Ring" 
- the Ring is a circular, ordered, 160 bit integer space (i.e. integers from 0 to 2^160)
- the elements of the Ring are the hash values computed of the key from the "key/value" pair (in fact they are computed of the pair "bucket"+"key", but this is not so important at the moment)
- the number of partitions is always the same, whether you add a new node to the Cluster or take one down (i.e. a bit of a planning is needed upfront)
- the vnode can be used to store data or to perform computations (executing commands)

I'll not go further into details, as [wiki.basho.com](http://wiki.basho.com) is your best source of information, if you want to dive in. I'll just outline the "scalability" thing and how Riak will make our life easier:

As you see in the second bullet from the bootom - the number of partitions is constant. It is set in the app.config's _ring_creation_size_ parameter. Default number is 64 and there is no formal way to calculate it. It would be more or less empirically set by the developer. According to the Riak's wiki - for a mid-sized cluster of 8 to 16 nodes, the number should be between 128 and 512 partitions (always a power of 2), but this is definitely not "set on stone". Let's assume that we have set the optimal ring size already. At some point of time when the system is fully operational, we might need more computational power. Then, what we should do is: 

1. Deploy the application on a new node. 
2. "join" (one-line command) the new one to a randomly chosen node from the cluster (you can really pick any member, there is no "master node"). 
3. ....Wait...there is no other steps... :)

The new node will automatically claim a certain number of partitions, offloading the other nodes. Under the hood, Riak will use "handoff" procedures, "Gossip" protocol and other magics to do the job. What is important to us is that, due to the "consistent hashing" (more on this in some other article), the relocation of the vnodes will be transparent to the application(s) using the Riak cluster, as all the hash values are still on the same vnodes, even though, some of the vnodes are now residing on a different phisical host. Cool, isn't it? The bottom line is: Scaling by adding a new host to the cluster is dead simple, but it will relocate a number of vnodes - not add new ones. So, be careful when you plan your Ring size.



generated rebar.conf:
``` erlang
%% -*- erlang -*-
{sub_dirs, ["rel", "apps/aggregation"]}.
{cover_enabled, true}.
{erl_opts, [debug_info, warnings_as_errors]}.
{edoc_opts, [{dir, "../../doc"}]}.
{deps, [{riak_core, "HEAD",
         {git, "https://github.com/basho/riak_core", {"HEAD"}}}
       ]}.
```

changed rebar.conf:

``` erlang
%% -*- erlang -*-
{sub_dirs, ["rel", "apps/aggregation"]}.
{cover_enabled, true}.
{erl_opts, [debug_info, warnings_as_errors]}.
{edoc_opts, [{dir, "../../doc"}]}.
{deps, [{riak_core, "1.1.2",
         {git, "https://github.com/basho/riak_core", {"1.1.2"}}}
       ]}.
{lib_dirs, ["apps", "deps"]}.
```



``` bash
<project_root>/erlang-rts-part-2 $ ./rebar compile
==> lager (compile)
==> poolboy (compile)
==> protobuffs (compile)
==> basho_stats (compile)
==> riak_sysmon (compile)
==> mochiweb (compile)
==> webmachine (compile)
==> riak_core (compile)
==> rel (compile)
==> aggregation (compile)
Compiled src/aggregation_sup.erl
Compiled src/aggregation_node_event_handler.erl
Compiled src/aggregation_app.erl
Compiled src/aggregation_console.erl
Compiled src/aggregation.erl
Compiled src/aggregation_vnode.erl
Compiled src/aggregation_ring_event_handler.erl
==> erlang-rts-part-2 (compile)

```




Credits
---------
Ryan Zezeski for the Riak [tutorial](https://github.com/rzezeski/try-try-try) and rebar [templates](https://github.com/rzezeski/rebar_riak_core) 


References
------------
(*) ["Standing on the shoulders of giants"](http://en.wikipedia.org/wiki/Standing_on_the_shoulders_of_giants)
