<!--
SPDX-FileCopyrightText: 2021 Anders Rune Jensen

SPDX-License-Identifier: CC-BY-4.0
-->

# SSB set replication notes

This contains loose thoughts on SSB feeds, meta feeds and set
replication for partial replication. For the purpose of this, set
replication refers to the algorithm decribed in
[here](https://hackmd.io/VuJqxRtFTCyby7JtAbeZJA) while meta feeds
refers to [this](https://github.com/ssb-ngi-pointer/ssb-meta-feed).

In SSB the main abstraction is a linear single writer feed, because
each message is signed and linked, these messages can be distributed
without fear of a third party tampering or leaving out messages. The
downside is that feed needs to be replicated in full.

[Bamboo](https://github.com/AljoschaMeyer/bamboo) allows partial
replication by using lipmaalinks, while an upgrade compared to flat
feeds, this only guarantees that you got the particular subset of
messages without any tampering that is defined by that one chain of
lipmaalinks, but not necessary all the messages of a different subset,
like responses to a thread. This is similar to partial replication
guarantees of meerkle trees in
[hypercore](https://github.com/mafintosh/hypercore).

Applications in SSB have started using another scheme to guarantee
integrity of a set of messages. By using tangles where messages are
linked across feeds to the previous messages on the same "topic" (root
message). This allows a non-linear multi writer datastructure to be
built. One of the downsides of this is that the link structure is
baked into the messages, this means that if you encode a private
groups this way, you will be able to replicate opaque group messages,
but not the subset of messages such as post messages in a particular
group.

The set replication algorithm mentiond above deals with message hashes
and the links between these. Using this, two peers are able, after
multiple rounds, to converge on the difference between the two
sets. Then the messages that are missing can then be exchanged in some
form.

Some good use cases for set replication: Threads (outside follow hops
or in the case of partial replication getting old threads), private
groups (authenticated group peers), claims (subsets).

Meta feeds describes a mechanism for defining "physical" subfeeds for 
a bigger, single "logical" feed. This allows partial replication of
that subset as claims.

This concept could be extended to tangles where instead of just
linking to the previous message of a certain kind on your own logical
feed, you would also link to messages in other logical feeds. In very
much the same way as tangles in threads function now, but instead of
including the actual content these subfeeds only include the linkage
between messages.

Multiple subfeed can then be combined to create a tangle set. Consider
the case of 3 feeds: A, B and C. First they define a common concept,
for example all post messages in a certain group. Then each feed
create a subfeed with their messages in this set.

Assuming the messages are tangles like this:

```
   a2      (the root, from the point of view of A, B, C)
  / \
 b1 c100   (two concurrent posts)
  \ /
  a10      (a message merging those two contexts together)
   |
   b3      (the current end-point)
```

The messages would be linked as:

```
a2: none
a10: a2, b1, c100

b1: a2
b3: a10, b1

c100: a2
```

By combining these 3 subfeeds, it would be possible to construct a
tree that could be synchronized using set replication.


One point to note is that meta feeds and set replication on messages
defined in feeds are both backwards compatible with existing
clients. The set replication algorithm is very general and can be used
in settings where there are not even feeds just single messages
written by a key.


Open problems:
 - same-as (linked feeds or identity tangles)
 - combine set replication with EBT (subscribe to tangles instead of feeds)

Latest set replication proposal: https://hackmd.io/mPmWbNaDR9-AfrM1mSYtHQ

FIXME: Compare the set replication with:
 - https://martin.kleppmann.com/2020/12/02/bloom-filter-hash-graph-sync.html
 - Twitter discussion about link above: https://twitter.com/pvh/status/1384635288836661252
 - Github PR to automerge for the algo above: https://github.com/automerge/automerge/pull/332#issuecomment-812485433
