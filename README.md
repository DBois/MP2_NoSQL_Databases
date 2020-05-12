# Mini Project 2: NoSQL Databases


## Chosen Databases
- [MongoDB](https://docs.mongodb.com/): Document-Oriennted  
- [Redis](https://redis.io/documentation): Key-Value  

## Chosen Dataset
- [E-commerce with 541.906 rows](https://www.kaggle.com/carrie1/ecommerce-data/data) on Kaggle

## Setup
- [Populate the Redis databse with this C# tool](https://github.com/DBois/RedisPopulator)  
- With MongoDB's Compass you can just load the CSV file. If using the console, you can use this command: `mongoimport --type csv -d <DBNAME> -c <COLLECTION_NAME> --headerline --drop <FILENAME>.csv`  


## Relevant database operations
### Search and update
**Redis**:  
With the Redis-CLI you can not search for or update multiple hashes at the same time, you are only able ot update a single hash at a time. If you want to update multiple hashes you would have to write a script in a programming language.   
![Image depicting a search query in Redis. After that the found item is updated. And then it is queried again to show the change](https://i.imgur.com/SmKsyPh.png)  


**MongoDB**:  
With MongoDB you can use the ManyUpdate command to update several several documents at the same time. `db.orders.updateMany({InvoiceNo: "536365"},{$set: {Country: "Denmark"}});`

### Insert


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

### [Indexing](https://redis.io/topics/indexes) 
**MongoDB:**  
Indexes in MongoDB are similar to indexes in other DB systems. They are defined at the collection level, and any field or sub-field of the documents can be indexed. By default the _id is a unique index. We tried adding an index on the *Country* field using the following command: `db.ecommerce.createIndex({Country: -1})`. We see below that we cut 1ms off the execution time, and only 8532 documents were examined.  
**Before Index**
![Image showing mongoDB is querying every document wihout an index](https://i.imgur.com/1IFHoAI.png)  
**After Index**
![Image showing mongoDB only querying the documents which were indexed](https://i.imgur.com/EMfPufX.png)

**Redis:**  
Redis only offers primary key access, it means any indexing would have to be done on the key. As Redis states themselves: `Implementing and maintaining indexes with Redis is an advanced topic, so most users that need to perform complex queries on data should understand if they are better served by a relational store.`  
For example, if we want to index our orders by country, we would have to include the country in the **key**.

## Security

**MongoDB**:
**Redis**: 

## Storage

**MongoDB**:
**Redis**: 

## Criteria for comparison

## Demo

## Results and conclusion
Redis is more troublesome than Mongodb to work with in relation to querying, but is great for supplying very fast responses to those queries.
Redis is limited by the amount of RAM on the machine. If the datasize exceeds the RAM, you need to incorporate clustering. 

### ACID and CAP
**MongoDB**: Consistency - Partition Tolerance, from version 4.0 MongoDB is ACID compliant _Multi-Document Transaction_.   
**Redis**: Consistency - Partition Tolerance, Redis is single threaded which allows it to be ACID compliant.



![Picture of CAP Theorem](https://i.stack.imgur.com/iEC8r.png)