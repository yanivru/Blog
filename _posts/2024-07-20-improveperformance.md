---
title: "Reading and writing Avro files using Apache.Avro library (C#)"
date: 2014-06-12
tags: [Avro, Dotnet, C#]
---

<h1>IMPROVING C# APPLICATION PERFORMANCE</h1>
Loader is a program that reads rows from CSV files, transforms them, and uploads them to Solr (a search engine) in groups of 1000 rows. Each row has around 200 fields. Loader was uploading only 500 rows per second while using 1 CPU core. I had to improve it to reach around 10,000 rows per second from each core.

<h3>Dictionary 0 - Array 1</h3>
It was evident from the profiler results that running hash functions wasted a lot of CPU time. So, a quick investigation of the code revealed that we transformed each CSV row into a dictionary, and the transformation stage then used that dictionary. Dictionaries offer an easy way to work with data with changing schema (I don't know the schema at compile time), as I can access the row's properties by their names. But surprisingly, it has a big performance hit when accessing the dictionary 200 times per row. 

The obvious solution was to replace the dictionaries with arrays. The programs copy the CSV rows to arrays. Create one dictionary per CSV file that contains the mapping between the property name and its index in the array. That way, the code that transforms the data can use the dictionary once per file to get the indexes. Then, it uses the indexes to access the data in the arrays. So, instead of using a dictionary once per row, I now use it once per file.

Replacing the dictionary with an array improved performance dramatically. Of course, there are many cases in which we can't replace dictionaries with arrays, or the ease of use is more important than the performance impact. Keep in mind that they come with a performance hit if used excessively.

You shall not parse
Now, I turned the spotlight to string parsing. Since we save data in CSV files as strings, the code had to parse all the fields with other types, like integers and dates. Replacing CSV with Apache Avro solved this issue and gave another performance boost, as Avro stores data in a binary format.

This wasn't an easy change. It required changing the service that produced the CSV file and all the other consumers of the CSV files.

Alot of garbage
But the performance was still far from our goal. Further investigation proved that the bottleneck is now the GC. It makes sense since we were creating around 200 objects per row. And those objects survived GC and were copied to higher generations.

Array 0 - Enumerable 1
The code was reading 1000 rows from a file, converting them to Solr objects, and then uploading them. Since it was waiting for 1000 rows, most of those objects lived long enough to survive GC. Same as for the Solr objects. The solution was to create each Solr object just before the program streamed them to Solr. In this way, the program created each Solr object for a short time, and it didn't survive GC. I changed the array of Solr objects to Enumerable. The Enumerable creates the Solr objects lazily.

Stop the world
After the last improvement, the program performance met the demands. The next step was scale testing. Meaning to run the code on several threads. But it seems that adding more threads didn't improve performance. And the limit was 18k rows per second, no matter how many threads we added. The limit was a result of GC locking the application too often.

Changing GC mode to Server was supposed to improve performance, but Server mode only worsened it.

A stream of memories
The solution that helped was to read 1000 rows one by one and serialize each row immediately to a MemoryStream. And then push this entire byte array (from the steam) to Solr. In this case, objects read from the file didn't survive GC. And the only object that was hanging around for a while was the memory streams. However, since it was just one object, it was easier for the GC to handle, even though it goes to the large object heap.

Further improvement will be to use a MemoryStreamPool. To reduce the pressure on GC even more.

Another option is using a RowsPool instead of using MemoryStream. This way, the program won't create the rows each time. But this seems much more complicated to implement.

Summary
Arrays offer much better performance than dictionaries, mainly because it saves the need to calculate hash values. Dictionaries might be slow if used in performance-critical code

Binary formats such as Avro and Parquet are more performant than CSV or string-based formats.

Short-living objects are less expensive in terms of GC.

A single big object performs better than many small objects.
