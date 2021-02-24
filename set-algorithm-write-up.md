---
tags: ssb, protocol
---
$$
\newcommand{\mathfn}[1]{\mathsf{#1}}
\newcommand{\agg}{\mathfn{agg}}
\newcommand{\split}{\mathfn{split}}
%
\newcommand{\mathtype}[1]{\mathfn{#1}}
\newcommand{\Message}{\mathtype{Message}}
\newcommand{\nimplies}{\nRightarrow}
\newcommand{\depth}{\mathfn{depth}}
%
\newcommand{\dotrm}{\mathrm{.}}
\newcommand{\depth}[1]{#1 \dotrm d}
$$

# Set Replication Trees

## Introduction

Scuttlebutt is a protocol that replicates messages that were signed by their authors between computers. These messages each have an ID, a cryptographic hash of the message. Often, messages reference each other by that ID. We say that the messages referenced by a message are its children.

Since Scuttlebutt started, we have replicated all messages of an author linearly, in the sense that within their _feed_, we receive one message after the other, always in the same order. The limitation of this approach is that it is a single-writer datastructure. If two concurrent processes append message to their data without any coordination, their replication state will diverge. This condition can not be cleanly recovered from.

Most SSB applications (likely all relevant ones) operate on messages that form a directed acyclic graph (DAG). The more recent applications have adopted certain patterns to encode the DAG structure in a consistent way. One strategy that has emerged is including the transitive reduction of the DAG in the message. This means every message created by the application contains references to the messages that are not yet referenced by another message, i.e. all the messages that do not have incoming edges. Furthermore, we require that starting from any node in the DAG, by following outgoing edges we eventually end up at the same node, which we call the root. In the Scuttlebutt community, this kind of DAG is often referred to as a tangle. In this article, we will use the term _transitive reduction DAG_, or, in short, trDAG.

This suggests there is an opportunity for improvement. If we are only using one application but another user publishes many messages irrelevant to us, we download (and store) a lot of data that we don't need or want. Being able to sync data per application-level object would allow sharing an identity between applications without draining the resources of users that only use a small subset of them.

In this article we construct a protocol for efficient reconciliation of DAGs in which links conform to the transitive reduction pattern.

The approach we take is a variant of the traditional divide-and-conquer technique. We have an interactive protocol where the input sets are split (along the same boundary) into two disjoint subsets, and for both subsets an aggregate value is sent to the other party. That party can then compare the digests of the two subsets with its own, and recurse if they are different. Eventually the difference will be found.

Let's take a closer look at the central parts here.

### Aggregate Value

<!-- TODO: figure out where to mention aggregate subtraction -->

In order to efficiently check equality of sets of messages, we compute a short aggregate values from the set elements that we can compare in place of the sets themselves. This aggregate value is computed using an aggregate function.

We also need a function that takes two aggregate values and returns the aggregate value of the union of the sets the two values represent. We call that function an aggregate union function, and it is required for two central reasons: Firstly, it would be inefficient if every computation based on aggregates would have to take all the aggregated messages as input. Secondly, the parties involved in the protocol need to be able to verify the consistency of the data received from their communication partner, as this is used as an indicator for their honesty. Therefore, when receiving the aggregates of two subtrees rooted at neighboring nodes as well as the parent of these nodes, participants need to be able to verify the consistency without having access to the aggregated tree leaves.

<!-- TODO: This is already in the previous paragraph and I have to pick which one I want to keep -->
Consider a tree, where each inner node contains an aggregate, comprising the union of the aggregates of the children. Using an aggregate union function we can infer the aggregate value of a node from those of it's children. If we did not have such a function, the leaf data of the entire subtree would have to be re-read for every aggregate computation.

In this system, we use an aggregate tuple $(c, x)$, containing two values: the number of messages aggregated $c$, and the XOR of the IDs of all these values $x$. XOR and addition are among the cheapest computations there are, and we can trivially construct an aggregate and an aggregate union function for these tuples.

Note that we can use other properties of XOR and addition for optimizations, as is shown in the Aggregators section. We also describe the security of this scheme in the Security section.

<!--
Using $\setminus_{agg}$, we can infer the aggregate value of the second child of a node for which we know the aggregate, as well as that of one child node.
-->

### Tree Construction

To construct the tree for a set of messages in a trDAG $M$, consider a function $\split : M \to (M_0, \dots, M_k), k \leq |M|$. In order to be useful in this context, the function needs to (i) produce disjunct sets, the union of which needs to be $M$, (ii) always assigns each message to the same set, independent of what $M$ contains otherwise, and finally (iii) should ensure that messages which likely only one of the parties has should be in a set $M_i$, $k-i$ small.

Solving this problem for the general case is difficult, but we are working with trDAGs, which come with a certain structure. The general idea is that we define the depth of a message to be its Lamport timestamp. Note that the Lamport timestamp of a message is the same under all circumstances, and that, due to the nature of timestamps, new messages (which are more likely to only be owned by one party) have a high depth. Furthermore, every message gets assigned exacly one depth. Therefore we choose it as a foundation to implement a simple $\split$ function.

The next step is to construct a search tree for these depths. In that tree, each inner node contains the aggregate of the subtree that node is a root of, as well as links to the children. One simple option for the structure of this tree would be a binary trie, where the left/right sequence of the path from the root to a leaf $M_i$ is the binary representation of $i$.

This approach is explored further in the Tree Construction section.

### Protocol

After constructing the replication tree, we need two computers to communicate in order for them to efficiently find and resolve differences. Let's take a close look at the word efficient here. There are two main metrics for the efficiency of the protocol:

The first metric is cumulative message size, i.e. the total amount of data sent over the network during a session. The second metric is the number of communication rounds until both parties complete the protocol. We aim to minimize both of these metrics.

It is clear that we can build a very round-efficient protocol by sending all data we know in a single message. That message however would be very large. We could also imagine protocols that are size-efficient, at the cost of a lot of a lot of communication rounds. 

<!-- TODO: once the protocol is fixed and the metric as known, update this to be specific -->
We propose a protocol for which both metrics are be sublinear, maybe something like $O(log^3)$ or so.

Approaches we take in order to achieve these numbers are laid out in the Protocol section.

## Aggregates

Aggregates allow two parties to efficiently check whether two sets of messages are equal. These aggregate values are computed, sent to the communication partner and compared. If the aggregate values match, the sets are equal; or, more formally:

Definition: An aggregate function is an efficiently computable function $\agg: S \to A$, with $S$ a random set of leaves and $A$ an aggregate value, such that for all $S_1, S_2$ probabilistically holds:

$$
\agg(S_1) = \agg(S_2) \implies S1 = S2
$$

<!-- TODO: I don't know if probabilistically is the right word here
What I mean is that as long as the hashes are random, the probability
that this holds is high. In general I think the definition is poorly worded
-->

Additionally, we would like to combine two aggregate values into a joint aggregate, that represents the union of both sets.

Definition: An aggregate union function is an efficiently computable function $\cup_{\agg}: A \times A \to A$ such that for all $S_1 \cap S_2 = \{\}$ probabilistically holds:

$$
\cup_{\agg}(\agg(S_1), \agg(S_2)) = \agg(S_1 \cup S_2) \\
% \setminus_{\agg}(\agg(S_{1} \cup S_2), \agg(S_1)) = \agg(S_2)
$$

or, possibly more readable, in infix notation

$$
\agg(S_1) \cup_{\agg} \agg(S_2) = \agg(S_1 \cup S_2) \\
%\agg(S_{1} \cup S_2) \setminus_{\agg} \agg(S_1) = \agg(S_2)
$$

Note that we only require this holds probabilistically. We do not require the function to be secure if an attacker can select one of the sets based on the other. We discuss this problem in the Security section and propose a solution.

We use a two-component aggregate value. The first component is the number of elements aggregated. We call this component the cardinality-component, and define that of a set of messages $S$ to be $|S|$, and the union functions of two aggregates $a_1, a_2$ to be $a_1+a_2$. This is correct, assuming that all sets are disjunct. Since each set represents a different subtree on the same tree level, and that each message only appears in one leaf, this assumption holds. However, it is obvious that for this is not a well-defined aggregate value since clearly it does not hold that $|S_1| = |S_2| \nimplies S_1 = S_2$. We therefore need a different mechanism in order to construct a well-defined aggregate.

The second component is the XOR of the hashes of all the contained messages. As long as these hashes are random and have length $l$, the probability that given a set of messages, finding a distinct set of messages that results in the same XOR is $2^{-l}$, and we can choose l large enough for that to be negligible. Note that we do not care about birthday attacks here, since one of the two values is fixed, by virtue of being computed by an honest party (if neither party is honest, security is meaningless).

Based on these, we define the aggregate function $\agg$ and the aggregate union function $\cup_{\agg}$ as follows:

$$
\agg: S = \{s_1, \dots, s_n\} \mapsto (n, s_1 \oplus \cdots \oplus s_n) \\
\cup_{\agg}: ((n_1, x_1), (n_2, x_2)) \mapsto (n_1 + n_2, x_1 \oplus x_2)
$$

You may have noticed that addition and XOR give us an additional property: we can also compute the difference $-_{\agg}$of two aggregates, defined by

$$
-_{\agg}: ((n_1, x_1), (n_2, x_2)) \mapsto (n_1 - n_2, x_1 \oplus x_2).
$$

This operation will be helpful for maximizing the information learned from the aggregate values a node receives.

An important note here is that the assumption that hashes are random does not hold up in reality. An adversary can craft many messages and then only send those that fit certain criteria, introducing an adversarially-controlled bias. We present a solution to this problem based on per-session randomization is the Security section, where we will also present a good choice for the hash length $l$.

## Tree Construction

The next step is to construct a replication tree that represents a trDAG. In order to be able to design a good algoithm, we first need to formalize what a trDAG is. From the structure of the trDAG we derive a $\split$ function, and finally we construct an efficient search tree.

Afterwards, we define a $\split$ function, which is used to partition the messages into buckets, which in turn will be the basis for generating the replication tree.

### Transtitive-Reduction DAGs

A trDAG is a directed acyclic graph with exactly one root node, i.e. one node that does not have any child nodes. When a new node $n$ is created, for each existing node $m$ that does not have an incoming edge, an edge pointing from $n$ to $m$ is added as well. This means that each node has edges to the transitive reduction of the DAG as known at the time of adding the node.

Definition: A trDAG is a DAG that has exactly one node that does not have outgoing edges, such that for all nodes A, B, C with a path existing from A to B and from B to C, there is no edge from A to C. The depth of a trDAG node $n$ is the length of the longest path to the root node. We refer to it as $\depth{n}$.

We do not find the depth through searching the graph. Instead, trDAG nodes keep record of their own depth. When inserting a new node $n$ with children $C$, we set $\depth{n} = \max_{c \in C}(\depth{c}) + 1$ (except for the root, which has a depth of $0$). rote that this correlates with the Lamport timestamp scheme.

There is one important consideration for designing algorithms in this setting. When working with these graphs in the context of Scuttlebutt (and may other modern decentralized protocols), a graph is encoded such that edges are part of the node they originate from. When replicating nodes naively, it is possible to end up in a situation where a party knows about an edge, but not the node the edge points to, and, by extension, the edges originating from that node. This means that the graph can not be fully traversed, which also means that the depth of that node can not be computed. We avoid this issue by only allowing parties to process nodes that only have edges where destination nodes are part of the graph already.

We now turn to how this strucutre can be exploited in order to achieve efficient replication.

### The $\split$ function

The $\split$ function distributes the set of hashes of all locally known messages into a sequence of sets and will be the base of our divide-and-conquer approach.

The resulting sets will be the leaves of the replication tree, so in order to require that each message is in exactly one leaf, we require that the resulting sets are disjunct, and that no message gets lost, i.e. that the union of the resulting sets is equal to the original set.

Note that two parties need to agree on which messages results in which leaf, i.e. resuting subset. In order to ensure this holds, the function additionally needs to assign each message to the resulting set with the same index, disregarding any other messages in the original set. A precise definition follows.

Definition: A $\split$ function is a function $M \to (M_0, \dots, M_d)$, such that that $M_0, \dots, M_d$ disjunct, $M_0 \cap \dots \cap M_d = M$, and $\split$ stable, i.e.

$$
\begin{align}
\forall M, M'\ ,\\
\forall m \in M \cap M'\ ,\\
            M_{0..d} = \split(M)\ ,\\
          M'_{0..d} = \split(M') :\\
   \exists j : m \in M_j \wedge m \in M'_j
\end{align}
$$

In order for the tree search to be able to quickly rule out large parts of the tree, we design the split function in such a way that it is very likely for the difference between the sets of the two parties to reside in subsets with a high index. This way, the parts of the tree with low indices do not need to be traversed --- in most cases at least.

Notice that if, from each message $m$, we derive a number $i_m$, and then assign each message to a set $S_{i_m}$, the resulting sets are both disjunct and their union is the original set. If $i_m$ is computed purely from the message itself, it even is stable. We can therefore use this strategy to construct a $\split$ function.

It is clear that $i_m = \depth{m}$ is a great choice, as it is simple, maintains locality and is efficiently computable. Therefore, we define $\split$ as

$$
\split : M \mapsto (M_0, \dots, M_d), \quad M_i = \{m \in M: \depth{m} = i\}.
$$

Note that $d \leq |M|$, and that $M_0 = \{ \text{root} \}$. We assume that the root is already known to both parties, so we don't include $M_0$ in the replication process.

### Tree Structure

Now that we have a scheme in place to compute the leaves, we go ahead and propose a structure for the tree. Before diving into that, we will take a look at a trick that is both a simplification and an optimization. This will make the construction of the actual tree significantly simpler.

As it stands now, we need to construct a tree with some integer number of leaves. For the purpose of the argument, let us assume the tree we construct is a binary tree. In a binary tree, having a leaf count that is not a power of 2 will result in a sparse tree, which is more difficult to reason about. Having a tree with a leaf count that is a power of 2 clearly would be helpful.

Additionally, the problem we are working on rewards increasing the information density (i.e. the amount of information sent per round per leaf) around subsets with high depth. One trick to achieve this is to construct multiple trees, each holding $2^k$ leaves. For subsets $S_1, \dots, S_d$, we decompose $d = \sum_i d_i \cdot 2^i, d_i \in \{0,1\}$ and for each $i: d_i =1$ we construct a tree with $2^i$ leaves. For example, if we have $d=23 = 10111_2$ subsets, we have trees with 16, 4, 2 and 1 elements (in order, i.e. the trees contain subsets 1..16, 17..20, 21..22, 23). Therefore, the term "replication forest" would technically be more appropriate.

The trick here is that we can now perform the searches in parallel, and the density of information about subsets with higher indices has increased. At the same time, we don't have to worry about how traversal of sparse trees should work. The maximum number of trees generated is logarithmic in $d$, so this method does not result in an unacceptable amount of network traffic.

Note that we assumed that we'd be using binary trees. This is in fact the case, because it allows us to fully make use of the inference capabilities our aggregator provides: If we know the aggregate values of a node and one of its children, we can use the $-_{agg}$ operation to compute the aggregate value of the other child as well.

The construction of the trees themselves is very simple now. For the largest tree in a forest with $2^k$ leaves, we construct a prefix tree over the least significant $k$ bits of the depth of the subset. Meaning, going left $k$ times leads to the subset with depth 1, going right $k$ times leads to the subset with depth $2^k$ (again, the subset with depth 0 is the root and is implicit).

A noteworthy fact is that we get most trees (as local maxima) when $d = 2^k -1$, for all $k$. In this case, all the bits are set and we get $log_2(d)$ trees. In all other cases, at least one bit is cleared and we have less trees. To give an example for this, let's consider a trDAG of depth 15. From this we compute four trees, with leaf counts 8, 4, 2 and 1, resulting in depths 3, 2, 1 and 0.

## Protocol

Using these building blocks, we design an efficient protocol for learning the missing messages. The protocol we propose can be broken down into components that implement the following functionalities: (1) decide on the trDAG that should be synchronized, (2) learning the IDs of missing messages (3) fetching missing messages. These components operate in part sequentually, and in part concurrently. Specifically, the fetching of the missing messages is done in parallel to learning the IDs of the missing messages.

The first component could be implemented using a query engine, where e.g. the initiator sends a description of a trDAG (e.g. the root hash). A detailed of this component is considered beyond the scope of this article and will not be discussed in-depth, as it seems to be a rather practical engineering problem, compared to the abstract nature of the proposed protocol (and therefore not really what I'm super good at).

The second component consists of two phases. In the first phase, the two parties compare what the maximul depth of their local copy of the trDAG is. If it is not equal, the parties know that all messages with higher depth than the lower maximum depth need to be transmitted. They then go on to perform the more complex tree-based set reconciliation, as described in Algorithm 1.

The final component is concerned with bulk transmission of the messages that were observed only at one party. There are two possible approaches here. Either the party looking for a particular message queries their communication partner, or both parties compute the sets of messages they think their communication partner wants and eager-push them. This also seems like a problem that needs to be explored and the solutions compared in the real world. Therefore, no detailed discussion is performed.

Algorithm 1: Pseudocode of naive algorithm. 

```go

type ID []byte

type Data []byte

type TrDAGNode struct {
    ID    ID
    Root  *TrDAGNode
    Prev  []*TrDAGNode
    Depth int
}

type TreeNode struct {
    Prefix    []byte
    Aggregate []byte
    Depth     int // in the trDAG
    Height    int // in the tree, matches prefix bit-length
    
    // If this is an internal node, Left and Right are set and Leaves is nil.
    // If it is a leaf node, Left and Right are nil and Leaves is set.
    Left, Right *TreeNode
    Leaves      []*TrDAGNode
}

type Session interface {
    SendDepth(int)
    ReceiveDepth() int
    
    SendTreeNodes([]*TreeNode)
    ReceiveTreeNodes() []*TreeNode
    
    SendData(ID, Data)
    ReceiveData() (ID, Data, bool)
}

func RunProtocol(db DB, s Session) {
    // Component 1: Query
    var (
        qry   Query      = s.ReceiveQuery()
        trdag *TrDAGNode = db.Query(qry)
    )

    // Component 2: Find difference
    var (
        pullList []*TrDAGNode
        pushList []*TrDAGNode
    )
    
    // Phase 1: Find lower maximum depth
    s.SendDepth(trdag.Depth)
    rxDepth := s.ReceiveDepth()
    minDepth := trdag.Depth
    if rxDepth < trdag.Depth {
        minDepth = rxDepth
        curDepth := rxDepth
        for curDepth := rxDepth; curDepth <= trdag.Depth; curDepth += 1 {
            pushList.Append(trdag.GetLeavesByDepth(curDepth)...)
        }
    }
    
    // Phase 2: Tree-Based Set Reconciliation
    
    // only consider messages with a depth that both parties have
    trdag = trdag.SubTrDAGWithDepthLte(minDepth)

    var (
        // construct tree
        treeRoots  []*TreeNode = trdag.BuildTrees()
        
        // nodes being sent out this iteration
        roundNodes []*TreeNode = treeRoots
        
        // nodes being sent out next iteration
        nextRoundNodes []*TreeNode
        
        // end flag, loop breaks if true
        isEnd = false
    )
   
   
    for len(roundNodes) > 0 {
        s.SendTreeNodes(roundNodes)
        
        rxNodes = s.ReceiveTreeNodes()
        assert len(rxNodes) == len(roundNodes)
        for i := range rxNodes {
            if rxNodes[i].Aggregate == roundNodes[i].Aggregate {
                continue
            }
            
            if roundNodes[i].Leaves == nil {
                nextRoundNodes.Append(roundNodes[i].Left)
                nextRoundNodes.Append(roundNodes[i].Right)
            } else {
                pullList.Append(
                    rxNodes[i].Leaves.Minus(
                        roundNodes[i].Leaves))
                pushList.Append(
                    roundNodes[i].Leaves.Minus(
                        txNodes[i].Leaves))
            }
        }
        
        roundNodes = nextRoundNodes
    }
    
    // Component 3: Transmit data of respective nodes
    // Phase 1: send data
    for _, id := range pushList {
        s.SendData(id, db.GetData(id))
    }
    
    // Phase 2: receive data
    for {
        id, data, isEnd := s.ReceiveData()
        if isEnd {
            break
        }
        
        assert db.Validate(id, data)
        db.SetData(id, data)
        pullNodes.Remove(id)
    }
    
    // finally, check that all data was received
    assert len(pullNodes) == 0
}
```


### More Round-Efficient Variant at Higher Per-Round Costs

## Security

- hash randomization and length
- verification and consistency checks of received data