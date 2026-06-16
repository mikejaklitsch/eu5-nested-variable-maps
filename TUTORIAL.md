# Nested Variable Maps: Tutorial

We're going to build a horse racing minigame. Horses are class instances with stats like speed and stamina. Jockeys link to horses bidirectionally. Stables hold collections of horses and track records. Countries own stables.

In most languages you'd model this with classes and objects, but EU5 doesn't give you a way to allocate your own data containers or define custom types. NVM changes that: it turns dead characters into reusable storage nodes, letting you build class instances, object graphs, and collections on top of the game's variable system. For the full API reference, see [README.md](README.md).

---

## How NVM nodes work

Every NVM node is a dead character repurposed as a key-value store. Dead characters are ideal for this because they persist in the save game, retain their variable maps, each one is a unique referenceable scope, you can create as many as you need with no fixed ceiling, and they carry little to no performance cost. Each node can hold four types of data at any key:

- **Values**: numbers (speed = 8, wins = 0)
- **Scopes**: references to game objects or other nodes (a jockey, a sire)
- **Children**: links to other NVM nodes (sub-dictionaries)
- **Lists and sets**: indexed collections or membership collections

You allocate nodes with `nvm_new_map` and free them with `nvm_delete_map`. The library recycles nodes automatically through a pool. Before any allocation, your mod must call `nvm_init_pool = yes` once before the first `nvm_new_map` call. It doesn't have to be at game start; mods aiming for save-game compatibility can call it later. It's idempotent, so multiple mods calling it is safe.

Here's what we're building:

```
Horse (flag:nvm_tutorial_horse)
  values:  speed, stamina, endurance, wins, losses
  scopes:  jockey (Jockey node), sire (Horse), dam (Horse)
  child:   maintenance (dict of goods costs)
  list:    race_history (locations raced at)

Jockey (flag:nvm_tutorial_jockey) — stamped directly on a character
  values:  skill, experience
  scopes:  mount (Horse node)

Stable (flag:nvm_tutorial_stable)
  values:  total_wins, total_losses
  set:     horses (unordered collection of Horse nodes)
  set:     venues (locations raced at, no duplicates)
```

Each "class" is an NVM node stamped with a type flag, and its "fields" are key-value entries on that node. Flags are lightweight game-defined constants that work well as field names and type identifiers.

---

## Writing the constructors

A constructor is a scripted effect that stamps a type and initializes fields. For abstract concepts like horses and stables, it allocates a pool node first. For things that already exist in the game, like a character who becomes a jockey, you skip allocation and stamp the data directly on them. Whatever the constructor writes defines the class schema.

### Horse

```
nvm_tutorial_new_horse = {
    nvm_new_map = yes
    scope:nvm_map = {
        nvm_set_type = { type = flag:nvm_tutorial_horse }
        nvm_set_value = { key = flag:nvm_tutorial_speed value = $speed$ }
        nvm_set_value = { key = flag:nvm_tutorial_stamina value = $stamina$ }
        nvm_set_value = { key = flag:nvm_tutorial_endurance value = $endurance$ }
        nvm_set_value = { key = flag:nvm_tutorial_wins value = 0 }
        nvm_set_value = { key = flag:nvm_tutorial_losses value = 0 }
    }

    scope:nvm_map = { nvm_get_or_set_list = { key = flag:nvm_tutorial_race_history } }
}
```

`nvm_new_map` allocates a node and stores it in `scope:nvm_map`. We scope into it to write fields: `nvm_set_type` stamps the node as a Horse, and each `nvm_set_value` stores a number at a key. `$speed$`, `$stamina$`, and `$endurance$` are scripted effect parameters passed by the caller.

`nvm_get_or_set_list` pre-creates an empty list for race history so that recording methods can append to it without checking if it exists first. Maintenance costs are assigned separately after construction (see "Assign maintenance categories" below).

NVM operations communicate results through reserved variables. Each call overwrites its result, so copy the value before calling the same operation again:

| Variable               | Set by                                                    | Contains                  |
|------------------------|-----------------------------------------------------------|---------------------------|
| `scope:nvm_map`        | `nvm_new_map`                                             | Newly allocated node      |
| `scope:nvm_cursor`     | `nvm_get_map`, `nvm_get_or_set_map`, and other navigation | The node navigated to     |
| `scope:nvm_scope`      | `nvm_get_scope`, `nvm_get_type`                           | Retrieved scope reference |
| `local_var:nvm_result` | `nvm_get_value`                                           | Retrieved number          |
| `local_var:nvm_found`  | Most operations                                           | `yes` or `no`             |

### Jockey

A jockey IS a character, so there's no reason to allocate a separate node. NVM operations work on any scope; we stamp and initialize the character directly.

```
nvm_tutorial_init_jockey = {
    nvm_set_type = { type = flag:nvm_tutorial_jockey }
    nvm_set_value = { key = flag:nvm_tutorial_skill value = $skill$ }
    nvm_set_value = { key = flag:nvm_tutorial_experience value = 0 }
}
```

No `nvm_new_map` needed. The character is the jockey.

### Stable

```
nvm_tutorial_new_stable = {
    nvm_new_map = yes
    scope:nvm_map = {
        nvm_set_type = { type = flag:nvm_tutorial_stable }
        nvm_set_value = { key = flag:nvm_tutorial_total_wins value = 0 }
        nvm_set_value = { key = flag:nvm_tutorial_total_losses value = 0 }
    }
    scope:nvm_map = { nvm_get_or_set_set = { key = flag:nvm_tutorial_horses } }
    scope:nvm_map = { nvm_get_or_set_set = { key = flag:nvm_tutorial_venues } }
}
```

The stable creates two sets: one for its horses and one for venue tracking. Sets are unordered collections with instant membership checks and no duplicates.

---

## Building the world

Now we instantiate objects and wire them together. This code runs once at game start.

### Create a stable and attach it to France

```
nvm_tutorial_new_stable = yes
c:FRA = { set_variable = { name = nvm_tutorial_stable value = scope:nvm_map } }
```

NVM nodes live on dead characters, separate from the game world. To access a stable from a country, we store a reference using `set_variable` on the country. Now `c:FRA.var:nvm_tutorial_stable` reaches the stable node from anywhere.

### Create horses and add them to the stable

```
nvm_tutorial_new_horse = { speed = 8 stamina = 6 endurance = 5 }
scope:nvm_map = { save_scope_as = horse_1 }

c:FRA.var:nvm_tutorial_stable.nvm_get_set = { key = flag:nvm_tutorial_horses }
scope:nvm_cursor.nvm_set_add = { value = scope:horse_1 }
```

Note the dot-chain syntax: `c:FRA.var:nvm_tutorial_stable.nvm_get_set = { ... }` chains through the country, its variable, and into the effect in one expression. This is equivalent to nesting scope blocks but much cleaner for single operations. Both work everywhere, and we'll use whichever reads better from here on.

We navigate to the stable's horses set (created in the constructor), then add the horse to it. Sets are unordered collections where the item itself is the key, so you get membership checks for free (`nvm_set_has`) and don't need to invent arbitrary slot names. We'll cover sets in more detail in the racing section.

### Assign maintenance categories

Rather than giving every horse its own maintenance data, we create shared cost templates and attach them by name:

```
nvm_new_map = yes
scope:nvm_map = { save_scope_as = warhorse_maint }
scope:warhorse_maint = {
    nvm_set_value = { key = goods:wheat value = 3 }
    nvm_set_value = { key = goods:iron value = 2 }
}

nvm_new_map = yes
scope:nvm_map = { save_scope_as = packhorse_maint }
scope:packhorse_maint.nvm_set_value = { key = goods:wheat value = 1 }

scope:horse_1.nvm_set_map = { key = flag:nvm_tutorial_maintenance child = scope:warhorse_maint }
scope:horse_2.nvm_set_map = { key = flag:nvm_tutorial_maintenance child = scope:packhorse_maint }
```

`nvm_set_map` links an existing node as a named child on another node. Multiple horses can share the same maintenance template; if the base costs change, every horse with that template sees the update. This is the counterpart to `nvm_get_or_set_map` (which always allocates a new child).

### Link a jockey to a horse

```
scope:some_character.nvm_tutorial_init_jockey = { skill = 4 }
scope:horse_1.nvm_tutorial_assign_jockey = { jockey = scope:some_character }
```

The assign method creates a bidirectional link using scope references:

```
nvm_tutorial_assign_jockey = {
    nvm_set_scope = { key = flag:nvm_tutorial_jockey value = $jockey$ }
    save_scope_as = parent_node
    $jockey$ = {
        nvm_set_scope = { key = flag:nvm_tutorial_mount value = scope:parent_node }
    }
}
```

`nvm_set_scope` stores a reference to any game scope or NVM node at a key, separate from numeric values. The same key can hold both a value and a scope reference without conflict. Here the horse stores a reference to the character, and the character stores one back to the horse.

---

## Writing methods

Methods are scripted effects called on a node. The node is `this` inside the effect.

### Computing power

Two patterns to watch for in this method: every `nvm_get_value` overwrites the same `local_var:nvm_result`, so we copy each read before the next one clobbers it. And `nvm_get_scope` may not find a jockey, so we check `local_var:nvm_found` before following the reference.

```
nvm_tutorial_get_power = {
    nvm_get_value = { key = flag:nvm_tutorial_speed }
    set_local_variable = { name = nvm_tutorial_power value = local_var:nvm_result }

    nvm_get_value = { key = flag:nvm_tutorial_stamina }
    change_local_variable = { name = nvm_tutorial_power add = local_var:nvm_result }

    nvm_get_value = { key = flag:nvm_tutorial_endurance }
    change_local_variable = { name = nvm_tutorial_power add = local_var:nvm_result }

    nvm_get_scope = { key = flag:nvm_tutorial_jockey }
    if = {
        limit = { local_var:nvm_found = yes }
        scope:nvm_scope = { nvm_get_value = { key = flag:nvm_tutorial_skill } }
        change_local_variable = { name = nvm_tutorial_power add = local_var:nvm_result }
    }
}
```

The jockey's skill contribution is conditional: `nvm_get_scope` retrieves the jockey into `scope:nvm_scope`, but only if one is assigned.

### Training

```
nvm_tutorial_train = {
    nvm_change_value = { key = $stat$ operation = add value = $amount$ }
}
```

`nvm_change_value` modifies a value in place. It supports `add`, `subtract`, `multiply`, and `divide`. Missing keys start from 0. Usage:

```
scope:horse_1.nvm_tutorial_train = { stat = flag:nvm_tutorial_speed amount = 1 }
```

---

## Running a race

### Compare power and record results

```
scope:horse_a.nvm_tutorial_get_power = yes
set_local_variable = { name = power_a value = local_var:nvm_tutorial_power }

scope:horse_b.nvm_tutorial_get_power = yes
set_local_variable = { name = power_b value = local_var:nvm_tutorial_power }

if = {
    limit = { local_var:power_a >= local_var:power_b }
    scope:horse_a.nvm_tutorial_record_win = { location = scope:race_venue }
    scope:horse_b.nvm_tutorial_record_loss = { location = scope:race_venue }
} else = {
    scope:horse_b.nvm_tutorial_record_win = { location = scope:race_venue }
    scope:horse_a.nvm_tutorial_record_loss = { location = scope:race_venue }
}

c:FRA.var:nvm_tutorial_stable = { nvm_tutorial_record_venue = { location = scope:race_venue } }
c:ENG.var:nvm_tutorial_stable = { nvm_tutorial_record_venue = { location = scope:race_venue } }
```

Each power call overwrites `nvm_tutorial_power`, so we save horse A's result before computing horse B's.

### Recording wins with lists

```
nvm_tutorial_record_win = {
    nvm_change_value = { key = flag:nvm_tutorial_wins operation = add value = 1 }
    nvm_get_list = { key = flag:nvm_tutorial_race_history }
    if = {
        limit = { local_var:nvm_found = yes }
        scope:nvm_cursor = { nvm_list_add = { value = $location$ } }
    }
}
```

`nvm_get_list` navigates to the race history list (result in `scope:nvm_cursor`). `nvm_list_add` appends the race location. Lists are indexed collections of scope references: use append-only for stable ordering, or add/remove freely as an unordered bag.

Note: if you remove from a list, the last element swaps into the gap to avoid shifting everything. This means indices are not stable after removal.

### Tracking venues with sets

```
nvm_tutorial_record_venue = {
    nvm_get_set = { key = flag:nvm_tutorial_venues }
    if = {
        limit = { local_var:nvm_found = yes }
        scope:nvm_cursor = { nvm_set_add = { value = $location$ } }
    }
}
```

Sets are unordered collections with instant membership lookup. Adding the same venue twice is harmless since sets ignore duplicates. Check membership with `nvm_set_has`:

```
scope:my_set.nvm_set_has = { value = scope:some_location }
# local_var:nvm_found = yes if present
```

---

## Managing the stable

### Transferring a horse between owners

```
c:FRA.var:nvm_tutorial_stable.nvm_get_set = { key = flag:nvm_tutorial_horses }
scope:nvm_cursor.nvm_set_remove = { value = scope:horse_2 }

c:ENG.var:nvm_tutorial_stable.nvm_get_set = { key = flag:nvm_tutorial_horses }
scope:nvm_cursor.nvm_set_add = { value = scope:horse_2 }
```

Remove from one set, add to another. The horse node itself is unchanged; only its membership moves.

### Retiring a jockey

```
nvm_tutorial_retire_jockey = {
    nvm_get_scope = { key = flag:nvm_tutorial_mount }
    if = {
        limit = { local_var:nvm_found = yes }
        scope:nvm_scope = { nvm_remove_scope = { key = flag:nvm_tutorial_jockey } }
    }
    nvm_remove_scope = { key = flag:nvm_tutorial_mount }
    nvm_remove_value = { key = flag:nvm_tutorial_skill }
    nvm_remove_value = { key = flag:nvm_tutorial_experience }
}
```

Before clearing a node's data, clean up references that point to it. Here we follow the jockey's mount reference back to the horse and remove the horse's jockey reference, then clear the jockey's own fields. Since the jockey is a real character (not a pool node), we remove its fields individually rather than calling `nvm_delete_map`.

### Deleting pool nodes

Before deleting a node, remove it from any sets, lists, or scope references that point to it. `nvm_delete_map` only cleans up structural ownership (parent-child links created by `nvm_get_or_set_map`, `nvm_set_map`, etc.). Non-owning references like set membership, list items, and scope references are the caller's responsibility.

```
# 1. Remove from stable's horses set
c:FRA.var:nvm_tutorial_stable.nvm_get_set = { key = flag:nvm_tutorial_horses }
scope:nvm_cursor.nvm_set_remove = { value = scope:horse_1 }

# 2. Unlink jockey's mount reference (if any)
scope:horse_1.nvm_get_scope = { key = flag:nvm_tutorial_jockey }
if = {
    limit = { local_var:nvm_found = yes }
    scope:nvm_scope = { nvm_remove_scope = { key = flag:nvm_tutorial_mount } }
}

# 3. Now safe to delete
scope:horse_1.nvm_delete_map = yes
```

`nvm_delete_map` wipes the node's data and returns it to the pool. Children are orphaned, not deleted. `nvm_delete_tree` does the same recursively, deleting the node and all its children. Be careful with shared children: if a horse's maintenance template is shared with other horses, `nvm_delete_tree` would destroy the template and break every horse that references it. Use `nvm_delete_map` (which orphans children) when nodes share children, and `nvm_delete_tree` only when you own the entire subtree exclusively.

### Iterating all horses in a stable

```
nvm_tutorial_sum_power_callback = {
    scope:nvm_iter_item = { nvm_tutorial_get_power = yes }
    change_local_variable = { name = nvm_tutorial_stable_power add = local_var:nvm_tutorial_power }
}

nvm_tutorial_get_stable_power = {
    set_local_variable = { name = nvm_tutorial_stable_power value = 0 }
    nvm_get_set = { key = flag:nvm_tutorial_horses }
    scope:nvm_cursor = { nvm_for_each_set_item = { callback = nvm_tutorial_sum_power_callback } }
}
```

`nvm_for_each_set_item` iterates every item in a set, calling your callback with `scope:nvm_iter_item` set to each item. Here the callback computes each horse's power and accumulates the total. NVM also provides `nvm_for_each_value`, `nvm_for_each_scope`, and `nvm_for_each_list_item` for iterating other data types.

### Querying all horses across the entire game

```
my_count_callback = {
    change_local_variable = { name = count add = 1 }
}

set_local_variable = { name = count value = 0 }
nvm_for_each_of_type = {
    type = flag:nvm_tutorial_horse
    callback = my_count_callback
}
# local_var:count = total horses in the game
```

`nvm_set_type` (used in our constructors) auto-registers each node in a global type registry. `nvm_for_each_of_type` iterates all live nodes of a given type anywhere in the game, calling your callback with `scope:nvm_iter_item` set to each node. When a node is deleted, it's automatically deregistered.

---

## Traversing the object graph

Once objects are wired together, you can chain through references to reach any point in the graph. Each step uses a different result variable, and the next step reads from the previous one.

```
# Horse > jockey > experience
scope:horse_1.nvm_get_scope = { key = flag:nvm_tutorial_jockey }
scope:nvm_scope.nvm_get_value = { key = flag:nvm_tutorial_experience }
# local_var:nvm_result now holds the jockey's experience
```

`nvm_get_scope` writes to `scope:nvm_scope`. `nvm_get_value` writes to `local_var:nvm_result`. The chain works because each operation reads from the result of the previous step. For longer traversals involving navigation, `nvm_get_map` writes to `scope:nvm_cursor`.

---

## Try it yourself

The complete, runnable implementation lives in four files:

| File                            | Content                                        |
|---------------------------------|------------------------------------------------|
| `nvm_tutorial_constructors.txt` | Horse, Jockey, Stable constructors             |
| `nvm_tutorial_methods.txt`      | Training, racing, assignment, power calculation |
| `nvm_tutorial_setup.txt`        | Game start initialization and race simulation  |
| `nvm_tutorial_tests.txt`        | Assertion-based validation of the data model   |

Enable the tutorial by uncommenting the `on_game_start` block in `nvm_test.txt`. Grep for `NVM_TUT` in `logs/error.log` for output.

At this point you've seen the full pattern: allocate nodes, stamp types, store values and references, link objects into sets, iterate and query across them, and clean up when done. Everything else in the library is a variation on these building blocks. See the [README](README.md) for the complete API.
