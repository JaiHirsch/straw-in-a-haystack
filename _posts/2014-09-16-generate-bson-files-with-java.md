---
layout: post
title:  "Generating BSON files with java"
categories: mongoDB
---
<h2>Example of generating BSON files with java</h2>
<p>So I have been pondering various ways to import data into MongoDB lately. While it is possible to import .json files or 
.csv files directly, they do not carry the type data with them.  Mongo's default behavior in this case appears to be to 
store the data in the "least expensive" format. Thus fields that are intended to be longs may be stored as integers and 
dates will be stored as strings, etc. If we are using a strongly typed language this can lead to issues when we retrieve 
the data back out and it is not in a type that we expect.</p>
<br>
<h3>So what are our alternatives?</h3>
<br> 
Many legacy systems may send something like a .csv file with something like:
<br><br>
"Bob","Pants","07/14/1986"
<br><br>
We would then need a file descriptor of some sort to interpret the file, historically in xml:
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<fields>
   <field-1>
      <field-name>f_name</field-name>
      <field-type>String</field-type>
   </field-1>
   <field-2>
      <field-name>l_name</field-name>
      <field-type>String</field-type>
   </field-2>
   <field-3>
      <field-name>birthdate</field-name>
      <field-type>Date</field-type>
      <date-format>MM/dd/yyyy</date-format>
   </field-3>
</fields>
{% endhighlight %}

Or, perhaps we can try to use the file header to carry the information:
<br><br>
if_name,String","l_name,String","birthdate,Date,MM/dd/yyyy"
<br>
"Bob","Pants","07/14/1986"
<br><br>
<p>The problem with this approach is that we must update the header or the meta file every time there is a change in the incoming 
data and requires custom code to ingest the file and load it into MongoDB.
<br><br>
<h4>Using JSON files that carry their type information with them.</h4>
<br>
{% highlight json %}
{
   "names": {
      "f_name":"Bob",
      "l_name":"Pants",
      "$type":"String"
   },
   "dates": {
      "bday":"07/14/1986",
      "$type":"date",
      "$date_format":"MM/dd/yyyy"
   }
}
{% endhighlight %}
<p>This is very verbose, but it works well. If we generate a contract with the consuming code on what the $types mean 
then should be able to safely transmit our JSON files and have our types preserved. The layout of the data should be able to
change and we can use JSON libraries to ingest the files and load them into MongoDB and preserve our type information.
 It will, however, still require some custom ingestion code</p>
<br>
<h4>Lets take this a step further</h4>
<p>what if we could generate a <a href="http://bsonspec.org/" target="new">BSON</a> file that 
could be loaded directly into MongoDB? BSON is a binary JSON specification and is how MongoDB natively stores its data. The 
<a href="http://docs.mongodb.org/manual/reference/program/mongodump/" target="new">mongodump</a> and 
<a href="http://docs.mongodb.org/manual/reference/program/mongorestore/" target="new">mongorestore</a> utilities generate 
and consume the BSON files respectively.</p>
There are several very important things to keep in mind.
<ul>
   <li>The file structure is very important, it must be for the form: dump/database-name/collection-name.bson
   <li>Index information is carried in a file of the form: dump/database-name/collection-name.metadata.json
</ul>
<p>The index file is interesting because we can not only transmit the type information along with the BSON file but we can 
pass along the expected indexes as well. However, be very careful when adding indexes to existing collections!
</p>
<p>Writing the bson file itself is not very difficult, the java driver includes a BSON encoder out of the box that we can use.
 <a href="https://github.com/JaiHirsch/mongo-bson-writer/blob/master/src/main/java/org/mongo/bson/BSONFileWriter.java" target="new">
Here</a> is an example that uses the <a href="http://api.mongodb.org/java/2.7.2/org/bson/BasicBSONEncoder.html" target="new">
BasicBSONEncoder</a> to write out a BSON file:
</p>
{% highlight java %}
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import org.bson.BasicBSONEncoder;
import com.mongodb.DBObject;

public class BSONFileWriter {

   private final String path;
   private final BasicBSONEncoder encoder;

   public BSONFileWriter(String path) {
      this.path = path;
      this.encoder = new BasicBSONEncoder();
   }

   public void write(DBObject dbo) throws IOException {

      Files.write(Paths.get(path), encoder.encode(dbo),
            StandardOpenOption.CREATE, StandardOpenOption.APPEND);

   }

}
{%  endhighlight %}

Here we are using java 7 and the Files helper class to write the encoded MongoDB DBObject to our file an object at a time. 
<br>
Lets consider a use case for this strategy, extracting rows from MySQL and loading them into MongoDB. 

{% highlight java %}
...
public class MySqlDao implements Closeable {

   private final Connection conn;

   public MySqlDao(String connString) throws SQLException {
      conn = DriverManager.getConnection(connString);
   }

   public void exportMySqlToBSON(String query, String path)
         throws SQLException, IOException {
      BSONFileWriter bsonWriter = new BSONFileWriter(path);
      Statement st = null;
      try {
         Map<String, Object> mapper = new HashMap<String, Object>();
         st = conn.createStatement();
         ResultSet rs = st.executeQuery(query);
         // use the result set meta data to populate the keys for the hashmap
         // this will allow us to use the column names as the field keys in
         // MongoDB
         ResultSetMetaData metaData = rs.getMetaData();
         while (rs.next()) {
            for (int i = 1; i <= metaData.getColumnCount(); i++) {
               mapper.put(metaData.getColumnName(i), rs.getObject(i));
            }
            bsonWriter.write(new BasicDBObject(mapper));
         }
      } finally {
         if (st != null)
            st.close();
      }
   }
...
}
{% endhighlight %}

Here we are using the meta data carried by the MySQL result set to extract keys and the values are pulled as Object types. The 
encoder will then maintain the type as extracted from the MySQL database.
<br>
<h3>The meta data file</h3>
<br>
The meta data file has uses the following structure:
{% highlight json%}
{ "indexes" : [ { "v" : 1, "key" : { "_id" : 1 }, "name" : "_id_", "ns" : "test.foo" }, { "v" : 1, "key" : { "abc" : 1 }, "name" : "abc_1", "ns" : "test.foo" } ] }
{% endhighlight %}
<p>We can see that this JSON document is an array of indexes.</p>
<p>Lets take a look at how we can extract information from our MySQL table and carry that index over to our MongoDB collection.</p>
<br>
<p>First we will use a wrapper around DBObject to map out the key value pairs in the correct format:</p>
{% highlight java%}
mport com.mongodb.BasicDBObject;
import com.mongodb.DBObject;

public class MetaData {
   private DBObject meta;

   public MetaData(int v, String key, int dir, String name, String ns) {
      meta = new BasicDBObject();
      meta.put("v", v);
      meta.put("key", new BasicDBObject(key, dir));
      meta.put("name", name);
      meta.put("ns", ns);

   }

   public DBObject getMetaData() {
      return meta;
   }

   public String toString() {
      return meta.toString();
   }
}
{% endhighlight %}
<br>
<p>Next we need to obtain the index information from the MySQL database:</p>
{% highlight java%}
...
   public BasicDBList getIndexInfoForTable(String schema, String tableName)
         throws SQLException {
      BasicDBList rtn = new BasicDBList();
      Statement st = conn.createStatement();
      String query = "SHOW INDEX FROM %s";
      ResultSet rs = st.executeQuery(String.format(query, tableName));
      while (rs.next()) {
         MetaData md = new MetaData(1, rs.getString("COLUMN_NAME"), 1,
               rs.getString("COLUMN_NAME")+"_", schema + "." + tableName);
         rtn.add(md.getMetaData());
      }
      return rtn;
   }
...
{% endhighlight %}
<p>This is a rather simplistic approach for pulling the index information from MySQL and more advanced
or compound indexes will require additional logic to handle.</p>
<p>Once we have our list of indexes we can pass it along to the meta data file writer:</p>
{% highlight java %}
...
   public static synchronized void writeMetaDataFile(String DBName,
         String DBCollectionName, Indexizer idx) throws IOException {
      ensurePathExists(DBName, DBCollectionName);
      BufferedWriter bw = null;
      try {
         bw = new BufferedWriter(new FileWriter(String.format(
               Dumps.PATH_PATTERN, DBName, DBCollectionName, "metadata.json")));
         StringBuilder sb = new StringBuilder();
         sb.append("{ \"indexes\" : ");
         sb.append(idx.getMetaData(DBName, DBCollectionName));
         sb.append(" }");
         bw.write(sb.toString());
         bw.newLine();
      } finally {
         if (bw != null)
            bw.close();
      }
   }
...
{% endhighlight %}
<p>Now we can create a main class to run all of our classes together and create our import file!</p>
{% highlight java%}
package org.simple.mysql;

import java.io.IOException;
import java.sql.SQLException;

import org.mongo.bson.Dumps;
import org.mongo.bson.MetaDataWriter;

public class MySqlDirectRunner {

   public static void main(String[] args) throws SQLException, IOException {
      Dumps.createDumpDirectories("test");
      MySqlDao dao = new MySqlDao("jdbc:mysql://localhost/test?");
      dao.exportMySqlToBSON("select * from foo", "dump/test/foo.bson");
      MetaDataWriter.writeMetaDataFile("test", "foo", new MySqlIndexizer(dao));
      dao.close();

   }
}

{% endhighlight %}
<p>I hope you have enjoyed my ramblings on importing data into MongoDB. All source code found in these examples
may be found <a href="https://github.com/JaiHirsch/mongo-bson-writer" target="new">here</a>
