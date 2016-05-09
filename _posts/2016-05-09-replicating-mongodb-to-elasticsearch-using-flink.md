---
layout: post
title:  "Replicating the MongoDB oplog to Elasticsearch using Apache Flink"
categories: mongoDB, elasticsearch, apache flink
---
<h2>Replicating the MongoDB oplog to Elasticsearch using Apache Flink</h2>
<p>I have been tinkering around with RxJava and reactive streams lately, and I decided to revisit one of my older
projects for tailing the MongoDB oplog and used RabbitMQ to broadcast out changes to a sharded MongoDB Cluster.  In this
    installment of tailing the oplog I decided not to use RabbitMQ, instead I decided to try out Apache Flink.  Flink
    appears to play in the same domain as Spark Streaming.  I am not endorsing one over the other in this blog, and
    it is always important to be constantly evaluating different technologies to ensure you make the right decisions.
    I am using Apache Flink to wire the streaming data source (MongoDB oplog tail) and an Elasticsearch sink.
    </p>
    <p>All source code for this project may be found
        <a href="https://github.com/JaiHirsch/flink-mingo-tail" target="new">here</a>.  Also, before we get started:  All code used here
        is not production quality.  It was not test driven and was written as a thought experiment on a rainy
        Friday afternoon as a way to learn more about
        <a href="http://www.reactive-streams.org" target="new">reactive streams</a>,
        <a hreaf="https://github.com/ReactiveX/RxJava" target="new">RxJava</a>,
        <a href ="https://flink.apache.org" target="new">Apache Flink</a>, and
        <a href="https://www.elastic.co" target="new"> Elasticsearh.</a>

</p>
<p>I will not go deep into the dirty details of oplog tailing other than to say it is much easier with the reactive
    driver. Lets do, however, take a quick look at getting an event publisher using the reactive driver and some of
    syntax differences using the new generation of the MongoDB driver.</p>
<p>This snippet of code is inside a loop that is getting a publisher for every replica set inside of a sharded
    cluster.</p>
{% highlight java %}
FindPublisher</Document> oplogPublisher = client.getClient().getDatabase("local")
    .getCollection("oplog.rs").find().filter(getQueryFilter(tsCollection, client))
    .sort(new Document("$natural", 1)).cursorType(CursorType.TailableAwait);
{% endhighlight %}

<p>It is also important to understand the filters that are generated though the getQueryFilter() call:
<ul>
    <li>In this example I am using the same MongoDB cluster to track the last operation timestamps. So filtering out
        the collection these are being written to is VERY important, otherwise this creates a pretty bad ass infinite
        loop. On a side note, it is probably not a good idea to have your ops time collection on the same server you
        are tailing as it takes up a lot of space in the oplog.
    </li>
    <li>
        The next filter filters out “no-ops” that show up in the oplog, this will keep down the clutter as we are
        only looking for CRUD ops.
    </li>
    <li>
        Next we filter out fromMigrate, this removes operations from chunk migration and prevents false positives
        while tailing.

    </li>
    <li>
        Finally we get the last operation timestamp, this prevents the entire oplog from replaying every time we connect.
    </li>
</ul>
</p>
{% highlight java %}
private Bson getQueryFilter(MongoCollection</Document> tsCollection, MongoClientWrapper client) {
return and(ne("ns", "time_d.repl_time"), ne("op", "n"), exists("fromMigrate", false),getFilterLastTimeStamp(tsCollection, client));
}
{% endhighlight %}


<p>The time stamp filter keeps us from replaying the entire oplog every time we connect, if you notice this is
    querying MongoDB by the host, this corresponds to each of the host's server info that comes from a the sharded
    cluster.</p>
{% highlight java %}
private Bson getFilterLastTimeStamp(MongoCollection</Document> tsCollection, MongoClientWrapper client) {
    Document lastTimeStamp = tsCollection.find(new Document("host", client.getHost())).limit(1).first();
    return getTimeQuery(lastTimeStamp == null ? null : (BsonTimestamp) lastTimeStamp.get("ts"));
}

private Bson getTimeQuery(BsonTimestamp lastTimeStamp) {
    return lastTimeStamp == null ? new Document() : gt("ts", lastTimeStamp);
}
{% endhighlight %}
<p>
The MongoDB reactive driver uses the reactive streams standard. Reactive streams is rumored to be implemented in the Java 9
    release. This posed a bit of an issue while I was trying to bind it the RxJava, but there is a great translation
    library that wires the two approaches together,
    <a href="https://github.com/ReactiveX/RxJavaReactiveStreams" target="new">RxJavaReactiveStreams</a>.
    The bindPublisherToObservable method is called in a loop and attaches a subscriber onto an oplog tail per mongod
    that has been passed in.  This will create a thread per mongod. The code then uses a ConcurrentHashMap to
    increment the number of times a given operation is observed.  Once the map count is equal the replica set depth
    (the number of replicas per shard) the given operation has fully replicated to all replica sets.  At this point
    the operation is placed on an ArrayBlockingQueue opsQueue and removed from the map.  The opsQueue is used by the
    Flink data source to collect the operation and place it on the Flink work flow.
</p>
{% highlight java %}
private static final String OPLOG_TIMESTAMP = "ts";
private static final String OPLOG_ID = "h";

...

private void bindPublisherToObservable(Entry</String, FindPublisher</Document>> oplogPublisher,
    ExecutorService executor, MongoCollection</Document> tsCollection) {
    RxReactiveStreams.toObservable(oplogPublisher.getValue())
      .subscribeOn(Schedulers.from(executor)).subscribe(t -> {
        try {
            putOperationOnOpsQueue(oplogPublisher, tsCollection, t);
        } catch (InterruptedException e) {}
    });
}

private void putOperationOnOpsQueue(Entry</String, FindPublisher</Document>> publisher,
    MongoCollection</Document> tsCollection, Document t) throws InterruptedException {
    updateHostOperationTimeStamp(tsCollection, t.get(OPLOG_TIMESTAMP, BsonTimestamp.class), publisher.getKey());
    putOperationOnOpsQueueIfFullyReplicated(t);
}

private void putOperationOnOpsQueueIfFullyReplicated(Document t) throws InterruptedException {
    Long opKey = t.getLong(OPLOG_ID);
    documentCounter.putIfAbsent(opKey, new AtomicInteger(1));
    if (documentCounter.get(opKey).getAndIncrement() >= replicaDepth) {
        opsQueue.put(t);
        documentCounter.remove(opKey);
    }
}

{% endhighlight %}
<p>
    The code to run the oplog tail to elastic search replication is fairly simple:
</p>

{% highlight java %}
public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment see = StreamExecutionEnvironment.getExecutionEnvironment();
    DataStream</Document> ds = see.addSource(new MongoDBOplogSource("host", port));
    ds.addSink(new PrintSinkFunction</Document>());
    ds.addSink(new ElasticsearchEmbeddedNodeSink("cluster.name").getElasticSink());
    see.execute("MongoDB Sharded Oplog Tail");
}
{% endhighlight %}
<p> First we create the stream execution environment, next we add the MongoDB oplog source, and finally print and
    Elasticsearch sinks are added.  The PrintSinkFunction function was used to show how easy it is to attach multiple
    sinks to the execution environment.
</p>
<p>
    For this example I have not fully flushed out the Elasticsearch sink.  It is actually indexing the oplog documents
    and not creating a synchronized copy of the MongoDB collections.  I decided not to fully implement the Elasticsearh
    code as there are already a few decent projects out there that do this.  Also, I was only able to the the 1.7x
    Apache Flink connector for Elasticsearch working, as I unable find the 2.x connector on Maven Central.
</p>
<p>
    I had quite a bit of fun taking the reactive driver for MongoDB for a spin around the block.  Stream and event
    driven programming is something we all need to be aware of and it is just as important in the server side and
    big data worlds as it is in for Android and other font end development.
</p>