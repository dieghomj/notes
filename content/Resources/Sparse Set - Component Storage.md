---
tags:
  - ECS
  - DataStructures
  - Cpp
  - Performance
created: 2026-06-10
related:
  - "[[ECS Design]]"
  - "[[ECS_Shooter]]"
---
# Sparse Set — Component Storage

> [!abstract] TL;DR
> A **sparse set** stores one component type so that **both** core ECS queries are O(1) and cache-friendly:
> 1. *Random access* — "does entity `N` have this component? give me a ref" (by id)
> 2. *Iteration* — "walk every component of this type, packed, back-to-back" (for systems)
>
> It does this with **two arrays that point at each other**, holding the invariant `dense[sparse[id]] == id`.

Part of [[ECS Design]] · used by [[ECS_Shooter]].

---

## 1. The problem storage must solve

An **entity** is just an id (a number). **Components** (`Transform`, `Velocity`, `Health`…) are plain data kept in **per-type containers**. For one component type, storage must answer two questions *every frame*:

1. **Random access** — by id: `does entity 42 have a Transform? give me a ref`
2. **Iteration** — walk every `Transform` that exists, contiguously

> [!important] These two pull in opposite directions. That tension is the whole story.

---

## 2. Why the two naive solutions hurt

### Option A — flat array indexed by id  `transforms[42]`
- Random access: **perfect**, one index op, O(1).
- Iteration: **broken**. Entities `3, 900, 50000` → array of 50001 slots, ~49997 empty holes. Walking it trashes the cache and wastes memory.

### Option B — `unordered_map<EntityId, Transform>`
- Random access: O(1)-ish, fine.
- Iteration: **slow**. Entries scattered across the heap → a cache miss per element. Death for thousands of entities/frame.

> [!summary] The dilemma
> **Array** = fast iteration but wastes memory on sparse ids.
> **Map** = handles sparse ids but iterates slowly.
> We want **both**: dense packed iteration *and* O(1) lookup by arbitrary id.

---

## 3. The key idea — split *where it is* from *what it is*

The flat array wastes space because it forces **two unrelated things to share one index**:
- the entity's **identity** (id — can be huge/sparse), and
- the component's **storage position** (which we *want* packed 0,1,2,3…).

Stop forcing that. Keep component data **tightly packed** (great for iteration) and keep a **separate lookup table** translating `entity id → packed slot`.

```
sparse[]  : indexed by entity id  → packed slot     ("where is it?")
dense[]   : packed 0..N           → which entity owns each slot ("whose is it?")
data[]    : packed 0..N           → the component itself, mirrors dense[]
```

> [!tip] Why `sparse` can be huge & holey but `data` cannot — *the cost of a hole*
> - A hole in `sparse` = one `uint32` (**4 bytes**). And we **never iterate `sparse`**, only random-access it — holes don't hurt random access. Cheap.
> - A hole in `data` = a **whole `Transform`** (40+ bytes), *and* we **do** iterate `data`, so holes wreck both memory and cache.
>
> **Move the holes to the cheap array we never walk; keep the array we walk perfectly dense.**

> [!note] Optional optimization
> "Paged" sparse arrays chop `sparse` into ~4 KB pages, allocating a page only when an id in its range appears. Same idea, lazier allocation.

---

## 4. Concrete trace — `add(5)`, `add(2)`, `add(9)`

```
            slot:   0    1    2
dense  = [   5,   2,   9 ]      // packed, insertion order
data   = [  T5,  T2,  T9 ]      // T5 = entity 5's Transform, etc.

            id:  0  1  2  3  4  5  6  7  8  9
sparse = [  -,  -,  1,  -,  -,  0,  -,  -,  -,  2 ]   // '-' = INVALID
                     ↑           ↑              ↑
                e2→slot1     e5→slot0       e9→slot2
```

- Random access "get entity 9": `slot = sparse[9] = 2; return data[2] = T9`. Two reads, O(1).
- Iteration: `for slot in 0..size: use data[slot]` — contiguous, prefetcher-friendly. `dense[slot]` tells you *which entity* owns each element.

---

## 5. The invariant  `dense[sparse[id]] == id`

> [!important] The round-trip
> Start with an id → `sparse` sends you **out** to a slot → `dense` sends you **back** to an id → you land on the **same id**.
> `id → slot → id`, the loop closes.

```
sparse[9] = 2     // entity 9 lives at slot 2
dense[2]  = 9     // slot 2 is owned by entity 9
⇒ dense[sparse[9]] == 9   ✓
```

`dense` and `data` are **parallel arrays of different types**: `dense[slot]` = the owning **id** (a number), `data[slot]` = the **component** (a struct). They are never equal to each other.

> [!warning] What a *broken* round-trip means
> If `dense[sparse[id]] != id`, it almost always means **`id` is simply not a member** (never added, or removed) — `sparse[id]` holds stale/garbage that points at a slot owned by *someone else*. This is the **normal, expected** answer for any id you didn't add, **not** memory corruption. (Genuine corruption is the rare second cause → guard with `assert`.)

---

## 6. Operations

### `add` — 3 writes, O(1)
New element goes at slot `N = size`:
```cpp
sparse[id] = N;              // (1) id → new slot
dense.push_back(id);         // (2) slot N records "owned by id"
data.push_back(component);   // (3) slot N holds the payload
```
Keeps the invariant true *by construction*: we write `sparse[id]=N` and `dense[N]=id` in the same breath.

### `has` — the invariant *is* the membership test
```cpp
bool has(EntityId id) const {
    return id < sparse.size()          // A: id in range of sparse
        && sparse[id] != INVALID       // B: not an explicit hole
        && dense[sparse[id]] == id;    // C: round-trip closes → real member
}
```
> [!tip] Why check **C** and not just **B**
> **B** catches holes we *explicitly* marked INVALID. **C** catches the sneaky case where `sparse[id]` holds a *stale-but-plausible* index that accidentally points at a live slot **owned by someone else**. Without C you'd say "yes" and hand back the wrong entity's data.
> *(C alone can actually carry membership; B is a cheap early-out and makes intent obvious.)*

### `get` — 2 reads, O(1)
```cpp
T& get(EntityId id) { return data[sparse[id]]; }   // assert(has(id)) in debug
```

### `remove` — swap-pop, 4 writes, O(1)
Goal: keep `data` **gap-free** without an O(n) shift. We **overwrite the hole with the last element, then shrink**.

> [!info] Why the **last** element is free to move
> Nothing lives *after* it, so relocating it disturbs no other element's position. Every other element has neighbors depending on it staying put.

Trace `remove(2)` (slot 1) on the set above:
```cpp
idx  = sparse[2] = 1;          // the hole we create
last = dense.back() = 9;       // the free agent to move

dense[idx] = last;             // dense[1] = 9
data[idx]  = data.back();      // data[1]  = T9   (move)

sparse[last] = idx;            // sparse[9] = 1   (fix the MOVED element)
sparse[id]   = INVALID;        // sparse[2] = INVALID  (kill the removed one)

dense.pop_back();              // drop the now-duplicate tail
data.pop_back();
```
Result — packed, gap-free, invariant intact:
```
dense  = [ 5,  9 ]
data   = [ T5, T9 ]
sparse = [ -, -, -, -, -, 0, -, -, -, 1 ]
```

> [!danger] The ordering trap (the #1 bug here)
> When you remove the element that is **already last**, then `id == last`, so `sparse[id]` and `sparse[last]` are the **same slot** — the "fix the moved element" write and the "invalidate" write **collide**. Whichever runs **second wins**.
>
> **Rule: invalidate LAST.** Do `sparse[last] = idx;` *then* `sparse[id] = INVALID;` so in the self-remove case INVALID wins. Reverse the order and you "un-kill" the entity → `has` returns true on a dead id and `get` reads out of bounds.
>
> *(Alternative: special-case `if (id == last) { pop + invalidate; return; }` — also fine.)*
>
> **Always unit-test:** "remove the only element" and "remove the last element." These **aliasing edge cases** — two logical things collapsing onto one physical slot — are where the bugs hide.

### Iteration
```cpp
for (size_t i = 0; i < data.size(); ++i)
    fn(dense[i], data[i]);     // dense[i] = owning id, data[i] = component
```
`data` is a flat `std::vector<T>` → sequential reads, near-max cache throughput. **This is the whole payoff over `unordered_map`.**

> [!caution] Iteration order is NOT stable
> Swap-pop moves the last element into the removed slot, so `dense`/`data` order changes on every `remove`. Systems must **not** depend on iteration order.

---

## 7. Complexity summary

| Op | Cost | Notes |
|---|---|---|
| `add` | O(1) | 3 writes (+ amortized `sparse` resize) |
| `has` | O(1) | the 3-check invariant test |
| `get` | O(1) | 2 array reads |
| `remove` | O(1) | 4 writes, swap-pop, **invalidate last** |
| iterate | O(n) | contiguous, cache-friendly |
| memory | flat | holes only in cheap `sparse` (4 B each) |

---

## 8. Reference implementation (single type)

```cpp
template<typename T>
struct SparseSet {
    static constexpr uint32_t INVALID = UINT32_MAX;

    std::vector<uint32_t> sparse;  // EntityId → index in dense/data
    std::vector<EntityId> dense;   // packed list of alive EntityIds
    std::vector<T>        data;    // component data, mirrors dense

    void add(EntityId id, T c) {
        assert(!has(id));
        if (id >= sparse.size()) sparse.resize(id + 1, INVALID);
        sparse[id] = (uint32_t)dense.size();
        dense.push_back(id);
        data.push_back(std::move(c));
    }

    bool has(EntityId id) const {
        return id < sparse.size()
            && sparse[id] != INVALID
            && dense[sparse[id]] == id;
    }

    T&  get(EntityId id)    { assert(has(id)); return data[sparse[id]]; }
    T*  tryGet(EntityId id) { return has(id) ? &data[sparse[id]] : nullptr; }

    void remove(EntityId id) {
        assert(has(id));
        uint32_t idx  = sparse[id];
        EntityId last = dense.back();
        dense[idx] = last;
        data[idx]  = std::move(data.back());
        sparse[last] = idx;        // fix moved element FIRST
        sparse[id]   = INVALID;    // invalidate LAST (wins on self-remove)
        dense.pop_back();
        data.pop_back();
    }

    void each(auto&& fn) {
        for (size_t i = 0; i < data.size(); ++i) fn(dense[i], data[i]);
    }
    size_t size()  const { return dense.size(); }
    bool   empty() const { return dense.empty(); }
};
```

---

## 9. Implementation status in [[ECS_Shooter]]

> [!success] Implemented (2026-06-15 ECS refactor)
> - [x] **`SparseSet<T>`** itself → `SparseSet.h` (Add / Remove with self-remove case / 3-check Has / Get / iterate / Clear, `INVALID = UINT32_MAX`, raw slot — matches option (b) above).
> - [x] **Type erasure / `ComponentStore`** → `CComponentStore.h` — many `SparseSet<T>` held via `IComponentPool` base + `std::type_index` map + `static_cast` in `GetOrCreate<T>()`.
> - [x] **Queries** → `CComponentStore::QueryIds<...>` / `Query<...>` — drives the loop from the **smallest** matching pool, filters the rest with `HasAll<T...>`.

> [!done] Resolved (2026-06-15)
> - [x] **Carried-over storage bugs fixed** — `Get` now returns `T&` (asserts membership), `IterateOverId` is a concrete `std::function … const override`, `GetSize` returns `size_t`, iterate helpers renamed. Full write-up: [[ECS Render Pipeline - Bugs & Fixes]].

> [!todo] Still to do
> - [ ] **Entity lifecycle + generations** — `CEntityManager` recycles ids via a free-list, but `EntityId` is still a bare `uint32_t`, so a stale handle can alias a recycled entity. Pack `index + generation` into `EntityId`; bump generation on destroy so stale handles fail `IsAlive()`. *High value for [[ECS_Shooter]] (projectiles churn fast).* → next topic

---

### Key takeaways to remember
1. **Move holes to the cheap array (`sparse`) you never iterate.**
2. **`dense[sparse[id]] == id`** is the invariant — and *is* the membership test.
3. **Swap-pop** keeps `data` packed in O(1); **invalidate last** to survive the self-remove aliasing case.
4. Iteration is contiguous → the real win over `unordered_map`.
5. Iteration order is unstable — never depend on it.
