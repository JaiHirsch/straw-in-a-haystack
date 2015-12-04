---
layout: post
title:  "Thoughts on MongoDB 3.2 Document Validation"
categories: mongoDB
---
<h2>Thoughts on MongoDB 3.2 Document Validation</h2>
<p>Sometimes you want completely free form documents and sometimes you don’t. In the 3.2 release of MongoDB the idea of
    document validation was introduced.  One area I have encountered problems in the past is with dates being inserted
    using different data types.</p>
<b>Example:</b> (command line notation used for brevity)
<br>
<ul>
    <li>The first team was using numeric dates: {% highlight javascript %} db.date.insert({date:20150321}) {% endhighlight %}</li>
    <li>The second team was using string format: {% highlight javascript %} db.date.insert({date:"03/21/2015"}){%  endhighlight %} </li>

<li>The third team was using ISO dates: {% highlight javascript %} db.date.insert({date:new Date('Mar 21, 2015')}) {% endhighlight %}</li>
</ul>
<br>
<p>The communication issues that lead to this problem is a story for another time, but needless to say a fair amount of
    data cleanup was required to fix the issue as the primary application that read the data was expecting an ISO date. 
    The code issue was solved by using a shared library that defined the database mapping object.  Now let’s take a
    look at how document validation could be used to solve this problem.</p>
<p>We will create a collection with the following command:</p>
{% highlight javascript %}
db.createCollection( "date", {
    validator: { date: {$type:"date"}}
} )
{%  endhighlight %}
<p>With the collection created with the validator one can now only insert using the new Date() format:</p>
{% highlight javascript %}
db.date.insert({date:new Date('Mar 21, 2015')})
{%  endhighlight %}
<p>The numeric and string inserts will give the following error:</p>

{% highlight javascript %}
WriteResult({
    "nInserted" : 0,
    "writeError" : {
        "code" : 121,
        "errmsg" : "Document failed validation"
    }
})
{%  endhighlight %}
<p>In my mind this error message leaves something to be desired. We do not know why we failed the validation, just
that the document failed.</p>
<p>There is another interesting side effect in the preceeding validation. Now all documents in the date collection are
    required to contain the date field.  What if, however, all of your documents do not contain the date field?  We
    can get around this problem with the following validator:</p>
<br>
{% highlight javascript %}
db.createCollection( "date", {
    validator: { $or:
        [
            { date: { $type: "date" } },
            { date: { $exists: false } }
        ]
    }
} )
{%  endhighlight %}
<p>The date field is now optional, but if it exists it must be of type date.</p>
<p>Let’s make it a bit more interesting put an additional requirement that if the date exists it must be after
January first 2015:</p>
<br>
{% highlight javascript %}
db.createCollection( "date", {
    validator: { $or:
        [
            { date: { $type: "date",
              $gte: new Date('Jan 1, 2015') } },
            { date: { $exists: false } }
        ]
    }
} )
{%  endhighlight %}
<p>By using the $or operator mixed with the implicit "$and" operator we can now logically divide the validation
requirements.</p>
 
<h3>Some other interesting cases.</h3>

<p><b>Integers vs floating-point values</b></p>

<p>Almost every developer new to MongoDB will make this "mistake" on the command line:</p>
{% highlight javascript %}
db.numbers.insert({number:2})
{%  endhighlight %}
<p>One would think that they have inserted a document containing the integer number two into the collection, when
in fact the document <a href="https://docs.mongodb.org/manual/core/shell-types/#numberint">now contains</a> a
floating-point value.</p>

<p>I can attest to the pain this can cause (especially if you are using a strongly typed language such as JAVA).</p>

<p>We can use the validator to safeguard against this:</p>

{% highlight javascript %}
db.createCollection( "numbers", {
    validator: { number: {$type:"int" }}
} )
{%  endhighlight %}
<p>The preceding insert will now give us the 121 error code and you need to use the NumberInt constructor:</p>
{% highlight javascript %}
db.numbers.insert({number:NumberInt(2)})
{%  endhighlight %}

<p><b>Oh noes, my collection can't haz documents!!!</b></p>
<p>At the time of writing it is quite possible to mess up your validation and make the collection non-writeable.
Consider the following example:</p>

{% highlight javascript %}
db.createCollection( "dates", {
    validator: {
        date: {$type:"date"},
        date: {$type:"int"}
    }
} )
{%  endhighlight %}
<image src="http://orig07.deviantart.net/47d0/f/2012/240/d/c/oh_noes_cat_plz_by_ohnoescatplz-d5crycw.png"></image>
<br>
<p>We have made the dates collection require a date field that is both an integer and ISO date type. So who validates
the validator? At this point it appears to be you at definition time. In future releases this may be something that
MongoDB Compass addresses</p>
<br>
<iframe width="480" height="360" src="https://www.youtube.com/embed/XfLeO_LE6ic" frameborder="0"></iframe>
<br>
<p>Perhaps, however, all we really wanted to do was say that the date field can be either an integer or an ISO date.
 In that case we can solve the problem with the following:</p><br>
{% highlight javascript %}
db.createCollection( "date", {
    validator: { $or:
        [
            { date: { $type: "date" } },
            { date: { $type: "int" } }
        ]
    }
} )
{%  endhighlight %}
<br>
<p><b>Almost referential integrity?</b></p>
<p>MongoDB 3.2 also introduces some new aggregation operators. One of the more interesting ones is $lookup. This
allows you to run aggregations across multiple collections. However, you do not appear to be able to hack it
into a foreign key constraint for validation (yet).</p>
<h3>But what about performance?</h3>
<p>I was expecting to see some slow downs on loads when using validation so I set up some load tests using
<a href="http://jmeter.apache.org">Apache JMeter</a>.  JMeter now comes with nice built in MongoDB testing tools
and I was all excited about posting pretty graphs showing how much validation slowed down the loading process.
However, there was next to no cost for using it!  As such, I decided not to include these graphs but instead simply
recommend that you profile your own application as necessary.</p>
<br>
<p>I hope you have enjoyed this quick overview of MongoDB 3.2 validation. More information may be found in the
<a href="https://docs.mongodb.org/manual/release-notes/3.2/">Development Release Notes for 3.2.0 Release
    Candidate</a> </p>
