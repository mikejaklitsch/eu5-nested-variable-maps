# Nested Variable Maps for EU5

Nested dictionaries, ordered lists, unordered sets, and object-oriented data structures for EU5. Built on variable maps, backed by dead characters as storage nodes. No fixed pool ceiling.

Store a reference on any scope with `set_variable`. Access via dot chain: `c:FRA.var:my_data.nvm_get_value = { key = goods:wheat }`.

See [TUTORIAL.md](TUTORIAL.md) for a progressive walkthrough from basic dictionaries through OOP patterns with a horse racing minigame.

---

## Quick Start

```
# Once at game start
nvm_init_pool = yes

# Allocate a node, store it on France
nvm_new_map = yes
c:FRA = { set_variable = { name = my_data value = scope:nvm_map } }

# Write and read a value
c:FRA.var:my_data.nvm_set_value = { key = goods:wheat value = 42 }
c:FRA.var:my_data.nvm_get_value = { key = goods:wheat }
# local_var:nvm_result = 42
```

---

## API Reference

### Lifecycle

| Operation | Result |
|-----------|--------|
| `nvm_init_pool = yes` | `global_var:nvm_pool_size` |
| `nvm_new_map = yes` | `scope:nvm_map`, `local_var:nvm_found` |
| `nvm_delete_map = yes` | Cleans parent refs, orphans children |
| `nvm_delete_tree = yes` | Recursive delete of entire subtree |

### Values

| Operation | Result |
|-----------|--------|
| `nvm_set_value = { key = X value = Y }` | |
| `nvm_get_value = { key = X }` | `local_var:nvm_result` |
| `nvm_has_value = { key = X }` | `local_var:nvm_found` |
| `nvm_remove_value = { key = X }` | |
| `nvm_clear_values = yes` | |
| `nvm_change_value = { key = X operation = add/subtract/multiply/divide value = Y }` | |
| `nvm_swap_values = { key_a = X key_b = Y }` | |

Arithmetic on missing keys starts from 0. Values can be bare literals or variable references.

### Sub-dictionaries

| Operation | Result |
|-----------|--------|
| `nvm_get_or_set_map = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_get_map = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_has_map = { key = X }` | `local_var:nvm_found` |
| `nvm_remove_map = { key = X }` | |
| `nvm_clear_maps = yes` | |

### Scope References

| Operation | Result |
|-----------|--------|
| `nvm_set_scope = { key = X value = Y }` | |
| `nvm_get_scope = { key = X }` | `scope:nvm_scope`, `local_var:nvm_found` |
| `nvm_has_scope = { key = X }` | `local_var:nvm_found` |
| `nvm_remove_scope = { key = X }` | |
| `nvm_clear_scopes = yes` | |

### Ordered Lists

| Operation | Result |
|-----------|--------|
| `nvm_get_or_set_list = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_get_list = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_has_list = { key = X }` | `local_var:nvm_found` |
| `nvm_remove_list = { key = X }` | |
| `nvm_clear_lists = yes` | |
| `nvm_list_add = { value = X }` | |
| `nvm_list_get = { index = N }` | `scope:nvm_scope`, `local_var:nvm_found` |
| `nvm_list_remove = { index = N }` | Swaps last item into gap |
| `nvm_list_find = { value = X }` | `local_var:nvm_result` (index), `local_var:nvm_found` |
| `nvm_list_has = { value = X }` | `local_var:nvm_found` |
| `nvm_list_remove_value = { value = X }` | Find + swap-remove |
| `nvm_list_clear = yes` | |

List size: `scope:my_list.var:nvm_list_size`

**Index instability:** Removal swaps the last element into the removed slot. Do not cache indices across remove operations.

### Unordered Sets

| Operation | Result |
|-----------|--------|
| `nvm_get_or_set_set = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_get_set = { key = X }` | `scope:nvm_cursor`, `local_var:nvm_found` |
| `nvm_set_add = { value = X }` | Silent no-op on duplicates |
| `nvm_set_has = { value = X }` | `local_var:nvm_found` (O(1)) |
| `nvm_set_remove = { value = X }` | |
| `nvm_set_clear = yes` | |

### Type Tagging

| Operation | Result |
|-----------|--------|
| `nvm_set_type = { type = X }` | Sets `var:nvm_type` |
| `nvm_get_type = yes` | `scope:nvm_scope`, `local_var:nvm_found` |
| `nvm_is_type = { type = X }` | `local_var:nvm_found` |

Types use `flag:` scopes as identifiers. Cleared on delete/recycle.

### Type Checking

| Operation | Result |
|-----------|--------|
| `nvm_check_key = { key = X }` | `local_var:nvm_has_map`, `nvm_has_value`, `nvm_has_scope`, `nvm_has_list` |

### Iteration

Use `every_key_in_variable_map` on the dictionary, specifying which internal map to iterate. Inside the iterator, `this` is the **key** (not the dictionary), so scope back to the dictionary to read the value.

Internal map names: `nvm_values` (numbers), `nvm_children` (sub-dicts), `nvm_scopes` (scope refs), `nvm_lists` (lists/sets).

See [TUTORIAL.md](TUTORIAL.md) Part 1 for examples iterating each type.

### Navigation

Each dictionary stores `var:nvm_accessed_from` for backtracking up the tree.

---

## Thread Safety

EU5 has two dispatch mechanisms in on_actions:
- **`on_actions = { }`** fires sub-actions sequentially in a single thread.
- **`events = { }`** fires events per-country in **parallel threads**.

NVM operations return results through local variables (`local_var:nvm_result`, `local_var:nvm_found`) and saved scopes (`scope:nvm_cursor`, `scope:nvm_map`). Parallel safety testing with 2161 countries across 50 iterations showed zero race failures on reads and read-writes. The engine appears to isolate execution context per event. NVM is safe under both sequential and parallel dispatch.

---

## Best Practices

- **Store references as variables, access via dot chain.** Works from any context.
- **Use `nvm_get_or_set_map` for write paths.** Creates on first access, navigates on repeat.
- **Use `nvm_get_map` for read paths.** Doesn't create empty nodes on missing keys.
- **Save `scope:nvm_cursor` immediately.** Every navigation call overwrites it.
- **Save `local_var:nvm_result` to a local variable immediately.** The next NVM call overwrites it.
- **Clean up with `nvm_delete_tree`.** Orphaned nodes waste pool space permanently.
- **Use `flag:` scopes as keys for OOP field names.** Unique, readable, content-independent.

## Things to Avoid

- **`save_temporary_scope_as`** for dictionary references. Use `save_scope_as`. (Under investigation; see T120-T126 tests.)
- **Bypassing the API.** Writing to `nvm_children`, `nvm_values`, etc. directly skips parent tracking and error handling.
- **Relying on `local_var:nvm_result` across multiple NVM calls without saving.** Each call overwrites the previous result.
- **Caching list indices across removals.** Swap-from-end makes indices unstable.

## Reserved Names

On pool nodes: `nvm_children`, `nvm_values`, `nvm_scopes`, `nvm_lists`, `nvm_parents`, `nvm_accessed_from`, `nvm_is_list`, `nvm_list_size`, `nvm_op_idx`, `nvm_type`

Scopes (overwritten each call): `nvm_map`, `nvm_cursor`, `nvm_scope`, `nvm_op_key`, `nvm_op_key_a`, `nvm_op_key_b`, `nvm_op_parent`, `nvm_deleting`, `nvm_del_parent`, `nvm_del_iter_key`, `nvm_del_remove_key`

Local variables (overwritten each call): `nvm_result`, `nvm_found`, `nvm_has_map`, `nvm_has_value`, `nvm_has_scope`, `nvm_has_list`, `nvm_nav`

Globals: `nvm_pool_size`, `nvm_pool_free`, `nvm_pool_active`, `nvm_keys_to_remove`

## Tests

All test output goes to `logs/debug.log`. Grep for the prefix to filter:

| Prefix | Content |
|--------|---------|
| `NVM` | Core API tests + stress tests |
| `NVM_CAP` | Engine capability probes |
| `NVM_TUT` | Tutorial horse racing |
| `NVM_PAR` | Parallel safety tests |

Enable tests by uncommenting the `on_game_start` block in `in_game/common/on_action/nvm_test.txt`.
