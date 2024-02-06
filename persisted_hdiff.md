# Persisted Hierarchical diff-states

Anτon Nashαtyrεv write-up about Teku's [Incremental diff SSZ storage](https://hackmd.io/G82DNSdvR5Osw2kg565lBA). Jimmy's detailed diagram of Lighthouse's particular scheme on the [SigP Notion / Tree States](https://www.notion.so/sigp/Storage-Tree-States-aa64cc30be084ce28e6f67f737243360).

The store computes two diffs, one for the entire beacon state and another one for balances. HD stands for Hierarchical diff.

```rust
impl HDiff {
    pub fn compute(source: &HDiffBuffer, target: &HDiffBuffer) -> Result<Self, Error> {
        Ok(Self {
            state_diff: BytesDiff::compute(&source.state, &target.state)?,
            balances_diff: CompressedU64Diff::compute(&source.balances, &target.balances)?,
        })
    }
}
```

The balances of the state are replaced with a list of zero length in the state.

- **:question:Q**: Why is there a separate balances diff?
  - A: balances are the highest entropy, and using the knowledge that they're u64s makes the diff massively smaller. Anton N from TXRX discovered this. [This spreadsheet](https://docs.google.com/spreadsheets/d/1EU4-CPvmoRTDEb81TlpYNsgldROrrbJbkSDmkoGCo1A/edit#gid=0) has a comparison of the binary diff algorithms. With 550k mainnet vals and a 76MB state a state diff is 91KB and and takes 100-200ms to compute/apply.

```rust
impl HDiffBuffer {
    pub fn from_state<E: EthSpec>(mut beacon_state: BeaconState<E>) -> Self {
        let balances_list = std::mem::take(beacon_state.balances_mut());
        let state = beacon_state.as_ssz_bytes();
        let balances = balances_list.to_vec();
        HDiffBuffer { state, balances }
    }
}
```

**Full state diff**

Leverages `xdelta` open-source binary diff, delta/differential compression tools, VCDIFF/RFC 3284 delta compression. Specifically Rust bindings of [`xdelta3`](https://docs.rs/xdelta3/latest/xdelta3/).

`xdelta` takes a source binary blob and applies a single patch.

```rust
impl BytesDiff {
    pub fn compute_xdelta(source_bytes: &[u8], target_bytes: &[u8]) -> Result<Self, Error> {
        let bytes =
            xdelta3::encode(target_bytes, source_bytes).ok_or(Error::UnableToComputeDiff)?;
        Ok(Self { bytes })
    }

    pub fn apply_xdelta(&self, source: &[u8], target: &mut Vec<u8>) -> Result<(), Error> {
        *target = xdelta3::decode(&self.bytes, source).ok_or(Error::UnableToApplyDiff)?;
        Ok(())
    }
}
```

- Note, `xdelta3-rs` is not mantained and Michael mantains a fork that could be brought under sigp org.

**Balances diff**

Computes the increment for each balance item from source to target. Then compresses the resulting concatenated u64 with `zstd` compression library. *(My asumption)* since most balances do not change at once their delta is 0, so diff + compression results in big savings.


### When is HDiff computed?

1. `HotColdDB::store_cold_state_as_diff(from_slot: Slot)`, the base buffer is decided by `from_slot`.
2. `HotColdDB::store_hot_state`, the base buffer is decided by `HotColdDB::state_diff_slot`

```rust
fn state_diff_slot(slot: Slot) -> Slot {
    // Only if slot % slots_per_epoch == 0
    slot - config.epochs_per_state_diff * slots_per_epoch
}
```

**Questions**

- **:question:Q**: In `HotStateSummary::new`, `diff_base_slot` is resolved into a state root looking up the `state.state_roots`. Is `diff_base_slot` never older than 8192 slots? 
  - A: epochs-per-state-diff limits that and SHOULD be set up to 8192, FIX: put a bound, add verify_epochs_per_state_diff.
- **:question:Q**: From `HotColdDB::is_stored_as_full_state`, will it store every new state that has been finalized? What's the writing frequency in practice? The fact that this function is dynamic, when is it called respective to the split point?
  - A: Yes, every time the finalization migration runs. There's a flag epochs-per-migration to make it less frequent and save IO.
- **:question:Q**: Why does `HotColdDB::store_hot_state :1268` need to do a `self.get_hot_state()` call to immediately compute a base buffer from it? 
  - A: We never store the buffer to disk, so we need to compute the diff. TODO: Rename HDiff to Diff, misnomer. TODO 1282 bad error name 
- **:question:Q**: Is there a limit on how many diff layers a state needs to regenerate the base state?
  - A: No it will just keep applying diffs. Part of the complexity of the diff algorithm is that the diff frequency for the hot DB is adjustable dynamically. So you can restart with a different `--epochs-per-state-diff` and everything should just work
- **:question:Q**: What are the exponents from `HierarchyConfig`
  - A: Explained here https://github.com/sigp/lighthouse/pull/3206/files#diff-fdb1b9480ecf7dbd64bd5acbb641a040103fb96cc366ad700adf23a4e6aa9c32R602-R614
- **:question:Q**: How does the setting `hierarchy-exponents` interact and differ from `epoch-per-state-diff`? AFAIU the logic to decide to write a diff is in `HotColdDB::state_diff_slot` and only considers `epoch_per_state_diff`.
  - A: They don't interact, epochs-per-state-diff only for hot db states, hierarchy is for the freezer. In the hot is still diffs but rooted in the finalized states and diff against the prev state of 4 epochs against. TODO 1576 should be metrics. TODO: not rebase and persist every state on load hot state
- **:question:Q**: Given a specific exponents setting, what's the ratio of total diff layers to full states?
  - A: With default settings `[5,9,11,13,16,18,21]` the most frequent diff layer is stored every 2^5 slots and a full snapshot every 2^21 slots, so the ratio is 2^16 = 65536.
- **:question:Q**: `--hierarchy-exponents` help says "Cannot be changed after initialization". Is that correct, and if so, why?
  - A: In the hot database the epoch-per-state-diff is dynamic. In the freezer DB hierarchy exponents is NOT dynamic, changing it requires a full sync.
- **:question:Q**: Why has `partial_beacon_state` been deleted?
  - A: We don't need it anymore, 
- **:question:Q**: `forwards_iter.rs` changes are due to blocks being stores in freezer?
  - A: Not just where the blocks are stored, but we abolished the chunked business
- **:question:Q**: What was `PartialBeaconState` and rationale?
  - A: Long ring vectors like block roots, state roots, historical roots, randao mixes, are stored separately. With complicated logic to re-create the state latter. For the origins of the network with smaller validator set it has decent disk space savings. Replaced completely with binary diffs which achieve the same for all fields. 
- **:question:Q**: Was are compact states and rationale?
  - A: Store the validator pubkey in a separate bucket since it's append only list. Saves 48 MB per 1M indexes. Still in the tree-states branch to save disk on full disk writes. Potentially only used by the hot db, freezer _should_ write the full disk a BeaconState since it's written once a year.
- **:question:Q**: Freezer states with checkpoint sync?
  - A: Not happening, freezer full snapshots writes only happen when you do a full backfill and "frontfill" of states replay. Or when you hit a snaphost point (once a year)


### Migration from v12 to v20

Open PR ["Tree states database upgrade 🏗️ #5067"](https://github.com/sigp/lighthouse/pull/5067)

- Upgrade
- Downgrade available also?


## Updated store

```diff
/// Data related to blocks.
///
/// - Key: `Hash256` block root.
+/// - Value in hot DB: SSZ-encoded blinded block.
+/// - Value in cold DB: 8-byte slot of block.
BeaconBlock,
+/// Frozen beacon blocks.
+///
+/// - Key: 8-byte slot.
+/// - Value: ZSTD-compressed SSZ-encoded blinded block.
+BeaconBlockFrozen,
+/// For beacon state snapshots in the freezer DB.
+BeaconStateSnapshot,
+/// For compact `BeaconStateDiff`s in the freezer DB.
+BeaconStateDiff,
```

**TODO(review)**: 1,700 line diff in `hot_cold_store.rs` :smile:
**TODO(review)**: `state_cache.rs`, replaces the previous cache in the beacon chain.
