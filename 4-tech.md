# computers

go faster pls

Note:

So, we're a software company.
We make money by selling access to software.
We save money by making our software as cheap as possible to run.
Let's talk about some software specific stuff!


```javascript
Latency Comparison Numbers
--------------------------
L1 cache reference                           0.5 ns
Branch mispredict                            5   ns
L2 cache reference                           7   ns                      14x L1 cache
Mutex lock/unlock                           25   ns
Main memory reference                      100   ns                      20x L2 cache, 200x L1 cache
Compress 1K bytes with Zippy             3,000   ns        3 us
Send 1K bytes over 1 Gbps network       10,000   ns       10 us
Read 4K randomly from SSD*             150,000   ns      150 us          ~1GB/sec SSD
Read 1 MB sequentially from memory     250,000   ns      250 us
Round trip within same datacenter      500,000   ns      500 us
Read 1 MB sequentially from SSD*     1,000,000   ns    1,000 us    1 ms  ~1GB/sec SSD, 4X memory
Disk seek                           10,000,000   ns   10,000 us   10 ms  20x datacenter roundtrip
Read 1 MB sequentially from disk    20,000,000   ns   20,000 us   20 ms  80x memory, 20X SSD
Send packet CA->Netherlands->CA    150,000,000   ns  150,000 us  150 ms
```

Note:

These are some latency numbers every programmer should know.
Reading a value from caches takes half a nanosecond.
So we can read a value from L1 cache two billion times per second.
That's crazy fast.
When we make a reference to main system memory, that's 200 times slower, taking 100 nanosends.
When we read a megabyte from memory, that takes us 250,000 nanoseconds!
But when we read that from disk, it takes four times as long.

Finally, sending stuff over the network is the very slowest thing we can do.
It takes 150 million nanoseconds for a tiny packet to get from California to Netherlands and back.
These numbers are hard to visualize.
Let's  multiply by a billion to put them on a human timescale.


```javascript
L1 cache reference    0.5 s      One heart beat
Branch mispredict     5 s        Yawn
L2 cache reference    7 s        Long yawn
Mutex lock/unlock     25 s       Making a coffee
Main memory           100 s      Brushing your teeth
Compress 1K           50 min     Game of Thrones
2K bytes over LAN     5.5 hr     Lazy workday
SSD random read       1.7 days   A normal weekend
Read 1 MB memory      2.9 days   A long weekend
datacenter round trip 5.8 days   A medium vacation
Read 1 MB SSD        11.6 days   Non-AMZ package delivery
Disk seek            16.5 weeks  A semester in university
Read 1 MB disk        7.8 months how is babby formed
CA->Netherlands->CA   4.8 years  College Degree
```
(source: https://gist.github.com/hellerbarde/2843375)

Note:

Grabbing some data from main memory takes about as long as brushing your teeth.
Waiting for something to come over the network from the same datacenter takes 6 days.
Waiting for something to come from a solid state drive takes 11 days.
Waiting for something to come from across the world is nearly 5 years.

One of the nice things about working in web application development is that our bottlenecks are really easy to spot.
The vast majority of the time we spend is simply waiting on network and datacenter requests to complete.
We spend a tremendous amount of time waiting on the SQL server to finish a request, or the authentication server to complete a request, etc.


# Amd-LOL's law

if you're a web developer, you don't have to care about performance

Note:

This gives us Amdlawl's law.
If you're a web developer, you don't really have to care about performance.
You can do some really dumb stuff on your application code, and since the actual computation stuff is such a small part of the overall time, it doesn't really matter.
If I optimize a routine for calculating some numbers to take 100 nanoseconds instead of 10,000 nanoseconds, no one cares, because we're still waiting 200 milliseconds for the page to load.

Unfortunately, us web developers take this too seriously, and the entire internet is way slower than it needs to be.
The popular web languages, like Ruby, Python, PHP, and Javascript, are incredibly slow.
So how can we optimize performance, as much as possible?


# Batching

Note:

This is kind of interesting.
If you pay attention to trendy programmer crap, everyone's extremely excited about streaming.
Stream! Realtime! Live data! Distributed Actor Systems!
These are systems that are developed to scale to absolutely massive systems -- trillions of requests, millions of users, tons of interacting systems.

Meanwhile, most of us don't have to deal with that.
So we can get away with much simpler solutions.
By batching things up, we can save a lot of time.


```php
# PHP
function saveObjects() {
    $objects = getObjects();

    foreach ($objects as $object) {
        $db->save($object);
    }
}
```

```haskell
-- Haskell
saveObjects = do
    objects <- getObjects

    for_ objects ( \object ->
        save db object
    )
```

Note:

Alright, let's talk about this code.
We get a collection of objects.
For each object, we save it to the database.
Saving stuff to the database is an extremely expensive operation, because for each database request, we need to make one of those "data center round trip" requests.
This takes 500 microseconds, or 6 days in human-scale terms.

We're paying that cost for every single object.
As the number of objects we try to save to the database grows, then the time it takes also grows linearly.


```php
# PHP
function saveObjects() {
    $objects = getObjects();
    $db->saveBulk($objects);
}

```

```haskell
-- Haskell
saveObjects = do
    objects <- getObjects
    saveBulk db objects
```

Note:

Now, we're only making a single request to the database.
This request contains all of the objects that we're trying to save.
We pay the huge network costs a single time.

As the number of objects we needs to save grows, this does still take more time.
The database has to do work for each object we save.
But databases are absurdly fast in comparison to network requests, so this is the right choice to make.


# MySQL

```sql
INSERT INTO `stuff` VALUES
    ('hello friend', 123, 2),
    ('have a nice day', 234, 3)
    ON DUPLICATE KEY UPDATE
        name = VALUES(`name`),
        price = VALUES(`price`)
```

```haskell
-- Haskell
insertManyOnDuplicateKeyUpdate
```

Note:

MySQL supports this sort of batch inserting/updating using the ON DUPLICATE KEY UPDATE syntax.
I've implemented this in the Haskell database library we use called persistent, so we can access these super fast bulk upserts.
The PHP library in Laravel does not support this, so you have to drop to raw SQL and manually convert all of the objects to do that.
This technique achieved something like 400x throughput increase in one of our workers.


# network batching

tired: `/v1/order-items?limit=100`

wired: `/v1/order-items?limit=1000`

Note:

The same applies to network requests.
Bigger network requests are better than more network requests.
The tired request up there only fetches 100 records.
The wired one fetches 1000 -- ten times as many!
The 1000 request will take slightly longer to respond, but much less so than 10 of the smaller requests.
We're saving a lot of time on the actual network overhead.
The web server also saves time making the SQL request, since most of the time spent in SQL queries is overhead (if it's well indexed).


# concurrency

Note:

When you make a web request, you're waiting, not doing anything.
For most of the workers I've touched, we spend the vast majority of our time sitting around and waiting on requests to finish.
This is really wasteful!
We can make better use of time by having multiple requests going concurrently.
That way, instead of waiting on a request, we're doing useful work in between requests, and when the reqeuests come in, we can handle them appropriately.

This technique allowed us to cut server resources by 90% for one of our workers.
