---
tags:
  - ECS
  - Performance
  - Cpp
  - DirectX
  - Postmortem
created: 2026-06-28
related:
  - "[[ECS_Shooter]]"
  - "[[ECS Render Pipeline - Bugs & Fixes]]"
  - "[[Entity Component System]]"
  - "[[Sparse Set - Component Storage]]"
---
# ECS Render Pipeline — Performance

> [!abstract] What this is
> Companion to [[ECS Render Pipeline - Bugs & Fixes]], but for **FPS**, not correctness. Symptoms here are "the game gets slower as enemies pile up," not "it crashes." Same format: **Symptom → Root cause → Fix → Lesson.** Ordered by impact.

Context: [[ECS_Shooter]] · theory: [[Entity Component System]] · storage: [[Sparse Set - Component Storage]].

The shooter spawns immense enemy waves, so **everything that costs *per live entity* gets multiplied by hundreds and by 60 fps.** A 5 µs waste × 300 enemies × 60 fps ≈ 90 ms/s of pure waste. The bottleneck is almost always the **CPU submitting work**, not the GPU drawing it — these meshes are tiny.

---

## Quick index

| # | Problem | Class | Cost shape | One-line fix |
|---|---------|-------|-----------|--------------|
| P1 | Enemies never despawn (too-tight `<= 0.1` check) | leak / unbounded | grows forever | despawn on a real condition (distance/`Lifetime`) |
| P2 | `CBUFFER_PER_FRAME` Mapped once **per mesh** | redundant upload | O(entities) | upload once per frame (frame-stamp guard) |
| P3 | `Query`/`GetOrCreate` double hash-lookup per entity | per-entity overhead | O(entities × types) | cache the pool pointer; single `find` |
| P4 | No instancing; every enemy is an animated skinned draw | draw-call bound | O(entities) | (future) instancing + animation LOD / frustum cull |

---

## P1. Enemies never despawn → unbounded entity growth ⭐ the real FPS killer

- **Symptom:** FPS degrades steadily the longer a wave runs; memory climbs; never recovers.
- **Root cause:** the despawn check only destroys an enemy when it's almost exactly on the target:
  ```cpp
  if (DXHelper::Length3(tf.Position) <= 0.1f) { /* destroy */ }
  ```
  A fast enemy moves more than 0.1 units per frame, so it **steps over** the 0.1-radius sphere and is never inside it. Enemies that reach the base are never removed → the live-entity count only ever **grows**. Since every downstream system (render, movement, collision, animation) is O(entities), the whole frame gets linearly slower forever.
- **Fix:** despawn on a condition the entity can't skip past:
  - distance threshold matched to speed (e.g. `Length3 <= radius` with `radius ≥ maxSpeed*dt`), **or** crossing the target plane, **or**
  - a `Lifetime` component (already exists) / explicit "reached base → apply damage → destroy" event.
  - Collect ids during the query, `DestroyEntity` **after** it (deferred destroy — see [[ECS Render Pipeline - Bugs & Fixes]] #19).
- **Lesson:** an "arrived?" test against a moving object must be a **half-space or a radius ≥ per-frame step**, never an exact point. In a spawner, *a despawn bug is a performance bug*: anything you forget to destroy is multiplied by every per-entity system every frame.

---

## P2. `CBUFFER_PER_FRAME` is uploaded once **per mesh**, not once per frame

- **Symptom:** frame time scales with enemy count more steeply than the draws alone explain; CPU-bound.
- **Root cause:** `CSkinMesh::Render` (and `CStaticMesh::Render`) each `Map`/`memcpy`/`Unmap` the per-frame constant buffer (camera, light, fog) **every call**. `RenderSystem` calls `Render` once per entity, so identical bytes are re-uploaded N times per frame.

  **Why re-uploading is expensive (not just "redundant"):**
  - **CPU↔GPU are async.** When you `DrawIndexed`, the GPU runs it *later*. A constant buffer the draw reads must stay valid until the GPU is done. Overwriting it mid-flight would corrupt earlier draws — so the driver must manage the hazard at `Map` time.
  - **`D3D11_MAP_WRITE_DISCARD` = buffer renaming.** "Give me fresh memory, ignore the old contents." The driver rotates through a **pool** of physical allocations so you can write without waiting. This is the *fast* path — but each call still pays a round-trip into the runtime + user-mode driver (validation, the immediate-context lock, rename-pool bookkeeping). One call is µs; the cost is **µs × N × 60 fps**.
  - **Rename-pool exhaustion → real stalls (the cliff).** Discard the same buffer hundreds of times per frame and the pool can run dry (GPU hasn't released in-flight blocks). Then `WRITE_DISCARD` can no longer hand you a fresh block immediately and **blocks the CPU** until the GPU drains. Smooth → cliff.
  - **It makes you CPU-bound.** Tiny meshes shade fast; if the CPU is busy Mapping/binding/drawing ×N, the GPU **starves**. Removing N−1 Maps frees CPU time → the CPU feeds the GPU sooner → higher FPS, identical image.
  - **`Map` vs *bind* asymmetry.** `VSSetConstantBuffers` just writes a handle into pipeline state — no allocation, no sync. `Map` touches the memory/hazard machinery. So hoist the **Map**, not the bind.
  - **Category error:** a *per-frame* buffer is constant for the frame; uploading it *per object* pays per-object cost for per-frame data. Per-object data (world matrix, bone palette) legitimately changes each draw — re-mapping *those* is correct and unavoidable.
- **Fix — frame-stamp guard** (minimal, keeps the shared `enCBSlot::Frame` slot working for both mesh classes):
  1. `Global.h`: add `uint64_t FrameIndex = 0;` to `SCENE_DATA`.
  2. `CScene::UpdateSceneInfo()`: `++m_SceneInfo.FrameIndex;` each frame.
  3. Thread `frameIndex` into the inner `Render` overload; add `uint64_t m_PerFrameStamp = UINT64_MAX;` to `CSkinMesh`/`CStaticMesh`.
  4. Guard the upload but **keep the bind outside** (the other mesh class rebinds the same slot):
     ```cpp
     if (frameIndex != m_PerFrameStamp) {
         // Map(WRITE_DISCARD) + memcpy_s + Unmap   // upload once
         m_PerFrameStamp = frameIndex;
     }
     VSSetConstantBuffers(enCBSlot::Frame, ...);     // bind every draw (cheap)
     PSSetConstantBuffers(enCBSlot::Frame, ...);
     ```
  - **Cleaner end-state (later):** one per-frame CB *owned by the render system*, bound once before the loop. Per-mesh classes stop owning `m_pCBufferPerFrame` entirely.
- **Lesson:** upload data at the frequency it actually changes — **per-frame data once per frame, per-object data per object.** In D3D11 the dominant cost is per-call CPU overhead, so deleting redundant `Map`s is pure win.

---

## P3. `Query` / `GetOrCreate` does two hash lookups per entity per type

- **Symptom:** death-by-a-thousand-cuts CPU overhead at high entity counts; not a cliff, just a tax.
- **Root cause:** `CComponentStore::GetOrCreate<T>()` does a `find` *and* (on the create path) an `emplace`, and `Query` calls it through `QueryIds` → `HasAll` → and again to build the `pools` tuple. Per entity per component type you pay repeated `std::type_index` hashing + `unordered_map` probes — for components that already exist.
- **Fix:** resolve each pool pointer **once** before the loop and reuse it:
  - `Query` already builds `std::tuple<SparseSet<T>*...> pools` — make `HasAll` and the membership test use those cached pointers instead of calling `GetOrCreate` again.
  - Single `find` in `GetOrCreate`; only `emplace` on the genuine miss (it already does, but the create-vs-find split can be tightened).
- **Lesson:** hot loops should hash **once** and iterate on raw pointers. `unordered_map<type_index, …>` is fine for *setup*, a tax for *per-entity* work. (Archetype/grouped iteration is the structural answer; pointer-caching is the cheap interim win.)

---

## P4. Every enemy is an individual animated skinned draw (future work)

- **Symptom:** even with P1–P3 fixed, hundreds of enemies = hundreds of `DrawIndexed` + per-mesh cbuffer churn.
- **Root cause:** no instancing; each enemy uploads its own world matrix + bone palette and issues its own draws per material subset. CPU skinning advances animation per entity every frame regardless of visibility.
- **Fix (directions, not yet done):**
  - **Hardware instancing** for static/identical meshes (one draw, per-instance world matrix in a vertex buffer).
  - **Frustum culling** — skip `Render` for off-screen entities (set `visible=false` or filter in the query).
  - **Animation LOD** — advance/skin distant or off-screen enemies less often (or not at all).
  - Batch by mesh so per-mesh state (vertex/index buffers, material) is set once per group.
- **Lesson:** the long-term ceiling for a wave shooter is *draw-call count and per-entity CPU work*. Instancing + culling + LOD are the levers; P1–P3 are the cheap wins to do first.

---

## Meta-lessons
1. **In a spawner, per-entity cost is the whole game.** Anything O(entities) is multiplied by the wave size and by 60 fps. Audit per-entity work first.
2. **A despawn/cleanup bug *is* a performance bug** — forgotten entities slow every system forever (P1).
3. **Upload at the frequency data changes**, not per object (P2). Per-frame once, per-object per-object.
4. **In D3D11 the bottleneck is CPU submission overhead**, not pixels, for many small meshes — kill redundant API calls (`Map`, `Set*ConstantBuffers`) before optimizing shaders.
5. **Hash once, iterate on pointers** in hot loops (P3).
6. **Profile to confirm** (PIX / RenderDoc): look for stacked `Map` calls on the CPU timeline and idle gaps on the GPU timeline — that's CPU-bound.
