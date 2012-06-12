---
layout: post
title: "Erlang Real Time Server - Part 1 - Diameter Server"
date: 2012-05-13 18:32
comments: true
categories: Erlang Diameter rebar
---

Diameter Basics
---------------

The [Diameter](http://en.wikipedia.org/wiki/Diameter\_(protocol\)) protocol is defined by [RFC 3588](http://tools.ietf.org/html/rfc3588), and defines the minimum requirements for an AAA protocol. The best way to build up some knowledge in Diameter is to read the RFC first. At the end of the Wikipedia article there are also a bunch of links to good introductory articles. Anyway, few very basic points:

* There are several types of Diameter nodes - Client, Server, Relay, Proxy (we will focus only on the first two)
* Diameter nodes communicate with "commands";
* Clients send a given command as "Request" and the server replays with the same command as "Answer", e.g. Accounting-Request and Accounting-Answer - both will have the same command code (271), but in the case of the Client the 'Command Flags' 'R' bit will be set;
* Each command has a predefined set of attributes. If there is a need for additional attributes, they can be specified as "Vendor Specific Attribute Value Pairs" (or AVPs for short);  
* RFC 3588 defines the basic set of AAA commands. Additional commands are defined in Diameter extensions called "Applications". Don't confuse them with the usual software applications (blame the Diameter creators for the poor choice of word here :), they are just declarative specifications (or 'contracts').


Diameter in Erlang
------------------

Among the other great things Erlang contains a Diameter library. This library has two important modules - diameter and diameter_app. With the help of the first one we can create a Diameter service, which receives and sends commands. The second one - diameter_app - is the callback module for the service started with the first. Please, note that I am using 'Diameter' to denote the protocol and 'diameter' for the Erlang module.

I will not explain this part in details as it is better to take a look at the exelent example provided by the Erlang creators in the Diameter lib [source](https://github.com/erlang/otp/tree/master/lib/diameter/examples/code). Just a couple of important things:

How to start the service

``` erlang
start() ->
        SvcName = ?MODULE,
        SvcOpts = [{'Origin-Host', atom_to_list(SvcName) ++ ".example.com"},
                        {'Origin-Realm', "example.com"},
                        {'Vendor-Id', 193},
                        {'Product-Name', "Server"},
                        {'Auth-Application-Id', [?DIAMETER_APP_ID_COMMON]},
                        {application, [{alias, diameter_base_app},
                                       {dictionary, ?DIAMETER_DICT_COMMON},
                                       {module, server_cb}]}],
        TransportOpts =  [{transport_module, diameter_tcp},
                                        {transport_config, [{reuseaddr, true},
                                        {ip, {127,0,0,1}}, {port, 3868}]}],
        diameter:start(),
        diameter:start_service(SvcName, SvcOpts),
        diameter:add_transport(SvcName, {listen, TransportOpts}).

```

Few things to be mentioned here:

- Service - the service will usually implement a given Diameter Application.
- ?DIAMETER_APP_ID_COMMON and ?DIAMETER_DICT_COMMON are defined in diameter.hrl 
- server_cb (line 10) will be our diameter callback module (implementation of diameter_app)

diameter_cb will handle Diameter requests. An example function, which handles "Accounting-Request" command (code=271) is given below:

``` erlang
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

            Ans = #diameter_base_ACA{'Result-Code' = ?'DIAMETER_BASE_RESULT-CODE_DIAMETER_SUCCESS',
		                             'Origin-Host' = OH,
		                             'Origin-Realm' = OR,
		                             'Session-Id' = Id,
		                             'Accounting-Record-Type' = RecType,
		                             'Accounting-Record-Number' = RecNum},
            {reply, Ans};

``` 

What we have here is:

- get the origin host and origin realm in OH and OR from the peer's "capabilities" (Caps)
- get some of the request parameters like SessionId, record type, etc
- constructing the Accounting-Answer message as #diameter_base_ACA record 
- reply with the answer

Note: Base Diameter protocol (rfc3588) can be used for Accounting only. For Authentication and Authorization it must be extended with other Diameter applications, which depends on the type of the applications, e.g. Diameter Network Access Application(rfc4005) and Diameter Mobile IPv4 Application(rfc4004). In the IMS architecture for example, there are many subsystems communicating each other via different interfaces. Each interface is based on a given "Diameter Application", which has a specific set of commands.

Using rebar as a build tool
--------------------------------

I am going to use [Basho's rebar](https://github.com/basho/rebar) to build my application. The project will contain at least two separately deployable applications - Diameter server and Aggregation module. I'll put them in separate app directories under my project's root. 

First, create .../apps/diaserver directory under your project dir and then, copy rebar executable in "diaserver" directory.

Now, let's create the project structure for our application. For this, we will need a project template. There are many in github. I'll use the templates generously provided by [Susan Potter](https://github.com/mbbx6spp/rebar-templates). Just clone the repository in your ~/.rebar/templates directory, then do:

``` bash Create erlang project
.../apps/diaserver $ ./rebar create template=project
==> diaserver (create)
Writing README
Writing rebar.config
Writing Emakefile
Writing Makefile
Writing .gitignore

```

Next, we will create the application files. We need 3 files: application resource file, application behavior implementation and application supervisor (see [here](http://learnyousomeerlang.com/building-otp-applications) for a detailed explanation of OTP applications). They all will be created by rebar: 

``` bash Create application using rebar
.../apps/diaserver $ ./rebar create-app appid=diaserver
==> diaserver (create-app)
Writing src/diaserver.app.src
Writing src/diaserver_app.erl
Writing src/diaserver_sup.erl
```

or alternatively, using template rtsapp.template (this is my version of Susan's finapp.template):

``` bash Create application skeleton using rebar template
.../apps/diaserver $ ./rebar create template=rtsapp name=diaserver
```

The above are the files needed to make our applications of terms of Erlang/OTP framework. The real job (starting diameter server, receieving commands, etc) will be done by a gen_server module, created with the following command:

``` bash Create gen_server module using template
.../apps/diaserver $ ./rebar create template=gen_server name=diameter
==> diaserver (create)
Writing src/diameter_srv.erl
```

This will create a file named diameter_srv.erl in the ...apps/diaserver/src. 
Again, gen_server template here is a modified (read - renamed and changed the names inside the template) version of finapp.template you downloaded before.
Here is what the file should look like:

``` erlang diameter_srv.erl
%% @author <your name> <your email>
%% @copyright 2012 <your name>
%% @doc gen_server callback module implementation:
%%
%% @end
-module(diameter_srv).
-author('<your name>').

-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).
-export([code_change/3]).
-export([stop/0, terminate/2]).

% TODO: If unnamed server, remove definition below.
-define(SERVER, ?MODULE).
%%%.
%%%'   PUBLIC API

%% @doc starts gen_server implementation and caller links to the process too.
-spec start_link() -> {ok, Pid} | ignore | {error, Error}
  when
      Pid :: pid(),
      Error :: {already_started, Pid} | term().
start_link() ->
  % TODO: decide whether to name gen_server callback implementation or not.
  % gen_server:start_link(?MODULE, [], []). % for unnamed gen_server
  gen_server:start_link({local, ?SERVER}, ?MODULE, [], []).

%% @doc stops gen_server implementation process
-spec stop() -> ok.
stop() ->
  gen_server:cast(?SERVER, stop).

% TODO: add more public API here...

%%%.
%%%'   CALLBACKS
%% @callback gen_server
init(State) ->
  {ok, State}.

%% @callback gen_server
handle_call(_Req, _From, State) ->
  {reply, State}.
%% @callback gen_server
handle_cast(stop, State) ->
  {stop, normal, State};
handle_cast(_Req, State) ->
  {noreply, State}.

%% @callback gen_server
handle_info(_Info, State) ->
  {noreply, State}.

%% @callback gen_server
code_change(_OldVsn, State, _Extra) ->
  {ok, State}.

%% @callback gen_server
terminate(normal, _State) ->
  ok;
terminate(shutdown, _State) ->
  ok;
terminate({shutdown, _Reason}, _State) ->
  ok;
terminate(_Reason, _State) ->
  ok.
%%%.
%%%'   PRIVATE FUNCTIONS
% TODO: Add private helper functions here.

%%%.
%%% vim: set filetype=erlang tabstop=2 foldmarker=%%%',%%%. foldmethod=marker:
``` 

Now, as we have the skeleton, we can start to flesh it out.
What we need to do is just to put the start() snippet from the above in the init() function:

``` erlang 
init(State) ->
        SvcName = ?MODULE,
        SvcOpts = [{'Origin-Host', atom_to_list(SvcName) ++ ".example.com"},
                        {'Origin-Realm', "example.com"},
                        {'Vendor-Id', 193},
                        {'Product-Name', "Server"},
                        {'Auth-Application-Id', [?DIAMETER_APP_ID_COMMON]},
                        {application, [{alias, diameter_base_app},
                                       {dictionary, ?DIAMETER_DICT_COMMON},
                                       {module, server_cb}]}],
        TransportOpts =  [{transport_module, diameter_tcp},
                                        {transport_config, [{reuseaddr, true},
                                        {ip, {127,0,0,1}}, {port, 3868}]}],
        diameter:start(),
        diameter:start_service(SvcName, SvcOpts),
        diameter:add_transport(SvcName, {listen, TransportOpts}),
       {ok, State}.
```

Then just add the headers to be included in the beginning:

``` erlang
-include_lib("diameter.hrl").
-include_lib("diameter_gen_base_rfc3588.hrl").
```

and that's pretty much it for the gen_server. It is of course a good idea to create a macros for the service options, as we will probably implement several Diameter Applications in this module, so, each one will need different name, svc and transport options. For simplicity's sake, I'll leave it as is for now.

Why we bother at all with those gen_server behaviour, templates, etc? Fist we need a gen_server in order to comply with OTP application requirements and second - we can use this behaviour later to control our server.

As you already know, there is another module we need to implement - the diameter_app (the callback) module. Here is how it looks like:

``` erlang server_cb.erl
-module(server_cb).

-include_lib("diameter.hrl").
-include_lib("diameter_gen_base_rfc3588.hrl").

%% diameter callbacks
-export([peer_up/3,
         peer_down/3,
         pick_peer/4,
         prepare_request/3,
         prepare_retransmit/3,
         handle_answer/4,
         handle_error/4,
         handle_request/3]).


-define(UNEXPECTED, erlang:error({unexpected, ?MODULE, ?LINE})).

peer_up(_SvcName, {PeerRef, _}, State) ->
            io:format("up: ~p~n", [PeerRef]),
            State.

peer_down(_SvcName, {PeerRef, _}, State) ->
            io:format("down: ~p~n", [PeerRef]),
            State.

pick_peer(_, _, _SvcName, _State) ->
            ?UNEXPECTED.

prepare_request(_, _SvcName, _Peer) ->
            ?UNEXPECTED.

prepare_retransmit(_Packet, _SvcName, _Peer) ->
            ?UNEXPECTED.

handle_answer(_Packet, _Request, _SvcName, _Peer) ->
            ?UNEXPECTED.

handle_error(_Reason, _Request, _SvcName, _Peer) ->
            ?UNEXPECTED.


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

            Ans = #diameter_base_ACA{'Result-Code' = ?'DIAMETER_BASE_RESULT-CODE_DIAMETER_SUCCESS',
                               'Origin-Host' = OH,
                               'Origin-Realm' = OR,
                               'Session-Id' = Id,
                               'Accounting-Record-Type' = RecType,
                               'Accounting-Record-Number' = RecNum

                },
            {reply, Ans}.
```

Here we handle only the Accounting-Request (ACR). In reality, this will be the place to invoke the "accounting" function once we get the Aggregation module ready (it will be subject of the next chapter). As for now, we simply replay with "DIAMETER_BASE_RESULT-CODE_DIAMETER_SUCCESS". There are plenty of other requests we have to handle, but as the purpose of this article is just to show some examples, I'll live the rest to the readers. As you see the other diameter_app functions just return "Unexpected Error".  

We have all the pieces needed to run the server, but there is few tweaks left before we get a working erlang applications.

First, add "diameter" application in the application resource file:

``` erlang diaserver.app.src
{application, diaserver,
 [
  {description, ""},
  {vsn, "0.1.0"},
  {registered, []},
  {applications, [
                  kernel,
                  stdlib,
                  diameter
                 ]},
  {mod, { diaserver_app, []}},
  {env, []}
 ]}.
```

 This will run diameter application on server startup. We can now remove the diameter:start() call from the diameter_srv.

 Our server itself, will be started by the supervisor, thus, we have to write the supervisor's "Child Specification". It will look like this:

``` erlang server supervisor's ChildSpec
         DiaServer = {diaserver,{diameter_srv,start_link,[]},
                     permanent,
                     5000,
                     worker,
                     [server_cb]}
```

The whole supervisor's init():

``` erlang server supervisor diaserver_sup.erl
init([]) ->
        DiaServer = {diaserver,{diameter_srv,start_link,[]},
                     permanent,
                     5000,
                     worker,
                     [server_cb]},

        {ok, { {one_for_one, 5, 10}, [DiaServer]} }.
```

In rebar.config erl_opts, add the location of diameter header files. Your final configuration should be similar to:

``` erlang rebar.config
{lib_dirs, []}.
{erl_first_files, []}.
{erl_opts, [{i,"include"},{i, "/usr/local/lib/erlang/lib/diameter-0.9/include"}, {src_dirs, ["src"]}]}.
{erlydtl_opts, []}.
{cover_enabled, true}.
{clean_files, ["ebin/*.beam", "priv/log/*", "rel/*"]}.
{target, "rel"}.
{app_bin, []}.
{deps_dir, ["deps"]}.
{deps, []}.
{edoc_opts, [{doclet, edown_doclet}]}.
{sub_dirs, []}.
```

The last step is to compile the application:
``` bash
./rebar compile
```
And we are done with the server.

Now, go to the project's root directory and copy rebar executable there. What we are going to do is to create Erlang "executable":

Run the following commands:

``` bash
mkdir rel
cd rel
 ../rebar create-node nodeid=diaserver
 ==> rel (create-node)
Writing reltool.config
Writing files/erl
Writing files/nodetool
Writing files/diaserver
Writing files/sys.config
Writing files/vm.args
Writing files/diaserver.cmd
Writing files/start_erl.cmd
```

The above will create Erlang node release configuration.

Almost done, but first we have to edit the reltool.config, which was just created by the rebar:

- put our "diaserver" application directory in the "lib_dirs":

```
 {lib_dirs, ["../apps","../apps/diaserver"]},
``` 

- add "diameter" and "diaserver" applications in the list of modules needed by the release:

```
 {rel, "diaserver", "1",
        [
         kernel,
         stdlib,
         sasl,
         diameter,
         diaserver
        ]},
```

- remove {incl_cond, exclude} clause. incl_cond="derived" will be used as defauld, which means that Reltool will include applications that it detects can be used by any applications in the "rel" tuple.

Full reltool.config should be similar to:

``` bash reltool.config
{sys, [
       {lib_dirs, ["../apps","../apps/diaserver"]},
       {erts, [{mod_cond, derived}, {app_file, strip}]},
       {app_file, strip},
       {rel, "diaserver", "1",
        [
         kernel,
         stdlib,
         sasl,
         diameter,
         diaserver
        ]},
       {rel, "start_clean", "",
        [
         kernel,
         stdlib
        ]},
       {boot_rel, "diaserver"},
       {profile, embedded},
       {excl_archive_filters, [".*"]}, %% Do not archive built libs
       {excl_sys_filters, ["^bin/.*", "^erts.*/bin/(dialyzer|typer)",
                           "^erts.*/(doc|info|include|lib|man|src)"]},
       {excl_app_filters, ["\.gitignore"]},
       {app, sasl,   [{incl_cond, include}]},
       {app, stdlib, [{incl_cond, include}]},
       {app, kernel, [{incl_cond, include}]},
       {app, diaserver, [{incl_cond, include}]}
      ]}.

{target_dir, "diaserver"}.

{overlay, [
           {mkdir, "log/sasl"},
           {copy, "files/erl", "\{\{erts_vsn\}\}/bin/erl"},
           {copy, "files/nodetool", "\{\{erts_vsn\}\}/bin/nodetool"},
           {copy, "files/diaserver", "bin/diaserver"},
           {copy, "files/sys.config", "releases/\{\{rel_vsn\}\}/sys.config"},
           {copy, "files/diaserver.cmd", "bin/diaserver.cmd"},
           {copy, "files/start_erl.cmd", "bin/start_erl.cmd"},
           {copy, "files/vm.args", "releases/\{\{rel_vsn\}\}/vm.args"}
          ]}.
```

Back in the top directory, we need a simple rebar.config file:

``` bash rebar.config
{sub_dirs, ["apps/diaserver",
            "rel"]}.

{erl_opts, [debug_info]}.

{lib_dirs, ["apps", "deps"]}.

{deps, []}.
```

Execute:

``` bash
$./rebar generate
```

At this point, if everything went well, you should have a working Erlang application. Let's try it:

``` bash
$ ./rel/diaserver/bin/diaserver start
$ ./rel/diaserver/bin/diaserver ping
pong
```

It is running, but is it capable to serve Diameter requests? We need a client to test this. I'm going to use the one provided in Diameter library's [examples](https://github.com/erlang/otp/tree/master/lib/diameter/examples/code), but I need to implement Accounting Request first. Just add the following code to client.erl:

``` erlang
call_ACR() ->
        call_ACR(?SVC_NAME).

call_ACR(Name) ->
        SId = diameter:session_id(?L(Name)),
        ACR = #diameter_base_ACR{'Session-Id' = SId,
                                 'Accounting-Record-Type' = ?'DIAMETER_BASE_ACCOUNTING-RECORD-TYPE_EVENT_RECORD',
                                 'Accounting-Record-Number' = 1},
        diameter:call(Name, client_base_app, ACR, []).
``` 

and change the client_cb.erl function "prepare_request" to:

``` erlang
prepare_request(#diameter_packet{msg = Rec} = Pkt, _, {_, Caps}) ->
    #diameter_caps{origin_host = {OH, DH},
                   origin_realm = {OR, DR}}
        = Caps,

    {send, Rec#diameter_base_ACR{'Origin-Host' = OH,
                                 'Origin-Realm' = OR,
                                 'Destination-Realm' = DR}}.
```

And finally - that's it. Compile the client, run the Erlang shell and try it:

```
Eshell V5.8.4  (abort with ^G)
1> diameter:start().
ok
2> client:start().
ok
3> client:connect(tcp).
{ok,#Ref<0.0.0.50>}
4> client:call_ACR().
{ok,{diameter_base_ACA,"client;1400340423;1;nonode@nohost",
                       2001,"diameter_srv.example.com","example.com",1,1,[],[],[],
                       [],[],[],[],[],[],[],[],[],[]}}
```

We happily got an Accounting-Answer(ACA) Diameter response :-)

Well, at this moment we can handle just one command, but implementing the rest of the "Diameter Base Application" spec shouldn't be much of a problem with everything we've done so far.

You can find the source code in github: [https://github.com/vascokk/diameter-test](https://github.com/vascokk/diameter-test/tree/master/erlang-rts-part-1)

In the next article I'll start with the implementation of the Aggregation module. See ya!

Credits
----------------------------------
Susan Potter for the rebar [templates](https://github.com/mbbx6spp/rebar-templates)

Frederic Trottier-Hebert for the amazing ["Learn You Some Erlang"](http://learnyousomeerlang.com/) book
