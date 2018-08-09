---
layout: post
title:  "MongoDB Transactions Demo"
categories: mongoDB
---
<h2>MongoDB Transactions Demo</h2>
<p>One of the major new features in the MongoDB 4.0 release is ACID transactions. For a quick refresher here is what
<a href="https://en.wikipedia.org/wiki/ACID_(computer_science)" target="new">Wikipedia</a>
    has to say on ACID transactions at the time of this article:</p>
<blockquote>
    In computer science, ACID (
    <a href="https://en.wikipedia.org/wiki/Atomicity_(database_systems)" target="new" >Atomicity</a>,
    <a href="https://en.wikipedia.org/wiki/Consistency_(database_systems)" target="new" >Consistency</a>,
    <a href="https://en.wikipedia.org/wiki/Isolation_(database_systems)" target="new" >Isolation</a>,
    <a href="https://en.wikipedia.org/wiki/Durability_(database_systems)" target="new" >Durability</a>
    ) is a set of properties of database
    transactions intended to guarantee validity even in the event of errors, power failures, etc. In the context of
    databases, a sequence of database operations that satisfies the ACID properties, and thus can be perceived as a
    single logical operation on the data, is called a transaction. For example, a transfer of funds from one bank
    account to another, even involving multiple changes such as debiting one account and crediting another, is a
    single transaction.
</blockquote>

<p>
    In this post I am going to cover three topics. First, a brief review of transaction syntax in other databases.
    Next, a quick overview of MongoDB transactions. Finally, a demonstration of MongoDB transactions using the Java
    driver.
</p>
<p>
    <h4>A brief review of transaction syntax</h4>
</p>
<p>
    If you are unfamiliar with database transactions it may be helpful to review how some of the popular relational
    databases handle them before we get into the MongoDB transactions. Let's take a look at the structure around MySQL,
    Oracle, and PostgreSQL for transactions. I will leave the more advanced topics for the reader to review. Perhaps
    I will prepare a deep dive in a later post if there is interest.
</p>
<p> <b>Example from the
    <a href="https://dev.mysql.com/doc/refman/8.0/en/commit.html" target="new">MySQL</a>
    docs:</b>

    {% highlight sql %}
    START TRANSACTION;
    SELECT @A:=SUM(salary) FROM table1 WHERE type=1;
    UPDATE table2 SET summary=@A WHERE type=1;
    COMMIT;
    {%  endhighlight %}

</p>
<p>
    <b>Example from the
    <a href="https://docs.oracle.com/database/121/CNCPT/transact.htm#CNCPT016" target="new">Oracle</a>
    docs:</b>
    {% highlight sql %}
    ...

    COMMIT;
    --  This statement ends any existing transaction in the session.

    SET TRANSACTION NAME 'sal_update2';
    -- This statement begins a new transaction in the session and names it sal_update2.

    UPDATE employees
    SET salary = 7050
    WHERE last_name = 'Banda';
    -- This statement updates the salary for Banda to 7050.

    UPDATE employees
    SET salary = 10950
    WHERE last_name = 'Greene';
    -- This statement updates the salary for Greene to 10950.

    COMMIT;
    -- This statement commits all changes made in transaction sal_update2, ending the transaction. The commit
    guarantees that the changes are saved in the online redo log files.

    {%  endhighlight %}
</p>
<p>
    <b>Example from the
    <a href="https://www.postgresql.org/docs/8.3/static/tutorial-transactions.html" target="new">PostgreSQL</a>
    docs:</b>
    {% highlight sql %}
    BEGIN;
    UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
    SAVEPOINT my_savepoint;
    UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
    -- oops ... forget that and use Wally's account
    ROLLBACK TO my_savepoint;
    UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Wally';
    COMMIT;
    {%  endhighlight %}

</p>
<p>
<br/>
<h4>A quick overview of MongoDB transactions</h4>
</p>
<p>
    Now, let us turn our attention to MongoDB transactions. For more information on MongoDB transactions, please see
    the official
    <a href="https://docs.mongodb.com/manual/core/transactions/" target="new"> documentation</a>.<br/>
    There are a few things from the documentation that I feel are very important to keep in mind.
<blockquote>
    <b>For transactions:</b>
<ul>
    <li>You can specify read/write (CRUD) operations on existing collections. The collections can be in different databases.</li>
    <li>You cannot read/write to collections in the config, admin, or local databases.</li>
    <li>You cannot write to system.* collections.</li>
    <li>You cannot return the supported operation’s query plan (i.e. explain).</li>
    <li>For cursors created outside of transactions, you cannot call getMore inside a transaction.</li>
    <li>For cursors created in a transaction, you cannot call getMore outside the transaction.</li>
</ul>

    <b>The following operations are not allowed in multi-document transactions:</b>
<ul>
    <li>Operations that affect the database catalog, such as creating or dropping a collection or an index. For
        example, a multi-document transaction cannot include an insert operation that would result in the creation of a new collection.</li>
    <li>The listCollections and listIndexes commands and their helper methods are also excluded.</li>
    <li>Non-CRUD and non-informational operations, such as createUser, getParameter, count, etc. and their helpers.</li>
    </ul>
</blockquote>

</p>
<p>
    The following code examples are from a MongoDB 4.0 transactions demo I wrote during the 3.7 beta testing and may
    be found
    <a href="https://github.com/JaiHirsch/mongo-4-demo" target="new">here</a>.
    This code base should be considered as demo/example quality.
</p>
<p>
    First, let’s take a look at the transaction structure as we have with the other databases. In this case, I am
    going to use the MongoDB java driver syntax as there is not a direct SQL translation.
</p>
<p>
    {% highlight java %}

    try (ClientSession clientSession = mongoClient.startSession()) {

        clientSession.startTransaction(TransactionOptions.builder()
            .writeConcern(WriteConcern.MAJORITY).build());

        inventoryCollection.updateOne(clientSession,
            Filters.eq("sku", "abc123"), Updates.inc("qty", amount));

        shipmentCollection.insertOne(clientSession, new Document("sku"
            , "abc123").append("qty", -amount)
            .append("tname", threadName));

        clientSession.commitTransaction();

    } catch (MongoException e) {
        throw new RuntimeException("Transaction failed: " + e);

    {%  endhighlight %}
</p>
<p>
    Once you get past the Java syntax, the structure is pretty much the same as in the other databases. Let's take a
    few of the key elements out and display them in less verbose pseudo code. (And yes, the MySQL, Oracle, or Postgres
    Java code is just as ugly if you are using the odbc/jdbc drivers directly.)<br/>
    {% highlight sql %}
    start transaction
        update an inventory record
        insert a shipment record
    commit transaction
    {%  endhighlight %}

</p>
<p>The structure of MongoDB transactions follows the same structure as the other examples.</p>
<p>
    <h4>A demonstration of MongoDB transactions using the Java driver</h4>
</p>
<p>
    We will begin by taking a look at the main class that runs the demo. One line to note is:<br/>
    <b>DemoMongoConnector dmc = new DemoMongoConnector()</b><br/>
    This class wraps
<a href="https://mongodb.github.io/mongo-java-driver/3.7/javadoc/com/mongodb/MongoClient.html" target="new">MongoClient</a>
        for convenience. The code may be found
<a href="https://github.com/JaiHirsch/mongo-4-demo/blob/master/src/main/java/org/mongo/DemoMongoConnector.java" target="new">
    here</a>.
</p>

{% highlight java %}
import com.mongodb.client.MongoDatabase;
import org.bson.Document;
import org.bson.assertions.Assertions;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

import static com.mongodb.client.model.Filters.eq;

public class MultiThreadTransactionRunner {

    private static final ExecutorService changeStreamExecutorService = Executors
        .newFixedThreadPool(2);
    private static final ExecutorService trnsactionExecutorService = Executors
        .newFixedThreadPool(10);

public static void main(String[] args) throws IOException {
        try (DemoMongoConnector dmc = new DemoMongoConnector()) {
            setUpMongoForTransactionTest(dmc);
            changeStreamExecutorService.submit(new ChangeStreamWatcher(dmc
                .getDatabase()));
            launchThreadsAndRunTransactions(dmc);
            Document skuAbc123 = (Document) dmc.getInventory().find(eq("sku",
                "abc123")).first();

            System.out.println("++++++++++ " + skuAbc123);

            Assertions.isTrue("qty should have been 500",
                500 == skuAbc123.getInteger("qty"));
            trnsactionExecutorService.shutdown();
        }
        changeStreamExecutorService.shutdown();
    }
{%  endhighlight %}
<p>
    The first thing we are doing after instantiating the connector is attaching a change stream watcher:
    {% highlight java %}
    changeStreamExecutorService.submit(new ChangeStreamWatcher(dmc
    .getDatabase()));
    {%  endhighlight %}
    <a href="https://docs.mongodb.com/manual/changeStreams/" target="new">Change streams</a>
    were introduced in MongoDB 3.6 and allow one to have an open query against the changes in the database. They
    also make several of my blog posts on oplog tailing mostly old news. (sad face) <br/>
    Change streams are outside the scope of this post, but we will use them to watch when the database accepts the
    writes during transactions. The change stream code may be found
    <a href="https://github.com/JaiHirsch/mongo-4-demo/blob/master/src/main/java/org/mongo/ChangeStreamWatcher.java" target="new">here</a>.
</p>
<p>
    Next, we set up the database environment for the transactional tests. The Collections to be used during a
    transaction <b>must</b> be created prior to the start of the transaction. For this testing scenario we will begin
    by dropping the existing "test" database and then recreating inventory and shipment collections. We will then
    insert an inventory document with a quantity value of 500.
</p>
{% highlight java %}
    private static void setUpMongoForTransactionTest(DemoMongoConnector dmc) {
        MongoDatabase db = dmc.getMongoClient().getDatabase("test");
        db.drop();
        db.createCollection("inventory");
        db.createCollection("shipment");
        dmc.getInventory().insertOne(new Document("sku", "abc123").append("qty", 500));
    }
{%  endhighlight %}
<p>
    Now it’s time to begin the transactions! The launch method uses the subsequent submit method to do the actual work,
    but the forEach calling <b>future.get()</b> is important here. This is a blocking call on the main thread that will
    cause the main thread to be blocked until each of the transaction threads have completed.
</p>
{% highlight java %}

    private static void launchThreadsAndRunTransactions(DemoMongoConnector dmc) {
        submitTransactionThreads(dmc).forEach(future -> {
            try {
                future.get();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    }
{%  endhighlight %}
<p>
    We’re almost to the good part, I swear! The <b>submitTransactionThreads</b> method is creating four transaction
    threads with the <b>TransactionRetryModule</b> that are going to increase the quantity of our item by one hundred,
    and four that will decrease the value by one hundred. Thus, a total of eight transaction threads will be created
    and placed into the <b>changeStreamExecutorService</b>. The second argument to the <b>iterateTransactions</b> will
    be used in a loop to fire the transaction multiple times. In this case, each thread will do its transaction five
    times. With the two combined arguments, each thread will increase or decrease the value by five hundred. This is
    done for testing only.
</p>
{% highlight java %}

    private static List<Future/> submitTransactionThreads(DemoMongoConnector dmc) {
        List<Future/> futures = new ArrayList<>();
        for (int i = 0; i < 4; i++) {
            futures.add(changeStreamExecutorService.submit(new TransactionRetryModule()
                .iterateTransactions(100, 5)));
        }
        for (int i = 0; i < 4; i++) {
            futures.add(changeStreamExecutorService.submit(new TransactionRetryModule()
                .iterateTransactions(-100, 5)));
        }
        return futures;
    }
{%  endhighlight %}
<p>
    Retries are <b>very</b> important for the MongoDB transaction logic.  The documentation gives
    <a href="https://docs.mongodb.com/manual/core/transactions/#retry-transaction" target="new">examples</a>
    for multiple languages.
</p>
<p>
    From the MongoDB transaction
    <a href="https://docs.mongodb.com/manual/core/transactions/#retry-transaction" target="new">documentation</a>:
    <blockquote>
    The individual write operations inside the transaction are not retryable, regardless of whether retryWrites is
    set to true.
    <br/><br/>
    If an operation encounters an error, the returned error may have an errorLabels array field. If the error is a
    transient error, the errorLabels array field contains "TransientTransactionError" as an element and the
    transaction as a whole can be retried.
    </blockquote>
    I highly recommend reading up on the
    <a href="https://martinfowler.com/bliki/CircuitBreaker.html" target="new">circuit breaker</a> pattern if you
    implement retry patterns in production code.  The
    <a href="https://github.com/Netflix/Hystrix" target="new"> Netflix Hystrix</a> library is one of the more popular
    implementations that I have worked with.  It is, however, rather heavy-weight for this demo.  A good post
    from DZone on retries can be found
<a href="https://dzone.com/articles/understanding-retry-pattern-with-exponential-back" target="new">here</a>.

</p>
<p>
    We are finally ready to review the
    <a href="https://github.com/JaiHirsch/mongo-4-demo/blob/master/src/main/java/org/mongo/TransactionRetryModule.java" target="new">
        TransactionRetryModule</a>.  The entry point into the class is the <b>iterateTransactions</b> method. This
    method creates a runnable lambda that is used in our multi-threaded model. Next, we create a new
    <b>DemoMongoConnector</b>. While one could use the connector from the main class, I wanted to imitate the
    transactions coming from multiple distinct applications. Next, we generate a random name for logging to show the
    order in which the transactions are being executed. Then, we loop and fire a new transaction for the number of
    iterations we passed into the runnable. Finally, the method calls

    {% highlight java %}
    handleTransactionClientSession(amount, dmc, threadName);
    {%  endhighlight %}

    which will begin the transaction session.
</p>

{% highlight java %}
import com.mongodb.TransactionOptions;
import com.mongodb.WriteConcern;
import com.mongodb.client.ClientSession;
import com.mongodb.client.model.Filters;
import com.mongodb.client.model.Updates;
import org.apache.commons.lang3.RandomStringUtils;
import org.bson.Document;
import org.mongo.utils.Retry;

public class TransactionRetryModule {

    private static final int MAX_RETRIES = 10;
    private static final long DELAY_BETWEEN_RETRIES_MILLIS = 30L;

    public Runnable iterateTransactions(final int amount, final int iterations) {
        return () -> {
            try (DemoMongoConnector dmc = new DemoMongoConnector()) {
                String threadName = RandomStringUtils.randomNumeric(4);
                for (int i = 0; i < iterations; i++) {
                    handleTransactionClientSession(amount, dmc, threadName);
                }
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };

    }
{%  endhighlight %}
<p>
    The <b>handleTransactionClientSession</b> method starts the ClientSession and enters the transaction retry loop.
</p>
{% highlight java %}
private void handleTransactionClientSession(int amount, DemoMongoConnector dmc,
        String threadName) {
    try (ClientSession clientSession = dmc.getMongoClient().startSession()) {
        transactionRetryLoop(amount, dmc, threadName, clientSession);
    } catch (Exception e) {

        throw new RuntimeException("Transaction failed: " + e);
    }
}
{%  endhighlight %}
<p>
    The <b>transactionRetryLoop</b> method begins by instantiating a
    <a href="https://github.com/JaiHirsch/mongo-4-demo/blob/master/src/main/java/org/mongo/utils/Retry.java" target="">Retry</a>
    object that is used to control the loop. Next, we enter the loop and attempt the transaction. If successful, we
    mark the retry loop as complete. If an error is thrown from the transaction, the transaction will be aborted and
    retried. The sysouts are for demo only. Friends don't let friends sysout in production. Finally, if the retry loop
    exits unsuccessfully after maximum attempts, an error will be thrown.
</p>
{% highlight java %}
private void transactionRetryLoop(int amount, DemoMongoConnector dmc, String threadName,
        ClientSession clientSession) {
    Retry retryLoop = new Retry().withAttempts(MAX_RETRIES)
        .withDelay(DELAY_BETWEEN_RETRIES_MILLIS);
    while (retryLoop.shouldContinue()) try {

        System.out.println(threadName + " : " + retryLoop.getTimesAttempted());

        doTransaction(amount, dmc, threadName, clientSession);
        retryLoop.markAsComplete();

        System.out.println(threadName + " complete : " + retryLoop.completedOk());

    } catch (Throwable e) {
        retryLoop.takeException(e);
        clientSession.abortTransaction();

        System.out.println(threadName + " Aborting transaction: " + e.getMessage());
    }
    if (!retryLoop.completedOk()) {
        throw new RuntimeException("Transaction failed after " + MAX_RETRIES
            + " retries.", retryLoop.getLastException());
    }
}
{%  endhighlight %}
<p>
    We made it! The <b>doTransaction</b> method is where the magic finally happens. We begin the transaction, we
    perform an update on the inventory collection, and make an insert to the shipment collection. Finally, we commit
    the transaction.
</p>
{% highlight java %}
private void doTransaction(int amount, DemoMongoConnector dmc, String threadName,
        ClientSession clientSession) {

   clientSession.startTransaction(TransactionOptions.builder()
        .writeConcern(WriteConcern.MAJORITY).build());

    dmc.getInventory().updateOne(clientSession, Filters.eq("sku", "abc123"),
        Updates.inc("qty", amount));
    dmc.getShipment().insertOne(clientSession, new Document("sku", "abc123")
        .append("qty", -amount).append("tname", threadName));

    clientSession.commitTransaction();
}
{%  endhighlight %}
<p>
    The standard out for a run of the demo can be found
    <a href="https://github.com/JaiHirsch/mongo-4-demo/blob/master/example_output.txt" target="new">here</a>.  It is
    rather verbose to paste into this post, but if examine it you will see the flow of aborted transactions and how
    the change streams show the successful ones.
</p>
<p>
    <h4>Conclusion</h4>
</p>
<p>
    Transactions are a big step forward for MongoDB. In the 4.0 release, they are limited to replica sets, but sharded
    clusters are on the road map. As with most new additions to a technology, I highly recommend doing extensive
    testing yourself prior to running it in production. Remember, even with transactions, it is your responsibility to
    ensure the integrity of your data. Any time your data model becomes more complex, so does the array of issues that
    can arise.
    </p>
<p>
    I hope this review has been helpful, and feel free to test out the code for yourself.
</p>

