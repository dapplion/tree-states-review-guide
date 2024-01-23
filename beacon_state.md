# Updated BeaconState type

## superstruct / metastruct

Milhouse does not support Containers. So the logic to hash, rebase, etc is handled in the consensus/types crate with the help of [`metastruct`](https://github.com/sigp/metastruct). The metastruct macro outputs macros to iterate ovand map over struct fields. For example:

```rust
#[superstruct(
    specific_variant_attributes(
        Deneb(metastruct(
            mappings(
                map_beacon_state_deneb_fields(),
            )
        )
    )
)]
```

superstruct 0.7 introduces the `specific_variant_attributes` feature, which transparently applies the macro to a single variant.

```rust
#[metastruct(
    mappings(
        map_beacon_state_deneb_fields(),
    )
)]
struct BeaconStateDeneb {}
```

Then `metastruct` outputs a macro that takes a `BeaconStateDeneb` as first argument and a clousure as second. It can be used in `compute_merkle_proof` to collect the leaves of each state variant.

```rust
BeaconState::Deneb(state) => {
    map_beacon_state_deneb_fields!(state, |_, field| {
        leaves.push(field.tree_hash_root());
    });
}
```

### Other macros

```rust
mappings(
    map_beacon_state_deneb_tree_list_fields(mutable, fallible, groups(tree_lists)),
),
```

`mutable` = `{ ref mut }` and `fallible` = `{ f()? }`. `groups(tree_lists)` **TODO**.

```rust
bimappings(bimap_beacon_state_deneb_tree_list_fields(
    other_type = "BeaconStateDeneb",
    self_mutable,
    fallible,
    groups(tree_lists)
)),
```

Creates a map function with itself (same type) that is mutable, fallible and only includes tree lists. Used in the rebasing operation to de-duplicate nodes on each list.

```rust
(Self::Capella(self_inner), Self::Capella(base_inner)) => {
    bimap_beacon_state_capella_tree_list_fields!(self_inner, base_inner,
        |_, self_field, base_field| { self_field.rebase_on(base_field) }
    );
}
```


## Validator mutable

- **:question:Q** Why do Validator fields become functions?
  - A: Put the pubkey inside an Arc and the rest of fields in [`ValidatorMutable`](https://github.com/sigp/lighthouse/blob/6262be72199d0cf81a8701076b8434c9914e211a/consensus/types/src/validator.rs#L22-L25). *TODO* I guess to dedup memory now that there are many more states in memory?

**TODO(review)** `Validator::could_be_eligible_for_activation_at` correctness and similar methods

- **:question:Q** Where is `ExecutionPayloadHeaderRefMut` needed?
  - A: Type safe wrapper to set the new payload header to the state, on `process_execution_payload`


## CompactBeaconState

Part of the pre-hdiff database idea, to de-dupe the fields like pubkey and withdrawal creds. Now it is used only when storing full states in the hot DB. Makes states 35% smaller in uncompressed serialized SSZ size.


## New caches consumer

- **TODO(review)** Consumer code for new caches for epoch transition single pass

- **:question:Q**: What are the consequences of not calling `state.build_caches`? And the costs of doing it? Is it free to re-call? This is called here pre-emptively, look if there are more places where it should be called https://github.com/sigp/lighthouse/pull/3206/files#diff-f1f4dab1ee31a59b5180e5d1baa0cd5dec78c10a03315816a8e20dc43dc227a8R72