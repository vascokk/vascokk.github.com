---
layout: post
title: "Erlang: How to create and compile module in run-time"
date: 2013-03-23 15:40
comments: true
categories: Erlang howto dynamic compile modules runtime
---


This is a very basic how-to, just to get you started with the topic and demonstrate that it is not so scary as you might think in the beginning.

Goal: Define and compile an Erlang module in run-time.

Let say we want to create the following module dynamically:

``` erlang
-module(div_test).
-export([divide/2]).

divide(A,B) when B =/= 0 ->
	A/B;
divide(_,_) -> 
	{error, division_by_zero}.

``` 

This is really simple module, just to illustrate the approach.

We are going to use the erl\_syntax module, which provides facilities to create Erlang source code in form of Abstract Syntax Trees.
Let's first take a look at our simple module. Here we have:

1. Module declaration where the name of the module is an Erlang atom (div\_test).
2. Export clause with a list of arity qualifiers. In our case - only one qualifier, which contains an atom (function name) and an integer (arity)
3. Function "divide" (again the name is an atom) and two function clauses

3.1. First clause contains:
	- two variables A and B
	- a guard, which is an infix operation ("=/=") between the variable B and the integer 0.
	- a body, which is an infix operation ("/") between two variables A and B

3.2. Second clause contains:
	- two anonymous variables ("_,_")
	- a body, which is a tuple of two atoms - "error" and "division_by_zero"


Now, we are going to create each of the above, using the functions provided by erl\_syntax module:


1. "module" attribute:

"-module(div\_test)" is a module attribute with argument - the name of the module:
 
``` erlang 
Module = erl_syntax:attribute(erl_syntax:atom(module),[erl_syntax:atom(div_test)]).
ModForm =  erl_syntax:revert(Module).
```

To make the Module AST compatible with the erl\_parse trees, which are used by the "compile" module (i.e. make a "Form" out of the AST), we have to invoke revert().

2. "export" attribute:

It is a bit tricky... I would happily use just the line: ExportForm = {attribute,2,export,[{divide,2}]}, but for the sake of completeness, let's go deeper:

``` erlang
Export = erl_syntax:attribute(erl_syntax:atom(export),[erl_syntax:list([erl_syntax:arity_qualifier(erl_syntax:atom(divide),erl_syntax:integer(2))])]).
ExportForm = erl_syntax:revert(Export).
```

Scary at a first sight, isn't it? In fact, it is quite straightforward:

First we are going to create an attribute. The attribute() function has two parameters - name of the attribute (which is an atom - "export") and a list of arguments. In this case the list of arguments has only one element, which is again - a list, containing arity qualifiers (pairs "name/arity") of the exported functions. So we have a call to attribute(atom, [Arguments]), where Arguments is "list([arity_qualifieris])". The arity qualifier is an atom and an integer.

2. Function:

If you grapsed the above, the function definition will be quite easy to understand:

``` erlang
% define the input variables
Var1 = erl_syntax:variable("A").
Var2 = erl_syntax:variable("B").

% define the Guard
Guard1 = erl_syntax:infix_expr(Var1, erl_syntax:operator("=/="), erl_syntax:integer(0)).

% define the function body
Body1 = erl_syntax:infix_expr(Var1, erl_syntax:operator("/"), Var2).

% define the first clause
Clause1 =  erl_syntax:clause([Var1,Var2],[Guard1],[Body1]).

% define the anonymous variable
Var3 = erl_syntax:variable("_").

% define the body of the second clause
Body2 =  erl_syntax:tuple([erl_syntax:atom(error),erl_syntax:atom(divsion_by_zero)]).

% define the second clause
Clause2 =  erl_syntax:clause([Var3, Var3],[],[Body2]).

% define the function
Function =  erl_syntax:function(erl_syntax:atom(divide),[Clause1,Clause2]).
FunctionForm = erl_syntax:revert(Function).
```

Now we just have to compile the module, load the binary and test:

``` erlang
15> {ok, Mod, Bin1} = compile:forms([ModForm,ExportForm, FunctionForm]).
{ok,div_test,
    <<70,79,82,49,0,0,2,16,66,69,65,77,65,116,111,109,0,0,0,
      55,0,0,0,5,8,100,...>>}
16> code:load_binary(Mod, [], Bin1).
{module,div_test}
17> div_test:divide(10,5).
2.0
18>

```

That's all folks! Happy hacking! :-)
