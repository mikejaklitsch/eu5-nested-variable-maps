# Nested Variable Maps for EU5

## What this is

EU5's variable maps are flat: one map name, one level of key-value pairs. This mod provides nested dictionaries on top of variable maps by using game locations as storage nodes. Each dictionary is a location drawn from a shared pool of ~28,000. The dictionary holds three internal variable maps: `nvm_children` for sub-dictionary references, `nvm_values` for numbers, and `nvm_scopes` for scope references. Parent tracking via an `nvm_parents` list enables automatic cleanup of references when a dictionary is deleted.

## How it works

Each dictionary is a game location with variable maps on it. The API wraps the engine's `add_to_variable_map`, `variable_map()`, `remove_from_variable_map`, and `is_key_in_variable_map` operations, handling the quirks (silent failure on duplicate keys, scope saving requirements, parameter substitution in quoted expressions) so you don't have to.

The pool collects all game locations at init. Allocating a dictionary picks a random location from the pool and wipes any stale data from previous use. Keys are scopes; values are numbers or scope references stored in variable maps on that location. Sub-dictionaries are more allocated locations, stored as scope references in the parent's variable map.

Store a dictionary reference on any scope that allows variables with `set_variable`. Access it via dot chain: `c:FRA.var:my_dict.nvm_get_value = { key = goods:wheat }`.

Deletion walks the parent list and scans each parent's variable maps to remove all references to the deleted node before returning it to the pool. `nvm_delete_tree` does this recursively through the entire subtree.

---

## Tutorial: Building a Royal Cookbook

Every country gets a cookbook. Each recipe is keyed by its main ingredient. The recipe stores ingredient quantities as values (keyed by goods), who invented it as a scope reference, and regional variants as sub-dictionaries. By the end of this walkthrough you'll have used every major operation.

The structure we're building:

```
country's cookbook > recipe (e.g. goods:wine) > {
    goods:wheat: 3                (value: ingredient quantity)
    goods:sugar: 2                (value: ingredient quantity)
    goods:fruit: 1                (value: ingredient quantity)
    goods:wine: c:FRA             (scope: who invented this recipe)
    sub_continent:western_europe > {  (sub-dictionary: regional variant)
        goods:marble: 1           (value: extra regional ingredient)
    }
}
```

### Step 1: Initialize the pool

Before allocating any dictionaries, the pool needs to be built. Call this once at game start. It collects all game locations into the free pool. To exclude specific locations, set `nvm_exclude` on them before this call.

```
nvm_init_pool = yes
```

### Step 2: Give every country a cookbook

Allocate a dictionary for each country and store it as a variable. `nvm_new_map` creates a new empty dictionary and writes it to `scope:nvm_map`. This scope is shared and overwritten on every call, so save it with `set_variable` immediately.

```
random_country = { # every_country requires country scope; random_country provides it
    every_country = {
        nvm_new_map = yes
        if = {
            limit = { global_var:nvm_found = no } # pool exhausted, no dictionary allocated
            error_log = "cookbook allocation failed: pool empty"
        } else = {
            set_variable = { name = cookbook value = scope:nvm_map }
        }
    }
}
```

In multiplayer, GUI-triggered scripted effects could race on `scope:nvm_map` if two players allocate in the same tick (untested).

### Step 3: Add a recipe

We'll add a wine recipe to France's cookbook. `nvm_get_or_set_map` opens or creates a child dictionary at the given key. If the recipe already exists it navigates to it; if not it allocates a new one. The result goes to `scope:nvm_cursor`, which is a reference to the child dictionary. It works just like any dictionary reference: you can call operations on it, store it, or save it.

`scope:nvm_cursor` is overwritten on every `nvm_get_or_set_map` and `nvm_get_map` call. `global_var:nvm_found` tells you whether the child already existed (yes) or was just created (no).

```
c:FRA.var:cookbook.nvm_get_or_set_map = { key = goods:wine }
# scope:nvm_cursor = the wine recipe dictionary
```

### Step 4: Add ingredients

Store the ingredient quantities on the recipe. Each ingredient is keyed by its goods scope. `nvm_set_value` creates or overwrites. `global_var:nvm_found` from step 3 tells us whether the recipe already existed; we only initialize ingredients on new recipes.

```
if = {
    limit = { global_var:nvm_found = no } # recipe was just created, safe to initialize
    scope:nvm_cursor.nvm_set_value = { key = goods:wheat value = 3 }
    scope:nvm_cursor.nvm_set_value = { key = goods:sugar value = 2 }
    scope:nvm_cursor.nvm_set_value = { key = goods:fruit value = 1 }
}
```

### Step 5: Record who invented it

Store a reference to the country that created this recipe. Scopes are stored separately from values, so `goods:wine` can hold both a scope (the inventor) and values (ingredient quantities) without conflict.

```
scope:nvm_cursor.nvm_set_scope = { key = goods:wine value = c:FRA }
```

### Step 6: Add a regional variant

Create a sub-dictionary inside the recipe for a Western European variant that adds marble as an extra ingredient. `scope:nvm_cursor` still points to the wine recipe from step 3. Since navigating deeper overwrites it, we save the recipe first.

```
# Save the recipe cursor before we navigate deeper.
# IMPORTANT: must use save_scope_as, not save_temporary_scope_as.
# Temporary scopes silently break variable map writes on re-entry.
scope:nvm_cursor = { save_scope_as = wine_recipe }

# Create the regional variant sub-dictionary.
# get_or_set_map creates it on the first call, navigates to it on repeat calls.
scope:wine_recipe.nvm_get_or_set_map = { key = sub_continent:western_europe }
# scope:nvm_cursor now points to the Western Europe variant.
# scope:wine_recipe still points to the recipe above it.

# Add the regional ingredient
scope:nvm_cursor.nvm_set_value = { key = goods:marble value = 1 }
```

### Step 7: Read it all back

Navigate from the country through the cookbook to the recipe. `nvm_get_map` navigates to an existing child without creating it. Check `global_var:nvm_found` before using `scope:nvm_cursor`; on a miss the cursor is unchanged from whatever it was before.

```
c:FRA.var:cookbook.nvm_get_map = { key = goods:wine }
if = {
    limit = { global_var:nvm_found = yes }

    # Read an ingredient quantity
    scope:nvm_cursor.nvm_get_value = { key = goods:wheat }
    set_local_variable = { name = wheat_qty value = global_var:nvm_result }

    # Read who invented it
    scope:nvm_cursor.nvm_get_scope = { key = goods:wine }
    # scope:nvm_scope = c:FRA

    # Navigate into the regional variant
    scope:nvm_cursor.nvm_get_map = { key = sub_continent:western_europe }
    if = {
        limit = { global_var:nvm_found = yes }
        scope:nvm_cursor.nvm_get_value = { key = goods:marble }
        set_local_variable = { name = marble_qty value = global_var:nvm_result }
    }
}
```

Dot chains also work from any starting scope. Here we read France's cookbook from a location owned by France:

```
scope:some_location.owner.var:cookbook.nvm_get_map = { key = goods:wine }
```

### Step 8: Check what's stored at a key

Before accessing, you might want to know what types of data exist at a key. Different keys hold different types in our recipe:

```
# goods:wheat has a value (ingredient quantity) but no scope or sub-dictionary
scope:wine_recipe.nvm_check_key = { key = goods:wheat }
# global_var:nvm_has_value = yes
# global_var:nvm_has_map = no
# global_var:nvm_has_scope = no

# goods:wine has a scope (the inventor) but no value or sub-dictionary
scope:wine_recipe.nvm_check_key = { key = goods:wine }
# global_var:nvm_has_scope = yes
# global_var:nvm_has_value = no
# global_var:nvm_has_map = no

# sub_continent:western_europe has a sub-dictionary (the regional variant)
scope:wine_recipe.nvm_check_key = { key = sub_continent:western_europe }
# global_var:nvm_has_map = yes
# global_var:nvm_has_value = no
# global_var:nvm_has_scope = no
```

For single-type checks: `nvm_has_map`, `nvm_has_scope`, `nvm_has_value`. All set `global_var:nvm_found`.

### Step 9: Modify a value

France improved the recipe and needs more wheat. `nvm_change_value` modifies a value in place using any arithmetic operation. Missing keys start from 0.

```
scope:wine_recipe.nvm_change_value = { key = goods:wheat operation = add value = 1 }
# wheat quantity is now 4

scope:wine_recipe.nvm_change_value = { key = goods:wheat operation = multiply value = 2 }
# wheat quantity is now 8
```

Supported operations: `add`, `subtract`, `multiply`, `divide`. Values can be bare literals or variable references.

To swap two values: `scope:wine_recipe.nvm_swap_values = { key_a = goods:wheat key_b = goods:sugar }`.

### Step 10: Iterate ingredients

To list all ingredients in the wine recipe, use `every_key_in_variable_map` directly. Inside the iterator the current scope is the key, not the dictionary. Scope back to the dictionary to read values.

```
scope:wine_recipe = {
    every_key_in_variable_map = {
        variable = nvm_values
        save_scope_as = ingredient
        scope:wine_recipe = {
            set_local_variable = { name = qty value = "variable_map(nvm_values|scope:ingredient)" }
        }
    }
}
```

Internal map names for iteration: `nvm_values`, `nvm_children`, `nvm_scopes`.

### Step 11: Delete a recipe

Remove the wine recipe from France's cookbook. `nvm_delete_tree` recursively deletes the recipe and its regional variant sub-dictionary, cleans up all parent references, and returns the nodes to the pool.

```
c:FRA.var:cookbook.nvm_get_map = { key = goods:wine }
if = {
    limit = { global_var:nvm_found = yes }
    scope:nvm_cursor = { nvm_delete_tree = yes }
}
```

To remove a recipe reference without deleting the recipe itself (e.g., if another country also has that recipe in their cookbook), use `nvm_remove_map`:

```
c:FRA.var:cookbook.nvm_remove_map = { key = goods:wine }
```

To delete just the recipe node without touching its children, use `nvm_delete_map`. Children become orphaned.

### Step 12: Clean up a country's entire cookbook and all its contents

```
c:FRA.var:cookbook = { nvm_delete_tree = yes }
```

---

## Best practices

- **Store references as variables, access via dot chain.** Works from any context without managing saved scope names.
- **Use `nvm_get_or_set_map` for write paths.** Creates on first access, navigates on repeat access. No existence check needed.
- **Use `nvm_get_map` for read paths.** Doesn't create empty nodes on bad keys. Check `nvm_found` before using the cursor.
- **Save `scope:nvm_cursor` when you need multiple children at once.** Every navigation call overwrites it.
- **Clean up with `nvm_delete_tree`.** Orphaned nodes waste pool space permanently.
- **Share children across parents** when the data is the same. Reduces pool consumption significantly.

## Things to avoid

- **Invalid scope keys.** If a scope doesn't exist (e.g., a misspelled goods name), reads silently return 0 and writes silently fail. Writes log an error, but there's no crash.
- **A dictionary per location.** ~13,000 roots leaves ~15,000 for children across all of them. Without heavy child sharing, no room for actual nesting.
- **Bypassing the API.** Writing to `nvm_children`, `nvm_values`, `nvm_scopes`, or `nvm_parents` directly skips parent tracking, remove-before-add handling, and error logging.

---

## API Reference

### Lifecycle

| Operation | Result |
|-----------|--------|
| `nvm_init_pool = yes` | `global_var:nvm_pool_size` |
| `nvm_new_map = yes` | `scope:nvm_map`, `global_var:nvm_found` |
| `nvm_delete_map = yes` | Cleans parent refs, orphans children |
| `nvm_delete_tree = yes` | Recursive delete of entire subtree |

### Values

| Operation | Result |
|-----------|--------|
| `nvm_set_value = { key = X value = Y }` | |
| `nvm_get_value = { key = X }` | `global_var:nvm_result` |
| `nvm_has_value = { key = X }` | `global_var:nvm_found` |
| `nvm_remove_value = { key = X }` | |
| `nvm_clear_values = yes` | |
| `nvm_change_value = { key = X operation = add/subtract/multiply/divide value = Y }` | |
| `nvm_swap_values = { key_a = X key_b = Y }` | |

Arithmetic on missing keys starts from 0. Values can be bare literals or variable references.

### Sub-dictionaries

| Operation | Result |
|-----------|--------|
| `nvm_get_or_set_map = { key = X }` | `scope:nvm_cursor`, `global_var:nvm_found` |
| `nvm_get_map = { key = X }` | `scope:nvm_cursor`, `global_var:nvm_found` |
| `nvm_has_map = { key = X }` | `global_var:nvm_found` |
| `nvm_remove_map = { key = X }` | |
| `nvm_clear_maps = yes` | |

### Scope References

| Operation | Result |
|-----------|--------|
| `nvm_set_scope = { key = X value = Y }` | |
| `nvm_get_scope = { key = X }` | `scope:nvm_scope`, `global_var:nvm_found` |
| `nvm_has_scope = { key = X }` | `global_var:nvm_found` |
| `nvm_remove_scope = { key = X }` | |
| `nvm_clear_scopes = yes` | |

### Type Checking

| Operation | Result |
|-----------|--------|
| `nvm_check_key = { key = X }` | `global_var:nvm_has_map`, `nvm_has_value`, `nvm_has_scope` |

### Navigation

Each dictionary stores which dictionary it was last accessed from in `var:nvm_accessed_from`. This enables backtracking up the tree:

```
scope:nvm_cursor.var:nvm_accessed_from                              # parent
scope:nvm_cursor.var:nvm_accessed_from.var:nvm_accessed_from         # grandparent
```

## Multi-parent

Multiple parents can reference the same child. Manual linking requires registering in both the parent's `nvm_children` map and the child's `nvm_parents` list. `nvm_delete_map` and `nvm_delete_tree` automatically clean all parent references.

## Reserved Names

On pool locations: `nvm_children`, `nvm_values`, `nvm_scopes`, `nvm_parents`, `nvm_accessed_from`

Scopes (overwritten each call): `nvm_map`, `nvm_cursor`, `nvm_scope`, `nvm_op_key`, `nvm_op_key_a`, `nvm_op_key_b`, `nvm_op_parent`, `nvm_deleting`, `nvm_del_parent`, `nvm_del_iter_key`, `nvm_del_remove_key`

Globals: `nvm_result`, `nvm_found`, `nvm_has_map`, `nvm_has_value`, `nvm_has_scope`, `nvm_nav`, `nvm_pool_size`, `nvm_free_pool`, `nvm_keys_to_remove`
