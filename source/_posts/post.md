---
title: Apache Phoenix and UDFs
date: 2016-12-13 23:07:12
tags: 'Software Development (Technical)'
---
In one of the projects at work, we've been using Apache Phoenix, a SQL layer on top of HBase. It's a pretty awesome system, though at the same time it's been a some what frustrating experience given some of the issues that the documentation glosses over. The issues that we encountered where inconsistent use of secondary indexes and the lack of documentation around important functionalities such as User-Defined Functions (UDFs). Now I'll be the first to admit that perhaps the documentation wasn't necessarily bad, but it assumes sufficient working knowledge with Java and the Hadoop eco-system which we did not have when we started the project. Thinking back on it, the success of platforms such as PHP or MongoDB despite everyone's gripes about the systems is due to the fact that their documentation assumed nothing about a coder's background and allowed coders of diverse backgrounds to get started with them, whereas systems such as HBase and Phoenix have lost out on user mindshare due to the fact that it's significantly harder to get up and running. Now while this is mainly a gripe post, I hope that there's sufficient information here to make your life a lot easier than mine was.

A brief description of our project. We were building a data warehouse that was supposed to ingest and store clinical data. While clinical data is generally tabular, different sets of clinical data will have different set of columns. Now before someone tells me to use a Key-Value schema, it's not quite that simple because different data sources can have different number of columns required to define a primary key, depivoting/pivoting in general is an expensive operation, and there's no guarantee that data you get isn't already pivoted or depivoted for you. We had a preliminary SQL server solution that stored all of the data in a KV schema, but it croaked when trying to get more than 50MB of clinical data back out of the system in a somewhat standard format. Oh, and the data that we were ingesting would sometimes be given in incremental form ODM in which updates, deletes, restores were given in the context of primary keys. Oh, and this was supposed to work on clinical data streaming into us, and given how the business process works in our industry, we only have an incomplete definition of the data that we are supposed to be getting. Finally we were supposed to be able to generate standard complaint output before all of the data has been received, which ruled out writing a custom ETL script through tools like Informatica, and update transformations on the fly without reloading all of the data from scratch.  

Now, given the hard won work that we've put into it, I actually do think that HBase/Phoenix are pretty awesome in terms of having a scalable No-SQL system that still allows for standard query paths. Now to describe the issues:

1. Apache Phoenix and HBase support Dynamic Columns i.e. Columns that don't have to be specified at table creation time. However, indices don't get used if you are using a dynamic column in the query. Therefore if we have a secondary index built on staticCol2:


    select "staticCol1" from "table" where "staticCol2"='value'

will use the index. However, using dynamic Columns

    select "staticCol1", "dynamicColumn" from "table"("dynamicColumn" varchar) where "staticCol2"='value' 

will not use the index. Even though they use the same table and are trying to use the same column for the index. Even using index hints does not force the system to use indexes.

2. So our solution to this problem was to make a static column, make it a varchar, and to basically pack a JSON object into that column. Probably not the best solution, but sufficient for our purposes. In this, we needed a user defined function to basically take as input the name of that static column, and the entry within the JSON object. For simplicity we assumed that our JSON object was a simple dictionary.

So, how exactly do you write a UDF function?

Well the [documentation](https://phoenix.apache.org/udf.html) isn't terribly helpful. They have code snippets, but what imports do I need? They tell you to go to a blog post for an actual [example](http://phoenix-hbase.blogspot.in/2013/04/how-to-add-your-own-built-in-function.html). Now, the first problem. It uses libraries from Salesforce (makes sense since Salesforce came up with Apache Phoenix). But where do I get the libraries? Furthermore, since it's been rolled under the Apache Projects, I'm pretty sure that there should be standard Apache Libraries and not Saleforce libraries that I need to use. A bunch of google-fu took me to another blog post, this time in Chinese (which though I am Chinese I cannot read), and I got the following imports needed


    import java.sql.SQLException;
    import java.util.List;
    import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
    import org.apache.phoenix.expression.Expression;
    import org.apache.phoenix.parse.FunctionParseNode.Argument;
    import org.apache.phoenix.parse.FunctionParseNode.BuiltInFunction;
    import org.apache.phoenix.schema.SortOrder;
    import org.apache.phoenix.schema.tuple.Tuple;
    import org.apache.phoenix.schema.types.PDataType;
    import org.apache.phoenix.schema.types.PVarchar;
    import org.apache.phoenix.util.StringUtil;
    import org.apache.phoenix.expression.function.*;

All right, so now I can write the function. But what if I wanted to include other libraries. I haven't used Java since I was an undergrad and I hated it then, preferring C/C++ and never really got the hang of ant. Now I'd love if in a central place they did things like tell me how to use the build tools mvn, create a jar and all that fun stuff. But they didn't. So googling a bit more, I do manage to figure it [out](https://maven.apache.org/guides/getting-started/index.html#How_do_I_make_my_first_Maven_project). I figure out how to include libraries in the POM file and the damn thing builds. 

Now how do I get it into my system? The instructions are:

* After compiling your code to a jar, you need to deploy the jar into the HDFS. It would be better to add the jar to HDFS folder configured for hbase.dynamic.jars.dir.

Not entirely clear unless you know what you're doing with hbase. So something like the bottom would have been helpful.

    /usr/lib/hadoop/hadoop-2.7.2/bin/hdfs dfs -mkdir /hbase/udf/
    #Remove all previous udf files (if the package changes for whatever reason)
    /usr/lib/hadoop/hadoop-2.7.2/bin/hdfs dfs -rm /hbase/udf/*
    /usr/lib/hadoop/hadoop-2.7.2/bin/hdfs dfs -copyFromLocal /path/to/the/uber/jar /hbase/udf

Finally, now that I have it in there, how do I test it? This actually took me the longest time since

    select JSONCOLUMN('{\"a\":\"hello\"}', 'a') 

should have worked (select 1+1 does). But it doesn't. It doesn't give me an error, it just tells me that the function isn't found. So I'm trying to figure out why it can't see my function. Turns out, that what you really need to do is the following:

select JSONCOLUMN("ColumnName", 'jsonname') from "sources"

You need to apply it on the result of an actual query that returns rows. This is not made clear anywhere in the documentation. Yes I wrote simpler functions that just added +1 to a given numeric value, and they all failed with the same cryptic message. Now at this point the frustration is over, our Phoenix system uses indexes, is fast, and handles a rather flexible schema. 

However, my gripe is that with relatively simple addendums to the documentation, the path to the happy place would have much easier to get to