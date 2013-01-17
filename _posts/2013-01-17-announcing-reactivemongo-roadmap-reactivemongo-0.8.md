---
layout: post
title: "Announcing ReactiveMongo Roadmap and ReactiveMongo 0.8"
date: 2013-01-17
---

> [ReactiveMongo](http://reactivemongo.org) is a brand new Scala driver for MongoDB. More than just yet-another-async-driver, it's a reactive driver that allows you to design very scalable applications unleashing MongoDB capabilities like streaming infinite live collections and files for modern Realtime Web applications.


ReactiveMongo was publicly introduced 4 months ago and I am glad to see the community very active and growing fast. Last month several pull requests were made by ReactiveMongo users, which means that the project is not only played with but used in a production context. ReactiveMongo is on the right track and I think that it's now the time to prepare the first stable release.

I am pleased to announce the official roadmap for the project. Given that ReactiveMongo is pretty much full-featured, and that some projects already rely on it, I am labeling the current snapshot as the 0.8 version.

The next versions (0.9 and 1.0) may be expected in the next couple of months. As usual all kinds of contribution are very welcome (pull requests of course, but also feedback, documentation and feature proposals).

Here are the main tasks I will focus on for theses versions:

* Polished BSON API
* Map/Reduce & Aggregation Framework Support *(note that it is already possible to send custom commands to the database)*
* Performance improvements
* More tests
* Bugfixing
* Documentation

Meanwhile, you can stick to the 0.8 release and wait for the next ones instead of using the snapshots.


### Roadmap

#### State of ReactiveMongo as of v0.8

The current version covers the following features:

* Replica Sets
* Authentication
* GridFS Support
* Tailable Cursors
* Bulk Inserts
* Index Management
* Failover
* Command Support, including:
	* Count
	* FindAndModify
	* CreateCollection
	* CreateCappedCollection
	* CollStats
	* DropCollection and DropDatabase
	* RenameCollection

#### Next version - v0.9

* BSON library and handlers improvements (to make their use less verbose and to ensure composability)
* Map/Reduce support
* Aggregation Framework

#### First stable release - v1.0

* Review of the whole API to ensure consistency
* Track down remaining bugs
* Focus on performance
* Complete documentation, new site

### ReactiveMongo packages are now hosted by [Sonatype](https://oss.sonatype.org/index.html#nexus-search;quick~reactivemongo)

ReactiveMongo is now available in the Sonatype OSS repository, in the group `org.reactivemongo` (was `reactivemongo` before). The previous bitbucket repository is deprecated (yet it is not going to be deleted right now).

If you want to use the current release, the only thing you need to do is to declare the dependency (since the Sonatype OSS Repository is synced with Maven Central, which is one of the default repositories of SBT). If you choose to live on the bleeding edge, you have to add the Sonatype snapshots repository.

{% highlight scala %}
// Using Release
libraryDependencies += "org.reactivemongo" %% "reactivemongo" % "0.8"

// Using Snapshots
resolvers += "Sonatype snapshots" at "http://oss.sonatype.org/content/repositories/snapshots/"
libraryDependencies += "org.reactivemongo" %% "reactivemongo" % "0.9-SNAPSHOT"
{% endhighlight %}

The Play 2.0 plugin jars have been moved to Sonatype as well. Note that the groupId of the plugin has changed and is now `org.reactivemongo`. For the same reasons as for ReactiveMongo itself, the Play 2.0 plugin current snapshot has been promoted to 0.8.

{% highlight scala %}
// Using Release
libraryDependencies += "org.reactivemongo" %% "play2-reactivemongo" % "0.8"

// Using Snapshots
resolvers += "Sonatype snapshots" at "http://oss.sonatype.org/content/repositories/snapshots/"
libraryDependencies += "org.reactivemongo" %% "play2-reactivemongo" % "0.9-SNAPSHOT"
{% endhighlight %}