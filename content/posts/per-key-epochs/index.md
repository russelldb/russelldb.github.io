+++
title = "Per-Key Actor Epochs In Riak"
date = "2015-12-31"
+++


# Per-Key Actor Epochs In Riak

[Last time](../dvv)
I wrote about how Dotted Version Vectors fix Sibling
Explosion. Sibling Explosion was relatively common bug, often seen in
the wild. This post is about a subtle bug that manifests in a number
of different ways, but all have the same result: silent data
loss. Don't panic, we believe this to be a very uncommon edge case,
and it is now fixed in Riak >= 2.1.

In order for the bug to occur we need a complex interplay of features,
so this post will have to cover Deletes, Read Repair and Handoff in
Riak.

The bug itself is very similar to the requirement to Read Your Own
Writes for Client Version Vectors, so if you haven't read
[part one](../vnode-vclocks)
now is a good time to do so.

## Why RYOW, again?

If you recall, an actor in a version vector must increment it's
counter each time it updates a value. If an actor can not read it's
own latest update in the version vector, it cannot issue a new event
that is certain to be larger than the last. The first post in this
series showed how this can lead to data loss. But how can it be that
some vnode (as actor) could fail to read it's own last write, isn't a
vnode where the database stores data? The simplest answer is: deletes.

## Deletes are hard

Before moving on we are going to have to have a least a passing
understanding of how deletes in Riak work. Deletes are hard in an
eventually consistent distributed system. Riak allows concurrent
writes to the same value, and it also allows a value to be
concurrently deleted and updated. Which operation "wins"? As with
concurrent updates to a key, version vectors arbitrate. You can read
more about
[deletes in the Riak documentation](http://docs.basho.com/riak/latest/ops/advanced/deletion),
but this post will cover what is needed to understand the bug.

### Tombstones

Rather than physically deleting a key, we can logically delete it. We
read the key and get it's version vector, and write a special
tombstone value back to Riak with the version vector as a context. A
delete then behaves like any other write. If the key was concurrently
updated, the version vector detects this, and the tombstone and
updated value become siblings. If there was no concurrent update only
the tombstone is recorded on disk. A request to read a key that only
has a tombstone value will return **`not_found`** as Riak understands
tombstones to mean logical deletion. A
[client](http://docs.basho.com/riak/latest/ops/advanced/deletion/#Client-Library-Examples)
can also ask Riak for the "deleted vclock" on a get, to ensure that a
new write supersedes a tombstone.

### Reaping: I want my disk space back!

Disks are cheap and disks are big, but they're neither free nor
infinitely large, and customers would like to see the deletion of
values reflected in increased disk space (I know! Crazy!) Even if a
tombstone is just a version vector and a small special value actual
removal of the key data is a requirement. The process by which Riak
reclaims space is called tombstone reaping. It's reasonably complex:

1. Client issues **`delete`** command
2. Riak writes special **`tombstone`** with Version Vector
3. Riak internally performs a **`GET`** on the key, with the [quorum](http://docs.basho.com/riak/latest/theory/concepts/glossary/#Quorum) value set to **`all`** (which means "ask all the replicas for the value, please.")
4. If the result of the get is a tombstone AND all vnodes that reply are [primaries](http://docs.basho.com/riak/latest/theory/concepts/glossary/#Sloppy-Quorum) the
    vnodes are told they may physically remove the key
5. The vnode will read the key, check it is a tombstone, and check the **`delete_mode`**
   and finally delete the tombstone.

Step 4 above amounts to Riak declaring unanimously that all primary
replicas agree on a tombstone value before issuing the final delete
that reaps the tombstone.

What is **`delete_mode`** as mentioned in step 5? It is a setting which can be one of

* keep - never delete the tombstone
* immediate - remove the tombstone at once
* integer - remove the tombstone after some time, eg 3 seconds

Read
[the documentation](http://docs.basho.com/riak/latest/ops/advanced/deletion/#Configuring-Object-Deletion)
for more details. I'll cover these settings as they pertain to the bug
later.

Clearly deletes are hard. The summary though is that Riak needs to see
all primary replicas agreeing on a logical delete before a physical
delete is considered.

## Read Repair

As mentioned in the introduction the process of
[Read Repair](http://docs.basho.com/riak/latest/theory/concepts/glossary/#Read-Repair)
has a part to play.

[Read repair](http://docs.basho.com/riak/latest/theory/concepts/Replication/#Read-Repair)
is an opportunistic anti-entropy mechanism. When a client reads data
from Riak, each replica responds with it's local copy. Using the
Version Vector Riak can detect when some replica is behind, or in
conflict, and send the most up-to-date value to that replica. The
replica then stores the correct value. It is a curious fact that _all_
the replying vnodes may need read repair as the "most up-to-date
value" could be the result of _merging_ all the responses. Imagine,
for example, 3 clients each writing a different value, concurrently,
to 3 vnodes.

## Hand Off

[Hand Off](http://docs.basho.com/riak/latest/theory/concepts/glossary/#Hinted-Handoff)
is a process that restores data after some failure or
[network partition](https://queue.acm.org/detail.cfm?id=2655736).

If the node **`A`** that some key **`X`** should be stored on is unavailable
when a client wishes to write **`X`** some other node **`A'`** will step in
and handle that write as a _fallback_. When the node **`A`** is available
again then the fallback node will _hand off_ any data it stored. All
this means is node **`A'`** sends any data that it stored for node **`A`
back to node **`A`**. Of course the magic of Version Vectors is how node
`A`** knows if it should add node **`A'`s data as a sibling, discard it,
or overwrite it's local data.

## Doomstones

Finally we have all the knowledge we need to understand the issue.

The example I'm going to use has deletes and read repair and fallbacks
in. It's a reasonably complex example, but one that has been seen in
the wild.

A client decides to delete key **`X`**. It reads key **`X`** and gets the Version Vector

```Erlang
    [{A, 2}]
```

Which means that actor **`A`** has issued two updates to **`X`**. The Client
sends the delete command and version vector to Riak. Riak creates a
`tombstone`** value and writes it. However, only primary nodes **`A`** and
`B`** are available, node **`C`** appears offline, maybe some congestion at
a network switch, or some other problem. Maybe an operator took **`C`**
offline to replace a faulty NIC. Whatever, node **`C'`** handles the write
of the tombstone as a _fallback_. The client is notified the delete
succeed, and its part is done.

Our cluster is in this state: the **`tombstone`** is on **`A`**  **`B`**  and
**`C'`**. As stated above, Riak now performs a **`GET`** operation to see if
all primaries unanimously agree on the tombstone value. By now **`C`** is
back online and returns **`not_found`**. It's OK, _Read Repair_ kicks in,
and sends the tombstone to **`C`**.

Now our cluster has the tombstone on **`A`**  **`B`**  **`C`** and **`C'`**. A read to
key **`X`** occurs. The client receives a **`not_found`**  and since all nodes
have the tombstone, and all are primaries, the _reap_ logic is run.

Nodes **`A`**  **`B`**  and **`C`** remove the tombstone. A client writes a new
value for **`X`**. Node **`A`** coordinates. The new value ends up with the
Version Vector.

```Erlang
    [{A, 1}]
```

This would be fine, except lingering on that fallback node **`C'`** is a
tombstone with the version vector

```Erlang
    [{A, 2}]
```

Handoff kicks in, **`C'`** sends its value to **`C`**. Node **`C`** looks at its
local Version Vector for key **`X`**  sees that the incoming hand-off
value dominates it, and writes the tombstone.

Later, a read will cause read repair to spread the tombstone to nodes
**`A`** and **`B`**. The tombstone _looks_ like it is from a later causal
time, but it's actually from an earlier time. The tombstone has
managed to silently delete the new value of key **`X`**  which is bad.

This example is convoluted but the essence of the problem is much like
the RYOW problem from part 1. If a vnode forgets the version vector
for a key (say by deleting the key) then it re-issues some event, in
this case **`{A, 1}`**.

In this case the answer could be as simple as use **`delete_mode =
keep`**. But that's not the whole answer.

## Fault Tolerant?

There are other ways for this issue to manifest. In the real world
disk errors occur. If your database is a replicated, fault tolerant
database, and it gets an error from a disk, should it fail a write
operation? When a Riak vnode can't read a local value for a key, it
treats the value as **`not_found`**  after all, there are **`n`** replicas of
the data, a write shouldn't fail if one is lost.

Turns out this can be bad, too. Failing to read the local version
vector for _whatever_ reason leads to a new version vector being
created for the key. Imagine some key **`X`** is on replicas **`A`**  **`B`**  and
`C`** with version vector

```Erlang
    [{A, 3}, {B, 2}, {C, 5}]
```

For _whatever_ reason, a coordinating write on node **`A`** fails to read
**`X`**. Solar flare, disk error, operator removes the data directory 'cos
"3 replicas, it's OK!", _anything_. At this point we assign the
Version Vector

```Erlang
    [{A, 1}]
```

To the new write. When it reaches replicas **`B`** and **`C`** they will
discard the new value as "already seen". Next Read Repair will cause
**`A`** to remove the value too. Same outcome (silent data loss), and from
a similar case: a vnode failing to read its own writes.

## Epochs is the answer

As with so many things, the answer is conceptually very simple, but
the engineering a little harder.

In each of the cases above, and other more complex manifestations of
the issue, the problem arises when a local **`not_found`** leads to the
creation of a new Version Vector when an older one from a _later
logical time_ still exists in the system.

One possible answer is to have an _Epoch_ for each key. When key **`X`**
is created the _first_ time call that "Epoch 1". Then, if later it is
deleted, and re-created the new **`X`** will be in "Epoch 2." If some
replica has key **`X`** in Epoch 1 and we try and compare it with Epoch 2,
we can ensure that Epoch 1's Version Vector at **`[{A, 3}]`** does not
dominate Epoch 2's Version Vector at **`[{A, 1}]`**

In practical terms this means that whenever some vnode gets a local
**`not_found`** when coordinating a write, it creates a new epoch for that
key. Why might a vnode get a local **`not_found`**? As above, maybe a
delete, maybe disk error, maybe operator error. Maybe this key has
_never_ been written before. It doesn't matter. A local **`not_found`**
means a new Epoch for the key.

### Does Epoch Two Dominate Epoch One?

NO! We've already seen an example above where Epoch 1 and 2 contain
siblings. So what do we do then? How do we merge a clock like

```Erlang
    [{A, 2}, {B, 4}]
```

From Epoch 1 with one like

```Erlang
    [{A, 1}]
```

From Epoch 2?

### Per Key Actor Epochs

Instead of having an epoch per key, we have an actor epoch per
key. Each vnode keeps a counter. Every time the vnode gets a local
**`not_found`** when coordinating a write it increments it's counter and
_for that key only_ creates an actor ID from the pair of values **`vnode
id`** and **`counter`**. This means that every time a key is (re)created it
gets a unique actor ID. This turns the clocks above into the pair:

```Erlang
    [{A:1, 2}, {B, 4}]
    [{A:2, 1}]
```

Meaning we _can_ merge the clocks into a single version vector, and we
can treat as concurrent the values assigned to each clock. No more
silent data loss. We consider the pair **`A:N`** as a single actor ID, thus
each vnode gets a per-epoch-id for each key without having a general
explosion in the number of actors in the system. This scheme isolates
the growth of actors to individual keys.

In the tombstone example above, the new Version Vector for the
re-created 'X' key would be **`[{A:2, 1}]`** and would therefore conflict
with the "doomstone" version vector. This ensures the new value
survives. Is this _strictly_ correct? No! Ideally in the "doomstone"
case the new epoch would dominate, but in the local fail to read case,
the new epoch would be concurrent. But in either case no data is lost
with this scheme.

This scheme has the benefit of being entirely backwards compatible and
requiring no logical changes elsewhere in Riak: all Version Vector
logic remains the same.

### Updating A Version Vector

How does the vnode **`A`** update it's entry in the Version Vector now? It
finds the entry **`A:N`** for the highest **`N`** in the Version Vector and
updates that.

#### Mechanics

It turns out keeping a simple, durable, strictly increasing counter is
not so simple. In order to work no pair **`vnode id:counter`** can ever be
used twice for a new key. Which means the counter must be
durable. Durable data is expensive, as this
[talk](https://www.usenix.org/node/186195) illustrates. In Riak each
vnode uses a leased counter approach. At a configurable interval each
vnode will asynchronously flush a lease (say 10k) to disk, and will
continue to increment it's counter up to that ceiling. As the ceiling
approaches, the vnode will store a new value (+10k) to disk, and
continue to issue new writes. If the vnode should be restarted or
crash between flushes it reads the ceiling value as the starting point
and creates a new lease from there. This ensures that a vnode never
res-uses a counter value. The size of the lease, and therefore
frequency of flushing/leasing is a configurable parameter.

## Summary

In this series of blog posts we've gone from 1970s seminal works to
modern academic/industry collaboration. We've seen how one of the
simplest and most venerable data structures in computer science can be
complex and error prone in the real world, and how even the classics
can be improved with some necessary innovation for time to time.

If you've read the whole series, thank you! You're fully caught up
with logical clocks in Riak from back in client-side vclock days to
the present world of per-key-actor-epoch-vnode-dotted-version-vectors!
