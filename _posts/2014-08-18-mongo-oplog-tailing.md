---
layout: post
title:  "Tailing the MongoDB oplog and RabbitMQ"
categories: mongoDB
---
<h2>Tailing the MongoDB Oplog and Broadcasting With RabbitMQ</h2>
<p>The MongoDB operations log, or oplog, is a powerful yet somewhat mystifying part of the MongoDB core. 
The oplog is the basis for replication and understanding its nuances in your production environment is very 
important for success over the long run.  For more information I recommend reading the 
<a href="http://docs.mongodb.org/manual/core/replica-set-oplog/" target="new"> official documentation</a>
<br>
<p>I have been tinkering with “tailing” the oplog for some time and while reading various blogs, stack 
overflow questions, and talking with other MongoDB users I decided to write up some of my findings. 
So what do I mean by tailing the oplog?  A tailable cursor in MongoDB is a query that has had the tailable 
option added to it.  This causes the cursor to continue “listening” for new data to be returned. This is of 
interest on the oplog because all operations on the database can be seen here. Thus, we can use the oplog 
to replicate action to other databases, set up filter alerts, or do anything else we can dream up. In this 
article I am going to show how to connect to the oplog using java and then send messages based on what is 
going on using RabbitMQ. Broadcasting messages will allow us to implement information 
consumers completely independently of the code base we will be using to produce the messages.
<br>
<p><b>The code in these examples should be considered <u>example</u> code and not production quality.</b>
<br>
<p>All code used in the examples may be found on my <a href="https://github.com/JaiHirsch/mongo-tail" target="new">github site</a>
<br>
<p>There are plenty of excellent tutorials out there on how to set up tailable cursors and tailing the oplog 
and I do not claim this will be any better than others. One thing I have found missing from many of them is 
how to connect into a sharded cluster and auto detect the replica sets inside of it and then begin the 
tailing automagically.
<br>
<p>So, with that in mind, lets take a look at what will be discussed in this writeup. Below you will find the 
system flow we will be creating. We will be using a sharded replicated MongoDB instance.  We will then use a 
multi-threaded java process to connect into each shard's oplog. Once we open the tailable cursor to each oplog 
we will then broadcast each operation to RabbitMQ. 
<br>
<h3>Why RabbitMQ you ask?</h3>
<p>By moving the consumers of the MongoDB operations from the tailing program itself we can separate our concerns. We can attach 
any sort of consumer to the RabbitMQ server we like. Infact, using fanout techniques we can attach as many RabbitMQ consumers as 
as we like. Perhaps we want to replicate the MongoDB data to MySQL or set up filters to watch for specific information flowing 
through the system. In the example code we will
 be using RabbitMQ routing keys to divert different types of MongoDB operations to different queues. The examples 
are designed to all run in a local environment and when kept to a small scale do run on my laptop.
<br><br>
<img src="{{site.baseurl}}/images/tailable-mongo-rabbit.png" alt="Tailing MongoDB Oplog to RabbitMQ">
<br><br>
<p>To initiate the sharded replicated MongoDB environment I am using a <a href="https://github.com/JaiHirsch/mongo-tail
/blob/master/mongo-tail/init_sharded_env.sh" target="new">script</a> written by Andrew Erlichson for MongoDB's
online training courses (which I highly recommend). A windows version may also be found <a href="https://github.com
/JaiHirsch/mongo-tail/blob/master/mongo-tail/init_sharded_env.bat" target="new">here</a>.
<br>
<p>RabbitMQ is not the focus of this writeup so I will not be going into detail on how to set it up. For more 
information I recommend reading their <a href="http://www.rabbitmq.com/" target="new">documentation</a>. At the time 
of writing I am working with RabbitMQ version 3.3.5 for mac. I will say that getting RabbitMQ running on Windows is 
possible but not a lot of fun.
<br>
<p>On the java front I will be using java 7 and primarily running on mac.  I have tested the tailing code on Windows 7
and RHEL. There are quite a few topics being covered, so for this exercise I will not be using MongoDB authentication. While 
this is ok for examples, it is probably not good for your production environment.
<br>
<h3>Now that we have all that out of the way, lets have some fun.</h3>
<br>
<p> We will begin by obtaining a connection into our MongoS. The connection code is located in our main class 
<a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/runner/
ShardedReplicaTailer.java" target="new">ShardedReplicaTailer</a>
{% highlight java %}
Properties mongoConnectionProperties = loadProperties();
hostMongoS = new MongoClient(mongoConnectionProperties.getProperty("mongosHostInfo"));
{% endhighlight %}
<br>
<h3>Now we get the shard info for the cluster</h3>
<br>
<p>The properties file that is being loaded currently points to localhost. Once we have a connection to a MongoS we can 
use the <a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/util/ShardSetFinder.java" target="new">
ShardSetFinder</a> to obtain the shard information:
<br>
{% highlight java %}
DBCursor find = mongoS.getDB("admin").getSisterDB("config").getCollection("shards").find();
{% endhighlight %}
<br>
<p>To put this line of code into context, the overall software is going to obtain a map of the shards and 
return it back to the calling code. 
<ul>
   <li>From the MongoClient mongoS we get the “admin” DB
   <li>From the admin DB we get the sister DB “config”
   <li>From the config DB we get the collection “shards”
   <li>Finally we issue a find() command that will return cursor containing all of the shards
</ul>
<p>Here is the code block that is building the map of shard names to mongo clients.

{% highlight java %}
public Map<String, MongoClient> findShardSets(MongoClient mongoS) {
   DBCursor find = mongoS.getDB("admin").getSisterDB("config").getCollection("shards").find();
   Map<String, MongoClient> shardSets = new HashMap<String, MongoClient>();
   while (find.hasNext()) {
      DBObject next = find.next();
      String key = (String) next.get("_id");
      shardSets.put(key, getMongoClient(buildServerAddressList(next)));
   }
   find.close();
   return shardSets;
}
{% endhighlight %}
<br>
<p>Example output when running a local replicated sharded environment:

<code>
Adding { "_id" : "s0" , "host" : "s0/localhost:37017,localhost:37018,localhost:37019"}<br>
Adding { "_id" : "s1" , "host" : "s1/localhost:47017,localhost:47018,localhost:47019"}<br>
Adding { "_id" : "s2" , "host" : “s2/localhost:57017,localhost:57018,localhost:57019"}
</code>

<p>Here we can see that we have added the shard names (s0 - s2) and their associated server lists (replica sets).
<br>
<h3>Now the magic happens</h3>
<br>
<p>From here we need to create connections into the various replica sets to gain access to the oplog.  In this 
case we will spawn a thread for each shard so we can tail each shard set independently. One other <b>very</b> 
important thing to keep in mind is the time stamp of the operations. Unless you wish to replay the entire oplog
everytime the program starts you must persist the timestamps of the latest operation per shard. This is why I 
used a map to hold the connection information. Now each thread may keep track of latest time stamp for its 
specific shard and use the shard name (the map key) for this.
<br>
Code excerpt from: <a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/runner/OplogTail.java" target="new">OplogTail</a> 
{% highlight java %}
...

DBCollection fromCollection = client.getDB("local").getCollection("oplog.rs");
DBObject timeQuery = getTimeQuery();
DBCursor opCursor = fromCollection.find(timeQuery)
                                  .sort(new BasicDBObject("$natural", 1))
                                  .addOption(Bytes.QUERYOPTION_TAILABLE)
                                  .addOption(Bytes.QUERYOPTION_AWAITDATA)
                                  .addOption(Bytes.QUERYOPTION_NOTIMEOUT);
...

private DBObject getTimeQuery() {
      return lastTimeStamp == null ? new BasicDBObject() : new BasicDBObject("ts", 
         new BasicDBObject("$gt",lastTimeStamp));
}

...

{% endhighlight %}
<br>
<p>Ok, lets break down what went on here.
<ul>
   <li>From the MongoClient we got the DB local
   <li>From the local DB we got the DBCollection "oplog.rs"
   <li>We then get a timestamp - this is very important as we do not want to reply the entire oplog every time we connect 
       (at least in this case)
   <li>We want to sort the return by the "$natural" operator, this will return the documents in the order they appear on disc. 
       For more information see the <a href="http://docs.mongodb.org/manual/reference/operator/meta/natural/" target="new">MongoDB docs</a>
   <li>We then add three options to our query: <a href="http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol/" target="new">
       Full Definition</a> as found in the MongoDB documentation. ***
   <ul>
      <li><code>Bytes.QUERYOPTION_TAILABLE</code></li>
         <ul>
            <li>Tailable means cursor is not closed when the last data is retrieved. Rather, the cursor marks the final object's 
                position. You can resume using the cursor later, from where it was located, if more data were received. Like any 
                "latent cursor", the cursor may become invalid at some point (CursorNotFound) – for example if the final object 
                it references were deleted.
         </ul>
      <li><code> Bytes.QUERYOPTION_AWAITDATA </code></li>
         <ul>
            <li>Use with TailableCursor. If we are at the end of the data, block for a while rather than returning no data. 
                After a timeout period, we do return as normal.
         </ul>
      <li><code>Bytes.QUERYOPTION_NOTIMEOUT</code>
         <ul>
            <li>The server normally times out idle cursors after an inactivity period (10 minutes) to prevent excess memory use. 
                Set this option to prevent that.</ul>
   </ul>
</ul>
<p>At this point we are ready to start tailing. The first thing we will do is use an injection strategy to create our RabbitMQ tailing class. I will not 
go over this class now, but you may find the code here:  <a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/tail/TailTypeInjector.java" 
target="new">TailTypeInjector</a>. This injection stretegy will allow us to chain tailing operations together at a later date.
<br>
<p><h3>Lets take a look at our tailing classes:</h3><br>
First, we define the following interface: <a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/tail/TailType.java" target="new">TailType</a>:
{% highlight java %}
package org.mongo.tail;

import com.mongodb.DBObject;

public interface TailType {
   public void tailOp(DBObject op);
   public void close();
}
{% endhighlight %}

<p>Next we difine the following abstract class: <a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/tail/types/AbstractGenericType.java" target="new">
AbstractGenericType</a>
{% highlight java %}
package org.mongo.tail.types;

import org.mongo.tail.TailType;
import com.mongodb.DBObject;

public abstract class AbstractGenericType implements TailType {
   @Override
   public void tailOp(DBObject op) {
      switch ((String) op.get("op")) {
         case "u":
            if ("repl.time".equals((String) op.get("ns"))) {}
            else handleUpdates(op);
            break;
         case "i": handleInserts(op);
            break;
         case "d": handleDeletes(op);
            break;
         default: handleOtherOps(op);
            break;
      }
   }  
   protected void handleOtherOps(DBObject op) {
      System.out.println("Non-handled operation: " + op);
   }
   protected abstract void handleDeletes(DBObject op);
   protected abstract void handleInserts(DBObject op);
   protected abstract void handleUpdates(DBObject op);
   public void close() {}
}
{% endhighlight %}
Finally we create our concrete oplog to RabbitMQ tailing classe: 
<a href="https://github.com/JaiHirsch/mongo-tail/blob/master/mongo-tail/src/main/java/org/mongo/tail/types/RabbitProducerType.java" target="new">RabbitProducerType</a>
{% highlight java %}
package org.mongo.tail.types;

import java.io.IOException;
import com.mongodb.DBObject;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class RabbitProducerType extends AbstractGenericType {
   
   private static final String EXCHANGE_NAME = "mongo-tail";
   private Connection connection;
   private Channel channel;
   
   public RabbitProducerType() {
      ConnectionFactory factory = new ConnectionFactory();
      factory.setHost("localhost");
      try {
         connection = factory.newConnection();
         channel = connection.createChannel();
         channel.exchangeDeclare(EXCHANGE_NAME, "direct");
      } catch (IOException e) {
            e.printStackTrace();
      }
   }

   public void publishMessage(DBObject op, String routingKey) {
      try {
         String message = op.toString();
         channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
      } catch (IOException e) {
         e.printStackTrace();
      }
   }

   @Override
   protected void handleDeletes(DBObject op) {
      publishMessage(op, "d");
   }  
   @Override
   protected void handleInserts(DBObject op) {
      publishMessage(op, "i");
   }

   @Override
   protected void handleUpdates(DBObject op) {
      publishMessage(op, "u");
   }

   @Override
   public void close() {
      try {
         channel.close();
         connection.close();
      } catch (IOException e) {
          e.printStackTrace();
      }
   }
}
{% endhighlight %}

<h3>Thank you</h3>
I hope this this information was useful. If you have not fallen asleep yet, here is a video
that shows the systems linked together! My first ever attempt at a screen recording is a bit fuzzy so I hope to
post a higher quality video soon.
<br>
<iframe width="480" height="360" src="http://www.youtube.com/embed//K4okN32CgQA" frameborder="0"></iframe>


