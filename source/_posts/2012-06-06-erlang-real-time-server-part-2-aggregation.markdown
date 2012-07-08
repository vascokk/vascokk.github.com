---
layout: post
title: "Erlang-real-time-server-part-2-aggregation"
date: 2012-06-06 18:39
comments: true
categories: Erang Diameter Riak rebar RiakCore
published: true
---

High level overview of the RTS
---------------------------------------------

In a schematic view, the RTS will look like this:



![](/images/rts-part-2/RTS1.png)


The idea here is: As a result of Diameter request, command on Aggregation module will be invoked (e.g. "accounting()"). As the job, usually performed by this module is computationally intensive, the Aggregation will run on a Riak cluster and the command will be executed within this cluster.

Setting up a Riak cluster (or  "A very lame introduction to Riak")
-------------------------------------------------------------------

Setting up a cluster could be a tedious job. Fortunately, there are always some giant's shoulders you can step on\*. In this case I will use the Ryan Zezeski's RiakCore rebar [templates](https://github.com/rzezeski/rebar_riak_core)

Once you clone the templates repository from GitHub, make a new Erlang application directory in _apps_ and run _rebar_ from it:


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
- Riak is usually deployed on a cluster (set of physical hosts called "nodes")
- each node runs a certain number of "virtual nodes" (or "vnodes")
- the vnode is responsible for a partition of the Riak "Ring" 
- the Ring is a circular, ordered, 160 bit integer space (i.e. integers from 0 to 2^160 placed in circle)
- each element of the Ring represents a hash value computed of the key from a "key-value" pair (in fact it will be computed of the "bucket"+"key", but this is not so important at the moment)
- the number of partitions is always the same, whether you add a new node to the Cluster or take one off (i.e. a bit of a planning is needed upfront)
- the vnode can be used to store data or to perform computations (executing commands)

I'll not go further into details, as [wiki.basho.com](http://wiki.basho.com) is your best source of information, if you want to dive in. I'll just outline the "scalability" thing and how Riak will make our life easier:

As you see in the second bullet from the bottom - the number of partitions is constant. It is set in the app.config's ring_creation_size parameter. Default number is 64 and there is no formal way to calculate it. It would be more or less empirically set by the developer. According to the Riak's wiki - for a mid-sized cluster of 8 to 16 nodes, the number should be between 128 and 512 partitions (always a power of 2), but this is definitely not "set on stone". Let's assume that we have set the optimal ring size already. At some point of time when the system is fully operational, we might need more computational power. Then, what we should do is: 

1. Deploy the application on a new node. 
2. "join" (one-line command) the new one to a randomly chosen node from the cluster (you can really pick any member, there is no "master node"). 
3. ....Wait...there is no 3rd... :)

The new node will automatically claim a certain number of partitions, offloading the other nodes. Under the hood, Riak will use "handoff" procedures, "Gossip" protocol and other magics to do the job. What is important to us is that, due to the "consistent hashing" (more on this in some other article), the relocation of the vnodes will be transparent to the application(s) using the Riak cluster, as all the hash values are still on the same vnodes, even though, some of the vnodes are now residing on a different physical host. Cool, isn't it? The bottom line is: Scaling by adding a new host to the cluster is dead simple, but it will relocate a number of vnodes - not add new ones. So, be careful when you plan your Ring size.



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

and compile:


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


Create release nodes
-----------------------

At this moment _rel_ directory should contain the _aggregation_ release configuration file. Because we have 2 applications (_diaserver_ and _aggregation_) I'll put them in separate _rel_ subdirectories.

First, create _aggregation_ sub-directory and move the content of _rel_ there. Next, edit the second line of reltool.config to match the directory structure:

``` bash
 {lib_dirs, ["../../apps/", "../../apps/aggregation/", "../../deps/"]},

```


Create the diaserver node (the same way as in Part1, only in different directory):

``` bash
mkdir diaserver
cd diaserver
rel/diaserver $ ../../rebar create-node nodeid=diaserver
==> diaserver (create-node)
Writing reltool.config
Writing files/erl
Writing files/nodetool
Writing files/diaserver
Writing files/sys.config
Writing files/vm.args
Writing files/diaserver.cmd
Writing files/start_erl.cmd
```

Edit reltool.config the in the way already described in Part1.

Create 2 aggregation release nodes. They will work on different ports and we will be able to run them on the same physical host. This way we will simulate a 2-host Riak cluster:

``` bash
erlang-rts-part-2/rel/aggregation $ ../../rebar generate target_dir=./dev/dev1 overlay_vars=vars/dev1.config appid=aggregation
==> aggregation (generate)
erlang-rts-part-2/rel/aggregation $ ../../rebar generate target_dir=./dev/dev2 overlay_vars=vars/dev2.config appid=aggregation
==> aggregation (generate)

```

Now, let's test the Riak cluster. Open a terminal and start the first node:

``` bash First console
erlang-rts-part-2/rel/aggregation $ ./dev/dev1/bin/aggregation console

Exec: /home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev1/erts-5.8.4/bin/erlexec -boot /home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev1/releases/1/aggregation -embedded -config /home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev1/etc/app.config -args_file /home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev1/etc/vm.args -- console
Root: /home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev1
Erlang R14B03 (erts-5.8.4) [source] [rq:1] [async-threads:5] [hipe] [kernel-poll:true]


=INFO REPORT==== 21-Jun-2012::21:44:19 ===
    alarm_handler: {set,{{disk_almost_full,"/"},[]}}
** Found 0 name clashes in code paths
21:44:20.142 [info] Application lager started on node 'aggregation1@127.0.0.1'
21:44:20.236 [warning] No ring file available.
21:44:20.237 [info] Application riak_core started on node 'aggregation1@127.0.0.1'
21:44:20.249 [info] Waiting for application aggregation to start (0 seconds).
21:44:20.254 [info] Application aggregation started on node 'aggregation1@127.0.0.1'
Eshell V5.8.4  (abort with ^G)
(aggregation1@127.0.0.1)1> 21:44:20.351 [info] Wait complete for application aggregation (0 seconds)

```

We have the first node 'aggregation1@127.0.0.1' running.

Open a second terminal window, start the second node and join it to the first one:

``` bash Second console
erlang-rts-part-2/rel/aggregation $ ./dev/dev2/bin/aggregation start
erlang-rts-part-2/rel/aggregation $ ./dev/dev2/bin/aggregation-admin join aggregation1@127.0.0.1

Sent join request to aggregation1@127.0.0.1
```

Attach to the running second node and call aggregation:ping() few times. The result in my case is:

``` bash Second console
erlang-rts-part-2/rel/aggregation $ ./dev/dev2/bin/aggregation attach
Attaching to /tmp//home/vasco/w/dia-test/erlang-rts-part-2/rel/aggregation/dev/dev2/erlang.pipe.1 (^D to exit)
(aggregation2@127.0.0.1)1> aggregation:ping().
Ping received by node : 'aggregation2@127.0.0.1'
{pong,479555224749202520035584085735030365824602865664}
(aggregation2@127.0.0.1)2> aggregation:ping().
{pong,639406966332270026714112114313373821099470487552}
(aggregation2@127.0.0.1)3>
```

As you see, the first invocation of ping() is received by the second node: "Ping received by node : 'aggregation2@127.0.0.1'". The second ping(), however, did not print anything. If we open the first console, we'll see:

``` bash First console
21:45:23.394 [info] 'aggregation2@127.0.0.1' joined cluster with status 'valid'
Ping received by node : 'aggregation1@127.0.0.1'
```

Hey, distributed unicorns! :)


Credits
---------
Ryan Zezeski for the Riak [tutorial](https://github.com/rzezeski/try-try-try) and rebar [templates](https://github.com/rzezeski/rebar_riak_core) 


References
------------
(*) ["Standing on the shoulders of giants"](http://en.wikipedia.org/wiki/Standing_on_the_shoulders_of_giants)
