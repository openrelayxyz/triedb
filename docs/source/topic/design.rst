Design
======

This document outlines a storage engine optimized for state trie storage.

Current state trie storage solutions rely on conventional, general purpose
key-value storage engines such as LevelDB or RocksDB. These conventional
key-value databases are a convenient way to build a state trie implementation
quickly, but they are far from an optimal on-disk representation of a state
trie.

Shortcomings of Existing Solutions
----------------------------------

Most if not all state trie implementations to date use key-value stores based on
LSM trees. In general, LSM trees require 1 disk seek for each level of the tree,
possibly more for the last level of the tree depending on available memory vs
the number of records. With a seven level LSM tree, this means looking up a
single key takes at least seven disk seeks. For a state trie that can be up to
64 levels deep, looking up a single value from the state trie takes up to 7 * 64 =
448 disk seeks. In practice, because state tries are sparse, it's rare to need
to navigate all 64 levels, but the point stands that seven seeks are required
for each level of the state trie.

A State-Trie Optimized Storage Engine
-------------------------------------

At Rivet, much of our infrastructure cost is driven by state trie storage, both
in terms of the disk requirements and the lookup performance of the state trie.
Thus, we have spent a reasonable amount of time researching solutions that could
reduce costs and improve performance.

Preliminary research guided us to storage solutions such as `Spotify's Sparkey <https://github.com/spotify/sparkey>`_
and a storage engine called `Pogreb <https://github.com/akrylysov/pogreb>`_.
Both of these storage engines optimize for read performance at the expense of
write performance. This is appealing for Rivet's use case, as we take daily
snapshots where we can afford slower write operations, and in exchange we could
look up each trie node in two or three seeks instead of seven or more.

Both Sparkey and Pogreb store data in a single large log file, using hash-based
indexes to identify where in the log file the value for a particular key can be
found. The complexity in these solutions, and the reason write performance is
sacrificed, is that hash based indexes can be complex to maintain. Read
operations, however, are very quick, as the key can be hashed to find the
location of the entry in the log file, then it requires just one seek to read
that record out of the log.

This system can be modified slightly to optimize state trie storage
significantly. When a state trie is updated, all new nodes are appended to the
log. Each node, rather than storing just the hash of child nodes, also stores
the location in the log of any child nodes. The log location of the new state
trie root is stored in an index, but the locations of the child nodes need not
be stored in the index, as they are only read in the context of the rest of the
trie. When reading a value out of the state trie, the state root's location is
retrieved from the index, then the content is retrieved from the log file.
Traversing the trie, the log location of the root node's next child can be
determined directly from the root node. Rather than going back to an index, we
can simply seek to that location in the log and retrieve the next node, and so
on to the leaf. Because each node contains the exact log location of the next
node, each level of the trie beyond the root can be traversed with only one seek
operation. Because individual nodes need not be stored in the index, the index
can be kept smaller - possibly small enough to be kept in memory - allowing
state trie lookups to be performed with only one disk seek per trie node. This
means that a read that might take 448 seeks in LevelDB would take 65 seeks -
approximately an 85% decrease in disk activity. It's difficult to say for
certain until we have an implementation to test, but this performance improvment
may be sufficient to make storing Ethereum state trie storage on magnetic disks
(instead of SSDs) feasible.

State Trie Synchronization
--------------------------

This approach to trie storage should work very well for tries created by
starting with an empty trie and sequentially updating it to add key / value
pairs. In those situations, any time a key is written to the state trie the leaf
and any modified nodes back to the root will be written in the same operation,
and there is no situation where a node must be written without knowledge of the
in-log location of its child node.

State trie synchronization, as tend to be done with blockchain fast syncs,
become more complicated. In those cases the state trie tends to be written to
disk from the top down, starting with the root, then looking for any nodes
referenced by the root, then any nodes referenced by those nodes, and so forth.
Given this approach, nodes must be written before their child nodes have been
observed, and thus before their location in the log can be known.

To support state trie syncronization, it will be necessary to keep a separate
index of nodes containing unresolved children. That way as new nodes are
received from other sources, the parent node can be retrieved from the index and
updated in place to point to the imported child node. Once all of a node's
children have been resolved, it can be removed from the index, as the only time
a direct node reference is required is updating the child references during the
synchronization process.

Limitations
-----------

This approach has some notable limitations.

Pruning
.......

There's no obvious path to state trie pruning. With conventional solutions,
nodes can be deleted and their space reclaimed. With this approach to state trie
storage, reclaiming the space of deleted nodes would moving nodes, which would
require updating all references to those nodes, which is difficult given the
lack of a node index.

However, state trie pruning is already a significant challenge, and Geth
refrains from pruning nodes once written to disk. The general problem with state
trie pruning is that while you might want to remove older state tries, those
older state tries share nodes with newer state tries, and it would require
massive overhead to track what nodes are associated with what state tries to
know which ones can be deleted and which should not.

Instead of trying to prune out state entries that are no longer needed, it will
probably be more effective to copy across the tries we are interested in into a
new database. The synchronization process would not copy across nodes not listed
in tries we don't wish to retain, and any nodes shared between tries would
synchronize quickly.

Oustanding Decisions
--------------------

There are several considerations that will require further research to make a
decision. These include:

* What indexed storage system to use for storing references to the state trie
  roots and the index of incomplete nodes during syncronization.
* What serialization format to use to track nodes on disk.
* Possible approaches to compressing the data, considering storage savings and
  possible performance penalties

Implementation
--------------

We plan to implement the proposed state trie storage implementation in Golang as
a stand-alone library, with integrations into the go-ethereum project. As we
finalize the details of state trie storage, that will be written into a clearly
defined specification to allow for compatible implementations in other
languages.
