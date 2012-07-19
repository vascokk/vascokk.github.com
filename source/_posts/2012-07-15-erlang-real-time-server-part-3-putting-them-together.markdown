---
layout: post
title: "Erlang Real Time Server - Part 3 - Putting Parts Together"
date: 2012-07-15 17:16
comments: true
categories: Erlang Diameter Riak RiakCore
published: true
---

The need for load-balancing
-----------------------

As you have probably noted in [Part 2](http://vas.io/blog/2012/06/06/erlang-real-time-server-part-2-aggregation/) we always send Riak commands to one of the nodes. That doesn't seem to be right, so, what about some load balancing? There are at least two ways to solve this:

- send commands from Diameter Server to Riak cluster in load-balanced fashion, or:
- couple Diameter server with  RiakCore application in one node and send Diameter messages directly to the RiakCluster in a load-balanced fashion. 

If we choose the first approach, we can use HTTP or ProtocolBuffer interface to access Riak from _diaserver_ and use some HTTP/TCP proxy to load-balance the requests (e.g. HAProxy is a good one). The inconvenience here comes from the fact that we have to write a separate HTTP POST requests or implement a PB interface for every Diameter command. (But, this is probably a good idea for a future blog post) 

In the second case we can use a Diameter Relay/Forward agent to distribute Diameter messages among the cluster or (again) a TCP proxy, since one of the possible Diameter transports is TCP. For the sake of simplicity I'll use this approach. 

First of all, as our applications will be running on the same Riak cluster, I'll change the _diaserver_ callback module. At the moment this module is responsible for creating the Diameter Accounting Answer message. Instead, I'll just make a call to aggregation:accounting() function with parameter - the Diameter Accounting Request. 


Change the _accounting()_ function in _aggregation.erl_ module:

``` erlang
accounting(Req) ->
    DocIdx = riak_core_util:chash_key({<<"accounting">>, term_to_binary(now())}),
    PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, aggregation),
    [{IndexNode, _Type}] = PrefList,
    riak_core_vnode_master:sync_spawn_command(IndexNode, {accounting, Req}, aggregation_vnode_master).
```

Don't forget to change also the arity in the export clause to /1.

Next, the handle\_call() in the aggregation\_vnode module:

``` erlang
handle_command({accounting, Req}, _Sender, State) ->
    %%TODO Process accounting request
    Req,
    ResponseCode = ?'DIAMETER_BASE_RESULT-CODE_DIAMETER_SUCCESS',

    {reply, ResponseCode, State};
```

For the sake of the exercise, we are just going to return SUCCESS, but actually this is the place where you should do all the processing of the Diameter request.

The last step is to call aggregation:accounting(Req) from the diameter callback module:

``` erlang server_cb.erl
handle_request(#diameter_packet{msg = Req, errors = []}, _SvcName, {_, Caps})
                  when is_record(Req, diameter_base_ACR) ->

        #diameter_caps{origin_host = {OH,_},
                   origin_realm = {OR,_}}
            = Caps,

            #diameter_base_ACR{'Session-Id' = Id,
                       'Accounting-Record-Type' = RecType,
                               'Accounting-Record-Number' = RecNum,
                           'Acct-Application-Id' = AccAppId }
            = Req,

        ResultCode = aggregation:accounting(Req),

        Ans = #diameter_base_ACA{'Result-Code' = ResultCode, %%?'DIAMETER_BASE_RESULT-CODE_DIAMETER_SUCCESS',
                       'Origin-Host' = OH,
                           'Origin-Realm' = OR,
                           'Session-Id' = Id,
                               'Accounting-Record-Type' = RecType,
                               'Accounting-Record-Number' = RecNum,
                           'Acct-Application-Id' = AccAppId
        },

        {reply, Ans}.

```

Now, what is left is - to put _diaserver_ and _aggregation_ applications together in one erlang "release":


Building the "RTS" node
------------------------

I'm going to the same as I did in the previous post - using the rebar-riak-core templates, but this time I'll use a slightly modified ["multinode template"](https://github.com/vascokk/rebar_riak_core/blob/master/riak_core_multinode_simple.template):

``` bash
$ cd rel
$ mkdir rts
$ cd rts
$ ../../rebar -f create template=riak_core_multinode_simple nodeid=rts target_dir=.

==> rts (create)
Writing ./reltool.config
Writing ./files/erl
Writing ./files/nodetool
Writing ./files/rts
Writing ./files/app.config
Writing ./files/vm.args
Writing .gitignore
Writing Makefile
Writing ./files/rts-admin
Writing ./vars.config
Writing ./vars/dev1.config
Writing ./vars/dev2.config
Writing ./vars/dev3.config

``` 

Now, we have to edit _reltool.config_ and _app.config_:

For _reltool.config_ we need to:
 
- change lib\_dirs with appropriate paths 
- add _diaserver_ and _aggregation_ to the list of application need to be started by this release
- add application specific configuration 

``` bash
{sys, [	
       {lib_dirs, ["../../apps/", "../../deps/"]},
       {rel, "rts", "1",
        [
         kernel,
         stdlib,
         sasl,
         %% add our applications
         aggregation,
         diaserver
        ]},
       {rel, "start_clean", "",
        [
         kernel,
         stdlib
        ]},
       {boot_rel, "rts"},
       {profile, embedded},
       {excl_sys_filters, ["^bin/.*",
                           "^erts.*/bin/(dialyzer|typer)"]},
       {app, sasl, [{incl_cond, include}]},
       %%add application specific configurations
       {app, aggregation, [{incl_cond, include}]},
       {app, diaserver, [{incl_cond, include}]}
      ]}.

{target_dir, "rts"}.

{overlay_vars, "vars.config"}.

{overlay, [
           {mkdir, "data/ring"},
           {mkdir, "log/sasl"},
           {copy, "files/erl", "\{\{erts_vsn\}\}/bin/erl"},
           {copy, "files/nodetool", "\{\{erts_vsn\}\}/bin/nodetool"},
           {template, "files/app.config", "etc/app.config"},
           {template, "files/vm.args", "etc/vm.args"},
           {template, "files/rts", "bin/rts"},
           {template, "files/rts-admin", "bin/rts-admin"}
           ]}.
```

For the _app.config_ we need to configure the TCP port on which diaserver will listen.

{% gist 3146816 %}


I only added the last _diaserver_ tuple. As you see the {{diameter\_port}} is a template parameter. It will be substituted with the value specific for each "dev" node. For this purpose we have 3 "dev" configuration files in the _dev_ directory. Here is the content of the first one:

``` bash
%%
%% etc/app.config
%%
{ring_state_dir,        "data/ring"}.
{web_ip,                "127.0.0.1"}.
{web_port,              "8091"}.
{handoff_port,          "8101"}.
{diameter_port,         "7071"}.

%%
%% etc/vm.args
%%
{node,                  "rts1@127.0.0.1"}.
{cookie,                "rts"}.
```

In this case I put 7071 for Diameter port, for the other 2 nodes the ports will be 7072 and 7073. Using this "dev" configurations will allow us to run 3 (or more if we want) Riak nodes running both _aggregation_ and _diaserver_ applications on the same host.

Compile and build the development release:

``` bash
<project_root>$ make
<project_root>$ make devrel
==> rts (generate)
```

Last thing - we need the  _rel/rts/files/rts-admin_ script to join our Riak nodes once they are started. If you open this file you'll notice that it calls "myapp\_console" application, we have to change this to "aggregation\_console" (see the _app/aggregation/src_ directory).

And we are done with the RTS!

We can now run 3 nodes, each one listening for Diameter messages on a different port. As you remember, the initial idea was to distribute the messages to these nodes via TCP proxy, so let's:


Setup the HAProxy
--------------------

This part is relatively easy. First download and install the HAProxy:

``` bash
$ wget http://haproxy.1wt.eu/download/1.4/src/haproxy-1.4.20.tar.gz

.....

$ tar -xzf haproxy-1.4.20.tar.gz

.....

$ cd haproxy-1.4.20
$ make TARGET=linux26
$ sudo make install
```

The above _make_ has the option _TARGET=linux26_, which might be different in your case. To determine the linux kernel version use "uname -r" command.

If the HAProxy installation is okay, create a configuration file. In our case it looks like this:

``` bash dev.haproxy.config
global
  # specify the maximum connections across the board
  maxconn 2048
  # enable debug output
  debug

# now set the default settings for each sub-section
defaults
  # stick with http traffic
  mode http
  # set the number of times HAProxy should attempt to
  # connect to the target
  retries 3
  # specify the number of connections per front and
  # back end
  maxconn 1024
  # specify some timeouts (all in milliseconds)
  timeout connect 5000

########### Riak Configuration ###################

frontend diaserver
  mode tcp

  # bind to default DIAMETER port 3868
  bind 127.0.0.1:3868

  # Default to the riak cluster configuration
  default_backend riak_cluster

  # timeouts
  timeout client 1200000

# Here is the magic bit which load balances across
# our instances of riak_core which are clustered
# together
backend riak_cluster
  mode tcp
  balance roundrobin
  # timeouts
  timeout server 1200000
  timeout connect 3000

  server rts1 127.0.0.1:7071 check
  server rts2 127.0.0.1:7072 check
  server rts3 127.0.0.1:7073 check

```

The idea here is pretty obvious (right?): our "frontend" (i.e. what the external world will see) is the standard Diameter port 3868. All the TCP packets sent to this port will be forwarded on a round-robin fashion to the "backend", which is our 3 dev release erlang nodes, each of them listening to a different diameter port.

And that's it! We only need to run everything and see how it works:

Open a terminal window and start the first node:

``` bash
$ ./rel/rts/dev/dev1/bin/rts console

19:01:11.204 [info] Application crypto started on node 'rts1@127.0.0.1'
19:01:11.208 [info] Application riak_sysmon started on node 'rts1@127.0.0.1'
19:01:11.239 [info] Application inets started on node 'rts1@127.0.0.1'
19:01:11.243 [info] Application mochiweb started on node 'rts1@127.0.0.1'
** Found 0 name clashes in code paths 
19:01:11.277 [info] Application webmachine started on node 'rts1@127.0.0.1'
19:01:11.311 [info] Application os_mon started on node 'rts1@127.0.0.1'
19:01:11.322 [info] Application folsom started on node 'rts1@127.0.0.1'
19:01:11.470 [info] New capability: {riak_core,vnode_routing} = proxy
19:01:11.471 [info] New capability: {riak_core,staged_joins} = true
19:01:11.473 [info] Application riak_core started on node 'rts1@127.0.0.1'
19:01:11.509 [info] Waiting for application aggregation to start (0 seconds).
19:01:11.515 [info] Application aggregation started on node 'rts1@127.0.0.1'
19:01:11.568 [info] Application diameter started on node 'rts1@127.0.0.1'
19:01:11.575 [info] Starting diameter on IP: {127,0,0,1}
19:01:11.575 [info] Starting diameter on port: 7071
19:01:11.611 [info] Wait complete for application aggregation (0 seconds)
19:01:11.614 [info] Application diaserver started on node 'rts1@127.0.0.1'
Eshell V5.9.1  (abort with ^G)
(rts1@127.0.0.1)1> 
```

Open a new terminal and start the second node and join it to the first one::

``` bash
$ rel/rts/dev/dev2/bin/rts start
$ rel/rts/dev/dev2/bin/rts-admin join rts1@127.0.0.1
Sent join request to rts1@127.0.0.1

....on the first one you should get:

(rts1@127.0.0.1)1> 20:13:44.622 [info] 'rts2@127.0.0.1' joined cluster with status 'valid'

....attach to the second rts console:

$ rel/rts/dev/dev2/bin/rts attach

```

Now, run the HAProxy:

``` bash
 $ sudo haproxy -f dev.haproxy.config -d
Available polling systems :
     sepoll : pref=400,  test result OK
      epoll : pref=300,  test result OK
       poll : pref=200,  test result OK
     select : pref=150,  test result OK
Total: 4 (4 usable), will use sepoll.
Using sepoll() as the polling mechanism.
[WARNING] 200/202744 (10525) : Server riak_cluster/rts3 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 2 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.

```

Run another erlang shell and test with diameter client (see Part 2):

``` bash
Eshell V5.9.1  (abort with ^G)
1> diameter:start().
ok
2> client:start().
ok
3> client:connect(tcp).
{ok,#Ref<0.0.0.50>}
4> client:call_ACR().
{ok,{diameter_base_ACA,"client;1404231385;1;nonode@nohost",
                       2001,"diameter_srv.example.com","example.com",1,1,[],
                       [],[],[],[],[],[],[],[],[],[],[],[]}}
5> 


....you should see the connection request accepted on the console running the HAProxy:

00000000:diaserver.accept(0004)=0005 from [127.0.0.1:33979]
```

If you execute the client:call\_ACR() several times, you should see the request going to the first or the second node. The balance policy is not really a round-robin, because we have another routing layer in RiacCore, which sends the requests to the appropriate Riak partition.

You can run the 3rd node, if you want and here we go - a real-time billing in the cloud :-) Well, we are far, far from it - we need to implement all the Diameter commands and wee  need a backend system to manage our subscribers and tariffs, but at least we just built some foundations for a Diameter-based AAA system. 

Project files are available one GitHub in repository [diameter_test](https://github.com/vascokk/diameter-test)

Happy hacking! :-)


Credits
--------

- OJ Reeves for the HAProxy [mini-tutorial](http://buffered.io/posts/webmachine-erlydtl-and-riak-part-2/)

 
