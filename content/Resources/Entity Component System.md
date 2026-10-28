---
tags:
  - ECS
  - Architecture
  - GameDev
  - Cpp
created: 2026-06-10
related:
  - "[[Sparse Set - Component Storage]]"
  - "[[ECS_Shooter]]"
---
# Entity Component System (ECS)

> [!abstract] What it is in one breath
> An architecture that splits a game object into three separate things: an **Entity** (just an id), **Components** (plain data, no logic), and **Systems** (logic that runs over components). You **compose** behavior by attaching data, instead of **inheriting** it from a class tree.

This is the **hub note**. Deep dives: [[Sparse Set - Component Storage]] · seed: [[ECS Design]] · applied in: [[ECS_Shooter]].

---

## 1. Why ECS exists — the problem with inheritance

Classic OOP game objects use an inheritance tree:

```
GameObject
 └─ Character
     ├─ Player        (mesh + transform + input + health + gun)
     └─ Enemy         (mesh + transform + AI + health)
         └─ FlyingEnemy ... + flight ... + shielded ... ?
```

> [!warning] Where this breaks
> - **The diamond / "where do I put it?" problem.** A flying *shielded* enemy that also drops loot — which base class owns "flight"? "shield"? You end up with deep, tangled trees or duplicated code.
> - **Fat base classes.** `GameObject` grows every field anyone might need; most objects use a fraction of it.
> - **Rigid at runtime.** You can't easily make an entity *gain* flight or *lose* a collider mid-game — its type is fixed at compile time.
> - **Bad memory layout.** Objects are heap-scattered fat structs; iterating "all things that move" chases pointers → cache misses.

> [!success] ECS's answer — composition over inheritance
> Behavior = **the set of components you attach**, decided at runtime. A "flying shielded enemy" is just an entity with `{Transform, Mesh, AI, Health, Flight, Shield, LootDrop}`. Add/remove a component → change behavior live. Data is stored **packed by type** → fast iteration.

---

## 2. The three pillars

| Pillar        | What it is                                                      | Has logic? | Has data? | Example                             |
| ------------- | --------------------------------------------------------------- | ---------- | --------- | ----------------------------------- |
| **Entity**    | An **id** (a number). An identity, nothing more.                | ❌          | ❌         | `42`                                |
| **Component** | **Plain data** (a POD struct).                                  | ❌          | ✅         | `Transform { pos, rot, scale }`     |
| **System**    | **Logic** that queries entities and runs over their components. | ✅          | ❌         | `MovementSystem`: `pos += vel * dt` |

> [!important] The golden rule
> **Components hold no behavior. Systems hold no state.** Data and logic are fully separated. (In [[ECS_Shooter]], `Transform` is already a clean data-only struct — that's the right shape.)

```
Entity (id)  ──owns──▶  Components (data)  ──read/written by──▶  Systems (logic)
   42                    Transform, Health                       MovementSystem
```

---

## 3. The architecture — how the pieces fit

```
                     ┌─────────────────────────────────────┐
                     │              World / Registry         │
                     │   (the façade that ties it together)  │
                     ├───────────────────┬───────────────────┤
                     │  EntityManager    │   ComponentStore   │
                     │  - create() → id  │   map<type, pool>  │
                     │  - destroy(id)    │                    │
                     │  - isAlive(id)    │   ┌──────────────┐ │
                     │  - id recycling   │   │ SparseSet<Transform> │
                     │    + generations  │   ├──────────────┤ │
                     │                   │   │ SparseSet<Health>    │
                     │                   │   ├──────────────┤ │
                     │                   │   │ SparseSet<Velocity>  │
                     │                   │   └──────────────┘ │
                     └───────────────────┴───────────────────┘
                                    ▲
                                    │ query<Transform, Velocity>()
                     ┌──────────────┴───────────────────────┐
                     │               Systems                  │
                     │  RenderSystem · MovementSystem ·       │
                     │  CollisionSystem · ShootSystem ...     │
                     └────────────────────────────────────────┘
```

### 3.1 EntityManager — identities & lifetime
- Hands out ids on `create()`, marks them dead on `destroy()`, answers `isAlive(id)`.
- **Recycles** freed ids so they don't grow forever — which creates the *stale-handle* problem (see [[#6. The hard/remaining parts (forward links)|generations]] below).

### 3.2 ComponentStore — the data
- A map from **component type → a storage pool** for that type.
- Each pool is a [[Sparse Set - Component Storage|sparse set]]: O(1) `add`/`remove`/`has`/`get` and **contiguous, cache-friendly iteration**.
- Holding pools of *different* types in one container needs **type erasure** (`IComponentPool` base + `std::type_index` + `static_cast`) — see [[#6. The hard/remaining parts (forward links)|below]].

### 3.3 Systems — the logic
- Each frame, a system **queries** the store for entities that have a required set of components, then iterates and mutates their data.
- Systems are stateless transforms: `(components, dt) → updated components`.

### 3.4 World / Registry
- The front door. Bundles `EntityManager` + `ComponentStore` and exposes `createEntity()`, `add<T>(id, c)`, `get<T>(id)`, `query<...>()`, `destroyEntity(id)`.

---

## 4. The frame loop — how data flows

```
every frame:
  1. INPUT    → ShootSystem spawns a Bullet entity {Transform, Velocity, Damage}
  2. UPDATE   → MovementSystem:  for (id, tf, vel) in query<Transform,Velocity>:
                                     tf.pos += vel * dt
                → CollisionSystem: bullets vs enemies → apply Damage, mark dead
                → LifetimeSystem:  despawn expired bullets  (destroyEntity)
  3. RENDER   → RenderSystem:    for (id, tf, mesh) in query<Transform,Mesh>:
                                     draw mesh at tf
```

> [!tip] Why this is fast
> Each system iterates **one packed array** ([[Sparse Set - Component Storage|the sparse set's `data[]`]]) straight through. The CPU prefetcher predicts every access → near-max cache throughput. This **data-oriented** layout is the core performance argument for ECS over OOP object graphs.

---

## 5. Queries — "give me entities with X and Y"

A system rarely wants one component type; it wants the **intersection**.

```cpp
// iterate the SMALLEST matching pool, filter the rest
template<typename First, typename... Rest>
void each(auto&& fn) {
    for (auto [id, first] : pool<First>())   // ideally pick the smallest pool
        if ((has<Rest>(id) && ...))          // fold: must have ALL the rest
            fn(id, first, get<Rest>(id)...);
}
```

> [!note] Optimization
> Drive the loop from the **smallest** component pool, because the result can't be bigger than the smallest set. Iterating `Velocity` (50 bullets) and filtering by `Transform` beats iterating `Transform` (5000 objects) and filtering by `Velocity`.

---

## 6. The hard/remaining parts (forward links)

> [!success] Implemented in [[ECS_Shooter]] (2026-06-15)
> - [x] **Storage** — `SparseSet<T>` → `SparseSet.h`. See [[Sparse Set - Component Storage]].
> - [x] **Type erasure** — `CComponentStore` holds many `SparseSet<T>` via `IComponentPool` base + `std::type_index` key + `static_cast`. → `CComponentStore.h`
> - [x] **Queries** — `QueryIds<...>` / `Query<...>` drive from the smallest pool + `HasAll<T...>` filter.
> - [x] **World/Registry façade** — `CWorld` (`CreateEntity` / `DestroyEntity` / `AddComponent` / `Query`). → `CWorld.h`
> - [x] **Entity lifecycle** — `CEntityManager` with free-list id recycling. → `CEntityManager.h/.cpp`
> - [x] **First systems** — `RenderSystem` (`Query<Transform, MeshComponent>` → draw) and a `Spin` system (`Query<Transform, Spin>` → rotate). → `CRenderSystem.*`, `CTestScene.*`. Getting these working surfaced a chain of bugs → [[ECS Render Pipeline - Bugs & Fixes]].

> [!todo] Not yet done
> - [ ] **Generations** — recycled ids still alias dead entities (`EntityId` is a bare `uint32_t`). Pack `index + generation`; bump on destroy so stale handles fail `IsAlive()`. Important for [[ECS_Shooter]] (bullets/enemies churn fast). → next topic
> - [ ] **MovementSystem** — `Transform += Velocity * dt` (a `Velocity` component; `Spin` is the template).
> - [ ] **System scheduling** — ordering, and later parallelism (systems touching disjoint components can run concurrently).
> - [ ] **Archetypes vs sparse sets** — the *other* major ECS storage model (group entities by their exact component set). Trade-offs vs the sparse-set approach we built.

---

## 7. ECS vocabulary cheat-sheet

| Term                     | Meaning                                                                    |                |
| ------------------------ | -------------------------------------------------------------------------- | -------------- |
| **Entity**               | An id. No data, no logic.                                                  |                |
| **Component**            | A POD data struct attached to an entity.                                   |                |
| **System**               | Stateless logic iterating components each frame.                           |                |
| **Registry / World**     | Façade owning entities + component storage.                                |                |
| **Pool / Storage**       | Per-type container of components (here: a [[Sparse Set - Component Storage | sparse set]]). |
| **Query / View**         | Set of entities that have a required component combination.                |                |
| **Archetype**            | Alternative storage: group entities by their exact component set.          |                |
| **Generation / Version** | Counter packed into an id to detect stale handles.                         |                |

---

### Key takeaways
1. **Entity = id, Component = data, System = logic** — strictly separated.
2. **Composition over inheritance**: behavior is the *set of components*, changeable at runtime.
3. Storage is **per-type and packed** → data-oriented, cache-friendly iteration ([[Sparse Set - Component Storage]]).
4. Systems **query** for component combinations, then iterate and mutate.
5. The pieces: **EntityManager** (ids) + **ComponentStore** (data) + **Systems** (logic), bundled by a **World/Registry**.

---
**See also:** [[ECS Design]] · [[Sparse Set - Component Storage]] · [[ECS_Shooter]]
