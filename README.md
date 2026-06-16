# Nested Variable Maps for EU5

Nested dictionaries, ordered lists, unordered sets, and object-oriented data structures for EU5. Built on variable maps, backed by dead characters as storage nodes. No memory ceiling.

Store a reference on any scope with `set_variable`. Access via dot chaining: `c:FRA.var:my_data.nvm_get_value = { key = goods:wheat }`.

See [TUTORIAL.md](TUTORIAL.md) for a guided walkthrough building a horse racing minigame with OOP patterns.

---

## Quick Start

```
# Before first use (idempotent, safe to call from multiple mods)
nvm_init_pool = yes

# Allocate a node, store it on France
nvm_new_map = yes
c:FRA = { set_variable = { name = my_data value = scope:nvm_map } }

# Write and read a value
c:FRA.var:my_data.nvm_set_value = { key = goods:wheat value = 42 }
c:FRA.var:my_data.nvm_get_value = { key = goods:wheat }
# local_var:nvm_result = 42

# Save results before the next NVM call overwrites them
set_local_variable = { name = my_wheat value = local_var:nvm_result }
```

---

## API Reference

All operations are scripted effects called on a node scope (e.g., `scope:my_dict.nvm_get_value = { ... }`). NVM operations work on any scope, not just pool-allocated nodes; you can stamp values, types, and references directly on characters, countries, or any game entity.

### Lifecycle

| Operation                | Scope | Result                                            |
|--------------------------|-------|---------------------------------------------------|
| `nvm_init_pool = yes`    | any   | Initializes pool and type registry. Idempotent.   |
| `nvm_new_map = yes`      | any   | `scope:nvm_map`, `local_var:nvm_found`            |
| `nvm_delete_map = yes`   | node  | Cleans parent refs, wipes data, returns to pool   |
| `nvm_delete_tree = yes`  | node  | Recursive delete of entire subtree                |
| `nvm_is_valid = yes`     | any   | `local_var:nvm_found` (is this a tracked pool node?) |

### Values

| Operation                                                                      | Result               |
|--------------------------------------------------------------------------------|-----------------------|
| `nvm_set_value = { key = X value = Y }`                                       |                       |
| `nvm_get_value = { key = X }`                                                 | `local_var:nvm_result` |
| `nvm_get_value_or = { key = X default = Y }`                                  | `local_var:nvm_result` |
| `nvm_has_value = { key = X }`                                                 | `local_var:nvm_found` |
| `nvm_remove_value = { key = X }`                                              |                       |
| `nvm_clear_values = yes`                                                       |                       |
| `nvm_set_value_if_absent = { key = X value = Y }`                             | `local_var:nvm_found` |
| `nvm_change_value = { key = X operation = add\|subtract\|multiply\|divide value = Y }` |               |
| `nvm_swap_values = { key_a = X key_b = Y }`                                   |                       |

Arithmetic on missing keys starts from 0. Values can be bare literals or variable references.

### Sub-dictionaries

| Operation                          | Result                                     |
|------------------------------------|--------------------------------------------|
| `nvm_get_or_set_map = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_get_map = { key = X }`       | `scope:nvm_cursor`, `local_var:nvm_found`  |
| `nvm_has_map = { key = X }`       | `local_var:nvm_found`                      |
| `nvm_set_map = { key = X child = Y }` | Links existing node `Y` as child at `key` |
| `nvm_remove_map = { key = X }`    |                                            |
| `nvm_clear_maps = yes`            |                                            |
| `nvm_move_map = { key = X target = Y target_key = Z }` | Unlinks child at `key` from this node, relinks it under `target` at `target_key` |

### Scope References

| Operation                                  | Result                                     |
|--------------------------------------------|--------------------------------------------|
| `nvm_set_scope = { key = X value = Y }`   |                                            |
| `nvm_get_scope = { key = X }`             | `scope:nvm_scope`, `local_var:nvm_found`   |
| `nvm_has_scope = { key = X }`             | `local_var:nvm_found`                      |
| `nvm_remove_scope = { key = X }`          |                                            |
| `nvm_clear_scopes = yes`                  |                                            |

### Lists

| Operation                              | Result                                              |
|----------------------------------------|------------------------------------------------------|
| `nvm_get_or_set_list = { key = X }`   | `scope:nvm_cursor`, `local_var:nvm_found`            |
| `nvm_get_list = { key = X }`          | `scope:nvm_cursor`, `local_var:nvm_found`            |
| `nvm_has_list = { key = X }`          | `local_var:nvm_found`                                |
| `nvm_remove_list = { key = X }`       |                                                      |
| `nvm_clear_lists = yes`               |                                                      |
| `nvm_list_add = { value = X }`        |                                                      |
| `nvm_list_get = { index = N }`        | `scope:nvm_scope`, `local_var:nvm_found`             |
| `nvm_list_remove = { index = N }`     | Swaps last item into gap                             |
| `nvm_list_find = { value = X }`       | `local_var:nvm_result` (index), `local_var:nvm_found` |
| `nvm_list_has = { value = X }`        | `local_var:nvm_found`                                |
| `nvm_list_remove_value = { value = X }` | Find + swap-remove                                 |
| `nvm_list_pop = yes`                  | `scope:nvm_scope`, `local_var:nvm_found`. Removes last item. |
| `nvm_list_clear = yes`                |                                                      |

Lists are indexed collections of scope references. Use them as ordered arrays (append-only, stable indices) or as unordered bags (add/remove freely, indices shift on removal). The same structure serves both patterns depending on how you use it.

List size: `scope:my_list.var:nvm_list_size`

**Index instability:** `nvm_list_remove` swaps the last element into the removed slot. Do not cache indices across remove operations. `nvm_list_pop` removes from the end and preserves order.

### Unordered Sets

Sets store items as keys with a numeric value (default 0). The value slot can be used as a weight for weighted graph traversal.

| Operation                                          | Result                        |
|----------------------------------------------------|-------------------------------|
| `nvm_get_or_set_set = { key = X }`                | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_get_set = { key = X }`                       | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_set_add = { value = X }`                     | No-op on duplicates           |
| `nvm_set_add_weighted = { value = X weight = W }` | Inserts or updates weight     |
| `nvm_set_has = { value = X }`                     | `local_var:nvm_found`         |
| `nvm_set_get_weight = { value = X }`              | `local_var:nvm_result`        |
| `nvm_set_remove = { value = X }`                  |                               |
| `nvm_set_clear = yes`                             |                               |

### Type Tagging

| Operation                      | Result                                    |
|--------------------------------|-------------------------------------------|
| `nvm_set_type = { type = X }` | Registers in global type registry         |
| `nvm_get_type = yes`          | `scope:nvm_scope`, `local_var:nvm_found`  |
| `nvm_is_type = { type = X }`  | `local_var:nvm_found`                     |

Types use `flag:` scopes as identifiers. Auto-deregistered on delete/recycle.

### Multi-key Check

| Operation                      | Result                                                                    |
|--------------------------------|---------------------------------------------------------------------------|
| `nvm_check_key = { key = X }` | `local_var:nvm_has_map`, `local_var:nvm_has_value`, `local_var:nvm_has_scope`, `local_var:nvm_has_list` |

### Iteration

| Operation                                          | Scope | Callback receives                                                     |
|----------------------------------------------------|-------|------------------------------------------------------------------------|
| `nvm_for_each_value = { callback = X }`            | dict  | `scope:nvm_iter_key`, `local_var:nvm_iter_value`                      |
| `nvm_for_each_scope = { callback = X }`            | dict  | `scope:nvm_iter_key`, `scope:nvm_iter_value`                          |
| `nvm_for_each_child = { callback = X }`            | dict  | `scope:nvm_iter_key`, `scope:nvm_iter_child`                          |
| `nvm_for_each_set_item = { callback = X }`         | set   | `scope:nvm_iter_item`, `local_var:nvm_iter_weight`                    |
| `nvm_for_each_list_item = { callback = X }`        | list  | `scope:nvm_iter_item`, `local_var:nvm_iter_index`                     |
| `nvm_for_each_of_type = { type = X callback = Y }` | any   | `scope:nvm_iter_item` (iterates all live nodes of a type)             |

Define a named scripted effect as your callback:

```
my_handler = {
    # scope:nvm_iter_key = goods:wheat, goods:iron, etc.
    # local_var:nvm_iter_value = the numeric value
}
scope:my_dict.nvm_for_each_value = { callback = my_handler }
```

For manual iteration, use `every_key_in_variable_map` directly on the dictionary scope.

### Navigation

Each dictionary stores `var:nvm_accessed_from` pointing to the dictionary it was last navigated from.

---

## Best Practices

- **Store references as variables, access via dot chain.** Works from any context.
- **Use `nvm_get_or_set_map` for write paths.** Creates on first access, navigates on repeat.
- **Use `nvm_get_map` for read paths.** Doesn't create empty nodes on missing keys.
- **Save `scope:nvm_cursor` immediately.** Every navigation call overwrites it.
- **Save `local_var:nvm_result` immediately.** The next NVM call overwrites it.
- **Clean up with `nvm_delete_tree`.** Orphaned nodes waste pool space permanently.
- **Use `flag:` scopes as keys for OOP field names.** Unique, readable, content-independent.

## Things to Avoid

- **Bypassing the API.** Writing to `nvm_children`, `nvm_values`, etc. directly skips parent tracking and error handling.
- **Relying on `local_var:nvm_result` across multiple NVM calls without saving.** Each call overwrites the previous result.
- **Caching list indices across removals.** Swap-from-end makes indices unstable.

## Reserved Names

All NVM-internal names use the `nvm_` prefix. Avoid this prefix in your own variable and scope names.

On pool nodes: `nvm_children`, `nvm_values`, `nvm_scopes`, `nvm_lists`, `nvm_parents`, `nvm_accessed_from`, `nvm_is_list`, `nvm_list_size`, `nvm_type`

Scopes (overwritten each call): `nvm_map`, `nvm_cursor`, `nvm_scope`, `nvm_op_key`, `nvm_op_key_a`, `nvm_op_key_b`, `nvm_op_parent`, `nvm_deleting`, `nvm_del_parent`, `nvm_del_iter_key`, `nvm_del_remove_key`, `nvm_del_child_key`, `nvm_move_child`, `nvm_typed_node`, `nvm_old_type`, `nvm_releasing_node`, `nvm_release_type`, `nvm_iter_dict`, `nvm_iter_key`, `nvm_iter_value`, `nvm_iter_child`, `nvm_iter_item`

Local variables (overwritten each call): `nvm_result`, `nvm_found`, `nvm_has_map`, `nvm_has_value`, `nvm_has_scope`, `nvm_has_list`, `nvm_nav`, `nvm_current`, `nvm_i`, `nvm_last`, `nvm_target`, `nvm_op_idx`, `nvm_val_a`, `nvm_val_b`, `nvm_iter_value`, `nvm_iter_index`, `nvm_iter_weight`

Globals: `nvm_pool_size`, `nvm_pool_free`, `nvm_pool_active`, `nvm_type_registry`, `nvm_keys_to_remove`, `nvm_del_nodes`

## Tests

All test output goes to `logs/error.log`. Grep for the prefix to filter:

| Prefix    | Content                      |
|-----------|------------------------------|
| `NVM`     | Core API tests + stress tests |
| `NVM_TUT` | Tutorial horse racing         |
| `NVM_PAR` | Parallel safety tests         |

Enable tests by uncommenting the `on_game_start` block in `in_game/common/on_action/nvm_test.txt`.
