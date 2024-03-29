# Epoch processing single pass

A tree-states list/vector is much slower to iterate over a large amount of elements than a regular Vec. Lighthouse's epoch processing implementation iterates over lists of size N = `state.validators.len()` multiple times, unacceptably decreasing performance of the tree-states branch. Solution? only iterate each list of size N _once_. Enter single pass epoch processing.

- Description of the algorithm, with python code and informal proof: https://github.com/sigp/verified-consensus/blob/main/docs/description_and_informal_proof.md
- EthResearch post (summary of the above): https://ethresear.ch/t/formally-verified-optimised-epoch-processing/17359
- Main PR into `tree-states` "Single-pass epoch processing #4483": https://github.com/sigp/lighthouse/pull/4483

_**michaelsproul note**: Reviewers should look for discrepancies between the proof and the implementation, and issues in the proof itself_

The gist of the change is implementing epoch processing as:

```python
for (
    validator,
    balance,
    ...
) in zip(
    state.validators,
    state.balances,
    ...
):
    process_single_inactivity_update(...)
    process_single_reward_and_penalty(balance, ...)
    process_single_registry_update(validator, ...)
    process_single_slashing(balance, validator, ...)
    process_single_effective_balance_update(balance, validator...)
    ...
```

Before diving into the Rust code it's best to fimiliarize yourself with the [python spec for `process_epoch_single_pass`](https://github.com/sigp/verified-consensus/blob/main/docs/description_and_informal_proof.md#single-pass-epoch-processing-process_epoch_single_pass), it's good! The spec is being proofed formally so the chance of logic errors is low. Focus on the spec translation into Rust code.

## Caches

New caches appended to the beacon state struct (excluded from serialization ofc):

- `progressive_balances`: Balance sums for active validators & attestation participation flags.
- `activation_queue`: Cache of validators that *could* be activated soon.
- `exit_cache`: Cache of exits indexed by exit epoch.
- `base_reward_cache`: Cache of base rewards and effective balances.
- `num_active_validators`: Number of active validators in the current epoch


```rust
struct BeaconState {
    // ... caches only
    pub total_active_balance: Option<(Epoch, u64)>, // changes how it's computed
    pub committee_caches: [Arc<CommitteeCache>; CACHED_EPOCHS], // already present in unstable
    pub progressive_balances_cache: ProgressiveBalancesCache, // now tracks source,target,head balances
    pub pubkey_cache: PubkeyCache, // switch HashMap -> HashTrieMap
    pub exit_cache: ExitCache, // switch HashMap -> HashTrieMap
    pub slashings_cache: SlashingsCache, // new cache in tree-states
    pub epoch_cache: EpochCache, // new cache in tree-states
```

Since the caches are part of the state struct, they are set to default values on deserialization. Those values are incorrect, but all are guarded by checks of non-initialized. `total_active_balance` is an `Option` because 0 is a valid value, so the initialization check is cleaner with `None`.

- **:question:Q**: Why switch `pubkey_cache` and `exit_cache` from `HashMap` to `HashTrieMap`?
  - A: To have structural sharing, now there will be much more elements in memory
- **:question:Q**: What are the consequences of not calling `state.build_caches`? And the costs of doing it? Is it free to re-call? This is called here pre-emptively, look if there are more places where it should be called https://github.com/sigp/lighthouse/pull/3206/files#diff-f1f4dab1ee31a59b5180e5d1baa0cd5dec78c10a03315816a8e20dc43dc227a8R72
  - A: They should be free to call in loop, but not sure. With immutable state you can't build the cache, but with a mutable ref we always try to build pre-emptively. Now with tree-states there's a more top level wrapper `build_all_caches`, which builds caches that deal with state processing logic. All caches are structurally shared or cloneable.


### `total_active_balance`

On the single pass epoch processing `next_epoch_total_active_balance` is initialized to 0 and incremented on `process_single_effective_balance_update()`, then set into the state with `state.set_total_active_balance()`.

- **:question:Q**: Many codepaths consider `total_active_balance` could be `None`. In what cases do you expected that to happen?
  - A: See above, caches set to default value on load, None guards against them not being not initialized.
- **:question:Q**: Is total_active_balance recomputed every epoch, or progressively updated?
  - A: recomputed every epoch as part of the single pass
- **TODO**: Should add a metric for total iterations over the validators list


### `progressive_balances_cache`

Progressive balances cache tracks only target balance in stable. With tree-states it tracks source, target and head balances.
- **:question:Q**: Why is it necessary to track source,head now?
  - A: We use those total in process rewards and penalties. Justification and finality only cares about target, but now we need the rest for single pass rewards computation

PR diff extends existing interface to track the extra flag balances.


### `slashings_cache`

Contains the list of _all_ slashed validators. When initialized it iterates the entire validators list.

```rust
pub struct SlashingsCache {
    latest_block_slot: Option<Slot>,
    slashed_validators: HashTrieSet<usize>,
}
```

On attestation processing check against this cache instead of the state itself. Traversing the validators list for each attestation bit is not acceptable, so this cache prevents that with minimal memory cost.

```rust
let validator_slashed = state.slashings_cache().is_slashed(index);
```
_[process_operations.rs#L154)](https://github.com/sigp/lighthouse/blob/ae6620e59d3fc0f0928f4502a40f11f5027ddefa/consensus/state_processing/src/per_block_processing/process_operations.rs#L154)_


### `epoch_cache`

Cache of values which are uniquely determined at the start of an epoch.

- **:question:Q** Why is this cache necessary with tree-states?
  - A: `effective_balances` is necessary to speed up attestation processing, similar to the slashing cache
  - A: `base_rewards`: Just makes sense to cache, 6 or 7 arithmetics operations
  - A: `activation_queue`: Pre-computed list of validators that _could_ be eligible for activation.
  - A: `effective_balance_increment`: (@lion guess) to prevent having to reach for the `spec` object

```rust
struct EpochCache (Inner) {
    /// Unique identifier for this cache, which can be used to check its validity before use
    /// with any `BeaconState`.
    key: EpochCacheKey,
    /// Effective balance for every validator in this epoch.
    effective_balances: Vec<u64>,
    /// Base rewards for every effective balance increment (currently 0..32 ETH).
    /// Keyed by `effective_balance / effective_balance_increment`.
    base_rewards: Vec<u64>,
    /// Validator activation queue.
    activation_queue: ActivationQueue,
    /// Effective balance increment.
    effective_balance_increment: u64,
}
```


### `ActivationQueue`

```rust
pub struct ActivationQueue {
    /// Validators represented by `(activation_eligibility_epoch, index)` in sorted order.
    queue: BTreeSet<(Epoch, usize)>,
}
```

Speculative values added in `process_single_registry_update`, and read at the start of the single pass run checking the full eligibility criteria.

- **:question:Q**: Where is `queue` pruned?
  - A: Should be re-computed every epoch, **TODO**: try to find the codepath


### `ParticipationCache`

- **:question:Q**: Is this a leftover from unstable? It's not used in state transition code, but only indirectly in ef tests + rewards calculations.
  - A: Yes, it's a leftover, could be removed
- **:question:Q**: `initialize_progressive_balances_cache()` is never called with `Some(ParticipationCache)`, so on each epoch transition there's a full loop over all validators.
  - A: In `initialize_progressive_balances_cache()` the first condition will exit early most of the times, because the progressive balances cache is initialized.
- **TODO**: The rewards test does not test production code


### `PreEpochCache`

Used to collect active balances from validators. Its lifetime is just during `process_epoch_single_pass`, at the end of the function it's converted to `EpochCache`.


## Single pass fn

`process_epoch_single_pass` is called in `altair::process_epoch`. The original source of those steps (`altair/inactivity_updates.rs`, `altair/rewards_and_penalties.rs`, `effective_balance_updates.rs`, `slashings.rs`, `registry_updates.rs`) is kept for testing and non-performance critical calls (rewards calculation). They call into `process_epoch_single_pass` with configs to only execute that one step.

- **:question:Q**: Why is base not using single pass?
  - Phase0 attestation processing requires very different logic, not worth it to build a second single pass for phase0

**TODO**: Review

