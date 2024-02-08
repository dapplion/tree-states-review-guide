# Updated BeaconChain

_^ Look at the other sections first, start with milhouse_

`BeaconChain` diff is a less straightforward mix of moving the caches around + consuming the caches differently + HDiff changes.

```diff
struct BeaconChain {
-    /// A cache dedicated to block processing.
-    pub(crate) snapshot_cache: TimeoutRwLock<SnapshotCache<T::EthSpec>>,

     /// Caches a map of `validator_index -> validator_pubkey`.
-    pub(crate) validator_pubkey_cache: TimeoutRwLock<ValidatorPubkeyCache<T>>,
+    pub(crate) validator_pubkey_cache: Arc<RwLock<ValidatorPubkeyCache<T>>>,

+    /// A cache used to de-duplicate HTTP state requests.
+    pub parallel_state_cache: Arc<RwLock<ParallelStateCache<T::EthSpec>>>,

-    /// State with complete tree hash cache, ready for block production.
-    pub block_production_state: Arc<Mutex<Option<(Hash256, BlockProductionPreState<T::EthSpec>)>>>,
}
```

- **:question:Q**: Why is `snapshot_cache` removed?
  - A: It's replaced by the state cache inside the store. Current snapshot cache is suboptimal because it has to use heuristics to clone the state (slow) and if you get a cache miss you load from disk (very slow)
- **:question:Q**: Why is the `validator_pubkey_cache` no longer a timeout rwlock?
  - A: Was used in the past to notice deadlocks, but they can stop liveness. If the node is slow it will timeout doing work, then try to do the work again and loop. Michael used the opportunity to drop it from here. TODO: consider runing lockbud in CI to detect locks. Instructions in ./Code dir of the lockbud repo.
- **:question:Q**: Why is `block_production_state` removed?
  - A: It's a recent optimization [#4925](https://github.com/sigp/lighthouse/pull/4925) to eagerly clone the state since it takes ~500ms in mainnet. No longer an issue with tree-states since cloning is very cheap.
- **:question:Q**: Why is `parallel_state_cache` added?
  - A: Added to the BeaconChain to de-duplicate work from HTTP requests that need states. Used in `beacon_node/http_api/src/state_id.rs`. Parallel state cache is probably broken so should not push to add to unstable. It doesn't have any pruning logic, and it completely duplicates functionality from the committee cache. A better solution is a layer in front of the state_cache in the store, which would cover both use-cases. Only viable with tree-states or it too much RAM. Issue about this: https://github.com/sigp/lighthouse/issues/5112

**Misc changes (non-structured):**

Update `BeaconChain::import_block` and `BeaconChain::load_state_for_block_production` to not use the `snapshot_cache` or `block_production_state`.
- **:question:Q**: why it not necessary to fallback on them compared to before? The store has better guarantees?

On `get_expected_withdrawals` no longer fallback on `snapshot_cache`
- **:question:Q**: what are the consequences of a miss here? Why is it okay to not fallback in tree-states?
  - A: snapshot_cache is gone, going to the store hits the in-memory cache and fallbacks to disk (that's the fallback)
- **:question:Q**: in `BeaconChain::with_committee_cache` now it errors for shufflings that are too old, why?
- **:question:Q**: in `BeaconChain::with_committee_cache` the comment about `get_inconsistent_state_for_attestation_verification_only` and unreliable `state.state_roots` is removed. Why does it not apply anymore?
  - A: It should apply, but there's an intention to remove since Michael has found the function to not be correct. Not computing block_roots can affect attestation rewards, which may change effective balances.
  - A: EDIT: It does not apply, now replay_blocks fill state_roots with correct values always from the iterator to skip computing the root

ValidatorPubkeyCache moved to the store.
- **:question:Q**: Why?
  - A: We have compact beacon store, so the store is aware of the pubkey cache

Block verification changes
- **:question:Q**: In `ExecutionPendingBlock::from_signature_verified_components` it stores a state of every slot of the advance. How can  "marking it as temporary" in the tree-states implementation?
  - A: The temporary flag is _probably_ no longer necessary, replaced by this logic [`hot_cold_store.rs#L1353](https://github.com/sigp/lighthouse/blob/6262be72199d0cf81a8701076b8434c9914e211a/beacon_node/store/src/hot_cold_store.rs#L1353-L1367). Purpose of temp column was to avoid surfacing intermediate states until their block was committed. It was partly necessary because we also didn't have the ability to garbage collect states without an index, whereas now in tree-states we just iterate the states column.
- **:question:Q**: In `block_verification.rs::load_parent` if `block.slot() != state.slot()` you only log a warn "Parent state is not advanced". How can that happen and why not hard error?
  - A: Log may happen if the state advance is not used, i.e. a re-org that invalidates the current advance.

`SszHeadTracker`
- **:question:Q**: `SszHeadTracker` fields became public, intentional or typo?
  - A: maybe, review

`beacon_node/beacon_chain/src/historical_blocks.rs`, updated to store historical blocks linearly by slot.

`beacon_chain/src/migrate.rs`, **TODO(review)**

ShufflingCache code has been generalized (deleted from `shuffling_cache.rs`) into the PromiseCache crate at `common/promise_cache`, **TODO(review)**.  

- **:question:Q**: Why `as_store_bytes` became a fallible operation?
  - A: because of the zstd compression on state diffs and blocks. But it is no longer used in favor of `blinded_block_as_cold_kv_store_ops`. So it is no necessary to be fallible now.

In `state_advance_timer.rs` no longer necessary to pre-emptively clone the state, tree-states now.

- **:question:Q**: `test_utils.rs::produce_unaggregated_attestation_for_block` why is complete_state_advance used now? Are state roots required to be accurate?
- **:question:Q**: How is `state-cache-size` default to 128 selected and its implications in high forking scenarios

In `consensus/fork_choice/src/fork_choice.rs` use the BeaconState's own progressive balance cache. Drop all code managing the cache on its own.
