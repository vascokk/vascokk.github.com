## Writing a DSL - tutorial for the mere mortals.

Writing a computer language is cool and can make you famouse - everybody in the IT industry knows who Dennis Ritchie, James Gosling and Guido van Rossum are. Ever wondered what it takes to create a new language? You probably think that it needs a serious theoretical background and you must be a computer scientist to do this. Well, that's true. However, sometimes you need a tool simplier than a full-blown language, something that has just the right subset of features to acomplish some task. Here comes the Domain Specific Language. DSL's come in many different shapes. It could look like a language (e.g SQL), or a special XML schema (e.g. XSLT), or an API like: 

''' java
DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);
'''