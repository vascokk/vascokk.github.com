## Writing a computer language - tutorial for the mere mortals.

Writing a computer language is cool and can make you famouse - everybody in the IT industry knows who Dennis Ritchie, James Gosling and Guido van Rossum are. Ever wondered what it takes to create a new language? You probably think that it needs a serious theoretical background and you must be a computer scientist to do this. Well, that's true. However, sometimes you need a tool simplier than a full-blown language, something that has just the right subset of features to acomplish some task. Here comes the Domain Specific Language. DSL's come in many different shapes. It could look like a language (e.g SQL), or a special XML schema (e.g. XSLT), or an API like: 

``` java
DataSet<Tuple2<String, Integer>>  wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);
```

In case you need something that looks more like a language than an XML or API, the good news is that you don't need to take (again?) a course in compilers and formal languages theory. There are tools, that implement all the "science" for you - lexers, parser generators, or also called "compiler-compiler" tools. Here is a list from [Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_parser_generators#Deterministic_context-free_languages). 
As you can see - almost every major programming language has tools you can yous to create your own "language". I'll use my favorite Erlang for this tutorial, but the principles are the same for any other language.

## Basics

Here is how it works:

Sterp 1:

![LangTutorial1.png]({{site.baseurl}}/source/_posts/LangTutorial1.png)

Step 2:

![LangTutorial2.png]({{site.baseurl}}/source/_posts/LangTutorial2.png)

In step 1 you are using an "alphabet" (specification of the allowed caracters and primitives) and a "grammer" (rules for building sentences in your language) to describe the language. Those two are used as an input to the lexical tools - in our case _leex_ and _yecc_. 
The _leex_ tool is called "lexer", "tokenizer" or "scanner". It will produce Erlang code specific to your "alphabet", which is used to convert the source text into valid language primitives - reserved words, strings, operators, delimiters, atoms (in case of Erlang), etc.
_yeec_ is called "compiler-compiler" or "parser-generator". It will produce Erlang code specific to your "grammar", which is used to produce an Abstract Syntax Tree (AST). In simple words, an AST is a structure representing your custom language source code, in a way you can use to produce some result. There are two ways you can use the AST, i.e. there are 2 types of AST "Tree Parsers":
- you evaluate the tree for every input, or
- you generate a source/binary code based on the AST and execute this code for every input

Yes, the first one is caled "interpretation", and the second is "compilation", respectively your "Tree Parser" is either interpreter or compiler.
