# Mini Project 2: NoSQL Databases


## Chosen Databases
- [MongoDB](https://docs.mongodb.com/): Document-Oriented  
- [Redis](https://redis.io/documentation): Key-Value  

## Chosen Dataset
- [E-commerce with 541.906 rows](https://www.kaggle.com/carrie1/ecommerce-data/data) on Kaggle

## Setup
- [Populate the Redis database with this C# tool](https://github.com/DBois/RedisPopulator)  
- With MongoDB's Compass you can just load the CSV file. If using the console, you can use this command: `mongoimport --type csv -d <DBNAME> -c <COLLECTION_NAME> --headerline --drop <FILENAME>.csv`  


## Relevant database operations
### Search and update
**Redis**:  
With the Redis-CLI you can not search for or update multiple hashes at the same time, you are only able to update values for a single key at a time. If you want to update on multiple keys you would have to write a script in a programming language.  
![Image depicting a search query in Redis. After that the found item is updated. And then it is queried again to show the change](https://i.imgur.com/SmKsyPh.png)  





**MongoDB**:  
With MongoDB you can use the ManyUpdate command to update several several documents at the same time. `db.orders.updateMany({InvoiceNo: "536365"},{$set: {Country: "Denmark"}});` This makes updating several items much easier than in redis.

**Benchmark:**  
The following script in python gets the average time for updating and searching for a key in redis.  
![Image showing benchmark of average search and update based on 100.000 iterations. Result shows average time of 0.19ms](https://i.imgur.com/C9LF6eE.png)  
The following script in C# gets the average time for finding and updating a document in MongoDB. 
```csharp
Console.WriteLine("Starting");
            
            const string connectionString = @"mongodb://localhost:27017/?readPreference=primary&appname=MongoDB%20Compass%20Community&ssl=false";
            var dbClient = new MongoClient(connectionString);
            var db = dbClient.GetDatabase("ecommerce");
            var collection = db.GetCollection<BsonDocument>("orders");
            var stopWatch = Stopwatch.StartNew();
            var filter = Builders<BsonDocument>.Filter.Eq("InvoiceNo", "536365");

            for (var i = 0; i < 100_000; i++)
            {
                var update = Builders<BsonDocument>.Update.Set("Quantity", i);
                collection.FindOneAndUpdate(filter, update);
            }
            stopWatch.Stop();
            Console.WriteLine($"Average time: {100_000/ (double)stopWatch.ElapsedMilliseconds:N5}ms");
```  
![Average time 3,02654ms](https://i.imgur.com/4Xui7Vc.png)

### Insert
**Redis**: Same is with search and update, you have to insert one at a time. Your options are to write a script in a programming language or find a 3rd party tool  
**MongoDB**: With MongoDB you can make bulk inserts with giving a list of objects with the a command like this: `db.orders.insertMany([list of objects])`
![Image depicting use of insertMany command](https://i.imgur.com/S8ZRgEZ.png)  
We compared the insertion performance of both MongoDB and Redis, by inserting 100.000 elements from a [C# program](https://github.com/DBois/RedisMongoComparator) and compared the averages. As we can see in the results below, Redis is faster when not using insertMany from MongoDB:  
![Average speed of inserting an element to Redis is 0,14ms and 0,22 for mongoDB](https://i.imgur.com/DwxJtqF.png)  
We compared MongoDB's insertMany vs inserting hashes into Redis, the result confirms Mongo is faster for bulk operations.  
**InsertMany vs Redis HSET in C#**  
![MongoDB insertMany took almost 9 seconds. Redis insertion took over a minute](https://i.imgur.com/ugCMXH3.png)



### Migrating data
**Redis**: Gives you a tool to dump files as a .rdb file, which gives you the option to easily migrate to another redis DB. However exporting to a more generic format such as CSV, JSON or XML requires you to find third party tools or commands. This makes it not as user friendly as MongoDB   
**MongoDB**: Gives you a tool to export files as either a .csv or .json. You can in the export tool write a query to narrow down what you want to export. This makes MongoDB export functionality easy to use compared to Redis.

### [Transactions](https://redis.io/topics/transactions)
**MongoDB**:

"In MongoDB, an operation on a single document is atomic. Because you can use embedded documents and arrays to capture relationships between data in a single document structure instead of normalizing across multiple documents and collections, this single-document atomicity obviates the need for multi-document transactions for many practical use cases.

For situations that require atomicity of reads and writes to multiple documents (in a single or multiple collections), MongoDB supports multi-document transactions. With distributed transactions, transactions can be used across multiple operations, collections, databases, documents, and shards."

ref: https://docs.mongodb.com/master/core/transactions/

Below you can see an example on how MongoDB transactions works in practice.

```csharp=
with client.start_session() as s:
    s.start_transaction()
    collection_one.insert_one(doc_one, session=s)
    collection_two.insert_one(doc_two, session=s)
    s.commit_transaction()
```

**Redis**: Transaction in Redis works around a few keywords;  
- Multi
- Discard
- Exec
- Unwatch
- Watch

These commands allows the execution of a group of commands in a single step. Using these commands guarantees; that all the commands in the transaction are serialized and executed sequentially. In other words its a executed as a single isolated operation. It also ensures that the transactions are atomic, meaning either all of the commands or none are processed.

A Redis transaction begins with the `Multi` command, from here Redis will queue the commands. The queue of commands are executed once the `Exec` command is called. However calling `Discard` will flush and exit the transaction.

The `Watch` command will make the `Exec` command conditional. By using `Watch` Redis will only perform the transaction if none of the watched keys were modified. In-addition you can use the `Unwatch` command to flush all the watched keys... However you can also add arguments to this command, to specify which keys to unwatch.  
**MULTI**
```
> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
```  
**WATCH failing**  
```
Console 1:
WATCH InvoiceNo:547223.22983
MULTI

Console 2:
hset InvoiceNo:547223.22983 UnitPrice 10
> (integer) 0

Console 1:
hset InvoiceNo:547223.22983 UnitPrice 3
EXEC
> (nil)
```

### [Indexing](https://redis.io/topics/indexes) 
**MongoDB:**  
Indexes in MongoDB are similar to indexes in other DB systems. They are defined at the collection level, and any field or sub-field of the documents can be indexed. By default the _id is a unique index. We tried adding an index on the *Country* field using the following command: `db.ecommerce.createIndex({Country: -1})`. We see below that we cut 1ms off the execution time, and only 8532 out of 541.9k documents were examined.  
**Before Index**
![Image showing mongoDB is querying every document wihout an index](https://i.imgur.com/1IFHoAI.png)  
**After Index**
![Image showing mongoDB only querying the documents which were indexed](https://i.imgur.com/EMfPufX.png)

**Redis:**  
Redis only offers primary key access, it means any indexing would have to be done on the key. As Redis states themselves: `Implementing and maintaining indexes with Redis is an advanced topic, so most users that need to perform complex queries on data should understand if they are better served by a relational store.`[-Redis](https://redis.io/topics/indexes)  
For example, if we want to index our orders by country, we would have to include the country in the **key**.

## Security

**MongoDB**:  

![Console displaying a message saying your DB has been stolen, and a randsom of 550USD is required.](https://i.imgur.com/9YKD8Me.png)

Authentication: SCRAM-SHA-1, MongoDB-CR. 

MongoDB can also employ externnal authentication protocols: LDAP, Kerberos.

MongoDB has RBAC (Role-based access control) - Theres alot of pre-made roles, however you can also make your own.

You can enable authorization using the --auth command. This enables it to control a user's access to a database and its resources.

[Enable Access Control](https://docs.mongodb.com/manual/tutorial/enable-authentication/)  
For our database we have created a readOnly role for the ecommerce database with the following command on the `admin` database.  
```
db.createRole(
   {
     role: "readOnlyEcommerce",
     privileges: [
       {
         actions: [ "find" ],
         resource: { db: "ecommerce", collection: "orders" }
       }
     ],
     roles: []
   }
);
```  
Then we create a user with the new role:  
`
db.createUser({user: "newUser1", pwd: "newUser1", roles: [{role: "readOnlyEcommerce", db: "ecommerce"}]});
`  
And finally we can login:  
`mongo --port 27017 -u newUser1 -p 'newUser1' --authenticationDatabase ecommerce`  
To test:  
![](https://i.imgur.com/kPi6tR8.png)  
![](https://i.imgur.com/ycQtpdk.png)  
![](https://i.imgur.com/ZAoG8ef.png)  
![](https://i.imgur.com/Fq6Z8lt.png)

[Reference](https://opensource.com/article/19/1/mongodb-security)  

**Redis**:  
To setup authentication on Redis, all you have to write in your console is: `CONFIG set requirepass "YourPasswordHere"`, and from now on you will authenticate yourself by writing `AUTH "YourPasswordHere"` when you have connected, or by connecting with the CLI with the following command: `redis-cli -a YourPasswordHere`

## Storage
Using mongoDB's default storage engine, *WiredTiger*, the data will be saved on the disk as JSON data. This is inherently slower than Redis in-memory storage. However if we use the In-Memory Storage Engine, mongoDB would not be far off from Redis' performance. ([source](https://scalegrid.io/blog/comparing-in-memory-databases-redis-vs-mongodb-percona-memory-engine/)) (In house testing of inMemory not possible, since it requires enterprise edition of MongoDB)


## Results and conclusion
Redis is more troublesome than MongoDB to work with in relation to querying, but is great for supplying very fast responses to those queries.
Redis is inherently faster than MongoDB when making individual queries since it is written into RAM. The difference depends on how fast read/write operation you can make to your SSD (or HDD) when using MongoDB. However Mongo is optimised for bulk operations, and can be faster than Redis in those situations. 


Since Redis is written in to RAM, it is limited by the amount of RAM you have on the machine. If the datasize exceeds the RAM, you need to incorporate clustering.  Therefore MongoDB has easier scaling. 
MongoDB is great for prototyping or startups since it is schemaless. 


### ACID and CAP
**MongoDB**:  
Mongo resolves network partitions by maintaining consistency, therefore compromising on the availabity- Which makes Mongo a CP (Consistency - Partition Tolerance) data store.

Since version 4.0 MongoDB has been ACID compliant, this is because it uses _Multi-Document Transaction_.   

**Redis**:   
Same as Mongo, Redis is a CP. However Redis still has, A (High Availability) this is because of the Master Slave architecture - if a master fails then Redis Sentinels will promote a Slave to be the new Master. Thus making the entire solution A. 

However, if P (Network Partition) happens then Redis becomes unvailable in the minority partition. This is why Redis is CP


Redis is single threaded which allows it to be ACID compliant.
![Picture of CAP Theorem](https://i.stack.imgur.com/iEC8r.png)

## Conclusion
As most choices, it depends on the use case. Overall, mongoDB is easier to work with, has more features and faster if you want to do bulkoperations. Mongo allows you to create indexes, which makes searching data sets much faster, compared to sequentially searching every element in the database. Moreover, creating Roles in MongoDB allows you to have varying levels of access to your data. On the other hand, what Redis wants to do - it does extremely well. Single read/write operations are insanely fast due to the DB saves objects in RAM. For purely bandwidth comparisons, DDR4-3200mhz RAM is around 50x faster than some of the fastest m2 NVME SSDs.

So if you need the fastest, cachelike functioning database, go for Redis. If you need a more fully fledged database, go for mongoDB. Alternatively, if speed matters but you like mongoDBs functionality too, a good compromise would be to make use of mongoDBs storage engine *inMemory*. This makes mongoDB almost as fast as redis, and you keep the existing functionality 

 

