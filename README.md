# hashmap.h

A small header-only hashmap for C. Open addressing, linear probing, fixed size (it never
grows). You get back a plain `T*`; the bookkeeping lives in a hidden header right before it.

```c
#include "hashmap.h"

typedef struct { token_t a, b; } Pair;
typedef struct { Pair key; uint32_t id; } MergeEntry;

MergeEntry *m = hash_table_create(MergeEntry, 8192, pair_hash);

MergeEntry e = { .key = {a, b}, .id = 42 };
hash_table_put(m, e);

MergeEntry *p = hash_table_get(m, ((Pair){a, b}));
if (p) printf("%u\n", p->id);

hash_table_free(m);
```

Needs GCC or Clang (it uses `__typeof__`).

## Your entry type

- It must be a struct with a field called `key`. Any type, anywhere in the struct.
- Only `key` is hashed and compared. Everything else is just data that gets copied along.
- Keys are compared with `memcmp`, so `key` shouldn't have padding bytes (or zero it first).
- Slots are 8-byte aligned. Don't store types that need 16 (`long double`, `__int128`, SIMD).

## API

| Macro | Returns | What it does |
|---|---|---|
| `hash_table_create(T, cap, hash)` | `T*` or `NULL` | `cap` is rounded up to a power of 2 (`0` → 64). `hash` can be `NULL` to use the built-in FNV-1a. |
| `hash_table_free(h)` | – | Frees everything. |
| `hash_table_put(h, item)` | `bool` | Inserts, or overwrites the entry with the same key. `item` must be a variable of type `T`. Returns `false` only if the table is full. |
| `hash_table_get(h, k)` | `T*` or `NULL` | Looks up key `k`. |
| `hash_table_remove(h, k)` | `bool` | `true` if the key was there. |
| `hash_table_clear(h)` | – | Empties the table, keeps the memory. |
| `hash_table_count(h)` | `size_t` | Entries stored. |
| `hash_table_capacity(h)` | `size_t` | Total slots. |
| `hash_table_full(h)` | `bool` | No free slots left. |

### Gotchas

- **Wrap struct keys in extra parentheses.** The commas inside `{a, b}` confuse the macro:
  ```c
  hash_table_get(m, ((Pair){a, b}));   // ok
  hash_table_get(m, (Pair){a, b});     // compile error
  ```
- **Don't change `p->key` after inserting.** The entry stays where the old key put it and
  you'll never find it again. Remove it and put it back instead. Changing other fields
  (`p->id = ...`) is fine.
- **Pointers from `get` survive `put`** (there's no resize), **but not `remove`**, which
  may slide other entries around. Just call `get` again after removing.

## Writing a hash function

```c
typedef size_t (*hash_func)(const void *key, size_t key_size);
```

You get a pointer to the `key` field and its size. The table only looks at the **low bits**
of what you return, so make sure those depend on the whole key.

For a pair of 16-bit tokens:

```c
static size_t pair_hash(const void *k, size_t len) {
    (void)len;
    const Pair *p = k;
    uint64_t x = ((uint64_t)p->a << 16) | p->b;         // one number per pair
    return (size_t)((x * 0x9E3779B97F4A7C15ull) >> 32); // scramble, keep the good bits
}
```

The multiply spreads similar pairs like `(3,7)` and `(3,8)` far apart, so they don't pile
up next to each other. The `>> 32` is there because the low bits of a product only depend
on the low bits of the input, which here would mean only `b`.

## How it works

### Memory layout

One `malloc`:

```
[ header | T slots[capacity] | used[capacity] ]
           ^ the pointer you get
```

`used[i]` is 1 if slot `i` holds something. You need it because fresh memory is garbage,
and a key of all zeros can be a real key.

### Finding the key inside a slot

The internal functions don't know what `T` is; they just see `sizeof(T)` bytes per slot.
The macros pass them two numbers, worked out by the compiler from your struct:

- `key_off`: how many bytes into the struct `key` starts (same as `offsetof(T, key)`)
- `key_size`: `sizeof(key)`

With those, comparing slot `i` is just
`memcmp(slots + i * sizeof(T) + key_off, key, key_size)`.

### Lookup

Hash the key, mask it to get a slot. If the slot is empty, the key isn't there (and `put`
writes it there). If it holds the same key, found it. Otherwise try the next slot, wrapping
around, at most `capacity` times.

### Removal

You can't just mark the slot empty: anything that was pushed past it during insertion
would become unreachable. So after freeing a slot, `remove` walks forward and pulls back
any entries that would otherwise get lost. No tombstones, no slowdown over time.

## Tips

- It never grows, so size it up front. Linear probing gets slow past ~70% full; just use
  about 2x the max number of entries.
- On a full table, looking up a missing key scans every slot.
- `put` copies the whole entry, so keep entries small.
- Not thread-safe.
