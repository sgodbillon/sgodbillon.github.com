---
layout: post
title: "ReactiveMongo for Scala: Unleashing MongoDB Streaming capabilities for Realtime Web"
date: 2012-08-30
---

<p class="notice">
I am very excited to introduce <a href="http://reactivemongo.org">ReactiveMongo</a>, a brand new Scala driver for MongoDB. More than just yet-another-async-driver, it's a <em>reactive</em> driver that allows you to design very scalable applications unleashing MongoDB capabilities like streaming infinite live collections and files for modern Realtime Web applications.
</p>

### What does reactive mean?

I/O operations may be handled in the following ways:
* synchronously: each time a request is sent, the running thread is blocked, waiting for the response to arrive. When the response is received, the execution flow resumes.
* asynchronously: the code handling the response may be not run simultaneously (often in a closure). Still, a thread may be blocked, but it may not be the same thread that did the request.
* non-blocking: sending a request does not block any thread.

#### Scalability - asynchronous and non-blocking requests

Synchronous database drivers do not perform very well in heavily loaded applications - they spend a lot of time waiting for a response. When you get 2000 or more clients performing each several database non-trivial requests, such a component becomes the main bottleneck of your application. Moreover, what’s the point of using a nifty, powerful, fully asynchronous/non-blocking web framework (like [Play!](http://www.playframework.org/)) if all your database accesses are blocking?

The problem is, almost all MongoDB drivers perform synchronous IO. Particularly, there are no fully implemented non-blocking drivers on the JVM. A year ago, [Brendan McAdams](http://blog.evilmonkeylabs.com/) from [10gen](http://www.10gen.com/) did actually start an initiative to implement the mongo protocol in a non-blocking manner, Hammersmith. This gave us motivation to work on ReactiveMongo.

ReactiveMongo is designed to avoid any kind of blocking request. Every operation returns immediately, freeing the running thread and resuming execution when it is over. Accessing the database is not a bottleneck anymore.

#### Streaming capability - Iteratee/Enumerator to the rescue

More than enabling non-blocking database requests, ReactiveMongo is streaming capable. Everywhere it makes sense, you can provide streams of documents instead of one, and you can get streams of documents instead of a classic cursor or list. It is very powerful when you want to serve many clients at the same time, without filling up the memory and in an efficient way.
This feature is crucial to build modern web applications. Indeed, the future of the web is in streaming data to a very large number of clients simultaneously. Twitter Stream API is a good example of this paradigm shift that is radically altering the way data is consumed all over the web.

ReactiveMongo enables you to build such a web application right now. It allows you to stream data both into and from your MongoDB servers. Using the well known [Iteratee/Enumerator](https://github.com/playframework/Play20/wiki/Iteratees) pattern, MongoDB can now be seen as a consumer (or producer) of streams of documents.

Here are some examples of what you can do with ReactiveMongo and Iteratees/Enumerators.

##### Stream a capped collection through a websocket

With ReactiveMongo, you can stream the content of a collection. That’s particularly useful when dealing with websockets. For example, you can get a tailable cursor over a capped collection, asynchronously fetch newly inserted documents and push them into a websocket. There is a good example of doing this here.

##### Perform bulk inserts from a streaming source

Imagine that your application receives a lot of documents to insert new documents continuously. Instead of making one database call for each, it is better to build a bulk of documents and then send it to the database. ReactiveMongo enables you to do this in perfectly non-blocking way.

##### Store and read files with MongoDB, the streaming way

MongoDB provides a way to deal with files, [GridFS](http://www.mongodb.org/display/DOCS/GridFS). That’s great when you want to read and write files, without directly using the filesystem and to profit of MongoDB replication/sharding capabilities. But in order to be very scalable, a web application should be able to stream files, without filling up the memory unnecessarily or blocking one thread per upload/download.
ReactiveMongo is the perfect tool to do this. It is designed to be natively streaming capable.

### ReactiveMongo’s design principles

#### All database operations must be non-blocking

All the database operations that ReactiveMongo performs are non-blocking. No thread may be blocked waiting for a response.

#### Implement Streams (Enumerators/Iteratees) everywhere it makes sense

Streaming capability is essential for building modern, reactive web applications. ReactiveMongo implements it with [enumerators and iteratees](https://github.com/playframework/Play20/wiki/Iteratees) (immutable Producer/Consumer pattern). The API is designed to provide this capability for every feature where it may be interesting: cursors, (bulk) insertion, GridFS...

#### Stay as close as possible of the Mongo Wire Protocol

Like modern web framework try to be conform to the HTTP protocol in their architecture (notably by being RESTful), ReactiveMongo is designed to stay as close as possible of the Mongo Wire Protocol.

#### Provide well designed API

ReactiveMongo’s API is designed to be very easy to play with, and to make easier to build libraries upon it.

### Developer preview, roadmap and examples

ReactiveMongo is still under heavy development, but the following are already implemented:
* ReplicaSet support
* Authentication support
* GridFS support (streaming capable)
* Cursors (providing a stream of documents)
* Bulk inserts
* Database commands support
* Indexing operations

You can get the code from the [Github repository](https://github.com/zenexity/ReactiveMongo). The scaladoc is also available.

Note that it depends on Scala 2.9.2, with the latest [Play 2.1](https://github.com/playframework/Play20) head. 

#### What remains to do

* Implement the missing database commands
* Complete the API
* Extensive testing

#### Examples

There are some good examples that cover ReactiveMongo’s capabilities:
* [Complete Web Application with basic CRUD and GridFS streaming](https://github.com/sgodbillon/reactivemongo-demo-app)
* [Streaming a Capped Collection through a WebSocket (tailable cursors)](https://github.com/sgodbillon/reactivemongo-tailablecursor-demo)

#### Community

There is a [ReactiveMongo Google Group](https://groups.google.com/forum/#!forum/reactivemongo). Don’t hesitate to post your questions, bug reports and comments. All kinds of contribution are very welcomed.