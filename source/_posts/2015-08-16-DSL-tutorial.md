## Writing a DSL - tutorial for the mere mortals.

Writing a computer language is cool and can make you famouse - everybody in the IT industry knows who Dennis Ritchie, James Gosling and Guido van Rossum are. Ever wondered what it takes to create a new language? You probably think that it needs a serious theoretical background and you must be a computer scientist to do this. Well, that's true. However, sometimes you need a tool simplier than a full-blown language, something that has just the right subset of features to acomplish some task. Here comes the Domain Specific Language. DSL's come in many different shapes. It could look like a language (e.g SQL), or a special XML schema (e.g. XSLT), or an API like: 

``` java
DataSet<Tuple2<String, Integer>>  wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);
```

If you need something that looks more like a language than an XML or API, the good news is that you don't need to take (again!) a course in compilers and formal languages theory. There are tools, that implement all the "science" for you - lexers, parser generators, or also called "compiler-compiler" tools. Here is a list from [Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_parser_generators#Deterministic_context-free_languages). As you can see - almost every major programming language has tools you can yous to create your own "language". I'll use my favorite Erlang for this tutorial, but the principles are the same for any other language.

## Basics

Here is how it works:

Sterp 1:

![LangTutorial1.png]({{site.baseurl}}/source/_posts/LangTutorial1.png)

Step 2:

![LangTutorial2.png]({{site.baseurl}}/source/_posts/LangTutorial2.png)

