---
tags:
  - ToDo
  - ECS
  - Shooter
  - Plan
  - GameDev
status: planned
created: 2026-06-23
related:
  - "[[ECS_Shooter]]"
  - "[[ECS_Shooter - Player & Shooting Slice]]"
  - "[[Entity Component System]]"
  - "[[Sparse Set - Component Storage]]"
  - "[[ECS Render Pipeline - Bugs & Fixes]]"
---
# ECS_Shooter — Collision Slice

> [!goal] Milestone
> Bullets ↔ enemies actually interact: a bullet that overlaps an enemy **damages it and despawns**, a dead enemy despawns. Turns *Spawn → Move → Cull* into a **game**. First cut is a **lightweight sphere broadphase** over `Transform` — *not* per-bullet `CCollider`.
> Parent: [[ECS_Shooter]] · follows [[ECS_Shooter - Player & Shooting Slice]].

> [!note] Mental model — two tiers of collision
> The swarm (hundreds of short-lived bullets + enemies) wants **cheap data, no objects**: a position + radius, tested by a pure function. The few **structural** collisions (player/base vs enemy, rotated shapes) want precise OBB math + a penetration vector for pushback. Same math, different weight. → **tag-based sphere broadphase for the swarm, precise path only for the handful.**

---

## Part A — The lightweight `CollisionSystem` (DONE — first cut)

Written to `Source/CollisionSystem.{h,cpp}` (already in the `.vcxproj`). Brute-force O(bullets×enemies) sphere test on `Transform.Position`, radii passed as params — **no `CCollider`, no per-frame collider sync, no heap.**

```cpp
namespace CollisionSystem
{
    void ResolveBulletHits(CWorld& world,
        float bulletRadius = 0.25f,
        float enemyRadius  = 1.0f,
        float bulletDamage = 1.0f);
}
```
Core loop (the three hard-won rules baked in: query by ref, deferred destroy, `CWorld&`):
```cpp
std::vector<EntityId> bullets = world.QueryIds<ProjectileTag, Transform>();
std::vector<EntityId> enemies = world.QueryIds<EnemyTag, Transform, Health>();
// ... snapshot ids, then:
std::vector<EntityId> toDestroy;
for (EntityId b : bullets) {
    const auto bp = world.GetComponent<Transform>(b).Position;
    for (EntityId e : enemies) {
        Health& hp = world.GetComponent<Health>(e);
        if (hp.health <= 0.f) continue;
        if (DistSq(bp, world.GetComponent<Transform>(e).Position) > hitDistSq) continue;
        hp.health -= bulletDamage;
        toDestroy.push_back(b);
        if (hp.health <= 0.f) toDestroy.push_back(e);
        break;                              // 1 bullet → at most 1 enemy
    }
}
for (EntityId id : toDestroy)
    if (world.IsAlive(id)) world.DestroyEntity(id);   // AFTER the loops
```

### Wiring still needed (`CDemoScene::Update`)
- [ ] Bullet spawn (~`CDemoScene.cpp:184`): `m_pWorld->AddComponent<ProjectileTag>(b, {});`
- [ ] Enemy spawn (~`CDemoScene.cpp:163`): `m_pWorld->AddComponent<EnemyTag>(e, {});`
- [ ] After `MovementSystem::Move` (~`:135`): `CollisionSystem::ResolveBulletHits(*m_pWorld);` (+`#include`)
- [ ] Build + confirm enemies take hits and despawn

---

## Part B — Why **not** today's `CCollider` for the swarm

`CCollider::Init()` (`CCollider.cpp:27`) `new`s a `CBoundingCube` **+** `CBoundingSphere` **+** `CDebugColliderRender` for **every** collider — even a sphere. Per-bullet at wave scale that's a flood of heap + GPU allocations for what is mathematically a distance check. Plus there's **no collider→transform sync system** yet (`UpdateCollider` is never called per frame), and `ColliderComponent` holds a **raw** `CCollider*` — ownership trap vs. the `unique_ptr` rule. The math in `CCollider` is *already* `static`/pure, so the cheap path needs none of the wrapper.

> [!info] Where `CCollider`'s math genuinely earns its place
> Rotated/precise hits with pushback — player/base vs enemy — via `SphereVsOBB` + penetration vector (`CCollider.cpp:287`). Few, long-lived, worth the shape math.

---

## Part C — Refactor toward a **lightweight `ColliderComponent`**

`CCollider` conflates four jobs. Split them:

| Responsibility | Now | Wants to be |
|---|---|---|
| Shape data (local center/radius, local half/min-max, enum) | `CCollider` members | **POD component** |
| World bounds (`m_pBSphere/m_pBCube`, `RecalculateWorldBounds`) | heap, rebuilt on every `Set*` | computed inline from `Transform` at test time |
| Collision math (`SphereVsSphere/OBB/AABB`) | `static` methods (already pure) | **free-function namespace** `Collide` |
| Debug render (`m_pColliderRenderer`, `DebugDraw`) | one renderer **per** collider | **one shared** renderer in a system |

**1. POD component** (`Component.h`) — trivially copyable, sidesteps the `unique_ptr`/`std::move`-on-`Remove` rule entirely:
```cpp
struct ColliderComponent
{
    enum Shape { Sphere, Box } shape = Sphere;
    DirectX::XMFLOAT3 localCenter = { 0, 0, 0 };
    float             localRadius = 0.5f;                 // sphere
    DirectX::XMFLOAT3 localHalf   = { 0.5f, 0.5f, 0.5f }; // box half-extents
    bool              isDebug     = false;
};
```
**2. Math namespace** `Collide` (new `Collision.h/.cpp`) — near-verbatim lift, on *values* not `CBoundingSphere*`:
```cpp
namespace Collide {
    bool SphereVsSphere(XMFLOAT3 cA, float rA, XMFLOAT3 cB, float rB, XMFLOAT3* pen = nullptr);
    bool SphereVsOBB(XMFLOAT3 sc, float sr, XMFLOAT3 boxC, XMFLOAT3 halfWS,
                     const XMFLOAT4X4& boxRot, XMFLOAT3* pen = nullptr);
}
```
**3. World bounds inline** — no `UpdateCollider`, no heap. Same formula as `RecalculateWorldBounds` (`CCollider.cpp:174`): `worldCenter = transform · localCenter`, `worldRadius = localRadius · maxScale`.
**4. Shared debug system** — one `CDebugColliderRender` owned by the scene; `ColliderDebugSystem` iterates `Transform + ColliderComponent` where `isDebug`. One GPU object, not N.

### Migration order (each step builds green)
- [ ] **1. Extract math** into `namespace Collide`; `CCollider` methods *delegate*. Pure refactor, zero behavior change — commit alone.
- [ ] **2. Add POD `ColliderComponent`** + a one-time `MakeColliderFromMeshData(MeshData&)` helper (reuse local-bounds extraction at `CCollider.cpp:83,111`).
- [ ] **3. Point `CollisionSystem` at POD + `Transform` + `Collide::`** (swap hardcoded radii for `localRadius`). Part A *is* this step for spheres.
- [ ] **4. Move debug render** to `ColliderDebugSystem`; drop `m_pColliderRenderer` from `CCollider`.
- [ ] **5. Delete `CCollider`** (and `CBoundingSphere`/`CBoundingCube` if unreferenced — grep first). Precise path lives in `Collide::SphereVsOBB`.

---

## ⚠️ Gotchas
> [!warning] Deferred destroy across **two** sets
> Collect bullet *and* enemy ids, destroy after both loops. Guard each `DestroyEntity` with `IsAlive` — a dead enemy can be queued by multiple bullets in one frame.

> [!warning] Generations come next, and collision is why
> This system *creates* heavy spawn/destroy churn → recycled `EntityId`s. Add **generations** on `EntityId` (next milestone) before this runs hot, or stale ids will alias fresh entities.

> [!warning] Don't rebuild bounds 3× per move
> Today `UpdateCollider` fires `RecalculateWorldBounds` once per `Set*` (×3). The POD path computes once, at test time, only for queried entities.

## ✅ Acceptance
Hold fire into the wave → bullets that reach an enemy make it vanish (and vanish themselves); enemies whose `Health` hits 0 despawn. No frame hitches at `MAX_SPAWNS`.

## Next — broadphase
- [ ] Precompute world AABBs into a scratch `vector` each frame, **uniform grid** bucket → O(N×M) → ~linear. POD's packed layout is what makes this cache-fast (pointer-chasing `CCollider*` wouldn't).
