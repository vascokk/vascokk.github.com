---
layout: post
title: "erlang-real-time-server-part-2-aggregation"
date: 2012-06-06 18:39
comments: true
categories: Erang Diameter Riak rebar
published: false
---

High level overview of the RTS
---------------------------------------------

In a schematic view, the RTS will look like this:



![](/images/rts-part-2/RTS1.png)


The idea here is: every Diameter request will be turned into a Riak command by the Diameter Server. The command then will be sent to the Aggregation, which as a computational intensive module will run on Riak cluster. 

Setting up a Riak cluster
----------------------------

Setting up a cluster could be kind of tedious. Fortunately, there are always some giant's shoulders you can step on(*). In this case I will use the  Ryan Zezeski's rebar [templates](https://github.com/rzezeski/rebar_riak_core)

Once you clone the templates repository from git hub, what is left to do is:


``` bash
<project_root>$ ./mkdir apps/aggregation
<project_root>$ cd apps/aggregation 
<project_root>/apps/aggregation$ ../../rebar -f create template=riak_core_multinode appid=aggregation nodeid=aggregation
```

and you should have the following files in the _src_ directory:

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

What you are looking at is an application, which will run the Riak's working horse - the vnode. The vnode (virtual node) is the building block of the Riak cluster. Riak itself is heavily influenced by Amazon's Dynamo paper - ["Dynamo: Amazon’s Highly Available Key-value Store"](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf). It is worth reading, but if you are not in an academic mood right now, here are some highlights, you have to know about Riak:

- Riak is a distributed, highly scalable, key-value store
- Riak is usually deployed on a cluster
- a Riak Cluster is run on a set of phisical hosts or "nodes"
- each node runs a certain number of virtual nodes or "vnodes"
- the vnode is responsible for a partition of the Riak "Ring"; 
- the vnode can be used to store data or perform computations ("commands")
- the Ring is a circular ordered 160 bit integer space [0-2^160]
- the elements of the Ring are the hash values computed of the key(in fact they are computed of the pair "bucket"+"key", but this is not so important at the moment)
- the number of partitions is always the same whether you add a new node in the Cluster or take one down (i.e. a bit of a planning is needed upfront)

I'll not go further into details, as [wiki.basho.com](http://wiki.basho.com) is your best source of information, if you want to dive in. I'll just outline the "scalability" thing and how Riak will make our life easier:

As you see in the last bullet - the number of partitions is constant (it is set in the app.config's _ring_creation_size_). Default number is 64. There is no formal way to calculate this number. It would be more or less empirically set. According to the Riak's wiki - for a mid-sized cluster of 8 to 16 nodes, the number should be between 128 and 512 partitions (always a power of 2), but this is definitely not "set on stone". Let's assume that we have set the optimal ring size. What we do, when we need more power is just: run another node and join it to a node from the cluster (you can pick any node, there is no "master). It will automatically claim a certain number of partitions, offloading the other nodes. Due to the "consistent hashing" (more on this in some other article) the relocation of the vnodes will be transparent to the application using our cluster, as all application calls will go to the same vnodes, even though they are now on a different phisical node. Simple as that. Operational cost is minimized.  Under the hood, Riak will use "handoff" procedures, "Gossip" protocol and other magics to do the job.


Credits
---------

Ryan Zezeski for the Riak [tutorial](https://github.com/rzezeski/try-try-try) and rebar [templates](https://github.com/rzezeski/rebar_riak_core) 