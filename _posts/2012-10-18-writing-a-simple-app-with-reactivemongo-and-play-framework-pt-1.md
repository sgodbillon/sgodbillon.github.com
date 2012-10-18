---
layout: post
title: "Writing Reactive Apps with ReactiveMongo and Play"
date: 2012-10-18
---

<p class="notice">
<a href="http://reactivemongo.org">ReactiveMongo</a> is a brand new Scala driver for MongoDB. More than just yet-another-async-driver, it's a reactive driver that allows you to design very scalable applications unleashing MongoDB capabilities like streaming infinite live collections and files for modern Realtime Web applications.
</p>

Play 2.1 has become the main reference for writing web applications in Scala. It is even better when you use a database driver that shares the same vision and capabilities. If you're planning to write a Play 2.1-based web application with MongoDB as a backend, then [ReactiveMongo](http://reactivemongo.org) is the driver to do this!

This article runs you through the process of starting such a project from scratch. We are going to write a simple application that manages articles. Each article has a title, a content, and may embed some attachments (like pictures, PDFs, archives...).

### Summary

<ul class="summary">
	<li><a href="#bootstrap">Bootstrap</a></li>
	<li><a href="#configuring_sbt">Configuring SBT</a></li>
	<li>
		<a href="#model">Model</a>
		<ul>
			<li><a href="#serializing_into_bson__deserializing_from_bson">Serializing into BSON / Deserializing from BSON</a></li>
			<li><a href="#play_form">Play Form</a></li>
		</ul>
	</li>
	<li>
		<a href="#show_a_list_of_articles">Show a List of Articles</a>
		<ul>
			<li><a href="#controller">Controller</a></li>
			<li><a href="#view">View</a></li>
			<li><a href="#route">Route</a></li>
		</ul>
	</li>
	<li><a href="#run_it">Run it!</a></li>
	<li><a href="#go_further">Go further</a></li>
</ul>

### Bootstrap

We assume that you have a running instance of MongoDB installed on your machine. If you don't, read the [QuickStart](http://www.mongodb.org/display/DOCS/Quickstart) on the MongoDB site.

Since Play is currently being refactored to integrate Scala 2.10, we will work with a [snapshot](https://bitbucket.org/sgodbillon/repository/src/9f0c4e40cca1/play-2.1-SNAPSHOT.zip). Let's download it and create a new Scala application:

{% highlight bash %}
$ mkdir reactivemongo-app
$ cd reactivemongo-app
$ curl -O https://bitbucket.org/sgodbillon/repository/src/9f0c4e40cca1/play-2.1-SNAPSHOT.zip
$ unzip play-2.1-SNAPHSOT.zip
$ ./play-2.1-SNAPSHOT/play new articles
       _            _ 
 _ __ | | __ _ _  _| |
| '_ \| |/ _' | || |_|
|  __/|_|\____|\__ (_)
|_|            |__/ 
             
play! 2.1-SNAPSHOT, http://www.playframework.org

The new application will be created in /Volumes/Data/code/article/articles

What is the application name? 
> articles

Which template do you want to use for this new application? 

  1             - Create a simple Scala application
  2             - Create a simple Java application
  3             - Create an empty project
  <g8 template> - Create an app based on the g8 template hosted on Github

> 1
OK, application articles is created.

Have fun!

{% endhighlight %}

### Configuring SBT

In order to use ReactiveMongo and the ReactiveMongo Play Plugin, we will set up the dependencies. Let's edit `project/Build.scala`:

{% highlight scala %}
import sbt._
import Keys._
import PlayProject._

object ApplicationBuild extends Build {

  val appName         = "mongo-app"
  val appVersion      = "1.0-SNAPSHOT"

  val appDependencies = Seq(
    "reactivemongo" %% "reactivemongo" % "0.1-SNAPSHOT",
    "play.modules.reactivemongo" %% "play2-reactivemongo" % "0.1-SNAPSHOT"
  )

  val main = PlayProject(appName, appVersion, appDependencies, mainLang = SCALA).settings(
    resolvers += "sgodbillon" at "https://bitbucket.org/sgodbillon/repository/raw/master/snapshots/"
  )
}
{% endhighlight %}

### Configure MongoDB Connection

Before going further, we should enable the ReactiveMongo Play plugin.

Let's create a `play.plugins` file in the `conf` directory:

    400:play.modules.reactivemongo.ReactiveMongoPlugin

Now we should configure it in the `conf/application.conf` file:

    # ReactiveMongo Plugin Config
    mongodb.servers = ["localhost:27017"]
    mongodb.db = "reactivemongo-app"

### Model

Our articles have a title, a content and a publisher. We will add a creation date and an update date to be able to sort them by date.

Let's create a file `models/articles.scala` and write a case class `Article`:

{% highlight scala %}
package models

import org.joda.time.DateTime
import reactivemongo.bson._
import reactivemongo.bson.handlers._

case class Article(
  id: Option[BSONObjectID],
  title: String,
  content: String,
  publisher: String,
  creationDate: Option[DateTime],
  updateDate: Option[DateTime]
)
{% endhighlight %}

The `id` field is an `Option` of `BSONObjectID`. An ObjectId is a 12 bytes long unique value that is the standard id type in MongoDB documents.

#### Serializing into BSON / Deserializing from BSON

Now, we may write the BSON serializer and deserializer for this case class. This enables to transform an `Article` instance into a BSON document that may be stored into the database and vice versa. ReactiveMongo provides two traits, `BSONReader[T]` and `BSONWriter[T]`, that should be implemented for this purpose.

Making a BSON document is pretty easy: the method `BSONDocument()` takes tuples of `(String, BSONValue)` as arguments. So, producing a very basic document could be written like this:

{% highlight scala %}
BSONDocument(
  "title" -> BSONString("some title"),
  "content" -> BSONString("some content")
)
{% endhighlight %}

The opposite can be achieved using the method `getAs[BSONValue]` on a `TraversableBSONDocument`:

{% highlight scala %}
val title :Option[String] = doc.getAs[BSONString]("title").map(_.value)
val content :Option[String] = doc.getAs[BSONString]("content").map(_.value)
{% endhighlight %}

Let's implement `ArticleBSONReader[Article]` and `ArticleBSONWriter[Article]` in the same file (`models/articles.scala`):

{% highlight scala %}
object Article {
  implicit object ArticleBSONReader extends BSONReader[Article] {
    def fromBSON(document: BSONDocument) :Article = {
      val doc = document.toTraversable
      Article(
        doc.getAs[BSONObjectID]("_id"),
        doc.getAs[BSONString]("title").get.value,
        doc.getAs[BSONString]("content").get.value,
        doc.getAs[BSONString]("publisher").get.value,
        doc.getAs[BSONDateTime]("creationDate").map(dt => new DateTime(dt.value)),
        doc.getAs[BSONDateTime]("updateDate").map(dt => new DateTime(dt.value)))
    }
  }
  implicit object ArticleBSONWriter extends BSONWriter[Article] {
    def toBSON(article: Article) = {
      val bson = BSONDocument(
        "_id" -> article.id.getOrElse(BSONObjectID.generate),
        "title" -> BSONString(article.title),
        "content" -> BSONString(article.content),
        "publisher" -> BSONString(article.publisher))
      if(article.creationDate.isDefined)
        bson += "creationDate" -> BSONDateTime(article.creationDate.get.getMillis)
      if(article.updateDate.isDefined)
        bson += "updateDate" -> BSONDateTime(article.updateDate.get.getMillis)
      bson
    }
  }
}
{% endhighlight %}


#### Play Form

We will also define a [Play Form](http://www.playframework.org/documentation/2.0.4/ScalaForms) to handle HTTP form data submission, in the companion object `models.Article`. It will be useful when we implement edition (in the next article).

{% highlight scala %}
val form = Form(
  mapping(
    "id" -> optional(of[String] verifying pattern(
      """[a-fA-F0-9]{24}""".r,
      "constraint.objectId",
      "error.objectId")),
    "title" -> nonEmptyText,
    "content" -> text,
    "publisher" -> nonEmptyText,
    "creationDate" -> optional(of[Long]),
    "updateDate" -> optional(of[Long])
  ) { (id, title, content, publisher, creationDate, updateDate) =>
    Article(
      id.map(new BSONObjectID(_)),
      title,
      content,
      publisher,
      creationDate.map(new DateTime(_)),
      updateDate.map(new DateTime(_)))
  } { article =>
    Some(
      (article.id.map(_.stringify),
      article.title,
      article.content,
      article.publisher,
      article.creationDate.map(_.getMillis),
      article.updateDate.map(_.getMillis)))
  }
)
{% endhighlight %}

### Show a list of articles

The ReactiveMongo Play Plugin ships with a mixin trait for Controllers, providing some useful methods and a reference to the configured database.

#### Controller

{% highlight scala %}
package controllers

import models._
import play.api._
import play.api.mvc._
import play.api.Play.current
import play.modules.reactivemongo._

import reactivemongo.api._
import reactivemongo.bson._
import reactivemongo.bson.handlers.DefaultBSONHandlers._

object Articles extends Controller with MongoController {
  def index = Action {
    Ok()
  }
}
{% endhighlight %}

We need to retrive all the articles from our collection `articles`. To do this, we get a reference to this collection and run a basic query:

{% highlight scala %}
  def index = Action { implicit request =>
    Async {
      implicit val reader = Article.ArticleBSONReader
      val collection = db.collection("articles")
      // empty query to match all the documents
      val query = BSONDocument()
      // the future cursor of documents
      val found = collection.find(query)
      // build (asynchronously) a list containing all the articles
      found.toList.map { articles =>
        Ok(views.html.articles(articles, activeSort))
      }
    }
{% endhighlight %}

Note the `Async` method: our controller action is actually return a future result.

#### View

Now, let's create a view file `app/views/index.scala.html` for this action result:

{% highlight html %}
@(articles: List[models.Article])
@if(articles.isEmpty) {
	<p>No articles available yet.</p>
} else {
  <ul>
    @articles.map { article =>
    <li>
      <a href="#edittoimplement">@article.title</a>
      <em>by @article.publisher</em>
    </li>
    }
</ul>
}
{% endhighlight %}

#### Route

Let's declare the matching route in the `conf/routes` file:

    GET     /                           controllers.Articles.index

### Run it!

Now, you can start play:
	
{% highlight bash %}
$ cd articles
$ ../play-2.1-SNAPSHOT/play
       _            _ 
 _ __ | | __ _ _  _| |
| '_ \| |/ _' | || |_|
|  __/|_|\____|\__ (_)
|_|            |__/ 
             
play! 2.1-SNAPSHOT, http://www.playframework.org

> Type "help play" or "license" for more information.
> Type "exit" or use Ctrl+D to leave this console.

[articles] $ run

[info] Updating {file:/Volumes/Data/code/article/articles/}articles...
[info] Done updating.                                                                  
--- (Running the application from SBT, auto-reloading is enabled) ---

[info] play - Listening for HTTP on /0.0.0.0:9000

(Server started, use Ctrl+D to stop and go back to the console...)
{% endhighlight %}

... and access http://localhost:9000 ;)

The list you get is empty. That is perfectly normal since our database does not contain any article. Let's open a mongo console, connect to the database and add an article:

{% highlight javascript %}
$ mongo
MongoDB shell version: 2.2.0
connecting to: 127.0.0.1:27017/test
> use reactivemongo-app
switched to db reactivemongo-app
> db.articles.save({ "content" : "some content", "creationDate" : new Date(), "publisher" : "Jack", "title" : "A cool article", "updateDate" : new Date()) })
{% endhighlight %}

### Go further

In the next articles, we will continue this application (edition, attachments submission...), and cover more advanced features of ReactiveMongo, such as running complex queries, building indexes, and using GridFS.

Meanwhile, you can grab [the complete application](https://github.com/sgodbillon/reactivemongo-demo-app) and start hacking with it.

Don't hesitate to post your questions and comments to the [ReactiveMongo Google Group](https://groups.google.com/forum/#!forum/reactivemongo).