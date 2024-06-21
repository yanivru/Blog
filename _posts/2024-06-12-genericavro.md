---
title: "Reading and writing Avro files using Apache.Avro library (C#)"
date: 2014-06-12
tags: [Avro, Dotnet, C#]
---
<a href="https://github.com/apache/avro">Apache Avro</a> is a data
  serialization library. It is ported to many languages including C#. Finding examples of how to read and write objects from files was difficult. And even more difficult to optimize it for performance. Most of the examples are in Java, and though the code is pretty similar there are still differences. All the code in this article can be found: <a href="https://github.com/yanivru/AvroExamples">here</a>.

When using Avro reader and writer you can use them either with GenericRecord or SpecificRecord. GenericRecord can be used when the schema is not known at compile time. You can access the fields of this record either by their name or by the index (the index is faster). SpecificRecord on the other hand is ideal when you know the schema at compile time. You can inherit from SpecificRecord and have explicit access to your fields.

<h3>Reading a file using generic records</h3>
The code first opens a DataFileReader. Notice that we use a fairly complicated constructor. This constructor allows us to provide a DatumReader. This class is responsible for reading a single object from the file. GenericDatumReader is faster than the default reader. We don't specify a reader schema (null), meaning we will read the data using the schema it was written with.

    static void Main(string[] args)
    {
        using (var stream = File.OpenRead(@"weather.avro"))
        {
            using (var reader = DataFileReader<GenericRecord>.OpenReader(stream, null, (ws, rs) => new GenericDatumReader<GenericRecord>(ws, rs)))
            {
                PrintHeader(reader);
    
                foreach (var entry in reader.NextEntries)
                {
                    Print(entry);
                }
            }
        }
    }
  
    private static void PrintHeader(IFileReader<GenericRecord> reader)
    {
        var schema = (RecordSchema)reader.GetSchema();
    
        foreach(var field in schema.Fields)
        {
            Console.Write(field.Name + ", ");
        }
        Console.WriteLine();
    
    }
    
    private static void Print(GenericRecord entry)
    {
        for(int i = 0; i < entry.Schema.Count; i++)
        {
            Console.Write(entry.GetValue(i) + ", ");
        }
        Console.WriteLine();
    }

<h3>Writing a file using GenericRecord</h3>
To write to an Avro file you first need to define a schema. The code defines a schema, creates a GenericRecord, fills it with data, and finally adds it to the DataFileWriter. Same with the reader, we specify here the GenericDatumReader which is the faster DatumReader.</div>

    using(var stream = File.OpenWrite(@"users.avro"))
    {
        var schemaJson = "{\"type\" : \"record\", " +
            "\"namespace\" : \"myNameSpace\", " +
            "\"name\" : \"User\", " +
            "\"fields\" : " +
            "[" +
            "{ \"name\" : \"Name\" , \"type\" : \"string\" }," +
            "{ \"name\" : \"ID\" , \"type\" : \"int\" }]}";
    
        var recordSchema = (RecordSchema)Schema.Parse(schemaJson);
        using (var writer = DataFileWriter<GenericRecord>.OpenWriter(new GenericDatumWriter<GenericRecord>(recordSchema), stream))
        {
            var record = new GenericRecord(recordSchema);
            record.Add(0, "user1");
            record.Add(1, 1234);
            writer.Append(record);
        }
    }
