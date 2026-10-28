---
tags:
  - ToDo
  - ECS
  - 3D
  - DirectX
  - Shooter
  - Project
status: in-development
updated: 2026-06-28
related:
  - "[[Entity Component System]]"
  - "[[Sparse Set - Component Storage]]"
  - "[[ECS Design]]"
  - "[[ECS_Shooter - Player & Shooting Slice]]"
  - "[[ECS_Shooter - Collision Slice]]"
  - "[[ECS Render Pipeline - Performance]]"
---
# ECS_Shooter

> [!info] At a glance
> A DirectX 11 FPS wave-shooter, mid **OOP → ECS** rewrite. Working now: render pipeline (`RenderSystem`), camera-relative player movement (`MovementSystem` + `InputSystem`), **shooting** (`WeaponSystem` w/ minigun spin-up), enemy spawn + velocity-facing, and bullet↔enemy collision (`CollisionSystem`). Next: enemy despawn / wave scaling + perf pass (see [[ECS Render Pipeline - Performance]]); then enemies damaging the base.
> ⚠️ Not leak-free: 2 known leaks open + 1 new (see *Known memory leaks* below). Architecture theory: [[Entity Component System]] · storage: [[Sparse Set - Component Storage]] · debugging log: [[ECS Render Pipeline - Bugs & Fixes]] · perf: [[ECS Render Pipeline - Performance]].

---
# Current Idea for gameplay/style/vibe:

==Make it fps give the player a minigun( **Mauga** type just mindless shooting ) and have to shoot inmense waves of enemies with different types of enemies ( if time allows it add upgrades to player stats). main objective is to protect a base.==

## Player

#### Movement
- WASD
- Camera
- Sprint
- Jump
#### Combat
- Melee
- Shoot
#### Gun
- Reload
- Shoot

---
# 🏗️ ECS Foundation  (the rewrite)

> [!success] Landed in `refactor!: large-scale refactoring to introduce ECS` (2026-06-15)

### Core building blocks
- [x] `EntityId = uint32_t`  → `Global.h` ✅ 2026-06-15
- [x] `IComponentPool` interface (type-erased base) → `IComponentPool.h` ✅ 2026-06-15
- [x] `SparseSet<T> : IComponentPool` — Add / Remove (self-remove case) / Has (3-check) / Get / iterate / Clear → `SparseSet.h` ✅ 2026-06-15
- [x] `CComponentStore` — `type_index → unique_ptr<IComponentPool>`, `GetOrCreate<T>`, smallest-pool `QueryIds`/`Query` → `CComponentStore.h` ✅ 2026-06-15
- [x] `CEntityManager` — `Create/Delete/IsAlive`, id recycling via free-list (`m_ListFree`) → `CEntityManager.h/.cpp` ✅ 2026-06-15
- [x] `CWorld` façade — entity + component API in one place → `CWorld.h` ✅ 2026-06-15
- [x] Delete legacy OOP god-objects (`CGameObject`, `CStaticMeshObject`, `CProjectileShot`, `CUIObject`, `CSpriteObject`, `CTextRect`, `CSkinMeshObject`, `CAnimCharacter`, `CPlayerController`) ✅ 2026-06-15

### 🎨 Render pipeline (landed `feat(ecs): wire RenderSystem...` — commit `e7075ab`)
- [x] `Component.h` — `MeshComponent { IMesh*; bool visible }`, `Transform` (moved here), `Spin` ✅ 2026-06-15
- [x] `RenderSystem::Render` — `Query<Transform, MeshComponent>` → per-entity world matrix → `IMesh::Render` → `CRenderSystem.h/.cpp` ✅ 2026-06-15
- [x] `CScene` wiring — split `UpdateSceneInfo`/`Draw`; base `Draw` runs `RenderSystem`; camera view/proj synced into `SCENE_DATA` ✅ 2026-06-15
- [x] `CTestScene` — 25-cube shared-mesh grid + `Spin` system (proves scaling, shared mesh, custom system, component filtering) ✅ 2026-06-15
- [x] Fixed all 4 carried-over storage bugs + 4 pipeline bugs → see [[ECS Render Pipeline - Bugs & Fixes]] ✅ 2026-06-15

### ⚙️ Gameplay systems + capsule (landed `feat(ecs): capsule mesh...` — commit `2c33eea`)
- [x] `CStaticMesh::GenerateCapsuleMesh` — procedural capsule (player/enemy placeholder) ✅ 2026-06-19
- [x] `MovementSystem` — `Query<Transform, Velocity>` → `Position += velocity·dt` ✅ 2026-06-19
- [x] `InputSystem` — raw `CInput` → `PlayerInput` intent component (`+ CInput::GetAxis`) ✅ 2026-06-19
- [x] Components: `Velocity`, `Lifetime`, `Weapon`, `PlayerInput` ✅ 2026-06-19
- [x] **Scene lifecycle fix** — build once in `AddScene`; `ChangeScene` only `Start`s (no more duplicate world/entities) ✅ 2026-06-19
- [x] Renamed `CRenderSystem` → `RenderSystem`; rename CScene render call ✅ 2026-06-19
- [x] Fixed 5 more bugs (C2461, `CWorld` by-value `construct_at`, query-by-value no-op, degenerate capsule, scene double-build) → [[ECS Render Pipeline - Bugs & Fixes]] ✅ 2026-06-19

### 🩸 Known memory leaks (2026-06-19 audit; status re-verified vs. code 2026-06-28) — full write-ups in [[ECS Render Pipeline - Bugs & Fixes]]
> [!bug] Leak audit of the entity/scene release paths. **2 of 4 original items fixed; 2 still open; 1 new one introduced in `ff461aa`.**
> - [x] **⭐ `MeshComponent` raw `AnimationController*` leaked on every `DestroyEntity`** — ✅ fixed: `unique_ptr` member (`Component.h:11`) **+** `SparseSet::Remove` does `std::move` (`SparseSet.h:47`).
> - [x] (crash, not leak) `RemoveAll`→`Remove` had no `Has` guard — ✅ fixed: `if (!Has(id)) return;` (`SparseSet.h:32`).
> - [ ] ❌ **STILL LEAKS — `new CSkinMesh()` in `CDemoScene::Create()` (`:46`)** — `LoadData` (`:59`) overwrites `m_pSkinDemoMesh` with manager-owned `loadSkin(...)`; `Release` (`:106`) only frees `m_pWorld`. Fix: drop line 46 (nothing reads it before `LoadData`; first use `:165`).
> - [ ] ❌ **STILL LEAKS — `CStaticMesh::Init` re-call orphans GPU buffers** — the `Release()` at `CStaticMesh.cpp:42` is in the *destructor*; `Init(...)` (`:53`) goes straight to `BuildFromMeshData` with no release. Fix: `Release();` at top of `Init`, or double-init guard.
> - [ ] 🆕 **NEW LEAK (introduced `ff461aa`) — `m_pTextRenderer = new CTextRenderer()` (`CDemoScene::Create :47`)** never deleted; `Release` (`:106`) doesn't touch it. Fix: `SAFE_DELETE` in `Release`, or make it a value / `unique_ptr` member.

### Next ECS steps
- [ ] **PlayerControlSystem** — turn `PlayerInput` + camera basis into camera-relative `Velocity` (flatten forward to XZ, normalize); pick one mover (ECS-driven, camera follows player) to avoid double-movement
- [x] **WeaponSystem (shoot + bullets)** — extracted from `CDemoScene`; ramping fire-rate (minigun spin-up), `ProjectileTag` bullets w/ `Velocity`+`Lifetime` (deferred destroy) → commit `e3fe9dd`, see [[ECS_Shooter - Player & Shooting Slice]] ✅ 2026-06-28
- [ ] **CollisionSystem** — bullet ↔ enemy sphere broadphase → damage + deferred destroy; later a POD `ColliderComponent` (split `CCollider`) → see [[ECS_Shooter - Collision Slice]]
- [ ] **Generations** on `EntityId` (index + version) so recycled ids can't alias dead entities — see [[Entity Component System]] §6. Critical once bullets/enemies churn.
- [ ] Fix `Transform::Rotate` quaternion bug (component-wise `*=`, not `QuaternionMultiply`) — see [[ECS Render Pipeline - Bugs & Fixes]]
- [ ] Migrate `CDemoScene`'s ground / robot / cube meshes to entities (currently only ECS entities render)
- [ ] Decide component set for Player / Bullet / Enemy / Base

### Architecture map
```
CWorld  ── façade
 ├─ CEntityManager   (ids, alive flags, free-list recycling)
 └─ CComponentStore  (type_index → IComponentPool)
        └─ SparseSet<Transform> · SparseSet<Velocity> · SparseSet<Health> ...
Systems → CWorld.Query<...>(fn)
```

---
# 🎮 Gameplay milestones
- [x] choose camera type for the game ✅ 2026-06-09
- [x] integrate basic ECS *(foundation — see ECS Foundation above)* ✅ 2026-06-15
- [x] setup camera ✅ 2026-06-22
- [x] put player in the scene ✅ 2026-06-22
- [x] make player move ✅ 2026-06-22
- [x] make player shoot ✅ 2026-06-22
- [x] appear random enemies ✅ 2026-06-22
- [ ] make enemies interact with bullets *(see [[ECS_Shooter - Collision Slice]])*
- [ ] make a lot of enemies appear *(perf: enemy-leak despawn, per-frame cbuffer hoist, instancing — see [[ECS Render Pipeline - Performance]])*
- [ ] make enemies damage main objective ⏳ 2026-06-15
- [ ] create start menu
- [ ] create clear/game over screen
- [ ] polish
- [ ] publish

---
**Active plan:** [[ECS_Shooter - Collision Slice]] — next milestone (bullets damage enemies). Prior slice: [[ECS_Shooter - Player & Shooting Slice]].

**See also:** [[Entity Component System]] · [[Sparse Set - Component Storage]] · [[ECS Render Pipeline - Bugs & Fixes]] · [[ECS Render Pipeline - Performance]] · [[ECS Design]]
