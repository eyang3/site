---
title: Why Open Source is Good
date: 2017-03-24 13:30:32
tags: 'Software Development (Technical)'
---
Delving deeper into Apache Phoenix, I had been looking for ways of loading data faster into my system. Our current system writes data utilizing the JDBC connection which is slower than using the bulk loader (this is true for any database system). So I had been playing around with various command line tools that were present with Apache Phoenix and realized that none of them worked if the table name was case sensitive. 

When writing the application we used camel case everywhere. I didn't much care what convention we picked, nor do I really care about conventions like that given the state of intellisense on modern editors. And so we happily marched along, double quoting tables and other identifiers and everything seemed to work. 

When playing with the map-reduce tool for doing deferred index creation, I ran into the issue of. How exactly do I specify my table name?

` ${HBASE_HOME}/bin/hbase org.apache.phoenix.mapreduce.index.IndexTool
  --schema MY_SCHEMA --data-table MY_TABLE --index-table ASYNC_IDX
  --output-path ASYNC_IDX_HFILES`

  In any case, every combination of "myTable", \"myTable\", '\"myTable\"', failed. Now the wonderful thing about Open Source Software is that the code is available on Github, so I can download the code and actually step through to figure out why it is or isn't working.

  Now in summary for anyone who is at all interested is that there are two bugs. The first bug is that the apache common cli library does not actually handle the presence of quotes within a command line parameter properly. The second bug is that the quote removal is done in two places in the Map Reduce Code. [Removal 1](https://github.com/apache/phoenix/blob/master/phoenix-core/src/main/java/org/apache/phoenix/mapreduce/index/IndexTool.java#L455), [Removal 2]((https://github.com/apache/phoenix/blob/master/phoenix-core/src/main/java/org/apache/phoenix/mapreduce/index/IndexTool.java#L455)) so even if it had been properly double quoted through the CLI parser, it would still have failed. Now the wonderful thing about open source is that the code is out there.

  So my solution was to disable those two lines, take the table name as case sensitive directly from the cli, and poof, the code works. Now at this point I can contribute back to the code by issuing a PR for the benefit of the world and I admit I haven't done it, because I need a way of introducing that change without breaking someone else's workflow (and I don't know that workflow). But for internal purposes, we have fixed an issue that affected the functioning of our system interally. And we didn't have to wait on someone else's release schedule to do it. 