---
layout: post
title: "Writing Reactive Apps with ReactiveMongo and Play, Pt. 3 - GridFS"
date: 2012-11-28
---

<p class="notice">
<a href="http://reactivemongo.org">ReactiveMongo</a> is a brand new Scala driver for MongoDB. More than just yet-another-async-driver, it's a reactive driver that allows you to design very scalable applications unleashing MongoDB capabilities like streaming infinite live collections and files for modern Realtime Web applications.
</p>

In the [previous article](/2012/10/29/writing-a-simple-app-with-reactivemongo-and-play-framework-pt-2.html), we saw how to insert and update documents, and to do more complex queries like sorting documents.

There is one feature that our application lacks sor far: the ability to upload attachments. For this purpose we will use GridFS, a protocol over MongoDB that handles file storage.

> #### How does GriFS work?
> When you save some file with GridFS, you are actually dealing with two collections: `files` and `chunks`. The metadata of the file are saved as a document into the `files` collection, while the contents are splitted into chunks (usually of 256kB) that are stored into the `chunks` collection. Then one can find files by performing regular queries on `files`, and retrieve their contents by merging all their chunks.

### Summary

<ul class="summary">
	<li>
    <a href="#using_gridfs_with_reactivemongo">Using GridFS with ReactiveMongo</a>
    <ul>
      <li><a href="#a_look_to_reactivemongos_gridfs_api">A Look to ReactiveMongo's GridFS API</a></li>
    </ul>
  </li>

	<li>
    <a href="#using_reactivemongo_gridfs_streaming_capability_with_play_framework">Using ReactiveMongo GridFS Streaming Capability with Play Framework</a>
    <ul>
      <li><a href="#save_a_file_into_gridfs_the_streaming_way">Save a File into GridFS, the Streaming Way</a></li>
      <li><a href="#stream_a_file_to_the_client">Stream a File to the Client</a></li>
    </ul>
  </li>
	<li><a href="#whats_next">What's Next?</a></li>
</ul>

### Using GridFS with ReactiveMongo

One of the main design principles of ReactiveMongo is that all the documents can be streamed both from and into MongoDB. The streaming ability is implemented with the Producer/Consumer pattern in its immutable version - as known as the Enumerator/Iteratee pattern:
- an `Iteratee` is a consumer of data: it processes chunks of data. An `Iteratee` instance is in one of the following states: done (meaning that it may not process more chunks), cont (it accepts more data), and error (an error has occurred). After each step, it returns an new instance of Iteratee that may be in a different state;
- an `Enumerator` is a producer, or a source, of data: applied on an iteratee instance, it will feed it (depending on its state).

Consider the case of retrieving a lot of documents. Building a huge list of documents in your application may fill up the memory quickly, so this is not an option. You could get a lazy iterator of documents, but the problem is that it's blocking. And its non-blocking counterpart - an iterator of future documents - may be difficult to handle. That's where Iteratees and Enumerators come to the rescue. The idea is to see the MongoDB server as the procucer (so, an Enumerator), and your code using these documents as the consumer.

The same vision can apply to GridFS too. When storing a file, GridFS is the consumer (so we will use an Iteratee); when reading one, it is seen as a producer, so an Enumerator.

Using Iteratees and Enumerators, you can stream files all along, from the client to the server and the database, keeping a low memory usage and in a completely non-blocking way.

#### A Look to ReactiveMongo's GridFS API

The `GridFS.save` is used to get an `Iteratee`:

{% highlight scala %}
/**
 * Saves a file with the given name.
 *
 * If an id is provided, the matching file metadata will be replaced.
 *
 * @param name the file name.
 * @param id an id for the new file. If none is provided, a new ObjectId will be generated.
 *
 * @return an iteratee to be applied to an enumerator of chunks of bytes.
 */
def save(name: String, id: Option[BSONValue], contentType: Option[String] = None)(implicit ctx: ExecutionContext) :Iteratee[Array[Byte], Future[ReadFileEntry]]
{% endhighlight %}

This `Iteratee` will be fed by an `Enumerator[Array[Byte]]` - the contents of the file.

When the file is entirely saved, a `Future[ReadFileEntry]` may be retrieved. This is the metadata of the file, including the id (generally a `BSONObjectID`).

{% highlight scala %}
/**
 * Metadata of a file.
 */
trait FileEntry {
  /** File name */
  val filename: String
  /** length of the file */
  val length: Int
  /** size of the chunks of this file */
  val chunkSize: Int
  /** the date when this file was uploaded. */
  val uploadDate: Option[Long]
  /** the MD5 hash of this file. */
  val md5: Option[String]
  /** mimetype of this file. */
  val contentType: Option[String]
  /** the GridFS store of this file. */
  val gridFS: GridFS
}


/** Metadata of a file that exists as a document in the `files` collection. */
trait ReadFileEntry extends FileEntry {
  /** The id of this file. */
  val id: BSONValue

  // ...
}
{% endhighlight %}

On a `ReadFileEntry`, we can call a function named `enumerate` that returns an `Enumerator[Array[Byte]]`... and the circle is complete.


{% highlight scala %}
/** Produces an enumerator of chunks of bytes from the `chunks` collection. */
def enumerate(implicit ctx: ExecutionContext) :Enumerator[Array[Byte]]
{% endhighlight %}

For example, here is a way to store a plain old `java.io.File` into GridFS:

{% highlight scala %}
// a function that saves a file into a GridFS storage and returns its id.
def writeFile(file: File, mimeType: Option[String]) :Future[BSONValue] = {
  // on a database instance
  val gfs = new GridFS(db)

  // gets an enumerator from this file
  val enumerator = Enumerator.fromFile(file)

  // gets an iteratee to be fed by the enumerator above
  val iteratee = gfs.save(file.getName(), None, mimeType)

  // we get the final future holding the result (the ReadFileEntry)
  val future = Iteratee.flatten(enumerator(iteratee)).run
  // future is a Future[Future[ReadFileEntry]], so we have to flatten it
  val flattenedFuture = future.flatMap(f => f)
  // map the value of this future to the id
  flattenedFuture.map { readFileEntry =>
    println("Successfully inserted file with id " + readFileEntry.id)
    readFileEntry.id
  }
}
{% endhighlight %}


### Using ReactiveMongo GridFS Streaming Capability with Play Framework

So, how can we use this for our attachments?

#### Save a file into GridFS, the streaming way

First, let's take a look at how file upload is handled in Play Framework. This is achieved with a [BodyParser](http://www.playframework.org/documentation/2.0/ScalaFileUpload).

A `BodyParser` is function that handles the body of request. For example, files are often sent with `multipart/form-data` (using a HTML form element).

A regular file upload in Play looks like this:

{% highlight scala %}
def upload = Action(parse.multipartFormData) { request =>
  request.body.file("file").map { file =>
    // do something with the file...
    Ok("File uploaded")
  }.getOrElse {
    Redirect(routes.Application.index).flashing(
      "error" -> "Missing file"
    )
  }
}
{% endhighlight %}

Obviously, you don't need to write your own `BodyParser` for ReactiveMongo. You can simply use the one that is provided in the [Play ReactiveMongo Plugin](https://github.com/zenexity/Play-ReactiveMongo). It is defined on the `MongoController` trait, which is a mixin for Controllers.

{% highlight scala %}
/**
 * A mixin for controllers that will provide MongoDB actions.
 */
trait MongoController {
  self :Controller =>
  // ...
  /**
   * Gets a body parser that will save a file sent with multipart/form-data into the given GridFS store.
   */
  def gridFSBodyParser(gfs: GridFS)(implicit ec: ExecutionContext) :BodyParser[MultipartFormData[Future[ReadFileEntry]]]
  // ...
}
{% endhighlight %}

Thanks to this function, all we need to do in our controller is to provide this `BodyParser` to the Action:

{% highlight scala %}
def saveAttachment(id: String) = Action(gridFSBodyParser(gridFS)) { request =>
    val future :Future[ReadFileEntry] = request.body.files.head.ref
    // ...
}
{% endhighlight %}

Simple, right?

In our case, we will need to update the `ReadFileEntry` to add a metadata: the article which this attachment belongs to. This can be achieved with a for-comprehension:

{% highlight scala %}
// save the uploaded file as an attachment of the article with the given id
def saveAttachment(id: String) = Action(gridFSBodyParser(gridFS)) { request =>
  // the reader that allows the 'find' method to return a future Cursor[Article]
  implicit val reader = Article.ArticleBSONReader
  // first, get the attachment matching the given id, and get the first result (if any)
  val cursor = collection.find(BSONDocument("_id" -> new BSONObjectID(id)))
  val uploaded = cursor.headOption

  val futureUpload = for {
    // we filter the future to get it successful only if there is a matching Article
    article <- uploaded.filter(_.isDefined).map(_.get)
    // we wait (non-blocking) for the upload to complete. (This example does not handle multiple files uploads).
    putResult <- request.body.files.head.ref
    // when the upload is complete, we add the article id to the file entry (in order to find the attachments of the article)
    result <- gridFS.files.update(BSONDocument("_id" -> putResult.id), BSONDocument("$set" -> BSONDocument("article" -> article.id.get)))
  } yield result

  Async {
    futureUpload.map {
      case _ => Redirect(routes.Articles.showEditForm(id))
    }.recover {
      case _ => BadRequest
    }
  }
}
{% endhighlight %}


#### Stream a File to the Client

The Play Plugin also provides a `serve` function that will stream the first `ReadFileEntry` available in the `Cursor` instance given as a parameter:

{% highlight scala %}
def serve(foundFile: Cursor[ReadFileEntry])(implicit ec: ExecutionContext) :Future[Result]
{% endhighlight %}

It returns a regular Play `Result`. The only thing to do is to look for an attachment matching the id with `gridFS.find(...)`, and give the resulting `Cursor[ReadFileEntry]` to the `serve` function:

{% highlight scala %}
def getAttachment(id: String) = Action {
  Async {
    serve(gridFS.find(BSONDocument("_id" -> new BSONObjectID(id))))
  }
}
{% endhighlight %}


### What's Next?

That's the last article of this series. But we did not cover all the features of ReactiveMongo, like bulk insert, using capped collections, run commands... So many topics that will be covered in future articles :)

Meanwhile, you can browse the [Scaladoc](http://reactivemongo.org/api/index.html), post your questions and comments to the [ReactiveMongo Google Group](https://groups.google.com/forum/#!forum/reactivemongo). And, of course, you are invited to grab [the complete application](https://github.com/sgodbillon/reactivemongo-demo-app) and start hacking with it!