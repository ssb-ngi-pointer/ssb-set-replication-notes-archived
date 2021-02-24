# SSB set replication notes

This contains loose thoughts on SSB feeds, meta feeds and set
replication for partial replication. For the purpose of this, set
replication refers to the algorithm decribed in
[here](https://hackmd.io/VuJqxRtFTCyby7JtAbeZJA) while meta feeds
refers to [this](https://github.com/ssb-ngi-pointer/ssb-meta-feed).

In SSB the main abstraction is a linear single writer feed, because
each message is signed and linked these messages can be distributed
without fear of a third party tampering or leaving out messages. The
downside is that feed needs to be replicated in full.

[Bamboo](https://github.com/AljoschaMeyer/bamboo) allows partial
replication by using lipmaalinks, while an upgrade compared to flat
feeds, this only guarantees that you got a particular subset of
messages without any tampering, but not necessary all the messages of
a particular subset. This is similar to partial replication guarantees
of meerkle trees in [hypercore](https://github.com/mafintosh/hypercore).

Applications in SSB has started using another scheme to guarantee
integrity of a set of messages. By using tangles where messages are
linked across feeds to the previous messages on the same "topic" (root
message). This allows a non-linear multi writer datastructure to be
built. One of the downsides of this is that the link structure is
baked into the messages, this means that if you encode a private
groups this way, you will be able to replicate the private group, but
not the subset of messages such as post messages in the group.

The set replication algorithm described deals with message hashes and
the links between these. Using this, two peers are able to, over
multiple rounds to converge on the difference between the two
sets. These messages can then be exchanged in some form.

Some good use cases for set replication: Threads (outside follow hops
or in the case of partial replication getting old threads), private
groups (authenticated group peers), claims (subsets).

Meta feeds describes a mechanism for defining subfeeds for a single
feed. This allows partial replication of that subset as
claims. Question: how do we define a similar mechasnism for tangles?
Preferably in a way that does not require them to be defined a priori.

One point to note is that meta feeds and set replication on messages
defined in feeds are both backwards compatible with existing
clients. The set replication algorithm is very general and can be used
in settings where there are not even feeds just single messages
written by a key.


Open problems:
 - same-as (linked feeds or identity tangles)
 - combine set replication with EBT
 - would it be possible to define views on top of messages instead of
   baking that into the messages. If so, how does that work in a group
   setting? Is that similar to [embedded
   indexes](https://github.com/hypercore-protocol/p2p-indexing-and-search/blob/main/problems/07.md)
   in hypercore?

FIXME: Compare the set replication with https://martin.kleppmann.com/2020/12/02/bloom-filter-hash-graph-sync.html
