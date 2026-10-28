---
tags:
  - ToDo
  - ECS
  - Shooter
  - Plan
  - GameDev
status: planned
created: 2026-06-16
related:
  - "[[ECS_Shooter]]"
  - "[[Entity Component System]]"
  - "[[ECS Render Pipeline - Bugs & Fixes]]"
---
# ECS_Shooter — Player & Shooting Slice

> [!goal] Milestone
> First **playable** vertical slice: the player (= the FP camera) holds left-mouse to **spray bullets** that fly out and despawn. No collision yet. Proves the full ECS gameplay loop — **Spawn → Move → Cull** — running as independent systems over shared pools.
> Parent: [[ECS_Shooter]] · pattern reference: `CTestScene` + `RenderSystem`.

> [!note] Mental model
> In an FPS **the player *is* the camera** — movement already works via `CCameraController` (WASD + mouse). New work = **shooting**: a player entity holding weapon state, and bullets that are just entities with `Transform + MeshComponent + Velocity + Lifetime`.

---

## Step 1 — New components (`Component.h`)
```cpp
struct Velocity { DirectX::XMFLOAT3 value{ 0.f, 0.f, 0.f }; };
struct Lifetime { float secondsLeft = 2.f; };
struct Weapon   { float fireInterval = 0.08f; float cooldown = 0.f; };  // ~12.5 shots/sec
```
- [x] Add `Velocity` ✅ 2026-06-19 *(member is `velocity`, not `value`)*
- [x] Add `Lifetime` ✅ 2026-06-19
- [x] Add `Weapon` ✅ 2026-06-19 *(fields named `cadence`/`damage`)*
- [x] Add `PlayerInput` + `InputSystem` (device → intent) ✅ 2026-06-19
- [x] `MovementSystem` (velocity → position) ✅ 2026-06-19
- [ ] **PlayerControlSystem** — `PlayerInput` + camera basis → camera-relative `Velocity` (flatten fwd to XZ, normalize); pick one mover to avoid double-movement

## Step 2 — Systems (namespace style, like `RenderSystem`)
```cpp
// MovementSystem: everything with a velocity moves
void MovementSystem::Run(CWorld& world, float dt)
{
    world.Query<Transform, Velocity>(
        [dt](EntityId, Transform& tf, Velocity& v) { tf.Position += v.value * dt; });
}

// PlayerShootSystem: weapon entities fire bullets along the camera aim
static constexpr float BULLET_SPEED = 60.f;
void PlayerShootSystem::Run(CWorld& world, CCamera& cam, float dt, bool fireHeld, IMesh* bulletMesh)
{
    world.Query<Weapon>([&](EntityId, Weapon& w)
    {
        if (w.cooldown > 0.f) w.cooldown -= dt;
        if (fireHeld && w.cooldown <= 0.f)
        {
            w.cooldown = w.fireInterval;
            const DirectX::XMFLOAT3 origin = cam.GetPosition();
            const DirectX::XMFLOAT3 dir    = cam.GetForward();   // normalized

            EntityId b = world.CreateEntity();
            Transform bt; bt.Position = origin; bt.Scale = { 0.2f, 0.2f, 0.2f };
            world.AddComponent<Transform>(b, bt);
            world.AddComponent<MeshComponent>(b, MeshComponent{ bulletMesh, true });
            world.AddComponent<Velocity>(b, Velocity{ dir * BULLET_SPEED });
            world.AddComponent<Lifetime>(b, Lifetime{ 2.f });
        }
    });
}

// LifetimeSystem: age bullets, despawn expired ones (DEFERRED destroy)
void LifetimeSystem::Run(CWorld& world, float dt)
{
    std::vector<EntityId> dead;
    world.Query<Lifetime>([&](EntityId id, Lifetime& lt)
    {
        lt.secondsLeft -= dt;
        if (lt.secondsLeft <= 0.f) dead.push_back(id);
    });
    for (EntityId id : dead) world.DestroyEntity(id);   // AFTER the query
}
```
- [ ] `MovementSystem::Run`
- [ ] `PlayerShootSystem::Run`
- [ ] `LifetimeSystem::Run`

## Step 3 — Scene wiring
Members:
```cpp
CStaticMesh m_BulletMesh;
EntityId    m_Player;
```
`LoadData()`:
```cpp
const MeshData bullet = CStaticMesh::GenerateCubeMesh(1.f);
if (FAILED(m_BulletMesh.Init(*m_pDx11, bullet))) return E_FAIL;

m_Player = m_pWorld->CreateEntity();
m_pWorld->AddComponent<Weapon>(m_Player, Weapon{ 0.08f, 0.f });
```
`Update()` — **order matters**:
```cpp
const float dt = static_cast<float>(m_pTime->GetFixedDeltaTime() / 1000.0);
CInput* in = CInput::GetInstance();

m_pCameraController->FirstPersonCamera(in->GetMouseDelta(), in->GetMouseSense());
m_pCameraController->Update(dt);

const bool fire = in->GetKey(VK_LBUTTON);             // held = minigun spray
PlayerShootSystem::Run(*m_pWorld, *m_pCamera, dt, fire, &m_BulletMesh);  // spawn
MovementSystem::Run(*m_pWorld, dt);                                      // move
LifetimeSystem::Run(*m_pWorld, dt);                                      // cull

CScene::Update();   // SCENE_DATA + Draw → RenderSystem renders everything
```
- [ ] Add `m_BulletMesh` + `m_Player` members
- [ ] Init bullet mesh + create player entity in `LoadData`
- [ ] Wire the 4 calls into `Update` in the right order
- [ ] Add the new system files to the `.vcxproj`

## ⚠️ Gotchas (the ECS lessons baked in)
> [!warning] Deferred destruction
> Never `DestroyEntity` inside a `Query` — destroy swap-removes from pools and corrupts the in-flight loop. **Collect dead ids, destroy after.** (`LifetimeSystem` does this.)

> [!warning] Don't mutate the pool you're iterating
> `PlayerShootSystem` spawns bullets but never gives them a `Weapon`, so the queried pool is untouched. If you ever add the *same* component you're querying, defer it. Also: don't reuse a queried `Transform&`/`Weapon&` after a spawn — `push_back` can reallocate the pool.

## ✅ Acceptance — what "done" looks like
Aim at the `CTestScene` cube grid, hold **left mouse** → a stream of small cubes flies out along your look direction at 60 u/s and vanishes after 2 s. That's `Spawn → Move → Cull` as three independent systems over shared pools.

---

## 🚀 Landed (commit `e3fe9dd`, 2026-06-28)
> [!success] Shooting extracted into its own system; diverged from the plan in a few useful ways.
- **`WeaponSystem::Shoot(world, mesh, origin, dir, dt)`** — replaces the inline shoot block that lived in `CDemoScene::Update` (and the planned `PlayerShootSystem`). Queries `<PlayerInput, Weapon>`, spawns bullets with `Transform + MeshComponent + Velocity + Lifetime + ProjectileTag`. Files added to the `.vcxproj`.
- **Ramping fire rate (minigun spin-up):** `Weapon` gained `fireInterval` / `deltaFireInterval` / `minFireInterval` / `maxFireInterval`, plus `bulletLifetime` and `bulletSpeed`. While fire is held, `fireInterval` ramps **down** from `max` → `min` (`max(min, fireInterval - delta)`); released, it snaps back to `maxFireInterval`. `cadence` is the per-shot countdown. *(Supersedes the flat `fireInterval`/`cadence` from Step 1.)*
- **`PlayerInput` field rename for clarity:** `move/look/fire/jump` → `moveDir/lookDir/isFire/isJump` (propagated to `InputSystem`, `MovementSystem`, and the `CScene` ImGui inspector).
- **Enemies:** now carry `EnemyTag`, spawn-facing fixed (yaw quaternion from velocity dir), `ENEMY_SPEED` constant; bullet↔enemy hookup via `CollisionSystem::ResolveBulletHits` (see [[ECS_Shooter - Collision Slice]]).
- **Perf win folded in:** `CComponentStore::Query` now caches pool pointers (one map lookup per **type**, not per **entity**) — this is **P3** in [[ECS Render Pipeline - Performance]].
- **Debug-only input grab:** ImGui entity inspector + cursor lock now gated behind `_DEBUG` so the editor build doesn't trap the mouse.
- Scene bookkeeping: `health` entity-id field renamed `m_PlayerId`; player given a `Weapon`; `CTextRenderer` member added; skybox DDS moved to `Data/Skybox/`, unused textures/collision meshes deleted.

> [!todo] Still open from this slice
> - **PlayerControlSystem** as a clean split (currently camera-relative velocity lives in `MovementSystem::UpdatePlayerVelocity`).
> - **Enemy despawn is still the buggy `<= 0.1f` point check** — fast enemies step over it → unbounded growth. This is **P1** (the top FPS killer) in [[ECS Render Pipeline - Performance]].
> - **Per-frame cbuffer hoist (P2)** not yet applied.

---

## Next slice — Collision (becomes a *game*)
> [!info] Promoted to its own note → [[ECS_Shooter - Collision Slice]]
> First cut **landed** as a lightweight sphere broadphase (`Transform` + radius, `ProjectileTag`/`EnemyTag`/`Health`) — deliberately *not* per-bullet `CCollider`. The note also covers the refactor that splits `CCollider` into a POD `ColliderComponent` + a `Collide::` math namespace + a shared debug-render system.
> *(Original stub — the early `Collider { float radius; }` idea — superseded by that note.)*

## Rough effort (see [[ECS_Shooter]] for full breakdown)
- This slice (player + shooting): **~1–2 days**
- Playable slice incl. enemies + collision: **~1–2 weeks** part-time
- The hard part (ECS plumbing) is **done** — everything here is "make entity → attach components → write a small system."
