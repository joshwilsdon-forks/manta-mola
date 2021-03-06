---
title: Mola GC Design Alternatives
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Intro

This document is meant to give a whirlwind tour of the options we considered for
manta garbage collection.  This assumes that you are familiar with Manta
architecture.

# Garbage Collection

A manta object is considered "live" when there is at least one key for the
object in manta.  These pointers are stored in moray shards and point to objects
on mako nodes.  A key exists on a moray shard based on the key's directory.
Since links exist in manta, it is possible (and likely) that pointers to an
object will exist on many moray shards.  When no links to an object exist on any
moray shards, the object is considered "dead" and can be garbage collected.  An
object dies when:

1. All keys are deleted (via the DELETE API).
2. All keys have been overwritten by new PUT requests (PUT object or PUT link)

The problem of garbage collection is determining which objects on mako nodes can
safely be removed to make room for new customer objects.  Obviously deleting
customer data when the data shouldn't be deleted is a Very Bad Thing (tm).  The
next section will explain our most problematic data and the sections following
will give an overview of the alternatives we considered.

# The Walking Link Problem

The most problematic data we face when performing garbage collection is a
"walking link".  A walking link is the term we coined for when a new link is
created for an object in a directory other than pwd and the old link is deleted.
Doing this as quickly as possible will result in references to the object
"walking" between moray shards.  This type of data is problematic because if one
were to look at moray dumps (which are all taken at different times) it is
possible that a reference to an object wouldn't exist in any dumps, but still be
live.  Since it isn't possible to take all moray dumps at exactly the same time,
this problem must be mitigated.

# GC Alternatives

## Synchronous Deletes

The first design we considered were synchronous deletes.  In order to make this
work, we would need the equivalent of an object reference count.  Due to the
distributed nature of moray shards and the links implementation we would either
need to keep a reference counter in a well known location or be able to take a
lock on all moray shards.  One way to implement a reference count would be to
rely on a second moray shard to keep the count, then implement two-phase commit
when performing write requests to Manta (PUT, DELETE, LINK).  The way to look at
"current" state is to synchronously ask each Moray shard if the shard has a
reference to the object on a delete or put, while locking the moray shard
against writes about that object.

Each of the above designs was rejected because taking a dependency on more than
one moray shard for every object write means increased latency, increased
complexity and decreased fault tolerance.  Think pain.  Lots of pain.

All the remaining designs are asynchronous designs.  Each asynchronous design:

1. Relies on periodic dumps of all Moray shards.  Moray is responsible for the
  key to object mappings and must be consulted for live objects in the system.
  The two ways of consulting it is through synchronous calls or asynchronous
  dumps.  We decided on the latter to keep load off of moray.  GC isn't critical
  and shouldn't be able to interfere with the customer-critical request path.
2. Runs as a Marlin job.  Otherwise we'd be hand-rolling a distributed system for
  crunching the moray dumps.  No reason to roll our own when Marlin exists.

## Manta Last Touch Time/List of Links

In this design any time a "write" operation is performed on an object Muskie
would "touch" the object in Mako.  Mako nodes would periodically publish
directory listings which would include the last update timestamp.  When the
timestamp for last touch no longer exists in dumps that were taken after the
last touch timestamp, the object is dead and can be GCed.  A variation on this
design was to keep the list of keys for the object with the object in fsattrs.
When all the keys were either deleted or pointing to other objects the object
could be GCed.

This design was rejected because it would require "touching" all mako nodes that
hold the object on a write.  This isn't acceptable from an availability
standpoint- to reject a request because one node in an exact subset of mako
nodes is begging for 500s.  Also, the first of the two designs didn't mitigate
the walking link problem.

## Muskie/Mako Input for Deletion Candidates

Each of the remaining designs attempt, in some way or another, to keep a record
of which objects could be dead and trying to reliably determine if the object is
dead.

The first thought was to process the logs from Muskie for the list to delete,
then check the moray dumps for references to that object.  Once the object
hasn't appeared in any dumps for n consecutive periods the object can be
collected.  The problems with this solution are two fold:

1. Muskie logs are unreliable.  Each muskie box isn't fault tolerant.  When a
  muskie host dies we would end up missing deletion candidates.  This isn't
  terrible, since we would only be leaving around objects on mako nodes (as
  opposed to deleting objects that shouldn't be).  What's worse...
2. This relies on probabilities rather than certainties.  At large scale even
  small probabilities happen every so often.  This would result in incorrect
  garbage collection (the Really Bad Thing (tm)).

A slight variation on the above is dumping directory listings from mako nodes to
be used as input to objects for GC.  In other words, all objects are candidates
for deletion until proven that they are live.  While this mitigates the
unreliability it is still subject to the problem of relying on probability.

## Moray Deletion Log

The final proposal is based on the realization that in our system, moray is the
highly consistent store.  We can rely on the consistency for assurance on when
to garbage collect an object.  The two variations on this design were: # Use
mako dumps as the source candidates for deletion # Use only moray data to
determine which objects can be deleted

Using mako is undesirable if there is a possible solution with moray only.
Since there is a solution using moray only, the first solution was discarded.

# Using Moray Deletion Log (aka "The Proposal")

This section explains the design for GC, using moray logs as the only source of
input.  Each time an object/link is overwritten or a key for the object is
deleted moray will record a deletion log.  The deletion log is another table in
moray (next to the already-existing "manta" table) that is written into using
either moray multi-PUT or moray triggers.  The deletion log contains the time
that each deletion was inserted.  The union of all deletion logs across moray
shards contains the candidate objects for GC.  Once we have a set of moray dumps
representing all moray shards we will look at each record in turn:

1. If the record time is greater than the least moray dump time we can't be sure
  if a record of the object is the latest and so it is ignored.
2. If there is a record of the object being "live" in a moray shard after the
  time of the deletion log, the deletion record can be deleted.
3. If there is a later deletion record in a moray shard, the record can be
  deleted.
4. If the time of deletion record came before the times of all dumps and it is
  the only record of the object in all shards, the object can be garbage
  collected.

Since we are guaranteed that the record of an object will either exist in the
live table or in the deletion logs we can be certain that if we find a record
where:

1. The deletion log happened earlier in time than when all dumps were taken
2. The deletion record is the only record of the object

We can be confident that it's a dead object.  Using this scheme we can also
define a grace period before we delete the object from the mako node.  The grace
period is meant to be useful when a customer calls up and needs to recover
accidentally deleted data.

We avoid the walking link problem since we would continually find deletion
records with greater record times.

To illustrate how this works, the following table shows the data on two moray
shard over a timeline.  The goal is to show that if the logic above is applied,
the object is only deleted when the record has passed the pre-designated grace
period.

<!-- Yuck -->
<table>
  <tr><td>Time </td><td>Action</td><td>Shard A</td><td>Shard B</td><td>Comments</td></tr>
  <tr><td>t0</td><td>(empty)</td><td></td><td></td><td></td></tr>
  <tr><td>t1</td><td>create&nbsp;/k</td><td><code>/k&nbsp;=>&nbsp;1234</code></td><td></td><td>/k points to object with id 1234.</td></tr>
  <tr><td>t2</td><td>create&nbsp;link</td><td><code>/k&nbsp;=>&nbsp;1234</code></td><td><code>/l&nbsp;=>&nbsp;1234</code></td><td>/l also points to object with id 1234.</td></tr>
  <tr><td>t3</td><td>/k&nbsp;deleted</td><td><code>1234,t3</code></td><td><code>/l&nbsp;=>&nbsp;1234</code></td><td>Shard A contains a record that 1234 became a deletion candidate at time t3.</td></tr>
  <tr><td>t4</td><td>/l&nbsp;deleted</td><td><code>1234,t3</code></td><td><code>1234,t4</code></td><td></td></tr>
  <tr><td>t5</td><td>GC</td><td></td><td><code>1234,t4</code></td><td>Shard A's record is cleaned up because there is a later record of object 1234.</td></tr>
  <tr><td>t6</td><td>GC</td><td></td><td><code>1234,t4</code></td><td>Even though the record of 1234 is the latest and only record of 1234, the time hasn't passed the grace period for deletion.</td></tr>
  <tr><td>t7</td><td>GC</td><td></td><td></td><td>Both the deletion record on shard B and the object on a mako node are removed.</td></tr>
</table>

Note that in the actual implementation dumps are taken at different times.  Any
dumps taken at any times should result in consistent behavior.

# Components to implement

A number of pieces need to come together to implement garbage collection:

1. Deletion records should be recorded in moray.  This will be implemented using
  moray triggers.
2. Marlin job that consumes moray dumps and produces:
- List of deletion records that can be removed for all moray shards
- List of objects that can be removed for all mako nodes
3. Agent that consumes the moray shard cleanup files and applies the deletes to
  moray (may run as part of the Marlin job)
4. Agent running on all mako nodes that will pull down the list of objects to
    purge and then purges.

# Edge cases to consider

## What if one Moray shard is cleaned up and anther is not before the next GC?

This wouldn't need to be considered if two shards had references to the same
object id.  Given that there are references to the same object on two shards,
the object could either be live or dead.  In the case that the object is still
live, the moray record on the shard that wasn't cleaned up will get cleaned up
during some future GC.  In the case where the object is the latest deletion
record it will get cleaned up during some future GC.  Finally, if the record on
the shard that didn't get cleaned up is the earlier of the deletion records, the
worst that can happen is the Mako node is signaled to clean up an object that
was already removed.  Since the code should handle this in any case, it is not a
problem.
