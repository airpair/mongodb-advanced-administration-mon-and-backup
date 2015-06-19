## Managing MongoDB: A DBA's Tale 

This blog post was based on (but is not identical to) my talk given at the MongoDB World 2015 conference.  If you would like to see the talk then luckily for you it was recorded and it can be found on Mongo's website.

### MongoDB Developers vs DBAs

The land of NoSQL is changing rapidly and it is easy to get caught up in a lot of the hype, especially with MongoDB.  I think that MongoDB is a great tool - but like all tools it can be misused or applied to the wrong task.  I'm going to be talking about how you can take control of a MongoDB deployment and begin imposing order, stability, and determinism when your stack gets out of hand.  If you are looking to make sure your MongoDB implimentation will not take a team to fix, or have a deployment and don't know where to start fixing it then I hope my experiences make your task a bit easier.

Whether you are a developer or a DBA this post can be useful to you, I know there are a lot of developers who know how MongoDB works.  I personally believe the line between DBA and Dev is going to blur more and more as these databases designed for developers in mind gain more and more traction.  I'm going to assume some working familiarity with MongoDB in this post so if you don't know what replica sets are or what the oplog is then don't worry, there are a lot of resources out there for you.  The free courses offered by [MongoDB University](https://university.mongodb.com/) are excellent both in the content covered and in the help from experts you can get during the courses.  There are also many [Mongo User Groups](https://www.mongodb.org/user-groups) around the country where you can get involved with other MongoDB users.


### MongoDB Deployment vs other Database

MongoDB isn't the only database that one can store unstructured data in, nor is it the only one that you can store unstructured JSON data in and query on the contents of that JSON (for example Postgres has a JSON data field and you can use it like Mongo in that regard.).  What really sets MongoDB apart is that replication and horizontal data distribution (sharding) come prepackaged with the database and are easy to use.  Other databases have to be horizontally scaled at the application level (put clients 1-10 on server A, put clients 11-20 on server B, etc.), however Mongo will do this for you at the database level in a sharded cluster.

#### MongoDB Sharded Clusters

In a sharded cluster several replica sets are assembled into a cluster which is connected to through Mongos (s for server, it is really just a routing proxy) instances.  The Mongos router will, depending on your query, figure out which one of the servers has the data you are interested in - route your query to those servers - then merge the results and return them to you.  The wire protocol and the driver calls to a Mongos are idential to that of calls that are going to individual nodes or replica sets, so your application does not need to know if it is talking to a cluster or not.

The data is distributed across all of the shards by the choice (Mongo doesn't choose this for you, it is up to you!) of a shard key.  The shard key is the value that determines what server a document will reside on.  For example if the shard key is chosen to be clientId then all documents for client 1 would reside on one server, and client 2 on one server, and client 3 on one server.  It could be that client 3 has a lot of documents so it takes up all the space in serverB, but that is ok because mongo will then put clients 1 and 2 on serverA to try and keep the servers as neatly balanced as possible.  It does this all for you automatically.

#### Mongo Shared Keys

There are a lot of subtlies in choosing the shard key and implications which could take up an entire other post, so we'll leave an in depth discussion of shard key choice for another day - we don't need to go down that road for the topics discussed here.  The [MongoDB University](https://university.mongodb.com/) course for DBAs has a very nice treatment of sharding if you would like to learn more.


### Why do I still need a DBA for Mongo?

A school of thought has emerged that with these schemaless datastores which are easy and fast to set up and tie into applications that the DBA is no longer needed.  No more will there be 100+ line SQL statements that need tuning, no more will there be parameters in the deployment of the database that need to be tweaked to get performance.  This has led many down the path of putting data in a location without care for what access controls are around the data, how it may later be used, and how to optimize the data for performance with specific use cases.  Monitoring the database and backing up the data also then fall onto the shoulders of perhaps the operations team.  Effectively the role of DBA was split up between the development team - which has their focus on things other than the data - and the operations team whose focus is again elsewhere.  With the coming of MongoDB 3.0 there are increasing number of parameters to tweak to optimize performance even beyond choosing the storage engine for the application or the shard key to distribute on.

If this is sounding daunting all of a sudden that is because it quickly becomes so.  The role of the DBA in the NoSQL future is as strong as it has been in the past.  The tools change and your codebase may change but your data will remain, and you need people who will speak for your data for it has no tounge.  These are the DBAs.

## Tales of DBA Struggle

When I joined Sailthru as a DBA the team was newly formed, the MongoDB deployment had been left to some combination of Operations members and Development for some time and many people had their hands in the pie but few knew or had ownership over what was going on.  Thus began several tales of struggle.

### Backups, MMS vs Hybrid Cloud

As a DBA the first thing to assess is how backed up is the data.  If your data access is slow it can be sped up later, if your monitoring is poor it can be shored up later, but if your backups are not working and you lose your data you cannot fix your backups later.  I'm going to discuss two options to backing up data for MongoDB: hybrid cloud and MMS backups.

#### Hybrid cloud backups

A hybrid cloud is where you have part of your application running on physical machines that you manage and the other part is running in a cloud provider such as AWS.  A hybrid cloud system with MongoDB is essentially having part of each replica set in the cloud syncing to part of the replica set that is running on your hardware.

![A schematic of a hybrid cloud setup](http://i.imgur.com/6zfybHh.png "Schematic of a hybrid cloud")

This has several benefits:

1. **Redundancy.** Disaster recovery is immediate.  You have a working copy of your data offsite so you can immediately begin using that set.
2. **Backups.**  Snapshots can be taken easily and are also stored off site.

However there are also downsides:
1. **Cost.**  This system could end up costing a lot of money, especially as you must make sure the MongoDB instances are provisioned so they can keep up with the instances in your datacenter.
2. **Security.**  You must maintain a secure connection to the cloud provider as replication in MongoDB occurs without encryption.
3. **Bandwidth.**  You must have enough bandwidth to be able to keep this data set in sync or it will quickly become useless, requiring manual intervention.
4. **Time.**  It will in all likelihood take a lot of time to maintain, especially if members in the cloud fall out of sync and a manual resync operation is required.

It can look like the costs outweight the benefits by sheer numbers, but let's be fair those two benefits are really big benefits.  What got me to give up a hybrid cloud was when we used this technique with a sharded cluster.

In a single replica set it is easy to say if we have a consistent backup of it or not, easy to monitor and if there is a problem it is easy to rectify.  However when you have a cluster all replica sets (shards) that compose the cluster must be backed up successfully for the entire cluster to be backed up.  As traffic and load increase it may be hard to do this, and if you have a single shard unable to keep up then your entire backup - no matter if it is a hundred shard cluster - is now incomplete.  This was a big problem, big enough to scrap our hybrid cloud architecture.

#### MongoDB managed MMS backups

We moved to MMS, MongoDB's backup solution.  MongoDB will stream your data to their datacenter in a compressed and encrypted format.  It will then take snapshots for you off site and store them, giving you very granular (between 15min and to the second in many cases) restore capabilities and redundancy.  Surprisingly this was also cheaper than our hybrid cloud solution and it was clear that migrating to this would give us more time to focus on other tasks.  The MMS backup has been relatively straightforward and has worked quite well.  It works by placing a MMS Backup agent within your datacenter / cloud which tails the oplog of the servers being backed up.  It then ships the data over the wire to MongoDB's data center where they keep an up to date copy, taking periodic snapshots and automating the point in time recovery operation through oplog replays.

![A schematic of the MMS Backup process](http://i.imgur.com/PrY781g.png)

Some things to keep in mind however:

1. **Automation.**  Currently there is no automation tied into the MMS backup solution, which means you will have to go through their relatively new API to try and get your backup tasks done and write your own automation, or you will have to use the UI.  For any sizeable deployment automation will be required for any speedy response time, so keep this in mind.
2. **Offsite.**  You will need to stream the data over the internet to its destination, these backups are not close to your servers, so it will take time to move the data.  Factor this into your disaster recovery plan.  However you will not need to execute mongorestore on the data that MongoDB ships you, it is all in datafiles that you can point mongod to immediately and begin using.

MMS backup can be a very good solution and it shouldn't be discounted, especially if your manpower is limited.  Not having backups is not an option for any organization.

### Monitoring with MMS and Zabbix

Monitoring is the second most crucial thing for a DBA to set up.  Being able to know if your system is behaving as expected or outside normal bounds, and being able to quickly narrow problems down to the source is necessary for confidence in the operation of the data layer.

#### MongoDB hosted MMS monitoring
Anyone playing with MongoDB out of the box should set up MMS, this will be an excellent monitoring and alerting system for you.  This gives you a lot of information but it might not be very helpful without context, especially if you are using this to alert you by SMS or by some third party such as PagerDuty (MMS does have integration with PagerDuty).  Here are some things to look out for:


1. **Read / Write Queues.**  These are real good indicators if you have overloaded what MongoDB can get done, even if your lock percent is high if you don't have queues you often don't have a problem.  If you do start to sustain a queue you'll see slowdown in your application. <br /><div style="text-align:center">![A MMS Queue graph](http://i.imgur.com/xJgk55j.png "A graph of MongoDB queues as recorded by MMS.  We can see that the queue length rises as times goes on, response time in the application would be seen as increasing")</div>
2. **Network Traffic.**  MongoDB can push as many bits as the network interface can handle, and at times this has been the only indicator as to why things are getting slow.  If you see your network graph plateau then there is a problem and likely where your performance bottleneck lies. <br /><div style="text-align:center">![A MMS Network Graph](http://i.imgur.com/m3Ev6Q9.png "Spikes in network egress can be a sign of queries pulling too much data and can impact performance as well")</div>
3. ** Background Flush Average.**  This represents how long it takes to flush the data to disk, a high value of the background flush will indicate that the disk is overloaded.  A background flush is executed every 60 seconds so keep that in mind, a background flush of 40 seconds is extremely high.<br /><div style="text-align:center">![A MMS background flush graph](http://i.imgur.com/n2kWHmC.png "The background flush average can warn you if your diskio cannot keep up with the demand being put on Mongo")</div>
4. **Lock Percentage.**  Depending on your workload a spike or a maintained high lock percentage can be a very bad sign.  This is very workload and even namespace dependent so I can't tell you what to look for, I'm sorry!<br /><div style="text-align:center">![A MMS Lock percent graph](http://i.imgur.com/l99X7ec.png "Increasing lock percentage means that there is an increasingly intensive write load coming into Mongo, perhaps due to a change in acces patterns.  As long as it does not impact other expected operations on the DB lock percent is perfectly fine")</div>

The most important thing is to know what your application needs in order to work effectively, and what will cause it to have problems.  Sometimes to do this you will need to implement custom alerting scripts that cannot be put into MMS currently.  Here you need to look elsewhere to new monitoring systems such as Nagios or Zabbix.  I have used Zabbix and I will talk a little bit about it here.

#### MongoDB monitoring with Zabbix

Zabbix is an [open source](http://www.zabbix.com/) monitoring framework that has been around for a long time, you can host it inside your own network and it allows you to utilize custom scripts.  A Zabbix template and script for monitoring MongoDB can be found [here](https://github.com/sailthru/mongodb-zabbix).  The alerting thresholds in the Zabbix template are set to set to some reaosnable defaults and should hopefully suit you well if you choose to go down this path.  This also allows you to consolodate all of your monitoring for your system into a single location, which can be very helpful.  There are many other monitoring frameworks and what works best for your orgnaization will vary.  Don't be afraid to strike out on your own, the benefits of having your monitoring exactly what is needed for your organization will outweigh the setup costs in the long run.

![An example Zabbix dashboard](http://i.imgur.com/hS9Yaqn.png, "An example Zabbix monitoring dashboard")

### Migrating data out of Mongo and into Mongo

Code changes, requirements change, and sometimes so must the data.  Many times migrating data from one store to another also falls on the DBA.  Even Mongo to Mongo migrations might be required to restructure the data or in order to shard effectively (by either changing the shard key or moving data into a cluster).  There are many ways to move data from Mongo instance to another.  I'll discuss utilizing some of the out of the box mongo tools in order to quickly move data.

1. **Mongodump and Mongorestore.**  These are good tools that come packaged with mongo, and especially in their 3.0 variants where much better multithreading and batching increases their performance as well as the vital ability of mongorestore to be able to restore from standard input rather than only from static files.  However it is important that if you are looking for speed that you pipe directly from mongodump to mongorestore.  This will only be possible if you are piping one collection to another collection, if you dump an entire database to a file then you are going to be working at the speed of your disk.  In the end if you touch disk you lose the performance battle.  An example of what this might look like (using the 3.0 tools): <br />
>`mongodump --host <SOURCE_HOST> --port <SOURCE_PORT> --db <SOURCE_DB> --collection <SOURCE_COLLECTION> --out - | mongorestore --host <TARGET_HOST> --port <TARGET_PORT> --db <TARGET_DB> --collection <TARGET_COLLECTION> -`
2. **Mongoconnector.**  This is an [open source](https://github.com/10gen-labs/mongo-connector) python tool from mongo that allows you to tail the oplog of a replica set or cluster and apply these changes to your target datastore.  Unlike mongodump and mongorestore it is possible to use mongoconnector for more than just mongo to mongo and you can write your own document manager to write to any datastore you like.  The downside is that the initial sync of data is slow, however it is possible to use the previous technique to copy data over quickly and then use MongoConnector to copy what you missed during that migration.  You can do this by recording the oplog timestamp just before you start the mongodump -> mongorestore operation in the file that mongoconnector specifies in the --oplog-ts argument.  Then when the dump and restore operation is complete you can start mongoconnector and it will run all operations from that timestamp forward (Be sure your oplog is long enough, if the oplog is not long enough to pick back up at that timestamp you will need to do something else.).

![A schematic of how Mongoconnector works](http://i.imgur.com/04JUSsg.png)

## What should I do to be a better Mongo User?

### Have a care for how your data will be used

Building your data to suit your access patterns is what you hear all the time with MongoDB, but that is because it is true.  If you are looking up many documents individually then a hashed shard key is ideal (as may be a keystore lookup, which you can hear described more in my talk from MongoDB World 2015).  If you are doing longer running queries more often then be sure to choose a shard key that has your query localized to a single shard.  You are the best one to choose how to structure your data as you know how it will be used.

### Design and document your schemas

MongoDB is heralded as having no schema, but if you talk to anyone who has worked with it they will tell you that their data has a schema.  All data has some sort of structure and defining it beforehand and documenting it will help you in the long run.  Maintinence will be easier, bringing people onboard will be much more rapid, and engaging external expertise will be much quicker and more productive when you know what you have.  This isn't to say that your data can't change over time, but if it does then document that and know what fields may or may not be there.  Additionally Mongo does allow you to have whatever data type you want in a field and it can vary from document to document - mixing datatypes in the same field name in the same collection is a very bad idea for performance and deterministic data access!  If your application expects a float and gets an int, especially if it is a statically typed language, you could force yourself to check and validate everything coming out of your database (a lot of overhead) or you might unexpectedly have your process crash, all bad things to have to do.

### Consider other datastores

Mongo is very powerful and flexible, what it is good at it is quite good at.  However it is not ideal for all datasets.  Keep an open mind when other datasets are better suited for your access patterns, it is dangerous to become so dependent on a single technology that you cannot innovate.  I'll list some brief patterns of access that could be examples of different datastores to use (but it is by far comprehensive in the access patterns or in the technologies listed).

1. **Link Shortening**.  If you are constructing some link shortening for your own domain then you might be creating some string that maps to a URL, effectively creating a hash map of these shortened links to the actual URL.  This can be stored in MongoDB as a document whose _id field is the shortened key and a single field of the URL, creating a key-value pair as a document.  You can get rapid insertion rates and speedy lookups from Mongo but there are other technologies, such as Redis or Aerospike, that may give you even greater performance and flexibility as key-value stores.
2. **Customer Accounts**.  The traditional problem of maintaing a balance or an account for a customer is frought with potential race conditions.  You want to put a credit or deduction on their account balance and make sure the event is recorded as well.  You cannot change the account balance without making sure the event is recorded or the other way around.  Instead of implimenting a system of locks using your application a database that supports transactions will save you (If you are trying to impliment ACID in your application, stop!  Someone else has already spent a long time working on this, just stand on the shoulders of giants!).  Tranditional relational databases such as Postgres, or MySQL would serve you better.
3. **Statistics Rollup**.  What if you would like to record the traffic on your site down to the number of clicks per minute and be able to quickly access this data by minute or hour granularity?  For this MongoDB's nested structures are extremely efficient and make sense.  You will almost always be pulling many of these data points at once to form reports or graphs.  Structuring the data so that your access requires you to only pull a single document from MongoDB when a more relational schema would require many rows to be queried then your use case is perfect for MongoDB.  There is a very well done and comprehensive blog post by Rick Copeland [here](blog.pythonisito.com/2012/09/mongodb-schema-design-at-scale.html) that describes this use case.
4. **Music Albums**.  What if you are storing data for music artists?  Do you want to use a relational system or a document store?  If you are often pulling just the albums and are rarely delving into the information of every song on the album it might be better for the application to make a single call and the album to be a single document with the tracks nested within it.  If your application will want to access tracks across albums often then the album should be another table that is joined to the tracks by a foreign key.  Here we see we can either go with a relational database such as MySQL or a document store such as MongoDB, what will sway us is how will the application actually use this data.

Using the right tool for the job, not just a tool that will work, can get you very far.  This is often easier said than done, there are many technologies (There are entire categories of datastores such as graph databases I haven't even mentioned) and other considerations such as in house expertise and current infrastructure also must weigh heavily on the decision.  So long as you consider other technologies for your problems your application will be stronger for it.

### Decouple and build in abstraction

My final piece of advice is to decouple and abstract your application as much as possible away from the datastore.  Taken to its end you could access the database through an entirely RESTful API that abstracts the database away from the application (the microservice architecture is extremely well suited to this and can solve a lot of tight-coupling issues before they start). A specific module where all database access is contained in the application would serve a similar purpose.  It is important to make sure that all parts of the application are as loosely coupled as possible.  Modifications in structure or access of the data as your application grows then become a localized task rather than a global task, leading to more flexibility and reliabilty in development.

## The landscape of Databases is changing

For decades SQL databases were the lords of the land and recently there has been a huge explosion of new datastores, NoSQL and MongoDB being only one facet of this.  The field of databases is changing rapidly and with it must come a changing of the idea of a DBA.  I have begun to use the term Database Engineer (DBE) to describe what is needed for this changing field.  A DBE must be able to choose a datastore that is ideal for the work of the application and cannot simply expect to be given what data is being stored and normalize it across many tables; a developer and a DBA at once (Indeed I find myself writing programs and reading code more often than I find myself optimizing queries, building indexes, or tailing logs!).  However as we have seen there are many facets of the old DBA mindset, such as backups, that are still just as vital as they once were.  Using MongoDB does not mean that an orgnaization does not need a DBA because someone must still manage the datastore with new skills and new insights that were not so needed even five years ago.