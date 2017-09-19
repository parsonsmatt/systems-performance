# SQL optimizations

Use the index, Luke! <!-- .element: class="fragment" -->

... dot com <!-- .element: class="fragment" -->

http://use-the-index-luke.com/ <!-- .element: class="fragment" -->

Note:

I received a few requests on optimizing SQL queries.
The gist is to use indexes.
They're incredibly powerful.
I'm not going to go into super detail here, but generally, an index will make reading data really fast, and it will make writing data really slow.
If you're writing data to a column a lot, don't put an index on it.

If you're interested in a fantastic resource, check out use-the-index-luke.com for a great book on SQL performance tips.
It doesn't take long to read, and it's really useful.
Well worth the cost.


# avoid limit/offset

```sql
SELECT *
FROM   things
LIMIT  100
OFFSET 100
```

Note:

When you go to implement pagination for a web reqeuest, it is really tempting to use `LIMIT/OFFSET`.
It's so easy for everyone involved.
However, this absolutely wrecks performance.
The database is much less able to take advantage of indexes to make this fast.
THe database must load limit + offset rows into memory, so as the offset grows, you have to load more and more data into memory, which is ultimately thrown away.
This is quite wasteful, and becomes a huge problem when the table gets huge and you're deep in the past.


# ranger danger

```sql
SELECT *
FROM   `things`
WHERE  `things`.`range` BETWEEN :start_range 
                            AND :end_range
LIMIT  1000;

CREATE INDEX things_range ON things(`range`);
```

Note:

The proper answer is to range over some column that has an index.
This gets to use an index, and is lightning fast.
Furthermore, no matter how far back we go in history, we still take the same small amount of time.
Where limit/offset must load all of the rows up until the offset plus limit, this only needs to load the rows that correspond with the range.

This is more complicated to implement.
Texas-Toast uses this technique to make queries and endpoints really fast.


# API side

```javascript
var response = {
    startRange: "2017-01-03",
    endRange:   "2017-01-04",
    count:      1000,
    payload:    [ ... ]
};

var response = getNext(
    { after: response.endRange, limit: 1000 }
):
```

Note:

This is what it looks like on the calling/receiving side. 
We get a response envelope that contains the start/end range parameters.
These parameters can be used to construct the next response.
It's a little more work than just incrementing the offset, but it's much faster and more robust.


# explain pls

```sql
EXPLAIN 
  SELECT *
  FROM   things
  WHERE  foo > 10
    AND  name = 'hello';
```

Note:

If a SQL query is slow, then you can use the EXPLAIN keyword.
MySQL's explain output is kind of awful, but it tells you whether or not it's using an index.
If it's not using an index, it can also help tell you why.
The SQL Performance Tips book goes into this in great detail.

Generally speaking, if you don't see an index being used, then that's the problem.
If there's no index on the table, then the SQL database must scan every single row in the table to identify whether or not it fits the query.
To make this query faster, we can place indexes on the two columns that we're querying: foo and name.
Which one should we use?

We can use logic to get at a first guess, but we must use scientists and measure/try/observe to know for sure.
If name is expected to be unique, or very rarely collide, then an index on name will allow this to work extremely fast.
If name is common, then an index on foo will very quickly allow us to eliminate the non-matching values.
