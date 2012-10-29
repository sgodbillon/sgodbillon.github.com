---
layout: post
title: "Writing Reactive Apps with ReactiveMongo and Play, Pt. 2"
date: 2012-10-29
---

<p class="notice">
<a href="http://reactivemongo.org">ReactiveMongo</a> is a brand new Scala driver for MongoDB. More than just yet-another-async-driver, it's a reactive driver that allows you to design very scalable applications unleashing MongoDB capabilities like streaming infinite live collections and files for modern Realtime Web applications.
</p>

In the [previous article](http://stephane.godbillon.com/2012/10/18/writing-a-simple-app-with-reactivemongo-and-play-framework-pt-1.html), we saw how to set up an application with ReactiveMongo and Play 2.1. This application  manages articles. Each article has a title, a content, and may embed some attachments (like pictures, PDFs, archives...).

Right now, we are able to list all the articles. This is done thanks to the `collection.find` method, which returns a Cursor of (future) documents.

Today, we will see how to insert and update articles, and how to sort the list of articles.

### Summary

<ul class="summary">
	<li><a href="#article_creation">Article Creation</a></li>
	<li><a href="#article_edition">Article Edition</a></li>
	<li><a href="#sorting_the_article_list">Sorting the article list</a></li>
	<li><a href="#coming_next_week">Coming next week</a></li>
</ul>

### Article Creation

First, we may create a form to create (and edit) articles. Let's create a view `editArticles.scala.html`:

{% highlight html %}
@(form: Form[models.Article])
@import helper.twitterBootstrap._

<div class="row">
  <div class="span8">
  <h2>Add an article</h2>
  @helper.form(action = routes.Articles.create, 'class -> "form-horizontal") {
    @helper.inputText(form("title"))
    @helper.inputText(form("publisher"))
    @helper.textarea(form("content"))
    <div class="form-actions">
      <input class="btn btn-primary" type="submit">
    </div>
  }
  </div>
</div>
{% endhighlight %}

This template is a function that takes a `Form[models.Article]` as a parameter. A `Form[T]`, in Play, is a helper that binds the HTTP params. In this case, it will allow us to get and validate an Article sent by the client. The `@helper.xxx` functions generate the matching form elements (like `<form>`, `<input>`, etc.) and fill them with the values it holds, if any.

In the previous article, we defined a `Form[Article]`. We will use it in the controller part.



The action to show the creation form is very simple. It uses the `form` we defined on the Article companion object, without filling it with a value, then renders the view.

{% highlight scala %}
def showCreationForm = Action {
  Ok(views.html.editArticle(Article.form))
}
{% endhighlight %}

Now, we may handle the creation itself.
The first thing we should do is to fill the `Article.form` with HTTP form data, then check if it is valid.
{% highlight scala %}
def create = Action { implicit request =>
  Article.form.bindFromRequest.fold(
    errors => Ok(views.html.editArticle(None, errors, None)),
    // if no error, then insert the article into the 'articles' collection
    article => // save it!
  )
}
{% endhighlight %}

To save a document with ReactiveMongo, we use the `collection.insert` method of signature `insert[T](document: T)(implicit writer: RawBSONWriter[T]): Future[LastError]`.

Before using this, we may ensure that `Article.ArticleBSONWriter` is the scope by importing it.

{% highlight scala %}
import models.Article._
{% endhighlight %}

Before saving the article, we set the creationDate and the updateDate, then we call `collection.insert()`:

{% highlight scala %}
val updatedArticle = article.copy(creationDate = Some(new DateTime()), updateDate = Some(new DateTime()))
collection.insert(updatedArticle) // returns a Future[LastError]
{% endhighlight %}

This snippet returns a Future of LastError. But what we want to get is a `Result` - if fact, we want to redirect back to the index to show the list of articles. Let's `map` our `Future[LastError]` to a `Future[Result]`:

{% highlight scala %}
val updatedArticle = article.copy(creationDate = Some(new DateTime()), updateDate = Some(new DateTime()))
collection.insert(updatedArticle).map( _ => // we don't care of the lasterror here (map is called on success)
  Redirect(routes.Articles.index)
)
{% endhighlight %}

... and we wrap this Future of Result in an AsyncResult that can be handled by a Play action. Our action eventually looks like this:

{% highlight scala %}
def create = Action { implicit request =>
  import models.Article._
  Article.form.bindFromRequest.fold(
    errors => Ok(views.html.editArticle(None, errors, None)),
    // if no error, then insert the article into the 'articles' collection
    article => AsyncResult {
      val updatedArticle = article.copy(creationDate = Some(new DateTime()), updateDate = Some(new DateTime()))
      collection.insert(updatedArticle).map( _ => // we don't care of the lasterror here (map is called on success)
        Redirect(routes.Articles.index)
      )
    }
  )
}
{% endhighlight %}


The last thing we should do is to declare two routes, one for rendering a creation form and the other for saving the created article.

    GET     /articles/new               controllers.Articles.showCreationForm
    POST    /articles/new               controllers.Articles.create

Now we can post new articles!

> #### What is a `LastError`?
> When a write operation is done on a collection, MongoDB does not send back a message to confirm that all went right or not. To be sure that a write operation is successful, one must send a `GetLastError` command. When it receives such a command, MongoDB waits until the last operation is done and then sends back the result. This result is a `LastError` message. Of course, ReactiveMongo do all this stuff for you by default - so you don't need to worry about this.

### Article edition



Editing an article is pretty much the same as creating it, except that the article may already exist in the collection. So, before rendering the edition form, we should fetch the article matching the id. If an article is found, the result will be the rendered template; or else it will be `NotFound`.

{% highlight scala %}
def showEditForm(id: String) = Action {
  implicit val reader = Article.ArticleBSONReader
  Async {
    val objectId = new BSONObjectID(id)
    // get the documents having this id (there will be 0 or 1 result)
    val cursor = collection.find(BSONDocument("_id" -> objectId))
    // ... so we get optionally the matching article, if any
    // let's use for-comprehensions to compose futures (see http://doc.akka.io/docs/akka/2.0.3/scala/futures.html#For_Comprehensions for more information)
    for {
      // get a future option of article
      maybeArticle <- cursor.headOption
      // if there is some article, return a future of result with the article
      result <- maybeArticle.map { article =>
        Ok(views.html.editArticle(Article.form.fill(article)))
      }.getOrElse(Future(NotFound))
    } yield result
  }
}
{% endhighlight %}

When an article is found, we call `fill(article: Article)` on `Article.form`. This function returns a new `Form[Article]` instance, holding the data of the given article. So, in the view, we can fill the fields of the edition form with the data of the article.

Now, let's write the `edit` action that will perform the update operation. The idea is to create a modifier document and call `collection.update` to update the article.

{% highlight scala %}
def edit(id: String) = Action { implicit request =>
  Article.form.bindFromRequest.fold(
    errors => Ok(views.html.editArticle(Some(id), errors, None)),
    article => AsyncResult {
      val objectId = new BSONObjectID(id)
      // create a modifier document, ie a document that contains the update operations to run onto the documents matching the query
      val modifier = BSONDocument(
        // this modifier will set the fields 'updateDate', 'title', 'content', and 'publisher'
        "$set" -> BSONDocument(
          "updateDate" -> BSONDateTime(new DateTime().getMillis),
          "title" -> BSONString(article.title),
          "content" -> BSONString(article.content),
          "publisher" -> BSONString(article.publisher)))
      // ok, let's do the update
      collection.update(BSONDocument("_id" -> objectId), modifier).map { _ =>
        Redirect(routes.Articles.index)
      }
    }
  )
}
{% endhighlight %}

First, we call `Article.form.bindFromRequest` that produces a new `Form[Article]` that will be filled with the HTTP form data. Then we call fold on it, that takes two functions as parameters: the first takes a `Form[Article]` that contains the errors (because the validation failed), and the other takes a valid instance of `Article`. This `fold` method will call either the first or the second function depending on the validation status.

If the validation fails, all we do is to render the form again with the errors. Else we perform the update operation and redirect to the articles list.

The `modifier` val is a document containing all the update operations to run on the matched document. Here, we set `updateDate` to the current date, and `title`, `content` and `publisher` to their new value.



Let's declare the matching routes in the `conf/routes` file:

    GET     /articles/:id               controllers.Articles.showEditForm(id)
    POST    /articles/:id               controllers.Articles.edit(id)



Last but not least, since we use the same view for the creation and the edition forms, we should switch the post action url. We will do this by adding a new parameter to our view, `id`, which is an `Option[String]`. If there is a value, then we are editing an article that already exists.

{% highlight html %}
@(id: Option[String], form: Form[models.Article])
@import helper.twitterBootstrap._

<div class="row">
  <div class="span8">
  <h2>Add an article</h2>
  @helper.form(action = (if(!id.isDefined) routes.Articles.create else routes.Articles.edit(id.get)), 'class -> "form-horizontal") {
    // ...
{% endhighlight %}

Then the `create` action becomes:

{% highlight scala %}
def showCreationForm = Action {
  Ok(views.html.editArticle(None, Article.form))
}
{% endhighlight %}

And the `showEditForm` action:

{% highlight scala %}
def showEditForm(id: String) = Action {
  // ....
      result <- maybeArticle.map { article =>
        Ok(views.html.editArticle(Some(id), Article.form.fill(article)))
      }.getOrElse(Future(NotFound))
    } yield result
  }
}
{% endhighlight %}

And we're done!

### Sorting the article list

Until now, our article list is unsorted. Well, that's not entirely true: it is sorted by the _natural_ order, ie the insertion order. But what if we want to sort them by publisher? Or last edition date?

Let's say that our `index` action, which lists the articles, will accept a 'sort' parameter. This parameter is a String composed of a field name, optionally prefixed by a '-' character to reverse the order.

For example, if we want to sort by updateDate, our URL will look like this:

    /?sort=updateDate

And if we want to get the more recently edited articles first:

    /?sort=-updateDate

In MongoDB, such a request would be written this way:

{% highlight javascript %}
db.articles.find({
  $query: {},
  $sort: {
    updateDate: 1 // or -1 for reverse order
  }
})
{% endhighlight %}

First, let's define a function that will handle the sort parameter. This function returns an option of `BSONDocument`: if there is a sort parameter, then some BSONDocument will be returned.

{% highlight scala %}
private def getSort(request: Request[_]) = {
  request.queryString.get("sort").map { fields =>
    val orderBy = BSONDocument()
    for(field <- fields) {
      val order = if(field.startsWith("-"))
        field.drop(1) -> -1
      else field -> 1

      if(order._1 == "title" || order._1 == "publisher" || order._1 == "creationDate" || order._1 == "updateDate")
        orderBy += order._1 -> BSONInteger(order._2)
    }
    orderBy
  }
}
{% endhighlight %}

This function is very simple: we get all the values of `sort`. For each of them, we extract the order (`1` or `-1`) by getting the first character and checking if it is a `-`. Then, we add this field name and its sort value into the resulting document.

That was the hardest part ;) All we need to do now is to use this function and, if there is some `sort` document, add it to our query. Now, our `index` action looks like this:

{% highlight scala %}
// list all articles and sort them
def index = Action { implicit request =>
  Async {
    implicit val reader = Article.ArticleBSONReader
    // empty query to match all the documents
    val query = BSONDocument()
    val sort = getSort(request)
    if(sort.isDefined) {
      // build a selection document with an empty query and a sort subdocument ('$orderby')
      query += "$orderby" -> sort.get
      query += "$query" -> BSONDocument()
    }
    val activeSort = request.queryString.get("sort").flatMap(_.headOption).getOrElse("none")
    // the future cursor of documents
    val found = collection.find(query)
    // build (asynchronously) a list containing all the articles
    found.toList.map { articles =>
      Ok(views.html.articles(articles, activeSort))
    }
  }
}
{% endhighlight %}

And we can add some links to our list view to sort it:

{% highlight html %}
@(articles: List[models.Article])
<ul class="sort">
  <li><a href="?sort=publisher">Sort by publisher</a></li>
  <li><a href="?sort=-publisher">Sort by publisher (reverse)</a></li>
  <li><a href="?sort=updateDate">Sort by update date</a></li>
  <li><a href="?sort=-updateDate">Sort by update date (reverse)</a></li>
</ul>
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

### Coming next week

In the next article, we will add the attachments management feature to this application using GridFS.

Meanwhile, you can grab [the complete application](https://github.com/sgodbillon/reactivemongo-demo-app) and start hacking with it.

Don't hesitate to post your questions and comments to the [ReactiveMongo Google Group](https://groups.google.com/forum/#!forum/reactivemongo).