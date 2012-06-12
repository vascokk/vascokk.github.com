---
layout: post
title: "erlang-real-time-server-part-2-aggregation"
date: 2012-06-06 18:39
comments: true
categories: Erang Diameter
published: false
---


/erlang-rts-part-2 $ ./rebar -f create template=riak_core_multinode appid=aggregation nodeid=aggregation


We have following files in the ./src directory:

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
- the vnode represents a partition from the Riak "Ring" 
- the Ring is a circular 160 bit integer space [0-2^160]
- key-value pairs are addressed by 160-bit hash values, calculeted of the keys
- every hash value is element of the Ring
- each "partition" (vnode) will take responsibility for part of the Ring space (i.e. if we think of hash values as "addresses", then the vnode will be responsible for part of the 160-bit address space)  


Credits
---------

Ryan Zezeski for the Riak [tutorial](https://github.com/rzezeski/try-try-try) and rebar [templates](https://github.com/rzezeski/rebar_riak_core) 