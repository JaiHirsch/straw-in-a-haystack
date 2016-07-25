---
layout: post
title:  "Mocking the MongoDB Java driver"
categories: mongoDB
---
<p>
<h2>Mocking the MongoDB Java driver</h2>
</p>
<p>While at MongoDB World 2016 I was involved in multiple conversations about unit testing and test driven development
of data access objects that wrap the MongoDB Java driver.  Not two days after returning to work from MongoDB World
one of the teams I work with was complaining that the 3.x Java driver broke a lot of their testing patterns and that
they were having difficulties mocking out the driver.  I had a few meetings canceled so I decided to sit down and
create a testing pattern that could be easily implemented and extended using the Mockito libraries.
</p>
<p>A side note before we get started. If you are mocking at the driver layer you may want to take a step back and
consider why you are doing it. I am not going to dive deep into the philosophy of unit testing. My general rule of
thumb is to only test logic, avoid the file system, and assume that the external systems work as advertised. Avoid
the pitfalls of only testing mocks for the sake of testing and make sure you set up integration layers to verify
that the external systems do, in fact, work as advertised.  That being said, all the code for the following examples
may be found <a href="https://github.com/JaiHirsch/mongoldb-testing">here</a>.</p>

<p>Lets begin with setting up our base mocks and injecting them into the class we will be working on.</p>

{% highlight java %}
@Mock
private MongoClient mockClient;
@Mock
private MongoCollection mockCollection;
@Mock
private MongoDatabase mockDB;

@InjectMocks
private DriverWrapper wrapper;

@Before
public void initMocks() {
   when(mockClient.getDatabase(anyString())).thenReturn(mockDB);
   when(mockDB.getCollection(anyString())).thenReturn(mockCollection);
   wrapper.init();
   MockitoAnnotations.initMocks(this);
}
{% endhighlight %}
<p>I am using the <a href="http://site.mockito.org/mockito/docs/current/org/mockito/InjectMocks.html" target="new">
   Mockito's InjectMocks</a> which will inject the mock MongoClient into the DriverWrapper class.
   DriverWrapper is the class under test in this case.  Using the Junit @Before annotation reduces the amount boilerplate
   code needed in each test as this method will run before each test.</p>

<p>For the first test lets try something simple with a test for a method that finds documents by a last name field</p>

{% highlight java %}
@Test
public void findBob() {
   FindIterable iterable = mock(FindIterable.class);
   MongoCursor cursor = mock(MongoCursor.class);
   Document bob = new Document("_id",new ObjectId("579397d20c2dd41b9a8a09eb"))
      .append("firstName", "Bob")
      .append("lastName", "Bobberson");

   when(mockCollection.find(new Document("lastName", "Bobberson")))
      .thenReturn(iterable);
   when(iterable.iterator()).thenReturn(cursor);
   when(cursor.hasNext()).thenReturn(true).thenReturn(false);
   when(cursor.next()).thenReturn(bob);

   List<Document> found = wrapper.findByLastName("Bobberson");

   assertEquals(bob, found.get(0));
}
{% endhighlight %}

<p>And here is the method under test:</p>

{% highlight java %}
public List<Document> findByLastName(String lastName) {
   List<Document> people = new ArrayList<>();
   FindIterable<Document> documents = collection.find(new Document("lastName", lastName));
   MongoCursor<Document> iterator = documents.iterator();
   while (iterator.hasNext()) {
      Document document = iterator.next();
      people.add(document);
   }
   return people;
}
{% endhighlight %}

<p>So this works, but the code feels pretty bulky. Following the red/green/refactor principles lets clean up the
implementation of the findByLastName method.  Looking over the Java api there is a into method that will will replace
the body of the code above:</p>

{% highlight java %}
public List<Document> findByLastNameUsingInto(String lastName) {
   return collection.find(new Document("lastName", lastName)).into(new ArrayList<>());
}
{% endhighlight %}

<p>Well, that reduced the line count of the method by a bit... It did however break the test as we need to mock out
different parts of the code and needs to be updated to reflect the new implementation:</p>

{% highlight java %}
@Test
public void findBobUsingInto() {
   FindIterable iterable = mock(FindIterable.class);
   MongoCursor cursor = mock(MongoCursor.class);
   Document bob = new Document("_id",new ObjectId("579397d20c2dd41b9a8a09eb"))
      .append("firstName", "Bob")
      .append("lastName", "Bobberson");

   when(mockCollection.find(new Document("lastName", "Bobberson"))).thenReturn(iterable);
   when(iterable.into(new ArrayList<>())).thenReturn(asList(bob));

   List<Document> found = wrapper.findByLastNameUsingInto("Bobberson");
   assertEquals(bob, found.get(0));
}
{% endhighlight %}

<p>While the code has been reduced down to one line the test still has a lot of boilerplate that will need to be
    "duplicated" in the next test. I am a fan of the builder pattern for problems like this so lets take a look at
    a way to implement this pattern to create a more generic tool for setting up the tests.
</p>

{% highlight java %}
public class MockCursorBuilder {

    private FindIterable iterable = mock(FindIterable.class);
    private MongoCursor cursor = mock(MongoCursor.class);
    private MongoCollection mockCollection;

    public MockCursorBuilder(MongoCollection mockCollection) {

        this.mockCollection = mockCollection;
        when(iterable.iterator()).thenReturn(cursor);
    }

    public MockCursorBuilder withQuery(Document query) {
        when(mockCollection.find(query)).thenReturn(iterable);
        return this;
    }

    public MockCursorBuilder usingInto(Document... documents) {
        when(iterable.into(new ArrayList<>())).thenReturn(asList(documents));
        return this;
    }
}
{% endhighlight %}

<p>Using this approach we are passing the mock MongoCollection we created in the test as a constructor parameter.
We then set collection as a field.  We internalize the mock MongoCursor and FindIterable classes that we needed to
set up per test. Finally we use the builder pattern to define the mocking of the find and into methods. The test
may now be rewritten as:</p>

{% highlight java %}
@Test
public void findBobUsingIntoAndMockBuilder() {
    Document bob = new Document("_id",new ObjectId("579397d20c2dd41b9a8a09eb"))
        .append("firstName", "Bob").append("lastName", "Bobberson");
    new MockCursorBuilder(mockCollection)
        .withQuery(new Document("lastName", "Bobberson")).usingInto(bob);
    assertEquals(bob,wrapper.findByLastNameUsingInto("Bobberson").get(0));
}
{% endhighlight %}
<p>
    Using this new builder it becomes much easier to create a test with multiple Documents returned by the query.
    The main difference in this test is the use of the Hamcrest library's IsIterableContainingInOrder class.  This
    allows us to test the order of a returned list against an expected list in a direct assertion.
</p>
{% highlight java %}
@Test
public void findMultipleUsers() {
    Document bob = new Document("_id",new ObjectId())
        .append("firstName", "Bob").append("lastName", "Bobberson");
    Document robert = new Document("_id",new ObjectId())
        .append("firstName", "Robert").append("lastName", "Bobberson");
    Document joe = new Document("_id",new ObjectId())
        .append("firstName", "JoeBob").append("lastName", "Bobberson");
    Document mark = new Document("_id",new ObjectId())
        .append("firstName", "MarkBob").append("lastName", "Bobberson");
    new MockCursorBuilder(mockCollection)
        .withQuery(new Document("lastName", "Bobberson"))
        .usingInto(bob, robert, joe, mark);
    Assert.assertThat(wrapper.findByLastNameUsingInto("Bobberson"),
        IsIterableContainingInOrder.contains(bob, robert, joe, mark));
}
{% endhighlight %}

<p>
    This concludes my basic implementation of using Mockito to test implementations of the MongoDB Java driver.  I
    hope you have found it useful. There are some other patterns and tests in the source code that I am working on
    and my publish more on this later.
</p>


