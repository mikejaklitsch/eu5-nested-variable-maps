# Nested Variable Maps: Tutorial

A progressive walkthrough from basic dictionaries through OOP patterns. For API reference, see [README.md](README.md).

---

## Part 1: Flat Dictionaries

A dictionary is a node that holds key-value pairs. Keys are game scopes (goods, countries, subcontinents, flags, etc.). Values are numbers.

### Writing and reading values

```
scope:my_dict.nvm_set_value = { key = goods:wheat value = 42 }
scope:my_dict.nvm_get_value = { key = goods:wheat }
# local_var:nvm_result = 42

scope:my_dict.nvm_has_value = { key = goods:wheat }
# local_var:nvm_found = yes

scope:my_dict.nvm_remove_value = { key = goods:wheat }
scope:my_dict.nvm_clear_values = yes
```

Values can be bare literals (`42`) or variable references (`local_var:v`).

### Scope references

Store references to any game scope separately from numeric values. The same key can hold both a value and a scope reference without conflict.

```
scope:my_dict.nvm_set_scope = { key = goods:wine value = c:FRA }
scope:my_dict.nvm_get_scope = { key = goods:wine }
# scope:nvm_scope = c:FRA, local_var:nvm_found = yes
```

### Arithmetic

Modify values in place. Missing keys start from 0.

```
scope:my_dict.nvm_change_value = { key = goods:wheat operation = add value = 10 }
scope:my_dict.nvm_change_value = { key = goods:wheat operation = multiply value = 2 }
scope:my_dict.nvm_swap_values = { key_a = goods:wheat key_b = goods:iron }
```

### Iteration

Each data type is stored in a separate internal variable map. You iterate them with `every_key_in_variable_map`, specifying which map to iterate.

There's one catch: inside the iterator, `this` becomes the **key** (e.g., `goods:wheat`), not the dictionary. To read the value at that key, you have to scope back to the dictionary and use the `variable_map()` expression with the saved key.

**Iterating values** (numeric data in `nvm_values`):

```
scope:my_dict = {
    every_key_in_variable_map = {
        variable = nvm_values
        save_scope_as = current_key
        # 'this' is now the key (e.g., goods:wheat), not the dictionary.
        # Scope back to the dictionary to read the value at this key.
        scope:my_dict = {
            set_local_variable = { name = qty value = "variable_map(nvm_values|scope:current_key)" }
        }
    }
}
```

**Iterating scope references** (stored in `nvm_scopes`):

```
scope:my_dict = {
    every_key_in_variable_map = {
        variable = nvm_scopes
        save_scope_as = current_key
        scope:my_dict = {
            set_local_variable = { name = nvm_nav value = "variable_map(nvm_scopes|scope:current_key)" }
        }
        # local_var:nvm_nav is now the stored scope reference
    }
}
```

**Iterating children** (sub-dictionaries in `nvm_children`):

```
scope:my_dict = {
    every_key_in_variable_map = {
        variable = nvm_children
        save_scope_as = current_key
        scope:my_dict = {
            set_local_variable = { name = nvm_nav value = "variable_map(nvm_children|scope:current_key)" }
        }
        # local_var:nvm_nav is now the child dictionary node.
        # You can scope into it to read its data.
        local_var:nvm_nav = { nvm_get_value = { key = goods:wheat } }
    }
}
```

**Iterating lists/sets** uses `nvm_lists` the same way. Sets can also be iterated by their elements directly since the elements ARE the keys in `nvm_children` (see Part 3).

---

## Part 2: Nesting

### Sub-dictionaries

Create or navigate to child dictionaries. Children are separate nodes stored in the parent's `nvm_children` map.

```
# Write path: create or navigate
scope:parent.nvm_get_or_set_map = { key = goods:wine }
# scope:nvm_cursor = the child dictionary
# local_var:nvm_found = yes if it existed, no if just created

# Read path: navigate only
scope:parent.nvm_get_map = { key = goods:wine }
# local_var:nvm_found = no if missing (cursor unchanged)
```

### Cursor management

Every navigation call overwrites `scope:nvm_cursor`. Save it before navigating deeper.

```
scope:root.nvm_get_or_set_map = { key = goods:wine }
scope:nvm_cursor = { save_scope_as = wine_recipe }

scope:wine_recipe.nvm_get_or_set_map = { key = sub_continent:western_europe }
scope:nvm_cursor.nvm_set_value = { key = goods:marble value = 1 }
```

**Always use `save_scope_as`, never `save_temporary_scope_as`.** Temporary scopes silently break variable map writes on re-entry. (Under investigation.)

### Dot-chain access patterns

Access dictionaries stored as variables from any starting scope.

```
# Dot chain directly into scripted effect
scope:my_country.var:my_dict.nvm_get_value = { key = goods:wheat }

# Chain through scope traversal
scope:my_location.owner.var:my_dict.nvm_get_value = { key = goods:wheat }

# Inline read in a value expression
set_local_variable = {
    name = v
    value = "scope:my_country.var:my_dict.variable_map(nvm_values|goods:wheat)"
}
```

### Navigation backtracking

Each dictionary stores which dictionary it was last accessed from.

```
scope:nvm_cursor.var:nvm_accessed_from                              # parent
scope:nvm_cursor.var:nvm_accessed_from.var:nvm_accessed_from         # grandparent
```

---

## Part 3: Collections

### Ordered lists

Lists store ordered sequences of scope references. Each list costs one pool node.

```
scope:parent.nvm_get_or_set_list = { key = goods:wine }
scope:nvm_cursor = { save_scope_as = my_list }

scope:my_list = { nvm_list_add = { value = c:FRA } }
scope:my_list = { nvm_list_add = { value = c:SPA } }
# scope:my_list.var:nvm_list_size = 2

scope:my_list = { nvm_list_get = { index = 0 } }
# scope:nvm_scope = c:FRA

scope:my_list = { nvm_list_remove_value = { value = c:SPA } }
# Swap-from-end: indices are unstable after removal
```

### Unordered sets

O(1) membership testing. Scope keys ARE the items.

```
scope:parent.nvm_get_or_set_set = { key = goods:salt }
scope:nvm_cursor = { save_scope_as = my_set }

scope:my_set = { nvm_set_add = { value = goods:wheat } }
scope:my_set = { nvm_set_has = { value = goods:wheat } }
# local_var:nvm_found = yes

scope:my_set = { nvm_set_remove = { value = goods:wheat } }
```

Iterate directly:

```
scope:my_set = {
    every_key_in_variable_map = {
        variable = nvm_children
        save_scope_as = item
        # scope:item IS the set element
    }
}
```

---

## Part 4: Advanced Data Patterns

### Multi-parent shared nodes

Multiple parents can reference the same child. Manual linking requires registering in both the parent's map and the child's parent list.

```
scope:parent_a.nvm_get_or_set_map = { key = goods:cloth }
scope:nvm_cursor = { save_scope_as = shared_child }

scope:parent_b = {
    add_to_variable_map = { name = nvm_children key = goods:silk value = scope:shared_child }
}
scope:shared_child = {
    add_to_variable_list = { name = nvm_parents target = scope:parent_b }
}
```

### Tree deletion

```
scope:my_node = { nvm_delete_map = yes }    # single node, orphans children
scope:my_node = { nvm_delete_tree = yes }   # recursive subtree delete
scope:parent.nvm_remove_map = { key = X }   # unlink child without deleting it
```

---

## Part 5: Object-Oriented Patterns

### The insight

An NVM node already has everything a class instance needs:
- **State** stored in variable maps (values, scopes, children)
- **Identity** (a specific dead character, unique and referenceable)
- **References** (stored as variables, passed between scopes, linked to other nodes)
- **Type** via `nvm_set_type` / `nvm_is_type`

A "class" is a convention: a set of expected keys. A "constructor" is a scripted effect that allocates, stamps, and initializes. A "method" operates on a typed node. A "comparator" reads fields from two nodes.

### Type tagging

```
scope:my_horse = { nvm_set_type = { type = flag:my_horse } }
scope:my_horse = { nvm_is_type = { type = flag:my_horse } }
# local_var:nvm_found = yes

scope:my_horse = { nvm_get_type = yes }
# scope:nvm_scope = flag:my_horse
```

### Writing a constructor

The constructor IS the class definition: whatever fields it writes are the class's schema.

```
my_new_horse = {
    nvm_new_map = yes
    scope:nvm_map = {
        nvm_set_type = { type = flag:my_horse }
        nvm_set_value = { key = flag:my_speed value = $speed$ }
        nvm_set_value = { key = flag:my_stamina value = $stamina$ }
        nvm_set_value = { key = flag:my_wins value = 0 }
    }
}

# Usage:
my_new_horse = { speed = 8 stamina = 6 }
scope:nvm_map = { save_scope_as = my_horse }
```

### Writing methods

Methods are scripted effects that take the node as `this`.

```
my_train = {
    nvm_change_value = { key = $stat$ operation = add value = $amount$ }
}

my_get_power = {
    nvm_get_value = { key = flag:my_speed }
    set_local_variable = { name = power value = local_var:nvm_result }
    nvm_get_value = { key = flag:my_stamina }
    change_local_variable = { name = power add = local_var:nvm_result }
}

# Usage:
scope:my_horse = { my_train = { stat = flag:my_speed amount = 1 } }
scope:my_horse = { my_get_power = yes }
# local_var:power = computed power
```

### Composition: objects referencing objects

```
# Horse has a Jockey (bidirectional scope reference)
scope:my_horse = { nvm_set_scope = { key = flag:my_jockey value = scope:my_jockey } }
scope:my_jockey = { nvm_set_scope = { key = flag:my_mount value = scope:my_horse } }

# Horse has parents (other Horse nodes)
scope:foal = { nvm_set_scope = { key = flag:my_sire value = scope:stallion } }

# Stable contains Horses (sub-dictionary links)
scope:stable.nvm_get_or_set_map = { key = flag:my_horse_1 }
```

---

## Part 6: Horse Racing Minigame

A complete working example in the mod itself. Files prefixed `nvm_tutorial_` demonstrate every OOP pattern above.

### Data model

```
Horse (flag:nvm_tutorial_horse)
  values:  speed, stamina, endurance, wins, losses
  scopes:  jockey (Jockey node), sire (Horse), dam (Horse)
  child:   maintenance (dict of goods costs)
  list:    race_history (locations raced at)

Jockey (flag:nvm_tutorial_jockey)
  values:  skill, experience
  scopes:  mount (Horse node)

Stable (flag:nvm_tutorial_stable)
  values:  total_wins, total_losses
  children: horses at flag:nvm_tutorial_horse_1, flag:nvm_tutorial_horse_2, etc.
```

### Creating a stable with horses

```
nvm_tutorial_new_stable = yes
c:FRA = { set_variable = { name = nvm_tutorial_stable value = scope:nvm_map } }

nvm_tutorial_new_horse = { speed = 8 stamina = 6 endurance = 5 }
scope:nvm_map = { save_scope_as = horse_1 }
c:FRA.var:nvm_tutorial_stable = {
    nvm_tutorial_stable_add_horse = { key = flag:nvm_tutorial_horse_1 horse = scope:horse_1 }
}

nvm_tutorial_new_jockey = { skill = 4 }
scope:nvm_map = { save_scope_as = jockey_1 }
scope:horse_1 = { nvm_tutorial_assign_jockey = { jockey = scope:jockey_1 } }
```

### Racing

```
scope:horse_a = { nvm_tutorial_get_power = yes }
set_local_variable = { name = power_a value = local_var:nvm_tutorial_power }

scope:horse_b = { nvm_tutorial_get_power = yes }
set_local_variable = { name = power_b value = local_var:nvm_tutorial_power }

if = {
    limit = { local_var:power_a >= local_var:power_b }
    scope:horse_a = { nvm_tutorial_record_win = { location = scope:race_venue } }
    scope:horse_b = { nvm_tutorial_record_loss = { location = scope:race_venue } }
} else = {
    scope:horse_b = { nvm_tutorial_record_win = { location = scope:race_venue } }
    scope:horse_a = { nvm_tutorial_record_loss = { location = scope:race_venue } }
}
```

### Traversing the object graph

```
# country > stable > horse > jockey > experience
c:FRA.var:nvm_tutorial_stable = { nvm_get_map = { key = flag:nvm_tutorial_horse_1 } }
scope:nvm_cursor = { nvm_get_scope = { key = flag:nvm_tutorial_jockey } }
scope:nvm_scope = { nvm_get_value = { key = flag:nvm_tutorial_experience } }

# horse > sire > speed (lineage query)
scope:foal = { nvm_get_scope = { key = flag:nvm_tutorial_sire } }
scope:nvm_scope = { nvm_get_value = { key = flag:nvm_tutorial_speed } }
```

### Files

| File | Content |
|------|---------|
| `nvm_tutorial_constructors.txt` | Horse, Jockey, Stable class definitions |
| `nvm_tutorial_methods.txt` | train, race, record, assign, power calculation |
| `nvm_tutorial_setup.txt` | Game start initialization and race simulation |
| `nvm_tutorial_tests.txt` | Assertion-based validation of the data model |

Enable the tutorial by uncommenting the `on_game_start` block in `nvm_test.txt`. Grep for `NVM_TUT` in `logs/error.log` for output.
