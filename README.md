# Tree-states review guide

This doc is meant to assist reviewers with [Upgrade in-memory and on-disk state representation with tree states #3206](https://github.com/sigp/lighthouse/pull/3206)

Overview of big items to review (in suggested order):

- **[New sigp/milhouse lib](./milhouse.md)**: underlying lib for in-memory representation
- **[Updated BeaconState](./beaconstate.md)**: new in-memory representation
- **[Epoch processing single pass](./epoch_singlepass.md)**: Epoch processing with a single iteration over big lists
- **[Updated BeaconChain](#./beacon_chain.md)**: Misc changes, updated caches and consumers of everything else
- **[Persisted HDiff](./persisted_hdiff.md)**: Persisted Hierarchical diff-states (HDiff) + blocks persisted in cold store indexed by slot


# Intro / motivation

Tree-states represents beacon chain state is merkleized representation so that multiple states can share the same data. Beacon chain mutation and access patterns make this approach efficient the vast majority of the data is the validators array, which is rarely mutated.

_TODO(lion): add more_

