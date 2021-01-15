% Advanced Caching Patterns with Django
% Andrew Brookins
% January 13, 2021

# Advanced Caching Patterns with Django

## Notes

* Read-through, write-through, and write-behind are typically features of commercial cashes
* Commercial examples: [Hazelcast](https://hazelcast.com/), [Oracle Coherence](https://www.oracle.com/middleware/technologies/coherence.html)
* (Sort of) non-commercial examples: [Redis Gears RGSync recipe](https://github.com/RedisGears/rgsync), [Redisson](https://github.com/redisson/redisson/wiki/7.-distributed-collections#write-behind-asynchronous-strategy)

----

## Look Aside

"Application reads from the cache and lazy-loads from the database."

----

```
    App    Cache
    ----------------------
    <- Read from cache
    .     .
    .     (Found?)
    .     <- Return data
    <- Return data
    .     .
    .     (Not found?)
    <- Read from database
    -> Set in cache
    <- Return data
```

----

Look Aside is the most common caching pattern in Django apps.

----

We'll start with it to provide a base of comparison.

----

* App reads from the cache
* If an entry is missing, the app reads from the database and updates the cache (AKA, "lazy loading")
* Why? Simple, uses less cache memory
* Why not? Only speeds up reads -- not writes


## Write Through

"Cache updates itself and the database synchronously."

----

```
    App    Cache
    --------------------------
    -> Read from cache
    .      .
    .      (Found)
    .      <- Return data
    <- Return data
    .      .
    .      (Not found)
    .      -> Set in cache (self)
    .      -> Set in database
    .      <- Return data
    <- Return data
```

----

* App always writes to the cache
* Cache writes synchronously to the database -- in other words, from the app's perspective, when a write succeeds, the cache and database are consistent
* Why? Strong consistency between cache and database
* Why not? Uses more cache memory, slow writes (have to write to the cache _and_ the database)

## Write Behind

"Cache updates itself and the database asynchronously."

----

```
    App   Cache
    --------------------------------------
    .     .
    -> Read from cache
    .     .
    .     (Found)
          <- Return data
    <- Return data
    .     .
    .     (Not found)
    .     -> Set in cache (self)
    .     -> Save job in background thread
    .     <- Return data
    <- Return data
    .     .
    .     ...(Later, in background thread)
    .     -> Write data to database
```

----

* App always writes to the cache
* Cache writes **asynchronously** to the database -- in other words, from the app's perspective, when a write succeeds, the change may not be written to the database
* Why? *Fastest* writes
* Why not? Uses more cache memory; consistency problems: chance that users won't see their own writes, etc.

## Write Through with Application Logic

* Write through usually means the cache handles the writes
* However, it's also a way of populating the cache with recent changes
* You can simulate Write Through by having Django update the cache when it saves/updates models

## Write Behind with Application Logic

* Write behind also usually means the cache handles the writes
* But the point is, ultimately, very fast writes
* You can simulate Write Behind by having Django update the cache and save a background job for the DB write

## Write Through with Redis Gears

* If you are using MySQL, SQLite, or Oracle, you can use the RGSync recipe with Redis Gears
* This is an actual write through -- the cache writes to the database
* But it's pretty new, and may be rough
* With Postgres, you need to write your own Gears function, which is a "project"

## Write-Behind with Redis Gears

* Same as write through with Gears
* Writing in the background, by using Streams or a List, gives you fast writes
* With Postgres, you need to write your own Gears function, which is a "project"

## Just Use Redis

"The cache is the database."

* Redis has two forms of persistence: RDB and AOF
* If you need extreme write speed, using Redis is going to beat a cache + database
* You could periodically sync data from Redis to Postgres for querying/reporting
* Or you might use a [Redis Foreign Data Wrapper](https://github.com/pg-redis-fdw/redis_fdw) to expose Redis data in Postgres
