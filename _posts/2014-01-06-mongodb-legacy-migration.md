---
layout: post
title:  "CARFAX vehicle history migration to MongoDB"
categories: mongoDB
---
<h3>Why we decided to move over eleven billion records from the legacy system</h3>
The legacy vehicle history database is a distributed key value store on <a href="http://en.wikipedia.org/wiki/OpenVMS">OpenVMS</a>. Compressed the database is roughly nine terabytes.  This system has served <a href="http://www.carfax.com/entry.cfx">CARFAX</a> very well since 1984 when the company was founded, but we were capping on the number of reports per second that could be delivered per production cluster.  The hardware cost and technical difficulties of scaling the legacy system were very high.  Thus we decided to start looking at different options.

<h3>Why we decided to use MongoDB</h3>

As an Agile and extreme programming development shop, CARFAX uses Friday afternoons as professional/personal development time.  Several developers began using this time to experimenting with traditional RDBMS and NoSQL solutions for the vehicle history database.  The complexity of the data structures compounded with the size of the database did not lend itself to <a href="http://www.oracle.com/index.html">Oracle</a> or <a href="http://www.mysql.com/">MySql</a>.  We also evaluated <a href="http://cassandra.apache.org/">Cassandra</a>, <a href="http://couchdb.apache.org/">CouchDB</a>, <a href="http://basho.com/riak/">Riak</a>, and several other NoSQL solutions (I really wanted to use <a href="http://www.neo4j.org/">Neo4J</a> to create a social network of cars).  In the end we decided that MongoDB gave us the best of where we wanted to be in the <a href="http://en.wikipedia.org/wiki/CAP_theorem">CAP theorem</a> and offered the level of commercial support we were looking for.

<h3>Challenges we faced during the migration</h3>
<p>
Cleansing, transforming, and inserting eleven billion complex records into a database is no small task. To avoid inconsistencies with the
 ongoing data feeds it was decided that we needed to load the existing records into MongoDB in under fifteen days. The legacy records were 
of multiple formats that required specific interpretations.  Many of the records were also inserted using PL/I data structures that have no
 direct mapping in Java or C and required bitmapping and other fun tricks.  Once we wrote the Java and C JNI algorithms to transform the old
 records to the MongoDB format we began our loads.
</p>
<<<<<<< HEAD
=======

>>>>>>> origin/gh-pages
<p>
MongoDB does not support OpenVMS and as such we had to use remote MongoS processes to load the data. Using direct inserts to MongoDB we opened 
up the queues.  The process was immediately computationally bound.  The poor processers on the OpenVMS boxes were running full tilt; we could 
barely even get a command prompt to check on the system.  We managed to throttle back the queues so we could get a stable session, but the projected
 extract, transform, and load time was at forty five days.  We let the loads continue but we went back to the drawing board to consider how to 
solve the computational limitations we had hit.
</p> 

<img src="{{site.baseurl}}/images/vms_direct_insert.jpg" alt="Direct load to MongoDB">

<h3>The Solution</h3>

The transformation and insertion logic was already pretty clean and stateless so with a modest amount work we were able to implement a multi-threaded
 and multi-JVM RabbitMQ solution.  We had some servers that were slated to become app-servers that were not being used yet so we deployed the RabbitMQ
servers and lots of consumer process onto them.  We released the queues.  The initial numbers did not look all that promising, but once the shard 
ranges on MongoDB settled in we began to see very impressive load numbers.  We averaged over eleven thousand plus writes a second to MongoDB.  
We finished the load in just under ten days.  Through judicious use of compression of certain fields that are not useful for normal queries or 
aggregation the MongoDB collection was actually slightly smaller than the original database on OpenVMS.

<img src="{{site.baseurl}}/images/rabbit_insert.jpg" alt="Direct load to MongoDB">
 
Successes of the Migration
<ul>
  <li>Each replica set in the production cluster is capable of producing an order of magnitude more reports per second than the old system.
  <li>They legacy system could only answer one question easily:  What are the records for a given vehicle?  Any others queries required a custom C crawler and could take weeks return the answers.  In MongoDB we are able to product many useful reports from the data much faster.
  <li>Setting up a new cluster in the legacy system was an extensive process.  We can stand up a new replica set in MongoDB in about two days.  This is only possible through the use of automation.  Automate EVERYTHING.
  <li>Cross datacenter data publication and synchronization was error prone and complicated under the old system.  In MongoDB we are generally only a few seconds behind in our various data centers.
</ul>
Other Thoughts:
<ul>
  <li>If you are using a Linux distribution; <b>check that you are not using the default ulimit settings!</b>  We suffered a massive cascading crash due to missing this in our initial automation settings.
  <li>Moving terabytes of data though an internal network is a challenge.  Work very closely with your network team to make sure everyone is ready for the traffic.
  <li>Naming conventions for the fields in your MongoDB schema are very important!  Due to the BSON format, remember that every character counts in large implementations.
  <li>Consider your shard keys very carefully!  Indexes on extensive collections use a lot of RAM.  We chose a shard key that maximized read and write performance but it was not the _id field.  This forced us into a two index situation.  The _id index is only used for updates and deletes which are relatively infrequent in this system.
  <li>If you are computationally bound, figure out how to grid out the problem.  RabbitMQ worked well in our case, but carefully consider your problem domain and figure out what is the best solution.  We also considered 0MQ and GridGain either would have been great choices but we decided upon RabbitMQ mostly due to in-house knowledge.  We are currently evaluating moving our distributed load process to Scala and Akka.
  <li>For high volume inserts the more shards the better (and LOTS of RAM)!  Managing those shards can be tricky but it pays off.
  <li>We attempted at one point to break the dataset down into a massive number of collections due to the collection level locking in MongoDB, however, the rebalancing of so many collections destroyed our ability to insert and we reverted back to a single large collection.
  <li>In a large sharded collection make sure that queries that need to return quickly use the shard key.  Otherwise you are forced into a scatter/gather query.
  <li>The project was ultimately only a success due the amazing team at Carfax.  If you love data and are looking for an excellent career opportunity we are always looking for good people to <a href="http://hire.jobvite.com/CompanyJobs/Careers.aspx?k=JobListing&c=qLi9VfwR&v=1">join us</a>.
</ul>

A special thanks goes out to Brendan McAdams (@rit on Twitter) for helping us in our initial development phase!

Here is a link to the official Carfax statement on the migration to MongoDB:

<a href="http://www.mongodb.com/press/carfax-selects-mongodb-power-11-billion-record-database">http://www.mongodb.com/press/carfax-selects-mongodb-power-11-billion-record-database</a>
