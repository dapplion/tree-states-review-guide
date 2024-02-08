# Updated BeaconState type

This page details the new meta-programing added to the `BeaconState` struct. The new caches are part of the epoch single pass page.

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

Before tree-states, the entire validators list is cloned. With tree-states we can clone only on write. Since pubkeys are immutable, we can de-duplicate them across validator record mutations. So `Arc` the pubkey and put the rest of mutable fields in [`ValidatorMutable`](https://github.com/sigp/lighthouse/blob/6262be72199d0cf81a8701076b8434c9914e211a/consensus/types/src/validator.rs#L22-L25).

- **:question:Q** Where is `ExecutionPayloadHeaderRefMut` needed?
  - A: Type safe wrapper to set the new payload header to the state, on `process_execution_payload`


## CompactBeaconState

Part of the pre-hdiff database idea, to de-dupe the fields like pubkey and withdrawal creds. Now it is used only when storing full states in the hot DB. Makes states 35% smaller in uncompressed serialized SSZ size.


