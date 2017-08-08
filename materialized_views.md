- Feature Name: Incrementally Updated Materialized Views
- Status:
- Start Date:
- Authors: Arjun Narayan
- RFC PR: TBD
- Cockroach Issue: None.

# Summary

Materialized views are a powerful feature for running realtime
analytics queries on a database. This RFC proposes a framework for
executing materialized views using differential dataflow and timely
dataflow, built on top of the existing distributed SQL (DistSQL)
infrastructure.


# Motivation: Denormalizations

Please read [this blog
post](https://hackernoon.com/data-denormalization-is-broken-7b697352f405). This
is the single best explained motivation, and I cannot possibly write a
better motivation.


Consider the following use case: we wish to implement an email
client:

![GMail inbox example](inbox_example.png)

To store the underlying data, we create a table "emails", with the
following schema[^reproducibility]:

    CREATE TABLE emails (
        id SERIAL,
        receiver INT NOT NULL,
        sender INT NOT NULL,
        received_at TIMESTAMP NOT NULL,
        subject INT NOT NULL,
        read BOOL DEFAULT TRUE NOT NULL,
        content TEXT,

This table stores every email received by our service (we use INTs
instead of strings for the receiver, sender, and subject for ease of
generating data). In order to facilitate rapid retrieval of emails to
render inboxes, we make the primary key an index on the receiver and
received_at columns:

    PRIMARY KEY (receiver ASC, received_at DESC));

We can now render the inbox for user "$user" with the query:

    SELECT sender, received_at, subject, read FROM emails WHERE receiver = '$user'::int ORDER BY received_at DESC LIMIT 100;

This query is efficient because we have an index on (receiver,
received_at), so finding emails by those two is rapid: the database
does a binary search for the user '$user', and then does a linear scan
to retrieve the first 100 rows. This is very quick:

TODO(arjun): Insert distsql query plan here.

But what about seeing our sent emails? Generating the "sent" page is
inefficient as we've ordered our data by receiver. If we were to run
this query to generate our "sent emails" page:

    SELECT receiver, received_at, subject, read FROM emails WHERE sender = '$user'::int ORDER BY received_at DESC LIMIT 100;


We would have to do a full scan of our entire table to get all the
emails, since they're sorted by the recipient, not the sender. But
this is where SECONDARY INDEXES are useful: we can create a secondary
index that is ordered for this query:

    CREATE INDEX sent_idx ON emails (sender ASC, received_at DESC);

This creates a copy of the sender and receiver_at fields, with a
pointer to the primary key. This is a good start: to execute this
query, we scan down the secondary index, but the result of that scan
only gives us the primary key fields[^secondaryindices]. In order to produce the "sent
emails" page, we still need the subject of each email, which requires
doing point lookups for every single of the (up to 100) primary keys
we get:

TODO(arjun): Insert distsql query plan here.

To avoid this, we can create a STORING secondary index:

    CREATE INDEX sent_itx_better ON emails (sender ASC, received_at DESC) STORING (subject, read);

This index stores a copy of the the "subject" and "read" fields in the
secondary index, so we can directly look it up. This removes the need
for the point lookups:

TODO(arjun): Insert distsql query plan here.


## Updating Materialized Views: Why?

We also want our email client to display the *number* of unread emails
in the inbox:

    SELECT count(*) FROM emails WHERE receiver = '$user'::int AND read = FALSE;

In particular, note that this number can scan through potentially
millions of emails: it doesn't have a LIMIT statement terminating the
scan after a hundred emails. This is problematic to compute every time.

We could build a second table storing unread counts, and make it the
developer's problem to ensure that they always update the two in
sync. Since we are a transactional database, they would then
permanently be burdened with writing code of the the following sort:

    BEGIN TRANSACTION;
    INSERT INTO emails VALUES<new email info goes here>;
    UPDATE unread_counts SET count = count + 1 WHERE receiver = '$user'::int;
    END TRANSACTION;

But we want to do better than that. Ideally, we would want some kind
of "index" that just stores this count, that would be transactionally
updated with every other operation in the database, and which the
developer could just query as needed, and it would be magically
correct and up-to-date.

What if we created a view?

    CREATE VIEW unread_count(receiver, unread) AS SELECT receiver, count(*) FROM emails WHERE (read = false) GROUP BY receiver;

We could then later run

    'SELECT unread_count FROM emails WHERE receiver = '$user'::int;

This isn't actually that useful computationally speaking: VIEWs are
just translated at runtime into the underlying query, which in this
case would be:

    SELECT unread_count FROM (SELECT receiver, count(*) AS unread_count FROM emails WHERE (read = false) GROUP BY receiver) WHERE receiver = '$user'::int;

However, take a look at the query plan:

TODO(arjun): Insert query plan here.

This is actually worse than our previous solution, because our query
planner is not smart enough to figure out that the WHERE clause can be
pushed up before the GROUP BY, and we don't actually need to GROUP all
the other values.

However, what if our VIEW wasn't a virtual mapping? What if it
resulted in the query results being computed ahead of time? This is
called a "materialized view". We want the user to write the following:

    CREATE MATERIALIZED VIEW unread_count(receiver, unread) AS SELECT receiver, count(*) FROM emails WHERE (read = false) GROUP BY receiver;

This is a feature supported by many databases (the above line runs on
Postgres). The materialized view is persisted by the database, and
updated more often than just at query runtime, reducing latency when
accessed.

## Updating Materialized Views: When?

There are largely two frameworks for updating materialized views:
update on a transactional commit to the underlying tables (optionally
transactionally), and on an explicit refresh command (which optionally
can be scheduled by a job scheduler to run at specified times).

There are many ways to update a materialized view in a controlled
fashion, because updating them for arbitrary SQL queries is
potentially very expensive. The most popular is UPDATE ON REFRESH:

    BEGIN TRANSACTION;
    REFRESH MATERIALIZED VIEW unread_count;
    SELECT unread FROM unread_count WHERE user = '$user'::int;
    END TRANSACTION;

This isn't actually useful to reduce query latency, but is useful if
we were using a materialized view for an analytics workload: every
night at 3am you REFRESH all your materialized views, and then
throughout the day users have access to analytics views that are
transactionally consistent, albeit stale. Another option is to refresh
the materialized view on a set schedule, or dedicate some resources to
always updating the materialized view, restarting the update job when
the previous one finishes, etc. If the materialized view update is
non-blocking (as with, e.g. online schema changes in F1), then read
queries can run on the latest finished materialized view.

Another option, available on Oracle is the eagerly updated
materialized view:

   CREATE MATERIALIZED VIEW (...) REFRESH ON COMMIT;

Materialized view updates are nontrivial: while it is easy for us to
see that maintaining the materialized view for the inbox unread-count
requires just incrementing or decrementing the backing integer when an
email is added or read respectively, most existing materialized view
implementations stop short of anything more complex than such
aggregations.

## Updating Materialized Views: How?

The most common form of materialized view update is the "complete
update". Upon a refresh request, we simply rerun the entire query on
the latest snapshot of the underlying data, and then swap out the old
materialized view for the new one.

Complete updates are easiest to reason about---most SQL engines can
support this form of update for every valid read-only SQL query, and
will typically just use the same qeury execution backend as
normal. This means that materialized views are essentially just a
convenient shorthand for running a query, stashing the results in a
new table, and periodically rerunning this and updating that table
transactionally.

*Incrementally updated* materialized views, on the other hand, involve
updating the table by only rerunning on the *delta* of tuples of the
underlying source data that have changed since the previous time the
materialized view was computed. Incrementally updated materialized
views are obviously more complex to reason about --- essentially you
have to build query pipelines that can take advantage of the existing
result table, and correctly compute the precise delta. This is
complicated by the fact that the delta can involve the *deletion* of
rows in the base tables. Thus, existing databases like Oracle only
really do this for a small subset of queries --- for instance,
supporting incremental updates on sums and counts (where dealing with
deletes is easy) but not for minimums (where the deletion of the
minimum value triggers a complete refresh).

## Updating Materialized Views Today

### Oracle

Oracle has incrementally updated materialized views, as well as fully
updated. The incrementally updated version is called "FAST
REFRESH". Contrast with "COMPLETE REFRESH".


A FAST REFRESH is done by taking all the transactions that mutate the
underlying data since the materialized view was created, and then
applying them incrementally to the view.

FAST REFRESH is only available for a subset of queries[^fastrefresh]:
for examplea, you cannot have JOINs with GROUP BYs, only some
aggregations are allowed, SELECTs cannot use columns from multiple
tables, MIN and MAX aggregations will fail to FAST REFRESH if the
underlying base data processes a DELETE row, ...

COMPLETE REFRESH is the option when a FAST REFRESH is not possible:
the database throws out the materialized view, and recomputes it from
scratch.

https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_6002.htm

### Microsoft SQL Server

Microsoft has "view indexes", no materialized views. More restricted
set of operations supported (basically sums, counts, joins, group
bys). No "on refresh", only on commit. Basically restricted only to
that set of queries that they can do incremental updates that are
transactionally consistent. They use the term VIEW INDEX in order to
drive home the point that they are always consistent.

https://technet.microsoft.com/en-us/library/ms187864(v=sql.105).aspx

### ETL into a data warehouse.

Another option is to ETL your transaction stream out to a data
warehouse. If you accumulate differences and batch them out, you
basically have a COMPLETE REFRESH. Very common to use a Kafka stream
to ETL into HDFS, and then do view computation with a Hadoop
cluster. Alternatively, can also use a dedicated columnar warehouse
like Vertica. If the columnar warehouse is fast enough, and you're
mostly computing scalar aggregates, you don't even really need views
(not true if you're computing a very JOIN/GROUP BY heavy workload,
where MapReduce shines TODO(arjun): make this concrete with examples).

### ETL into a stream processor

Similar to fast refresh, streaming updates to a stream processor
allows for incrementally updated views in real-time. Stream
processors, however, are currently limited in expressiveness. They
also have lower total throughput than batch processors if the data is
batched.

Add more examples.

### ETL into a "hybrid" processor

Batch processors like Spark have "streaming" mode support (Spark
Streaming) where they are able to deal with more incremental chunks of
data with lower latency. Spark claims to be a hybrid processing
engine. Spark is a lot more expressive, but its performance in
streaming situations is pretty awful. TODO(arjun); Cite the numerous
NSDI/SOSP works beating up on Spark for various graph computations
here.

Summingbird is a hybrid engine overlaid on Hadoop + Storm. Batch
workloads are routed through Hadoop, and incremental updates are then
sent through Storm. Again, query expressiveness is limited, due to the
reliance on Storm. TODO(arjun): chase this down, but we may have made
our point. Is it worth doing this much analysis of the competition?


# Updating Materialized Views: A Desideratum

I propose the following: we can implement incrementally updated
materialized views that deliver update latencies on the order of
hundreds of milliseconds, with optional transactional reads on
materialized views that deliver a "read-your-writes"
guarantee. Materialized views will be supported for every possible SQL
query, regardless of query complexity. As a goal, I propose the
following success criterion: a CockroachDB cluster should be able to
maintain an incrementally updated materialized view for every TPC-H
query, while the underlying base data is being mutated by the TPC-C
workload at high throughput.

As a side benefit, much of the framework should be usable as an
alternate execution path to Distributed SQL. For large queries with
query plans that have a large processor depth, it should be faster
than Distributed SQL due to eliminating the straggler problem
entirely.

I propose that this can be done, using the
[Timely Dataflow](http://sigops.org/sosp/sosp13/papers/p439-murray.pdf)
and
[Differential Dataflow](http://cidrdb.org/cidr2013/Papers/CIDR13_Paper111.pdf)
frameworks, built on the existing Distributed SQL[^distsql] (DistSQL)
code in CockroachDB.

# Introduction to Distributed SQL Execution

DistSQL is a package that provides a SQL execution layer that allows
for distributed computation. It follows a straightforward acyclic
dataflow execution model, along the lines of Sawzall, Hive, or
SPARQL.

In CockroachDB, A SQL query is first parsed, and converted into an
Abstract Syntax Tree (AST) which follows a straightforward logical
plan:

TODO(arjun): insert dataflow diagram of an example logical plan.

The logical plan is a directed acyclic graph, where the nodes are SQL
operations such as JOIN, GROUP BY, WHERE, etc. It has a single root
node (the final operator used), and directed edges that flow towards
the root. The leaf nodes are raw scans over one or more tables.

This logical plan can be executed as-is by the "classic SQL" execution
engine. Classic SQL proceeds as follows: the entire dataflow graph
exists on a single node, which takes every leaf node and issues an RPC
to the remote machine that holds the relevant tuples, asking for the
results of the scans (this service that locates the relevant remote
machine for each scan is known as the DistSender). It then proceeds to
execute the dataflow graph in a tuple-at-a-time lazy-evaluation
fashion, starting at the root node. The root node requests a tuple
from its in-edges (depending on its structure, of course, a HashJoin
operator might first request all tuples until exhaustion from its left
in-edge before requesting any tuples from its right in-edge. This is
just an example, CockroachDB does not have a HashJoin operator in
Classic SQL, only nested loop joins).

This execution is network and space inefficient: instead, this logical
plan can be transformed into a distributed "physical" plan. A simple
example is that instead of streaming all rows, a WHERE clause that
filters rows can be "pushed" to the source machines, where only those
rows that pass the filter are bubbled up. Second, many logical nodes
can be split into multiple filter nodes (for instance, in the previous
example of "pushing" a filter down, the underlying table may be split
across three machines, so that single logical filter would get
transformed into three physical filter nodes, one on each of those
machines).

Distributed SQL is the package in CockroachDB that performs this
transformation. It transforms each logical node into one or more
physical "processors". These processors are connected by "streams",
which sometimes cross machine boundaries. DistSQL still performs an
execution using a tuple-at-a-time model, however, processors keep
generating tuples asynchronously, unless they are sent a shutdown
signal. This reduces latency as compared to the lazy-evaluation model
of Classic SQL, especially as processors are potentially spread across
multiple machines.

Distributed SQL processors may take advantage of tuple ordering
information that they may know of. For example, one might naively
design a HashJoin processor to accumulate all left rows, construct a
HashMap, and then stream right rows, emitting output rows on every
match. However, this would have a high latency before the first row is
outputted. Instead, if the input streams are partially sorted (for
instance, if there are two join equality columns, but the input data
sources are sorted on only one of the equality columns), it can take
advantage of that information to only construct a HashMap on the set
of rows that are equal on the partial sort condition. It can do this
because it knows that there cannot be matches on subsequent rows due
to the ordering information.

Nevertheless, DistSQL processors can, in the worst case, have high
latency to "first output tuple emitted". In a complex dataflow graph
with a large depth, this can result in bad execution
performance. Downstream processors are essentially doing no work for a
large period of time until all upstream processors are
finished. Furthermore, a query that is distributed across many
machines can often involve the full N^2 bisection of streams at some
layers:

Consider TPC-H query 3, assuming no secondary indices on TODO(arjun),
such that MergeJoins cannot be planned, requiring HashJoins: The
HashJoiner processors require a full N^2 criss-crossing of streams
since a given tuple from any upstream processor could be hashed to any
of the HashJoiner processors.

One problem with this design is that if there is a single straggler
processor (which is the left input to the downstream HashJoiners) that
is not done emitting rows, all the HashJoiners are forced to wait and
accumulate rows, unable to emit rows. As the number of machines and
streams increases, the probability that one of the machines straggles
behind increases. Thus, this query runtime does not scale linearly
with the number of machines added, beyond the first few machines. At
about a 100 machines, *most machines will be waiting idly most of the
time*. This is the MapReduce straggler problem[^mapreduce-stragglers].

In order to do better, we must change the API between
processors. DistSQL processors must relax their requirements on input
streams, and be a little more flexible. One such flexible API, with
empirical support for its efficiency, is Timely Dataflow.

# Improving Execution with Timely Dataflow

Timely Dataflow changes the API between DistSQL processors. Instead of
just requiring that streams be ordered on some condition, and giving
up if they are unordered, it asks for some relaxations. Consider a
stream that has a total ordering of tuples (e.g. a sort condition on
tuples):

* Each tuple in this stream is passed along with a timestamp.

* Timestamps can optionally be "closed".

* When a processor outputs that timestamp *t* is closed, it guarantees
  that it will never ever send another timestamp with *t'* where *t'
  <= t*. (Condition 1)

* A processor that receives a timestamp *t* may only output timestamps
  *t'* where *t' >= t*. This guarantees that tuples do not go
  "backwards in time". (Condition 2)

* Close *t_2* where *t_2 >= t_1* implies close *t_1*.

* A processor should eventually close every timestamp it sends.

While it may not be obvious, this API relaxation is sufficient to
allow for processor designs that can make progress quickly, and do not
suffer from large amounts of buffering before making progress. They
may be more flexible in how they send their outputs, eagerly sending
tuples that violate ordering constraints, holding back only on close
notifications. Downstream processors can do preprocessing work, such
that upon receiving close notifications they only have sublinear
amounts of work to do, reducing end-to-end query execution latency,
even if the total amount of work done summed over all processors ends
up being greater.

The key engineering challenge in *using* timely dataflow is for
processors to do two things:

* carefully reasoning about when they can close a timestamp for their
  downstream processors and doing so as early as possible.

* making partial progress with input tuples from their upstream
  processors that are at unclosed timestamps such that when the close
  notification arrives, there is a sublinear amount of work that
  remains to be done.

## Timelystamps

So far, the conversation has kept the timestamps opaque. Everything
above is compatible with maintaining a single incrementing timestamp
integer. Instead, we are going to broaden them, such that timestamps
are tuples of arbitrary length. So timestamp *t* could be <1,2,6,3>.
We are also going to relax the inequalities in Conditions 1 and 2 to
be *partial orders*. Thus, two timestamps *t_1* and *t_2* may be
incomparable.

The partial order is defined as follows:

*t_1 = <a_1, a_2, ... a_m> <= t_2 = <b_1, b_2, ...,b_n>*

if and only if all following conditions hold:

* m <= n *

for all *0 <= i <= m, a_i <= b_i*

If *t_1 !<= t_2* and *t_2 !<= t_1*, then they are incomparable
timestamps.

Do note that Condition 1 promises that once a processor closes
timestamp *t_1*, it will never send a tuple with a timestamp *t_2 <
t_1*. It is still legal to send timestamp *t_3* that is incomparable
with *t_1*! We are, in fact, going to make liberal use of this
"loophole" when designing processors, for greater concurrency.

Finally, some special processors are going to modify timestamps by
adding or removing a column (always on the rightmost side). So a
processor may get as input timestamps of length 3, but output
timestamps of length 4, or length 2. It is, still, however, bound by
condition 1 and condition 2. In particular, this ability combined with
condition 2, allows us to make *cyclic* dataflow graphs, which can
make some queries much more efficiently executable.

It is important that we remember that these timestamps are not
totally-ordered integers, but a partially ordered semi-lattice of
vectors of integers. They are also distinct from the CockroachDB
concept of timstamps used in reasoning about consistency of database
transactions (they are, however, going to eventually incorporate
*those* timestamps as one of the vector dimesions). Thus, I am
henceforth going to call these timestamps "Timelystamps".

## Processor Design: General Principles

To reiterate, processors have two key goals:

* First: close timelystamps as fast as possible to its downstream
  consumers.

* Do as much preprocessing work as possible with tuples at unclosed
  timelystamps such that when those timelystamps are closed, the
  processor has a minimal amount of work left to do.

Processors will generally be structured as follows:

* If tuples are arriving such that the processor is "unblocked" on
  work (for instance, if it is receiving tuples in its incoming
  message queue at new timelystamps that are closed _and_ that queue
  is nonempty), it should just do the regular DistSQL work: emitting
  rows eagerly doing just a constant amount of work per tuple.

* If the regular DistSQL work cannot progress because all outstanding
  tuples have timelystamps that are unclosed, it should begin
  preprocessing those tuples, optimizing to minimize the amount of
  work left when the eventual close notifications arrive.

* Similarly, a processor will be designed to close outgoing
  timelystamps as fast as possible, to enable downstream processors to
  do work.

* Since close notifications apply to all smaller timelystamps, you
  should plan for the closing of the timelystamp of the _join_ of all
  its unclosed timelystamps. E.g. if you have unclosed timelystamps
  (2,3) and (3,2), the processor should do preprocessing work to make
  efficient the eventual close notification at (3,3). This, of course,
  gets complicated when there are lots of unclosed uncomparable
  timelystamps.

* A processor that simply waits until timelystamps are closed, and
  processes tuples in order recovers regular DistSQL behavior (which
  we are used to operating on a totally ordered stream of input
  tuples): it simply forgoes all opportunity to make partial progress.

* This realization also gives us a way to recover
  backwards-compatibility for full expressiveness: we have an "egress"
  processor that simply buffers until closure, and forwards closed
  tuples to a regular DistSQL processor.

Design principles for processors:

* The work on unclosed timestamps is as follows: processors are
  generally going to maintain state "at" many different timelystamps,
  specifically, at a set of timelystamps that form an antichain of
  unclosed timelystamps: An "antichain" is a set of timelystamps such
  that all the timelystamps in the antichain are incomparable with
  each other. You should think of this as the "frontier" of
  timelystamps that the processor can do no further work
  with[^frontier] until close notifications arrive. The goal is to
  compact the antichain down to a minimal efficient frontier.

* If we receive a tuple at timelystamp *t_1* that is comparable with
  an existing timelystamp *t_2* in our antichain, we will combine the
  information and only store the greater of the two. Do note that a
  timelystamp may combine *multiple* existing timelystamps in our
  antichain, for instance, if we have <1,2> and <2,1> in our
  antichain, receiving a timelystamp <2,2> will combine with both of
  the existing ones as it is greater than *both* of them.

* Once a timelystamp is closed, we will typically emit a corresponding
  timelystamp to our output, and then garbage collect the information
  at (or prior to) that timelystamp. Thus, the antichain is compacted
  by incoming close notifications.

## Working through example Timely processors

### Ingress and Egress

Ingress and Egress are two special Timely processors: they append and
strip the rightmost dimension of a timelystamp. The complication comes
in ingress operators having to assign timelystamps from multiple input
streams, with potentially different semantics.

Egress is simple: it simply strips the rightmost dimension, allowing
streams with different dimensionality to eventually merge. It
propagates close notifications only when it is sure that the
timelystamp

### Feedback



### HashJoiner



### Sorter


### GroupBy


# Pushing updates from the transactional layer

So far, we have discussed processors that find efficiencies in high
latency/deep dataflow settings. This creates (potentially) a more
efficient dataflow execution model for query execution. For
materialized views, however, we need to stream *change notifications*
from ranges as well.

Eagerly streaming change notifications to materialized views is not
sufficient on its own: this means that a materialized view updates
potentially out of order, and thus no longer provides any meaningful
consistency guarantee. It could also simply be _wrong_, if the
materialized view update function is not commutative over
transactions. Thus, we need a way to _order_ updates from the
transactional layer along with sending the updates. The complication
lies in the fact that an update can come from a wide variety of
ranges, which can be on any node, and those must be ordered.

Since all transactions contain a transaction timestamp, this will
serve as the root of the timelystamp ordering: a write on a key will
propagate along with the timelystamp of <nodeID, transaction
timestamp>. However, the recepient of this tuple has to treat this as
an unclosed tuple. The complexity now lies in closing tuples as fast
as possible --- which we can only do when all ranges are sure that no
writes will ever occur for that key at a lower timestamp.

CockroachDB nodes currently maintain a *timestamp cache*, which
contains a list of timestamps at which reads have taken place for each
key (TODO(toby/spencer): per key? per range?). This essentially
provides a "low watermark" per key, since writes cannot take place at
a timestamp earlier than a read. Currently, CockroachDB nodes do not
move this watermark eagerly: we allow transactions to proceed
arbitrarily far back in time (TODO(toby/spencer): is this true),
supporting very long running write queries. However, for materialized
views, we want to eagerly move the timestamp cache watermark upwards,
trailing behind at about 10 seconds or so. This effectively poses a
cap on the duration a write-transaction can take, but
ensures that materialized views can be sure that no additional
information will be coming.

Our solution is thus this: move the timestamp cache watermark upwards
(initially .e.g, once per second, at a 10 second lag) emitting _close_
notifications for <nodeID, transaction timestamp> as we
go[^watermark]. A processor might have to wait for all nodes to close
a given transaction timestamp (depending on the ordering requirement
that it is trying to preserve on the stream) before closing its
timelystamp(TODO(arjun): I'm wondering if we even care about orderings
because differential dataflow is designed to work without any
orderings. It remains to be seen if stream orderings even buy us
anything of any worth with the fully realized differential dataflow
processors...).

A final complication is that because of the structure of CockroachDB
transactions, a transaction can be emitted from _any_ range: due to
the nature of proposer-evaluated KV transactions, the actual
transaction might finally be committed at an unrelated range (and MVCC
intents are cleaned up on the local range on a lazy basis, so the
intent cleanup cannot be used as the trigger for sending eager
updates). Thus, materialized view processors need to listen to streams
from every CockroachDB node that has a range for the given database
(TODO(spencer/ben): can this ever be fixed? It displeases me
aesthetically, but I think its good for v1).

<work through an example here>

# Extension to Differential Dataflow

An astute reader might have caught that materialized views might
require us to handle the *removal* of an input tuple from the computed
view. This requires processors to store all intermediate state: consider the case of



# Zone configurations for OLTP/OLAP separation

One production usability concern is that materialized views (and
DistSQL analytics queries in general) can be computationally intensive
as well as storage intensive. This causes a lot of worry for database
administrators when run simultaneously with production OLTP
workloads. Thus, we need to support placing materialized view
processors on separate machines, obeying a zone configuration. Thus, a
CockroachDB cluster could be configurable with "OLTP" nodes (which
process queries and store the underlying data) and "OLAP" nodes (which
host the materialized view processors). Thus, the overhead (with
respect to the OLTP nodes) of supporting materialized views is limited
to performing the computation to stream out change notifications and
send timestamp watermark heartbeats to the OLAP nodes.

I think this would be a well-defined enterprise feature. CockroachDB
core will not be handicapped, and users can benefit from the full
feature-set of incrementally updated materialized views. But having
resource isolation for production critical data would require an
enterprise license.

#

# Footnotes

[^reproducibility]: In order to facilitate reproducibility of the SQL
execution diagrams used throughout this document, here are the
commands I ran:

I started a 5-node Cockroach cluster running
cockroachdb/cockroach#1d26483:

    ./cockroach start --insecure --background
    ./cockroach start --insecure --background --store=node2 --host=localhost --port=26258 --http-port=8081 --join=localhost:26257
    ./cockroach start --insecure --background --store=node3 --host=localhost --port=26259 --http-port=8082 --join=localhost:26257
    ./cockroach start --insecure --background --store=node4 --host=localhost --port=26260 --http-port=8083 --join=localhost:26257
    ./cockroach start --insecure --background --store=node5 --host=localhost --port=26261 --http-port=8084 --join=localhost:26257

I created the table and secondary index as described in the text
above. Then I ran the following statements to insert and split some
data across the 5 nodes:

    INSERT INTO emails (receiver, sender, received_at, subject) SELECT a,b,c,d FROM
    GENERATE_SERIES(1,100) AS A(a),
    GENERATE_SERIES(101,200) AS B(b),
    CURRENT_TIMESTAMP() + GENERATE_SERIES(60000000, 1, - 600000)::INTERVAL) AS C(c);
    GENERATE_SERIES(1001,1100) AS D(d);

TODO(arjun): fix the above syntax, I can't make it work.

[^secondaryindices]: Link to explanation of how secondary indices work at Cockroach.

[^fastrefresh]: http://docs.oracle.com/cd/B19306_01/server.102/b14223/basicmv.htm#i1007007

[^distsql]: Distributed SQL is best explained here. todo(Arjun): link
    to Distributed SQL blog post. The RFC at this point is not the
    best introduction.

[^mapreduce-stragglers]: Stragglers arise in MapReduce tasks for two
    general reasons: data partioning skew, or hardware. First, the
    partitioning scheme used to spread the load across multiple
    machines might have a single machine processing a disproportionate
    number of rows. This becomes more common as the concurrency level
    is increased, and most machines are given a very small amount of
    work to do. Second, the cluster scheduler might have scheduled
    other tasks on the machines, and one of the machines experiences
    resource starvation as a result (for more examples, see the
    MapReduce paper section 3.6).

[^frontier]: To be more specific, processors will attempt to construct
    an efficient frontier of Timelystamps which reduces the work
    required when close notifications arrive. This might or might not
    be just the greedily constructed antichain, or might involve more
    calculation to "compact" the frontier. For more on this topic,
    see
    [this blog post](https://github.com/frankmcsherry/blog/blob/master/posts/2017-03-22.md) by
    Frank McSherry.

[^watermark]: One additional thought is that we could dynamically vary
the watermark levels with an exponential backoff/linear creep as in
TCP window sizes: if a long running transaction has to abort because
the timestamp cache watermark advanced too far, we double the
watermark delay duration, and restart the transaction.