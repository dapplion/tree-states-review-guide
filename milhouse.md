# Milhouse

Inspired by [protolambda/remerkleable](https://github.com/protolambda/remerkleable), implements the SSZ but with persistent binary merkle trees.

Implements Vector and List only. Containers are implemented with regular structs and annonated with [`derive(TreeHash)`](https://github.com/sigp/tree_hash/blob/93d0e3ff58fb174f84dea8d2b5374787b7a4c85a/tree_hash_derive/src/lib.rs#L161-L170). Milhouse Vectors and Lists wrap an [`Interface`](https://github.com/sigp/milhouse/blob/ccaed7bb2902d28d85f7ddeec4999b65473ad4a8/src/interface.rs#L36) which holds:
- A backing tree, as a MutList
- Pending updates, as an UpdateMap

Any mutation is inserted into the UpdateMap as a pending value first, by value index (packing happens latter).

- `get_mut`: Reads from pending updates first, else reads from backing tree inserting a clone into the pending updates and returning a reference
- `push`: Inserts directly into pending updates

`Interface::apply_updates` takes the pending updates and applies them to the backing tree. In the case of a list it calls `ListInner::update`, then calls `Tree::with_updated_leaves` which has most of the logic.

### Update leaves

Example simple tree, `*` indicates a pending update at that or below that index.

```
        O Node
      /   \
     /     \
    O Node  O *Zero(1)  
  /   \
 /     \
O Leaf  O *Leaf
```

`Tree::with_updated_leaves` sequence:
- (0)   self == Node, left_prefix = 0, right_prefix = 2?. Check if there are updates on the left branch, and right branch by doing a 1 item iteration of the BTreeMap.
- (00)  self == Node, left_prefix = 0, right_prefix = 1. Left branch has no updates, right yes
- (001) self == Leaf, call `Tree::leaf_with_hash` with index and value
- (01)  self == Zero, Replace zero node at depth with two zero nodes at depth - 1.

### Rebasing

When loading a beacon state from disk, it's likely that most of the tree is shared with a recent state. Rebasing computes a "diff" of the new state against an existing tree and structurally shares as many nodes as possible between them.

Recursively split the tree and compare branches by cached hash if available else by leaf node value(s).

_TODO(lion): mid depth review, missing big logic blocks_