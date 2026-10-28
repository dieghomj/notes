---
tags:
  - ECS
  - Debugging
  - Cpp
  - DirectX
  - Postmortem
created: 2026-06-15
related:
  - "[[ECS_Shooter]]"
  - "[[ECS Render Pipeline - Performance]]"
  - "[[Entity Component System]]"
  - "[[Sparse Set - Component Storage]]"
---
# ECS Render Pipeline — Bugs & Fixes

> [!abstract] What this is
> A debugging log from wiring the first ECS `RenderSystem` into [[ECS_Shooter]] (commit `e7075ab`). Every bug below cost real time; each entry is **Symptom → Root cause → Fix → Lesson** so the *lesson* sticks, not just the patch.

Context: [[ECS_Shooter]] · theory: [[Entity Component System]] · storage: [[Sparse Set - Component Storage]].

> [!tip] For FPS/perf (not correctness) see the companion: [[ECS Render Pipeline - Performance]] — enemy-leak despawn, per-frame cbuffer hoisting, query hash-lookups, instancing.

---

## Quick index

| # | Bug | Class | One-line fix |
|---|-----|-------|--------------|
| 1 | `IterateOverId` didn't override the virtual | compile / vtable | concrete `std::function … const override` |
| 2 | `Get` return type (`T*` vs `T&`, `return nullptr`) | compile | `T& Get` + `assert(Has(id))` |
| 3 | `GetSize` `uint32_t` vs base `size_t` | compile / override | return `size_t` |
| 4 | `CComponentStore::Iterate` called a missing method | compile | rename/route to real iterate methods |
| 5 | Component owning a mesh **by value** | design / memory | store `IMesh*` (pointer, not value) |
| 6 | `static` free function in a header | link | drop `static` (external linkage) |
| 7 | Mesh path `\S` escape | runtime / silent | `\\` every separator |
| 8 | View/proj matrices never filled | runtime / **black screen** | `UpdateViewMatrix(m_mView,m_mProj)` each frame |
| 9 | `m_vRight` uninitialized | **red herring** | (rebuilt each frame; init defensively) |
| 10 | Cube outside the frustum | runtime / placement | put it in front of the camera |
| 11 | **Clear *after* draw** | runtime / **blue screen** | Clear → Render → Present |
| 12 | `Transform::Rotate` not a quaternion mult | latent / math | use `QuaternionMultiply` (worked around) |
| 13 | Popped the wrong stash | git | diff stash vs tree first; extract one line |
| 14 | Component member named same as struct | compile / C2461 | rename the member (`Velocity::value`, not `::Velocity`) |
| 15 | `CWorld` passed **by value** | compile / `construct_at` | pass `CWorld&` (it's move-only via `unique_ptr` pools) |
| 16 | Query lambda takes components **by value** | logic / silent | take them **by ref** (`Transform& tf`) — else you mutate a copy |
| 17 | Capsule **invisible** — `theta` never × slice index | runtime / degenerate | `theta = 2π * s / Slices` |
| 18 | Scene re-built on every switch | memory / duplicate | build once in `AddScene`; pair `Create`↔`Release` |
| 19 | `MeshComponent` raw `AnimationController*` **leaked on `DestroyEntity`** | memory / **leak on release** | own via `unique_ptr` **+** `Remove` must `std::move` |
| 20 | `new CSkinMesh()` in `Create`, overwritten in `LoadData` | memory / leak | drop it — `loadSkin` (manager) owns the mesh |
| 21 | `CStaticMesh::Init` re-called without `Release` | GPU leak | `Release()` at top of `Init` (or double-init guard) |
| 22 | `RemoveAll` → `Remove` with no `Has` guard | crash / UB | `if (!Has(id)) return;` at top of `Remove` |

---

## A. Storage layer — making `SparseSet<T>` actually instantiable

> The whole store hung on `SparseSet<T>` being a *concrete* class. Three separate mistakes kept it abstract or inconsistent, so `std::make_unique<SparseSet<T>>()` in `GetOrCreate` wouldn't compile.

### 1. `IterateOverId` silently failed to override
- **Symptom:** `SparseSet<T>` stayed abstract → `make_unique` error; adding `override` to the existing function errored too.
- **Root cause:** it was a **template** `void IterateOverId(auto&& fn)`, non-`const`. A *function template can never be virtual / never override*, and the signature didn't match the base (`std::function<void(EntityId)> … const`).
- **Fix:** a concrete, `const`, `std::function` override:
  ```cpp
  void IterateOverId(std::function<void(EntityId)> fn) const override {
      for (EntityId id : m_Dense) fn(id);
  }
  ```
- **Lesson:** `override` is your friend — it turns "silently doesn't override" into a compile error. Templates can't be virtual; the signature (incl. `const` and exact param type) must match the base **exactly**.

### 2. `Get` return-type contract
- **Symptom:** wouldn't compile — first `T*` declared but `T&` returned; then `T&` with `return nullptr;`.
- **Root cause:** you can't bind a **reference** to `nullptr`, and a reference return has no representation for "missing."
- **Fix:** pick one contract. Query already guarantees membership, so:
  ```cpp
  T& Get(EntityId id) { assert(Has(id)); return m_ComponentData[m_Sparse[id]]; }
  ```
  Keep a *separate* `T* TryGet` if a nullable lookup is ever needed.
- **Lesson:** don't merge "always present" and "maybe present" into one function. `operator[]` vs `.at()` is the same split.

### 3. `GetSize` return type
- **Symptom:** `uint32_t GetSize() const override` → "does not override".
- **Root cause:** base returns `size_t`; return types are **not covariant** for value types (only for pointers/refs). Different return type ⇒ not an override.
- **Fix:** return `size_t` to match the base.

### 4. `CComponentStore::Iterate` → nonexistent method
- **Symptom:** compile error calling `.Iterate(fn)` on a `SparseSet`.
- **Fix:** renamed/routed to the real methods: `IterateOverComponent` / `IterateOverId` / `IterateOverIdComponent`.

---

## B. Design — why a component holds a pointer, not a mesh

### 5. Never store a resource-owner by value in a sparse set
- **Symptom (would-be):** corruption / double-free if `MeshComponent` held a `CStaticMesh` by value.
- **Root cause:** the sparse set **relocates its elements** — `remove()` does swap-pop (`data[idx] = data.back()`) and `push_back` **reallocates**. Both copy/move every element. A `CStaticMesh` owns raw `ID3D11*` resources and `SAFE_RELEASE`s them in its destructor → copying it leads to double-free / dangling GPU handles, plus copying its vertex/index buffers.
- **Fix:** `struct MeshComponent { IMesh* pMesh; bool visible; };` — 8 trivially-movable bytes, and **many entities share one mesh** (25 cubes, 1 `CStaticMesh`).
- **Lesson:** components in a packed store must be cheap to move. Reference heavy/owning objects by pointer or handle.

---

## C. Linkage

### 6. `static` function declared in a header
- **Symptom:** unresolved/duplicate symbol; C4505 "unreferenced local function removed".
- **Root cause:** `static` at namespace scope = **internal linkage** → every TU including the header gets its *own private* declaration. `CScene.cpp`'s call had no definition in its TU; `CRenderSystem.cpp`'s copy was unused.
- **Fix:** drop `static` — a header-declared / `.cpp`-defined free function needs **external** linkage.
- **Note (7):** `SCENE_DATA` was used in the header without an include; this only worked because `Global.h` is the **precompiled header**, so the type is available everywhere. Fine here, but a header that doesn't rely on the PCH should include/forward-declare what it uses.

---

## D. The "nothing renders" saga 🟦

> Three things conspired; two were red herrings and one was the real killer. Debugged in the wrong order, which is itself the lesson.

### 8. View & projection matrices were never written
- **Symptom:** black/blank — `SCENE_DATA.mView/mProj` fed the WVP but produced nothing.
- **Root cause:** `CScene::m_mView`/`m_mProj` are **value-initialized to the zero matrix** (not identity!) and *nothing* ever pulled the live camera's matrices into them. `world * 0 * 0 == 0`.
- **Fix:** in `CScene::UpdateSceneInfo()`, before populating `SCENE_DATA`:
  ```cpp
  m_pCamera->UpdateViewMatrix(m_mView, m_mProj);
  m_Camera.vPosition = m_pCamera->GetPosition();
  m_Camera.vLook     = m_pCamera->GetForward();
  ```
- **Lesson:** a value-initialized `XMFLOAT4X4` is **all zeros**, not identity. If geometry vanishes, print/inspect the view & projection matrices first.

### 9. Uninitialized `m_vRight` — a red herring
- **Symptom:** ctor seeds `m_vLook`/`m_vUp` but not `m_vRight`, which feeds a Gram-Schmidt cross product in `UpdateViewMatrix`. Looked like the bug.
- **Reality:** `FirstPersonCamera` calls `Pitch`/`Yaw` **every frame before** the view update, and both call `RebuildBasisFromAngles()` which fully rebuilds the basis. So `m_vRight` is always overwritten before use.
- **Fix:** initialized it to `(1,0,0)` **defensively**, but it was never the cause.
- **Lesson:** *verify a suspect is actually on the execution path before "fixing" it.* Plausible ≠ guilty.

### 10. Cube outside the frustum
- **Symptom:** still blank after #8.
- **Root cause:** camera at origin looking **+Z**; cube placed at `(5,1,0)` — zero Z component ⇒ 90° to the side, outside a 45° FOV.
- **Fix:** place it in front (`{0,0,15}`) / aim the camera. Confirm by dragging to look ~90° right.
- **Lesson:** "nothing renders" can just be "nothing is *in view*." Sanity-check placement vs camera before assuming a pipeline failure.

### 11. 🟦 THE blue screen — clearing *after* drawing
- **Symptom:** solid clear-color (blue) no matter what — entities, placement, camera all irrelevant.
- **Root cause:** frame order in `CMain::Update()` was **Render → Clear → Present**:
  ```cpp
  m_pSceneManager->Update();   // draws all meshes
  m_pDx11->ClearBackBuffer();  // ❌ wipes them
  m_pDx11->Present();          // presents a blank buffer
  ```
- **Fix:** the canonical order — **Clear → Render → (UI) → Present**:
  ```cpp
  m_pDx11->ClearBackBuffer();
  m_pSceneManager->Update();
  CImGuiManager::GetInstance()->EndFrame();
  m_pDx11->Present();
  ```
- **Lesson:** when **nothing ever renders regardless of content**, suspect **global frame state** — clear/present ordering, render-target binding, viewport — not individual objects. (This regressed when rendering moved *into* the scene's `Update`, landing it before the clear.)

---

## E. Latent math bug

### 12. `Transform::Rotate` isn't a quaternion multiply
- **Symptom:** repeated rotation would distort/denormalize the orientation.
- **Root cause:** `Rotation *= DXHelper::StoreFloat4(...)` resolves to the **component-wise** `XMFLOAT4 operator*=` in `DXMathAlias.h`, **not** quaternion multiplication.
- **Workaround used:** the `Spin` system sets `Transform.Rotation` directly from `XMQuaternionRotationAxis(axis, accumulatedAngle)` each frame — no accumulation error.
- **Proper fix (TODO):** `Rotation = DXHelper::QuaternionMultiply(Rotation, delta);`
- **Lesson:** overloaded operators can silently do the *wrong* math for a type. Quaternions need real quaternion ops.

---

## F. Git

### 13. The stash was older than the working tree
- **Symptom:** expected a "camera fix" in `stash@{0}`; assumed `git stash pop` would restore it.
- **Reality:** the stash predated current work — its `IMesh` was `Render(SCENE_DATA)` (no world-matrix param), so popping would have **regressed** the render interface `RenderSystem` depends on. The camera-position migration was *already* in the tree; only **one line** (the `UpdateViewMatrix` sync, #8) was genuinely missing.
- **Fix:** `git diff stash@{0} -- <files>` to compare against the tree, then hand-apply only the needed line. Don't pop blind.
- **Lesson:** treat a stash as untrusted until diffed. Pop is all-or-nothing; cherry-pick the hunk you actually need.

---

## G. Components, systems & capsule mesh (2026-06-19)

### 14. Component member shares the struct's name → C2461
- **Symptom:** `error C2461: 'Velocity': constructor syntax missing formal parameters`.
- **Root cause:** the member was named the same as the struct (`struct Velocity { XMFLOAT3 Velocity; };`). A data member **can't** share its class's name — that name is reserved for the constructor, so the compiler read it as a malformed ctor.
- **Fix:** rename the member (`Velocity::value`). Rule: the component is the *container*, the field is the *data* — never the same name.

### 15. Passing `CWorld` by value → `std::construct_at: no matching overloaded function`
- **Symptom:** a cryptic error deep in the STL when calling a system: `InputSystem::HandleInput(CWorld world)`.
- **Root cause:** `CWorld` owns `unordered_map<type_index, unique_ptr<IComponentPool>>`. `unique_ptr` is **move-only**, so `CWorld` is **non-copyable**. Passing it *by value* forces a copy → the map's `construct_at` tries to copy a `unique_ptr` → no viable constructor.
- **Fix:** pass `CWorld&`. Never copy the world. → ties to lesson #3: move-only members make owners non-copyable *on purpose*.

### 16. Query lambda takes components by value → mutates a throwaway copy
- **Symptom:** compiled fine, but movement/input did nothing — the capsule wouldn't move, input wouldn't stick.
- **Root cause:** `Query` hands you `T&`, but `[](EntityId, Transform tf, Velocity v)` **copies** them. You mutate the copy; the real component is untouched.
- **Fix:** take them **by reference** — `Transform& tf, Velocity& v`. By-value in a system loop is almost always a silent no-op bug.

### 17. Capsule mesh invisible from every angle (not winding!)
- **Symptom:** cube/ground rendered, capsule never did, from any viewpoint, no error.
- **Root cause:** `const float theta = XM_2PI / Slices;` — the **`* s` was missing**, so every vertex in a ring used the *same* angle → all ring vertices collapsed to one point → every triangle had zero area → nothing to rasterize.
- **Fix:** `theta = XM_2PI * s / Slices`. The winding was correct all along.
- **Lesson:** **invisible from *everywhere* with no error ⇒ degenerate geometry; invisible only from *outside* ⇒ winding.** (Extends lesson #6.)

### 18. Scene re-built on every switch → duplicate world/entities (and a leak)
- **Symptom:** switching scenes duplicated the world + entities; `Create()` (which `new`s a `CWorld`) ran in both `AddScene` and `ChangeScene` → the first world leaked.
- **Root cause:** `Create()`/`LoadData()` weren't paired 1:1 with `Release()`. `LoadData` re-run on a populated world also duplicates entities.
- **Fix:** build each scene **once** in `AddScene` (`Create` + `LoadData`); `ChangeScene` only `Start()`s the target (and releases the old stack). Keep `Create`↔`Release` paired 1:1; `LoadData` must run on a fresh/empty world.
- **Note:** scenes are owned by the manager's `unique_ptr` dict; the stack holds non-owning raw views — so the *scene objects* never leaked, only the re-`new`'d `CWorld`s.

---

## H. Memory leaks on entity/scene release (2026-06-19 audit)

> Found by tracing the `DestroyEntity → RemoveAll → SparseSet::Remove` and scene `Create/Release` paths. The headline one leaks on **every entity destroyed** — deadly for a wave shooter.

### 19. `MeshComponent`'s raw `AnimationController*` leaks on every `DestroyEntity` ⭐ the one you hit
- **Symptom:** memory climbs as enemies spawn/die.
- **Root cause:** `MeshComponent` holds `AnimationController* pAnimController` (`Component.h:11`); each spawn does `...->Clone()` = `new AnimationController` (`CDemoScene.cpp:151`). `DestroyEntity` → `RemoveAll` → `SparseSet<MeshComponent>::Remove` just pops the struct, and `MeshComponent` has **no destructor** → the clone is never freed.
- **⚠️ Trap:** don't just add `~MeshComponent(){ delete pAnimController; }`. `SparseSet::Remove` swaps with **copy** (`m_ComponentData[idx] = m_ComponentData.back();`) → two components share the pointer → `pop_back` deletes it → dangling + the original idx pointer leaks. That's leak → **leak + double-free**.
- **Fix (two coupled changes):**
  1. Own it: `std::unique_ptr<AnimationController> pAnimController;` (move-only → copies become compile errors).
  2. `SparseSet::Remove` swap must **move**: `m_ComponentData[idx] = std::move(m_ComponentData.back());`.
  3. Call sites: wrap `Clone()` in `unique_ptr`, `AddComponent<MeshComponent>(e, std::move(mc))`, read with `.pAnimController.get()` (RenderSystem, CScene).
- **Lesson:** an owning raw pointer in a component placed in a packed store is a triple footgun (leak on remove, double-free on swap-copy, dangling). Use `unique_ptr` and make the store `std::move` its elements.

### 20. `new CSkinMesh()` in `Create()` is always leaked
- **Root cause:** `CDemoScene::Create()` does `m_pSkinDemoMesh = new CSkinMesh();` (line 44); `LoadData()` overwrites it with `m_MeshManager.loadSkin(...)` (manager-owned) without deleting, and `Release()` only frees `m_pWorld`. The `new`'d mesh is orphaned.
- **Fix:** delete line 44 — the `MeshResourceManager` owns the skin mesh; the scene just holds the non-owning pointer `loadSkin` returns.

### 21. `CStaticMesh::Init` re-call leaks GPU resources
- **Root cause:** `Init` creates DX11 buffers/shaders without releasing prior ones; a second `Init` orphans the first set (GPU memory, not CRT heap).
- **Fix:** `Release();` at the top of `Init`, or bail if already initialized.

### 22. `RemoveAll` calls `Remove` on pools that don't contain the entity (crash, not leak)
- **Root cause:** `CComponentStore::RemoveAll` loops **all** pools; `SparseSet::Remove` has no `Has` guard, so on an absent entity it touches `m_Dense.back()`/`m_Sparse[id]` → UB. Survives today only because entities share component sets.
- **Fix:** `if (!Has(id)) return;` at the top of `Remove`.

> **Non-issue:** `CComponentStore` has no explicit destructor, but `IComponentPool::~IComponentPool()` is `virtual`, so the `unique_ptr` pools destroy their `SparseSet<T>` correctly. No leak there.

---

## Meta-lessons (the transferable bits)
1. **`override` everywhere.** It converts whole classes of silent vtable mistakes into compile errors (#1, #3).
2. **One function, one contract.** Don't fuse present/absent semantics (#2).
3. **Packed stores move their elements** — components must be cheap, non-owning (#5). Move-only members make owners non-copyable; pass them by reference (#15).
4. **Zero-init ≠ identity** for matrices (#8).
5. **Confirm the code path before fixing a suspect** (#9).
6. **Invisible-everywhere ⇒ global state / degenerate geometry; invisible-sometimes ⇒ placement / winding** (#10, #11, #17).
7. **Diff a stash before popping** (#13).
8. **In a system loop, take components by reference** — by-value is a silent no-op (#16).
9. **Pair `Create`↔`Release` 1:1**; build once, mutate per-frame (#18).
10. **A component that *owns* a resource must use `unique_ptr` (or full rule-of-five) AND the store must `std::move` its elements** — a raw owning pointer in a packed store leaks on remove, double-frees on swap-copy, and dangles. `unique_ptr` turns the mistakes into compile errors (#19).
