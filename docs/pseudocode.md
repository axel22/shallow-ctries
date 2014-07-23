

Shallow Ctries
==============

This document describes the basic update and snapshot algorithm
for the shallow Ctrie data structure.
We use *pseudocode* below, not real Scala code.


# Data-types

Shallow Ctries work without I-nodes.
They are comprised of three basic node data-types,
represented by the abstract type `Node`.

## Node

Here are the three basic types of nodes:

- The `CNode` data-type represents an inner branching node.
It does not contain elements, only pointers to other nodes.

- The `SNode` data-type represents a key-value pair.

- The `LNode` represents a list of key-value pairs which have collisions.

Data-types:

    Node {
    }


    CNode: Node {
      @volatile var status: Status
      val bitmap: Int
      val children: Array[Node]
      val gen: Gen
    }


    SNode: Node {
      val key: K
      val value: V
    }


    LNode: Node {
      val collisions: ListMap[K, V]
    }


Note that `SNode`s and `LNode`s are immutable.
After they are created, they are never changed again.
The `CNode` cannot change its `bitmap` field,
the array object `children` or the `gen` field (more on the `Gen` type later).
However, the contents of the fixed-size array object `children` **can** be mutated.
Additionally, each `CNode` has a special `status` field.
The purpose of this field is to share information about the modifications in the `CNode`,
as we will when we introduce the pseudocode.


## Status

The `Status` objects contain information about the current modifications in a `CNode`
(or in a special `Root` object, as we will see later).
They are represented with the abstract type `Status`:

- The basic type of a `Status` is the singleton object `Idle`.
The `Idle` object means that nobody is modifying the node.

- The `Mutate` status object denotes that the node `parent` must replace
its child node `child` with the new child node `newChild`, and that
this change must occur at the position `index`.

- The `Snap` status object is only used at the root of the Ctrie,
and it denotes that the Ctrie generation `oldGen` needs to be updated
to the new generation `newTrieGen`.
Additionally, a new root with the generation `newSnapGen` is created.
When this happens, the `Node` object below the root is captured into a new root,
and stored into the `frozen` field as a new Ctrie
(more on this at the end of this document).

Data-types:

    Status {
    }


    Idle: Status


    Mutate: Status {
      val parent: Node
      val child: Node
      val newChild: Node
      val index: Int
    }


    Snap: Status {
      val oldGen: Gen
      val newTrieGen: Gen
      val newSnapGen: Gen
      @volatile var frozen: Root = null
    }

The `Status` data-type comes with a polymorphic `complete` method.
In a *virtual dispatch* implementation, these data-type may look as follows:

    trait Status {
      def complete(): Boolean
    }

    class Idle extends Status {
      def complete() = true // do nothing
    }

    class Mutate extends Status {
      def complete() = ??? // will be described later in this document
    }

    class Snap extends Status {
      def complete() = ??? // also, described later
    }


## Misc

There are several miscellaneous data-types used in the shallow Ctrie:

- The `Root` object represents the root of the shallow Ctrie.
It has the fields `status`, `child` and `gen`.
Much like the `CNode`, the `status` field is used to agree on the modifications
in the root.
The `child` field is the pointer to the root of the Ctrie.
The `gen` field represents the current generation of the Ctrie.

- The `Gen` object does not have any fields, but it has an identity.
The `Gen` objects are compared using reference equality to decide if
nodes are at the proper generation.
It cannot be implemented as a singleton object.

Data-types:

    Root: Node {
      @volatile var status: Status
      @volatile var child: Node
      @volatile var gen: Gen
    }


    Gen { // has reference identity
    }


## Pseudocode

We start by describing the pseudocode of the shallow Ctrie without the snapshot operation.
We will show the `insert` operation.
Other operations, like `remove`, conditional `remove`, `replace` or `putIfAbsent`,
have an almost identical flow.

The `insert` operation proceeds in two phases.
In the first phase, we search for the appropriate location in the Ctrie where the mutation should take place.
The second phase is the mutation phase.

The mutation phase proceeds in 5 steps:

1. change the status field of the parent to Mutate
2. change the status field of the child to Mutate
3. change the pointer in the parent to the new child
4. change the status field of the parent to Idle
5. change the status field of the new child to Idle

Note: the new child is **created in the `Mutate` status**.
This ensures that it has to be switched to the Idle status before it gets used.

Note: the old child **remains in the `Mutate` status after it is removed from the trie**.
This ensures that no other thread modifies the dangling node after it is removed.
Also, it encodes information on how the child was removed, i.e. by which operation.

Note: all algorithms assume there are no spurious `CAS` failures.
This is the case on the JVM.

After this basic idea, we present the concrete pseudocode below.
We write the atomic reads and writes like `READ` and `CAS` using capital letters,
to stress the important steps.
We treat the `root` object of type `Root` slightly differently.
We first compute the hashcode of the key `k` once.
We then read the `child` reference of the `root`, and store into a local variable `node`.
We proceed casewise:

- If we cannot place the new key-value pair `(k, v)` directly under the root (checked with `freeAt`),
we descend calling the overloaded `insert` method.

- Otherwise, we need to do the change directly below the root.
We create a new `Mutate` object to share information about the change
and attempt propose the change immediately with a `CAS` (common case, fast path).
If the assumption that the node is in the `Idle` state is correct,
we attempt to complete the operation by calling `complete` on the `Mutate` object, retrying if this fails.
If the assumption that the node is in the `Idle` state fails,
we read the current status, help the operation by calling `complete`, and retry the entire operation.

The top-level `insert` method:

    @tailrec def insert(root, k, v) {
      hash = hashCode(k)
      node = READ(root.child)
      node match {
        case CNode if !node.freeAt(hash, 0) =>
          insert(root, node, k, v, hash, 0)
        case _ =>
          nstatus = new Mutate(root, node, newNode(node, k, v), 0)
          if (CAS(root.status, Idle, nstatus)) {
            if (!nstatus.complete()) insert(root, k, v)
          } else {
            ostatus = READ(root.status)
            ostatus.complete()
            insert(root, k, v)
          }
      }
    }

Before proceeding, we note the following invariant,
which will be kept throughout the algorithm:

**INV1**: After a `CNode` is no longer reachable in the shallow Ctrie,
its `status` field remains forever set to the `Mutate` operation that removed it.

This invariant ensures that **a node that left the Ctrie is never modified again**,
thus preventing lost updates.

Now, we study the recursive `insert` method.
The structure is similar as before,
with the following differences:

- We cache the `hash` of the key and forward it around.

- We track the `level` of the Ctrie as we recurse.

- We calculate the position of the child branch
at every level and store it into a local variable `pos`.

Recursive `insert`:

    @tailrec def insert(root, cnode, k, v, hash, level) {
      pos = calcPos(hash, level, cnode.bitmap)
      node = READ(cnode.array(pos))
      node match {
        case CNode if !node.freeAt(hash, level + 5) =>
          insert(root, node, k, v, hash, level + 5)
        case _ =>
          nstatus = new Mutate(cnode, node, newNode(node, k, v), pos)
          if (CAS(cnode.status, Idle, nstatus)) {
            if (!nstatus.complete()) insert(root, cnode, k, v, hash, level)
          } else {
            ostatus = READ(cnode.status)
            ostatus.complete()
            insert(root, k, v)
          }
      }
    }

The gist of the modification phase is in the `complete` method of the `Mutate` data-type.
This method identifies the different status that the `parent` node and the old `child` node are in,
and proceeds with the mutation.

- `(this, Idle)`: The first time it is called, only the `parent` is set to the `Mutate` object `this`.
This is the case `(this, Idle)` in the pseudocode of the `complete` method.
In this case, we `CAS` the `child.status` to `this`.
If the `CAS` fails, we need to detect why it failed and resolve.
If the `CAS` succeeds, both the `parent` and the `child` are set to `this`, and we need to proceed to the next step.
In both cases, we call `complete` tail-recursively.

- `(this, this)`: When both the `parent` and the old `child` are set to `this`, the mutation to the `parent` can proceed.
The `CAS` at `parent(index)` tries to insert the `newChild`.
This can fail due to 2 reasons -- the `CAS` from `child` to `newChild` was already done by some other thread,
or, the `child` was replaced by some other node before this mutation got a chance to execute.
Alternatively, this `CAS` can succeed, indicating that the current thread inserted the `newChild` into the Ctrie.
In all these cases, we need to, **in this order**, first set the `child.status` from `this` to `Idle`,
and then set the `parent.status` from `this` to `Idle` -- both idempotent operations.
After them, we call `complete` recursively.

**INV2**: When a `CNode` object is added to the Ctrie,
its `status` field is set to `Mutate` object of the operation which added it.

- `(this, status)`: When the `parent.status` is `this`, but the `child.status` is neither `Idle`, nor `this`,
this indicates that the `child` has either been removed by some other operation,
or that some other operation is executing at `child`.
If the node at `parent(index)` is not `child`, we can `CAS` `parent.status` back to the `Idle` state
and return `false`, to report that the operation failed.
If the node at `parent(index)` is still `child`, there is a possibility to complete the operation --
we help the operation at `child` and retry by calling `complete` tail-recursively.

- `(_, _)`: When the `parent.status` is not `this`, this means that the thread either set the `parent.status` back to `Idle`,
or that it arrived very late to the game.
By this time, the mutation was either reverted or completed successfully.
In either case, the `insert` operation that called the `complete` needs to know whether the operation succeeded or not.
To find this out in a lock-free manner, we read the status of the `newChild` and the node at `parent(index)`.
If we detect that the `newChild` node is in the Ctrie, then the operation succeeded.
If the `newChild` node is not in the Ctrie, then it was either never placed in the Ctrie,
or it was already removed by another operation.
We disambiguate between the two by reading the `newChild.status` field --
recall that it had to be set to `Idle` **before** the `parent.status` was set to `Idle` in the `(this, this)` case.

The `Mutate` status:

    class Mutate(parent, child, newChild, index) extends Status {
      @tailrec def complete() {
        parentStatus = READ(parent.status)
        childStatus = READ(child.status)
        (parentStatus, childStatus) match {
          case (this, Idle) =>
            CAS(child.status, Idle, this)
            // this CAS admits a possibility of a race
            // we need to check if the update is still possible
            // in the (this, this) case
            complete()
          case (this, this) =>
            CAS(parent(index), child, newChild) // idempotent
            CAS(newChild.status, this, Idle) // idempotent
            CAS(parent.status, this, Idle) // idempotent
            complete()
          case (this, status) =>
            currChild = READ(parent(index))
            if (child ne currChild) {
              CAS(parent.status, this, Idle) // all hope is lost
              false
            } else {
              status.complete() // help lower
              complete()
            }
          case (_, _) =>
            // mutation has already completed -- see if the child is committed
            atIndex = READ(parent(index))
            newChildStatus = READ(newChild.status)
            val success = {
              if (atIndex ne newChild) newChildStatus ne this
              else true
            }
            if (success) true
            else {
              childStatus.complete()
              false
            }
        }
      }
    
    }

Note: as described here, the algorithm does recursive helping,
but we can easily change this with an extra parameter to `complete`,
if we notice that recursive helping is problematic.


## Snapshots

We now add the snapshot operation.

The `snapshot` operation replaces the `Gen` object in the root of the Ctrie.
The trie is then rebuilt lazily the first time some thread descends a path in the Ctrie.

    @tailrec def snapshot(): Root = {
      gen = READ(root.gen)
      status = READ(root.status)
      status match {
        case Idle =>
          snap = new Snap(gen, new Gen, new Gen)
          if (CAS(root.status, Idle, snap)) {
            snap.complete()
            READ(snap.frozen) // read the frozen node (which was written by the quickest helping thread)
          } else {
            snapshot()
          }
        case _ =>
          status.complete()
          snapshot()
      }
    }


    class Snap extends Status {
      def complete(): Boolean = {
        rootNode = READ(root.child) // note - cannot change during the snap
        frozen = new Root(rootNode, newSnapGen)
        CAS(snap.frozen, null, frozen) // set once by the quickest thread - idempotent
        CAS(root.gen, gen, newTrieGen) // can only succeed once - idempotent
        CAS(root.status, snap, Idle) // the snap can only complete once - idempotent
        true
      }
    }


Whenever we read a child at some position,
we need to replace the `READ` statement with the `READ_AT` method.
The `READ_AT` method checks if the node is at the proper generation,
and updates it if necessary, as it goes down the Ctrie.
This ensures that the shallow Ctrie is rebuilt lazily.


    // here one method with a pattern match for simplicity
    // in the implementation this should be 2 methods READ_ROOT and READ_AT
    def READ_AT(n, pos, gen): Node = {
        node = READ(n.child(pos)) // which means: n.child or n.array(pos)
        if (node.gen == gen) node
        else {
          refreshedNode = refreshed(node, gen)
          nstatus = new Mutate(n, node, refreshedNode, pos)
          if (CAS(n.status, Idle, nstatus)) {
            if (!nstatus.complete()) READ_AT(n, pos, gen)
          } else {
            ostatus = READ(n.status)
            ostatus.complete()
            READ_AT(n, pos, gen)
          }
        }
      }
    }


The top-level `insert` method and the recursive `insert` method are almost the same,
the only change is from `READ` to `READ_AT`, and the extra `gen` parameter in the recursive case.

    def insert(root, k, v) {
      hash = hashCode(k)
      gen = READ(root.gen)
      node = READ_AT(root, 0, gen)
      node match {
        case CNode if !node.freeAt(hash, 0) =>
          insert(root, gen, node, k, v, hash, 0)
        case _ =>
          nstatus = new Mutate(root, node, newNode(node, k, v), 0)
          if (CAS(root.status, Idle, nstatus)) {
            if (!nstatus.complete()) insert(k, v)
          } else {
            ostatus = READ(root.status)
            ostatus.complete()
            insert(root, k, v)
          }
      }
    }


    def insert(root, gen, cnode, k, v, hash, level) {
      pos = calcPos(hash, level, cnode.bitmap)
      node = READ_AT(cnode, pos, gen)
      node match {
        case CNode if !node.freeAt(hash, level + 5) =>
          insert(root, gen, node, k, v, hash, level + 5)
        case _ =>
          nstatus = new Mutate(cnode, node, newNode(node, k, v), pos)
          if (CAS(cnode.status, Idle, nstatus)) {
            if (!nstatus.complete()) insert(root, gen, cnode, k, v, hash, level)
          } else {
            ostatus = READ(cnode.status)
            ostatus.complete()
            insert(root, k, v)
          }
      }
    }




