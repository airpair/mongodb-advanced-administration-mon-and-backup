## Introduction

This blog post was based on (but is not identical to) my talk given at the MongoDB World 2015 conference.  If you would like to see the talk then luckily for you it was recorded and it can be found on Mongo's website.

### Who is this for?

The land of NoSQL is changing rapidly and it is easy to get caught up in a lot of the hype, especially with MongoDB.  I think that MongoDB is a great tool - but like all tools it can be misused or applied to the wrong task.  I'm going to be talking about how you can take control of a MongoDB deployment and begin imposing order, stability, and determinism when your stack gets out of hand.  If you are looking to make sure your MongoDB implimentation will not take a team to fix, or have a deployment and don't know where to start fixing it then I hope my experiences make your task a bit easier.  I'm going to assume some working familiarity with MongoDB in this post so if you don't know what sharding is or what replica sets are you might want to come back to this after you have gotten a little deeper into MongoDB.

### What is MongoDB?

MongoDB is the leading NoSQL database, NoSQL being a catch-all for any database that is not a relational SQL based database.  I might argue that it is more useful to call MongoDB a document store, because it stores JSON documents, and not so much as a database which invokes thoughts of schemas, foreign keys, transactions, and joins.  MongoDB does not enforce a schema, it does not allow you to do joins within the database, but it is very good at scaling horizontally and it is very developer friendly.  It can be used easily, deployed quickly, and get your application working very fast.  However with great power comes great responsibility.

### Why do I still need a DBA for Mongo?

A school of thought has emerged that with these schemaless datastores which are easy and fast to set up and tie into applications that the DBA is no longer needed.  No more will there be 100+ line SQL statements that need tuning, no more will there be parameters in the deployment of the database that need to be tweaked to get performance.  This has led many down the path of putting data in a location without care for what access controls are around the data, how it may later be used, and how to optimize the data for performance with specific use cases.  Monitoring the database and backing up the data also then fall onto the shoulders of perhaps the operations team.  Effectively the role of DBA was split up between the development team - which has their focus on things other than the data - and the operations team whose focus is again elsewhere.  With the coming of MongoDB 3.0 there are increasing number of parameters to tweak to optimize performance even beyond choosing the storage engine for the application or the shard key to distribute on.

If this is sounding daunting all of a sudden that is because it quickly becomes so.  The role of the DBA in the NoSQL future is as strong as it has been in the past.  The tools change and your codebase may change but your data will remain, and upi meed people who will speak for your data for it has no tounge.  These are the DBAs.

## Tales of DBA Struggle

When I joined Sailthru as a DBA the team was newly formed, the MongoDB deployment had been left to some combination of Operations members and Development for some time and many people had their hands in the pie but few knew or had ownership over what was going on.  Thus began several tales of struggle.

### Backups

As a DBA the first thing to assess is how backed up is the data.  If your data access is slow it can be sped up later, if your monitoring is poor it can be shored up later, but if your backups are not working and you lose your data you cannot fix your backups later.  I'm going to discuss two options to backing up data for MongoDB: hybrid cloud and MMS backups.

A hybrid cloud is where you have part of your application running on physical machines that you manage and the other part is running in a cloud provider such as AWS.  A hybrid cloud system with MongoDB is essentially having part of each replica set in the cloud syncing to part of the replica set that is running on your hardware.  This has several benefits:

1. **Redundancy.** Disaster recovery is immediate.  You have a working copy of your data offsite so you can immediately begin using that set.
2. **Backups.**  Snapshots can be taken easily and are also stored off site.

However there are also downsides:
1. **Cost.**  This system could end up costing a lot of money, especially as you must make sure the MongoDB instances are provisioned so they can keep up with the instances in your datacenter.
2. **Security.**  You must maintain a secure connection to the cloud provider as replication in MongoDB occurs without encryption.
3. **Bandwidth.**  You must have enough bandwidth to be able to keep this data set in sync or it will quickly become useless, requiring manual intervention.
4. **Time.**  It will in all likelihood take a lot of time to maintain, especially if members in the cloud fall out of sync and a manual resync operation is required.

It can look like the costs outweight the benefits by sheer numbers, but let's be fair those two benefits are really big benefits.  What got me to give up a hybrid cloud was when we used this technique with a sharded cluster.

In a single replica set it is easy to say if we have a consistent backup of it or not, easy to monitor and if there is a problem it is easy to rectify.  However when you have a cluster all replica sets (shards) that compose the cluster must be backed up successfully for the entire cluster to be backed up.  As traffic and load increase it may be hard to do this, and if you have a single shard unable to keep up then your entire backup - no matter if it is a hundred shard cluster - is now incomplete.  This was a big problem, big enough to scrap our hybrid cloud architecture.

We moved to MMS, MongoDB's backup solution.  MongoDB will stream your data to their datacenter in a compressed and encrypted format.  It will then take snapshots for you off site and store them, giving you very granular (between 15min and to the second in many cases) restore capabilities and redundancy.  Surprisingly this was also cheaper than our hybrid cloud solution and it was clear that migrating to this would give us more time to focus on other tasks.  The MMS backup has been relatively straightforward and has worked quite well.  Some things to keep in mind however:

1. **Automation.**  Currently there is no automation tied into the MMS backup solution, which means you will have to go through their relatively new API to try and get your backup tasks done and write your own automation, or you will have to use the UI.  For any sizeable deployment automation will be required for any speedy response time, so keep this in mind.
2. **Offsite.**  You will need to stream the data over the internet to its destination, these backups are not close to your servers, so it will take time to move the data.  Factor this into your disaster recovery plan.  However you will not need to execute mongorestore on the data that MongoDB ships you, it is all in datafiles that you can point mongod to immediately and begin using.

MMS backup can be a very good solution and it shouldn't be discounted, especially if your manpower is limited.  Not having backups is not an option for any organization.

### Monitoring

Monitoring is the second most crucial thing for a DBA to set up.  Being able to know if your system is behaving as expected or outside normal bounds, and being able to quickly narrow problems down to the source is necessary for confidence in the operation of the data layer.

Anyone playing with MongoDB out of the box should set up MMS, this will be an excellent monitoring and alerting system for you.  This gives you a lot of information but it might not be very helpful without context, especially if you are using this to alert you by SMS or by some third party such as PagerDuty (MMS does have integration with PagerDuty).  Here are some things to look out for:


1. **Read / Write Queues.**  These are real good indicators if you have overloaded what MongoDB can get done, even if your lock percent is high if you don't have queues you often don't have a problem.  If you do start to sustain a queue you'll see slowdown in your application.
2. **Network Traffic.**  MongoDB can push as many bits as the network interface can handle, and at times this has been the only indicator as to why things are getting slow.  If you see your network graph plateau then there is a problem and likely where your performance bottleneck lies.
3. ** Background Flush Average.**  This represents how long it takes to flush the data to disk, a high value of the background flush will indicate that the disk is overloaded.  A background flush is executed every 60 seconds so keep that in mind, a background flush of 40 seconds is extremely high.
4. **Lock Percentage.**  Depending on your workload a spike or a maintained high lock percentage can be a very bad sign.  This is very workload and even namespace dependent so I can't tell you what to look for, I'm sorry!

The most important thing is to know what your application needs in order to work effectively, and what will cause it to have problems.  Sometimes to do this you will need to implement custom alerting scripts that cannot be put into MMS currently.  Here you need to look elsewhere to new monitoring systems such as Nagios or Zabbix.  I have used Zabbix and I will talk a little bit about it here.

Zabbix is an open source monitoring framework that has been around for a long time, you can host it inside your own network and it allows you to utilize custom scripts.  A Zabbix template and script for monitoring MongoDB can be found [here](https://github.com/sailthru/mongodb-zabbix).  The alerting thresholds in the Zabbix template are set to set to some reaosnable defaults and should hopefully suit you well if you choose to go down this path.  This also allows you to consolodate all of your monitoring for your system into a single location, which can be very helpful.  There are many other monitoring frameworks and what works best for your orgnaization will vary.  Don't be afraid to strike out on your own, the benefits of having your monitoring exactly what is needed for your organization will outweigh the setup costs in the long run.

### Migrating Data

Code changes, requirements change, and sometimes so must the data.  Many times migrating data from one store to another also falls on the DBA.  Even Mongo to Mongo migrations might be required to restructure the data or in order to shard effectively (by either changing the shard key or moving data into a cluster).  There are many ways to move data from Mongo instance to another.  I'll discuss utilizing some of the out of the box mongo tools in order to quickly move data.

1. **Mongodump and Mongorestore.**  These are good tools that come packaged with mongo, and especially in their 3.0 variants where much better multithreading and batching increases their performance.  However it is important that if you are looking for speed that you pipe directly from mongodump to mongorestore.  This will only be possible if you are piping one collection to another collection, if you dump an entire database to a file then you are going to be working at the speed of your disk.  In the end if you touch disk you lose the performance battle.
2. **Mongoconnector.**  This is an [open source](https://github.com/10gen-labs/mongo-connector) python tool from mongo that allows you to tail the oplog of a replica set or cluster and apply these changes to your target datastore.  Unlike mongodump and mongorestore it is possible to use mongoconnector for more than just mongo to mongo and you can write your own document manager to write to any datastore you like.  The downside is that the initial sync of data is slow (bonus points for combining these two techinques for a fast and zero downtime data migration.).

## What should I do to be a better Mongo User?

### Have a care for how your data will be used

Building your data to suit your access patterns is what you hear all the time with MongoDB, but that is because it is true.  If you are looking up many documents individually then a hashed shard key is ideal (as may be a keystore lookup, which you can hear described more in my talk from MongoDB World 2015).  If you are doing longer running queries more often then be sure to choose a shard key that has your query localized to a single shard.  You are the best one to choose how to structure your data as you know how it will be used.

### Design and document your schemas

MongoDB is heralded as having no schema, but if you talk to anyone who has worked with it they will tell you that their data has a schema.  All data has some sort of structure and defining it beforehand and documenting it will help you in the long run.  Maintinence will be easier, bringing people onboard will be much more rapid, and engaging external expertise will be much quicker and more productive when you know what you have.  This isn't to say that your data can't change over time, but if it does then document that and know what fields may or may not be there.  Additionally Mongo does allow you to have whatever data type you want in a field and it can vary from document to document - mixing datatypes in the same field name in the same collection is a very bad idea for performance and deterministic data access!

### Consider other datastores

Mongo is very powerful and flexible, what it is good at it is quite good at.  However it is not ideal for all datasets.  Keep an open mind when other datasets are better suited for your access patterns, be it highly relational data where a more traditional SQL database will serve you better or a key value store like Redis.  Using the right tool for the job, not just a tool that will work, can get you very far.

### Decouple and build in abstraction

My final piece of advice is to decouple and abstract your application as much as possible away from the datastore.  Taken to its end you could access the database through an entirely RESTful API that abstracts the database away from the application (the microservice architecture is extremely well suited to this and can solve a lot of tight-coupling issues before they start). A specific module where all database access is contained in the application would serve a similar purpose.  It is important to make sure that all parts of the application are as loosely coupled as possible.  Modifications in structure or access of the data as your application grows then become a localized task rather than a global task, leading to more flexibility and reliabilty in development.

## Go build something!

MongoDB is a great tool that has a rapidly growing community around it.  Running and maintaining large distributed datastores can be daunting but it is very doable and can take your application to the next level.

