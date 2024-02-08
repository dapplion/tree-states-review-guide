# Milhouse

Inspired by [protolambda/remerkleable](https://github.com/protolambda/remerkleable), implements the SSZ but with persistent binary merkle trees.

Implements Vector and List only. Containers are implemented with regular structs and annonated with [`derive(TreeHash)`](https://github.com/sigp/tree_hash/blob/93d0e3ff58fb174f84dea8d2b5374787b7a4c85a/tree_hash_derive/src/lib.rs#L161-L170).

This library is being fuzzed, and it's probably mostly correct. The logic is not particularly complicated except for some highlights below and its caching layer, recommend to pay most attention to that. See example of a recent bug: [Vector to List conversion throws away pending changes #20](https://github.com/sigp/milhouse/issues/20).

## Overview

Milhouse Vectors and Lists wrap an 
- A backing tree, as a MutList
- Pending updates, as an UpdateMap

Quick overview of Milhouse's data-structures: both `Vector` and `List` wrap an [`Interface`](https://github.com/sigp/milhouse/blob/ccaed7bb2902d28d85f7ddeec4999b65473ad4a8/src/interface.rs#L36) and simply call into the interface and expose a vec-ish API.

```rs
pub struct Vector<T: Value, N: Unsigned, U: UpdateMap<T> = MaxMap<VecMap<T>>> {
    interface: Interface<T, VectorInner<T, N>, U>,
}
```

Wrapper around the `*Inner` struct plus a "cache" layer of pending updates. Any mutation is inserted into the UpdateMap as a pending value first, by value index (packing happens latter). Reads hit the cache layer first:
- `get_mut`: Reads from pending updates first, else reads from backing tree inserting a clone into the pending updates and returning a reference
- `push`: Inserts directly into pending updates

`Interface::apply_updates` takes the pending updates and applies them to the backing tree. In the case of a list it calls `*Inner::update`, then calls `Tree::with_updated_leaves` which has most of the logic.

```rs
pub struct Interface<T: Value, B: MutList<T>, U: UpdateMap<T>> {
    backing: B, // Contains all the commited data 
    updates: U,
}
```

`VectorInner` implements `MutList` with specific vector logic, like not allowing push.

```rs
pub struct VectorInner<T: Value, N: Unsigned> {
    tree: Arc<Tree<T>>,
    ...
}
```

Tree is a recursive data structure, caching all intermediary hashes and holding data only at the leaves.

### `Tree::with_updated_leaves`

Recursive function to insert a set of leaf nodes into the tree while minimizing tree traversals.

Example sequence flow for a simple tree, `*` indicates a pending update at that or below that index.

```
           (0) Node
          /   \
         /     \
     (00) Node  (01) *Zero(1)  
  /        \
 /          \
(000) Leaf  (001) *Leaf
```

`Tree::with_updated_leaves` sequence:
- (0)   self == Node, left_prefix = 0, right_prefix = 2?. Check if there are updates on the left branch, and right branch by doing a 1 item iteration of the BTreeMap.
- (00)  self == Node, left_prefix = 0, right_prefix = 1. Left branch has no updates, right yes
- (001) self == Leaf, call `Tree::leaf_with_hash` with index and value
- (01)  self == Zero, Replace zero node at depth with two zero nodes at depth - 1.

### `Tree::rebase_on`

When loading a beacon state from disk, it's likely that most of the tree is shared with a recent state. Rebasing computes a "diff" of the new state against an existing tree and structurally shares as many nodes as possible between them.

Recursively split the tree and compare branches by cached hash if available else by leaf node value(s). Somewhat spicy logic, worth a deeper review.

