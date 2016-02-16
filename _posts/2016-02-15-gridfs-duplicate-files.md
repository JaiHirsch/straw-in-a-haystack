---
layout: post
title:  "Finding Duplicate Files in GridFS"
categories: mongoDB
---
<h2>Finding Duplicate Files in GridFS</h2>
<p>I have been working on several large scale GridFS projects lately and the topic of finding, removing, and preventing
    duplicate files keeps coming up. As such, here are some of my findings and code samples around this topic.</p>
<br>
<p>First of all, what is GridFS?</p>
<blockquote cite="https://docs.mongodb.org/manual/reference/glossary/#term-gridfs">
    A convention for storing large files in a MongoDB database. All of the official MongoDB drivers support this
    convention, as does the mongofiles program.
    <p>...</p>
    GridFS stores files in two collections:
    <ul>
        <li>chunks stores the binary chunks. For details, see <a
                href="https://docs.mongodb.org/manual/core/gridfs/#the-chunks-collection">The chunks Collection.</a>
        </li>
        <li>files stores the file’s metadata. For details, see <a
                href="https://docs.mongodb.org/manual/core/gridfs/#the-files-collection">The files Collection.</a></li>
    </ul>
    GridFS places the collections in a common bucket by prefixing each with the bucket name. By default, GridFS uses two
    collections with a bucket named fs:
    <ul>
        <li>fs.files</li>
        <li>fs.chunks</li>
    </ul>
</blockquote>


<p>-- <a href="https://docs.mongodb.org/manual/reference/glossary/#term-gridfs">MongoDB Manual</a></p>

<p>
    Within the the files collection GridFS gives you the md5 hash of the file when it is uploaded.
    This does appear to
    a perfect candidate for identifying duplicate files and when you search google on this topic you will quickly find
    examples, gists and stackoverflow posts on how to use the md5 to identify duplicate files.
</p>


<p>Example file header:</p>
{% highlight javascript %}
{
   "_id" : ObjectId("56bf88d782419304bf212014"),
   "filename" : "foo.txt",
   "aliases" : null,
   "chunkSize" : NumberLong(261120),
   "uploadDate" : ISODate("2016-02-13T19:49:43.575Z"),
   "length" : NumberLong(541),
   "contentType" : null,
   "md5" : "3813e09a727083f74b45fe7f5b253c07"
}
{% endhighlight %}

<p>
    Here is an example using the aggregation pipeline that groups my md5, pushes the _ids into an array, and matches
    where there are two or more md5 matches:
    {% highlight javascript %}
    db.fs.files.aggregate([{$group:{_id:'$md5',count:{$sum:1},ids:{$push:'$_id'}}},{$match:{count:{$gte:2}}}])
    {% endhighlight %}
    It is also important to note that for this aggregation to work well there should be an index on md5 field. This
    could
    be a very costly index on a large implementation.
</p>

<p>
    So if there are so many posts out there and if this is a solved problem then why am I writing about it? Because
    you should never trust an md5 hash, especially for binary files. Lets take a moment for a closer look at the
    md5.
</p>

<blockquote cite="">
    The MD5 message-digest algorithm is a widely used cryptographic hash function producing a 128-bit (16-byte) hash
    value, typically expressed in text format as a 32 digit hexadecimal number. MD5 has been utilized in a wide variety
    of cryptographic applications, and is also commonly used to verify data integrity.
</blockquote>
<p>-- <a href="https://en.wikipedia.org/wiki/MD5">Wikipedia</a></p>

<p>There are many scholarly works written by much smarter people than I available on the probability of md5 collision,
    so I will not spend much time going over it. I have, however, encountered the consequences of hash
    collisions before and it is never fun.
</p>
<p>Unfortunately, the files that I have dealt with before, that had collisions, are proprietary and I can not share them
    here, so I took to the googles and found some interesting utilities and articles.
</p>
<p>Here are two that I liked in particular:
<ul>
    <li><a href="https://marc-stevens.nl/p/hashclash/">Project HashClash</a></li>
    <li><a href="http://www.mscs.dal.ca/~selinger/md5collision/">MD5 Collision Demo</a></li>
</ul>
</p>

<p>
    To perform my GridFS experiment I used a few jpg files that were created by Nat McHugh and may be found on his
    blog post:

    <a href="http://natmchugh.blogspot.com/2014/10/how-i-created-two-images-with-same-md5.html">How I created two images
        with the same MD5 hash</a> (He created the third one later)
</p>
<p><img src="http://www.fishtrap.co.uk/barry.jpg.coll"></p>
<p><img src="http://www.fishtrap.co.uk/james.jpg.coll"></p>
<p><img src="http://www.fishtrap.co.uk/black.jpg.coll"></p>
<br><br>

<p>You just can't go wrong with White, Brown, and Black. These images are obviously different, yet they all have the
    same md5 hash. What is more, they also show the same chunkSize and length information in the GridFS files document.
    Due to the fact that these three fields are identical we can not use the aggregation framework as noted thus far
    to identify that they are, in fact, not duplicates.
</p>
<p>There are many ways other ways to identify if two files are duplicates. Lets take a look at one such approach that
    also takes advantage the file streaming aspect of GridFS.</p>
<p><b>The following example code is written in Java and should be considered demo quality only</b>.</p>

<p>First you must get the input stream from the GridFS file</p>

{% highlight java %}
GridFSDBFile findFile1 = gridFS.find(docemnt);
InputStream is1 = findFile1.getInputStream();
{% endhighlight %}

<p>Once we get both of the input streams we want to compare we can do so as follows:</p>

{% highlight java %}
public boolean isDuplicateStream(InputStream is1, InputStream is2) {
   boolean isSame = true;
   int val1, val2 = -1;
   do {
      try {
         val1 = is1.read();
         val2 = is2.read();
         if (val1 != val2) {
            isSame = false;
            break;
         }
      } catch (IOException e) {
         logger.warn(e.getMessage());
         isSame = false;
         break;
      }

   } while (val1 != -1 && val2 != -1);  // important to check both to make sure both are completely read
   return isSame;
}
{% endhighlight %}

<p>I will be the first to admit, this code is rather ugly, but it works well enough for this.  Using streams has the
advantage that you do not have to materialize the file in order to check if they are duplicates.  The downside is that
it will take linear time to scan both files.</p>
<p> The GridFS documentation does mention ranged queries and the ability to skip to arbitrary locations in the file.
This should allow us to write a threaded file validator, so look forward to part two of the post on optimization.</p>
<p>The full project I used to may be found <a href="https://github.com/JaiHirsch/gridfs-duplicate-cleaner">here</a></p>
