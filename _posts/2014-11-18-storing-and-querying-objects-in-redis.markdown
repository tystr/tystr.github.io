---
layout: post
title: Storing and Querying Objects in Redis
summary: Redis is a super fast, in-memory, advanced key-value store capable of lightning quick operations. While it is commonly used for tasks such as caching, realtime leaderboards, analytics, and similar, in this post I am going to explain how you can use redis for storing and efficiently querying millions of objects.
keywords: redis
date: 2014-11-18
class: post-body
---
Redis is a super fast, in-memory, advanced key-value store capable of lightning quick operations. While it is commonly used for tasks such as caching, realtime leaderboards, analytics, and similar, in this post I am going to explain how you can use redis for storing and efficiently querying millions of objects.

## Storing Objects
Let's say we have a Car object that has some properties like "make", "model", "color", "topSpeed", and "manufactureDate". We can store this in a redis hash key quite easily:

    redis 127.0.0.1:6379> HMSET cars:1 make Ferrari model 458 color red topSpeed 202mph
    OK

## Finding Objects by its id
Notice that we've used `cars:1` as the key; in this case `1` is a unique identifier. It's up to the application to maintain unique Ids. So to look up a car by it's id, simply do

    redis 127.0.0.1:6379> HGETALL cars:1
    
But what happens when you want to find all cars made by Ferrari? Here's where the application code becomes a little more complicated. Since you cannot query on properties of a hash key, you have to manually build and maintain "indexes". For the purpose of this post an "index" is a key where the name is comprised of a concatenation of the property name and a value, (e.g. `make:Ferrari`) and whose value is a set containing ids of objects that match the property and value indicated by the key name. For example, the key `cars:Porsche` would contain a set of ids of all cars whose make is Porsche. This means anywhere in your application where you save your object, you must update the index keys accordingly:

    redis 127.0.0.1:6379> HMSET cars:1 make Ferrari model 458 color red topSpeed 202mph
    OK
    redis 127.0.0.1> SADD make:Ferrari 1
    (integer) 1

Finding all cars whose make is Ferrari is as simple as this:

    redis 127.0.0.1:6379> SMEMBERS make:Ferrari
    
## Finding Objects matching multiple criteria    
Now that there are index keys for the queryable properties, we can filter using multiple criteria with the redis commands `SINTER` and `SUNION`.

To find all Ferraris that are yellow, we would intersect the sets `make:Ferrari` and `color:Yellow`.

    redis 127.0.0.1:6379> SINTER make:Ferrari color:Yellow

To find all yello cars that are either Ferrari or Porsche:

    redis 127.0.0.1:6379> SUNION make:Ferrari make:Porsche color:Yellow
   
## Range Queries	
Querying based on a range such as a date gets a little trickier. We can accomplish this with redis sorted sets. A sorted set is simply a set where each element has a score by which it is sorted. If we wanted to be able to find cars using their manufacture date, we need to store the ids in a sorted set with the manufacture date timsestamp as the score:

    redis 127.0.0.1:6379> HMSET cars:1 make Ferrari model 458 color red topSpeed 202mph manufactureDate 1389772800 
    OK
    redis 127.0.0.1:6379> ZADD manufacture_date 1389772800 "1"
    (integer) 1
	redis 127.0.0.1:6379> HMSET cars:2 make Porsche model Carrera color black topSpeed 179mph manufactureDate 1391212800
    OK
    redis 127.0.0.1:6379> ZADD manufacture_date 1391212800 "2"
    (integer) 1
    
And then to query for cars manufactured between January 1st, 2014 and January 31st, 2014:

    redis 127.0.0.1:6379> ZRANGEBYSCORE manufacture_date 1388534400 1391212799
    1) "1"

## Mixing Criteria Types
We've seen that finding objects matching multiple criteria is as simple as performing an intersection or union of the appropriate set keys. But what if the query includes a range element, such as red cars manufactured during January? We just need to compute the intersection of the `color:red` and `manufacture_date` key; here we use the ZINTERSTORE command which operates on both sets and sorted sets:

    redis 127.0.0.1:6379> ZINTERSTORE tmpkey 2 color:red manufacture_date AGGREGATE MAX
    
Now we can pull out the range we want just as if we were working with the sorted set key in the previous example:

    redis 127.0.0.1:6379> ZRANGEBYSCORE tmpkey 1388534400 1391212799
    
## Conclusion
Although redis is "just" a key-value store, it can be an efficient tool for use as a primary database at scale. (There are a few configurable strategies for how and when data is flushed to disk, but this is outside the scope of this post. You can read about redis persistence <a target="_blank" href="http://redis.io/topics/persistence">here</a>). Querying is efficiently done using built in intersection and union functions. There are all the benefits of a schemaless database. So no columns to add or indexes to rebuild when adding new data points. Intersecting more sets (more complex queries) does not result in an exponentially longer query time. One caveat, though, is that redis is single-threaded, and these commands are blocking operations. So while most operations are quick, when dealing with sets that contain millions of records, you could end up blocking all operations while the intersections are performed. This could be trivially solved by reading form a slave replica.
