---
tags:
  - td_project
  - Cpp
  - DirectX
  - ImGui
  - ECS
  - Debugging
  - Tools
created: 2026-10-09
commit: 529b142
branch: save1
related:
  - "[[ECS_Shooter]]"
  - "[[Sparse Set - Component Storage]]"
  - "[[ECS Render Pipeline - Bugs & Fixes]]"
---
# td_project — Model Viewer, Orbit Camera & Asset Manifest Editor

> [!abstract] What this is
> Log of one working session on `td_project` (repo `dieghomj/td_project`, branch `save1`, commit `529b142`, not pushed).
> It started as *"skinned meshes only have one animation in `CViewer`"* and ended with an engine-wide **Asset Manifest editor**.
> Bugs are written as **Symptom → Root cause → Fix → How it was found**, features as **Goal → Design → Details**.

---

## Quick index

| # | Change | Kind | Main files |
|---|--------|------|------------|
| 1 | Clip picker never changed the clip | bug | `CScene.cpp` |
| 2 | Entity processed N times after N model picks | bug (ECS) | `SparseSet.h` |
| 3 | Orbit camera moved into `CameraUtil` + 4 camera bugs | refactor + bugs | `CameraUtil.*`, `CViewer.*` |
| 4 | Mouse-wheel zoom unreliable | bug (input) | `CInput.*`, `CMain.cpp` |
| 5 | Stale project entry breaks the build | build | `td_project.vcxproj` |
| 6 | Off-center models → movable asset position | feature | `CViewer.*` |
| 7 | **Asset Manifest editor (F9)** | feature | `Cinema/CAssetManifestEditor.*` |
| 8 | Manifest revision counter keeps every copy in sync | feature / data-loss guard | `CinemaJson.*`, `CCinemaScene.*`, `CViewer.*` |
| 9 | `CAssimpLoader::Probe` | feature | `CAssimpLoader.*` |
| 10 | `loadSkin` built a mesh from a failed load | bug | `MeshResourceManager.h` |

---

## 1. "Skinned meshes only have one animation"

**Symptom** — In the viewer, selecting a clip in the entity inspector did nothing; every skinned model kept playing clip 0.

**How it was found** — Followed the clip data end to end instead of guessing:
`CAssimpLoader::LoadSkinned` (loops over every `aiAnimation` ✅) → `MeshResourceManager::loadSkin` → `AnimationController::Init` / `Clone` (copies all clips ✅) → `AnimationSystem::Update` (evaluates `AnimState::clip`) → `CScene::ShowEntityProperties` ❌.
The data side was checked too: `grep "AnimationSet" ignis.X` → 42 sets, so the loader really receives all of them.

**Root cause** — The inspector edited a *copy* of the component:

```cpp
AnimState animState = m_pWorld->GetComponent<AnimState>(id); // copy
...
animState.clip = i;                                          // thrown away
```

**Fix** — `AnimState& animState = ...` (you also initialised `bool isClosed = true;` in the same function, which was uninitialised and passed to `ImGui::Begin`).

> [!note] Why it *looked* like the model only had one clip
> - Every `AnimationSet` in the Asymmetry `.X` files is named **`Anim1`**, so the list reads `0: Anim1 … 41: Anim1`.
> - Clips are stored in **reverse order**: clip 0 of ignis is `41 発射ポーズ(静止)`, a still pose — so the default looked like "no animation".
> - The real names only exist in `assets.json` → `actors[].clips`.

> [!tip] Lesson
> `T x = world.GetComponent<T>(id)` silently copies. When the goal is to mutate, always bind `T&`.

---

## 2. `SparseSet::Add` stored duplicates

**Symptom** — No visible error, but after picking N models in the viewer, the viewed entity was drawn and animated **N times per frame**, and every old `MeshComponent` (with its cloned `AnimationController`) stayed alive.

**Root cause** — `Add` always appended to `m_Dense` and re-pointed `m_Sparse[id]`, never checking `Has(id)`.
`QueryIds` walks `m_Dense`, so the id appeared N times; `Get(id)` goes through `m_Sparse`, so all N visits returned the *latest* component — hence N draws of the same mesh rather than old meshes.
The invariant `dense[sparse[id]] == id` from [[Sparse Set - Component Storage]] still held; what was missing was **uniqueness of dense**.

**Fix** — `Add` became an upsert:

```cpp
if (Has(id))
{
	m_ComponentData[m_Sparse[id]] = std::move(component); // old controller freed here
	return;
}
```

> [!tip] Lesson
> A sparse set's `Add` must be either *upsert* or *assert-not-present*. Silent append breaks iteration counts without breaking lookups, which makes it hard to spot.

---

## 3. Orbit camera → `CameraUtil`

**Goal** — Move every camera method out of `CViewer` into `CameraUtil`, and fix the viewer camera.

**Bugs found in the old `CViewer::UpdateOrbitCamera`:**

| Bug | Root cause | Fix |
|-----|-----------|-----|
| Camera spun whenever the mouse moved, even while clicking ImGui windows | Mouse delta applied with no button check (the comment said "drag") | Rotate only while LMB is held and `!io.WantCaptureMouse` |
| Zoom did nothing visible | `120 * -0.0005 = 0.06` units per wheel notch on a 5–60 range | Multiplicative zoom, 10 % per notch |
| Skybox lagged one frame | Skybox drew with `m_mView/m_mProj` from last frame's `UpdateSceneInfo` (and zeros on frame 1) | `ApplyOrbitCamera` rebuilds the matrices right away |
| Auto-rotation fought the drag | Always added | Only while idle |

**New API** (`CameraUtil.h`):

```cpp
struct OrbitParams { float Yaw, Pitch, Distance, TargetHeight, MinDistance, MaxDistance,
                     MaxPitch, AutoSpeed, RotateSense, PanSense, ZoomStep; };
void UpdateOrbitInput(OrbitParams& p, float dt);                  // mouse → params
void ApplyOrbitCamera(CCamera&, const XMFLOAT3& target, const OrbitParams&,
                      XMFLOAT4X4& mView, XMFLOAT4X4& mProj);       // params → camera + matrices
```

Input is separated from placement, matching the existing `ApplyFPCamera` / `ApplyTPCamera` style.
The orbit changed from *fixed camera height* to *yaw / pitch / distance*, so zooming keeps the viewing angle. The defaults reproduce the old start view: pitch `0.46 ≈ atan(10/20)`, distance `22 ≈ √(20² + 10²)`.

> [!example] Viewer controls
> **LMB drag** rotate · **RMB drag** raise/lower the look-at point · **wheel** zoom · idle = slow auto-rotate · ignored while over ImGui.

---

## 4. Mouse wheel only registered sometimes

**Root cause** — `MsgProc` called `SetScrollDelta(0)` on *every* window message, and the main loop handles either one message **or** one frame per iteration. A wheel notch only counted if no other message (mouse move, etc.) arrived before the next frame — and if none arrived for several frames, the same notch was applied repeatedly.

**Fix** — `WM_MOUSEWHEEL` now calls `AddScrollDelta` (accumulate); `CInput::Update` turns the accumulated value into this frame's `m_scrollDelta` and resets it. `SetScrollDelta` was removed (only the viewer used it).

---

## 5. Build broken by a stale project entry

The root-level `CameraUtil.cpp` (a duplicate of `Source/CameraUtil.cpp`) had been deleted in the working copy but was still listed in `td_project.vcxproj` → `C1083 cannot open source file`. With both files present it would have been `LNK2005` duplicate symbols instead. Removed the entry.

---

## 6. Off-center models → movable asset position

**Symptom** — Some assets were not centered in the viewer.

**Root cause** — The camera orbited the entity's own `Transform.Position`, so moving the model moved the camera with it. The geometry itself is offset inside the `.X` files: the ignis mesh frame sits at `y = 140`, extinger at `(0, 55.7, 20.9)`.

**Fix** — The camera now orbits a fixed `ORBIT_PIVOT` (world origin). The Mesh List window has a **Position** drag and **Reset Position**. Positions are remembered **per asset** (`m_AssetPositions`, keyed by model path) for the session, and edits made in the entity inspector are kept too.

> [!warning] Why not auto-frame from the mesh bounds?
> For skinned meshes `GetLocalBounds()` is computed from the raw vertex positions *before* the `.X` frame transforms — about 140 units off for ignis — so auto-framing would be wrong.

---

## 7. Asset Manifest editor (F9)

> [!info] First attempt was reverted
> I first built right-click context menus inside the viewer's Mesh List. That wasn't what you wanted — you wanted an **engine-wide window** that edits `assets.json` by **scanning the project resources**. The context-menu code was removed and the viewer went back to the "list + Position" version.

**Design**
- `Cinema::CAssetManifestEditor` is a singleton (same pattern as `CInput` / `CSoundManager` / `CEffect`). `CMain::Update` calls `Draw()` after the scenes, so it works in **every scene**.
- Opened with **F9** (F1 = cinema editor, F11 = perf, F12 = screenshot; F10 is avoided because it opens the Windows system menu) or the **Asset Manifest (F9)** checkbox in the Scene List window.
- **Scanning** — `std::filesystem::recursive_directory_iterator` over `Data/`, classified by extension:

| Category | Extensions |
|----------|-----------|
| Models (actors, stage pieces) | `.x .fbx .obj .gltf .glb` |
| Skies | `.dds` |
| Effects | `.efk .efkefc` |
| Sounds | `.wav .mp3 .ogg` |
| Cinematics | `Data/Cinema/*.json` except `assets.json` |

  Paths are converted through `u8string` so Japanese paths survive, and stored in the same `Data\\...` form as the manifest. A normalised set (lowercase, `\`) is kept for "does this file exist" checks, so there is **no disk access per frame**.

**UI**
- Tabs: Actors / Stages / Skies / Effects / Sounds (+ Menu BGM) / Cinematics (menu order with ↑ ↓).
- Each tab is split: **left** = entries in `assets.json` (editable), **right** = scanned files with **`+`** to add. Registered files are greyed; there is a filter box and *Hide registered*.
- **Red** = file not found in `Data`, **orange** = duplicate name (cinematics refer to assets by name).
- Adding a model as an actor runs `Probe` (see §9) to fill *Skinned* and one clip slot per animation; **Detect from model** re-runs it and keeps names you already wrote.
- New stage pieces copy the previous piece's scale (stage meshes in one set usually share a scale).

**Implementation details worth remembering**
- Erase/move operations are collected during the loop and applied **after** it (never mutate a vector while iterating it in ImGui code).
- `"Actors (%d)###Actors"` and `"Asset Manifest *###AssetManifest"` — the part after `###` is the ImGui ID, so labels can show counts and the unsaved `*` without resetting tab or window state.
- **Revert** loads into a temporary and swaps only on success. **Save** is disabled until the file has loaded once, so a failed load can never write an empty manifest.
- Closing the window keeps unsaved edits; reopening rescans `Data` but only reloads the file when there are no unsaved edits.

---

## 8. Manifest revision counter — keeping every copy in sync

**Problem** — Several places hold their own copy of the manifest:
- `CCinemaScene` loads it once at startup, and its cinema editor's **Save As** writes that copy back to disk → it would have **silently erased** edits made in the new window.
- `CViewer` would never show newly added actors.

**Fix** — `SaveManifest` is the only code that writes `assets.json`, so it increments `s_ManifestRevision` on success and exposes `GetManifestRevision()`. Each holder of a copy compares the number:

| Who | On change |
|-----|-----------|
| `CCinemaScene::Update` | `LoadAssets()` — reload, then register **new** sounds (`CSoundManager::Load` skips known names) and effects (`CEffect::Has` check) |
| `CViewer::Update` | Reload the Mesh List |
| Asset Manifest editor | Reload if there are no unsaved edits, otherwise show a warning |

> [!tip] Lesson
> If there is exactly one write path, a counter on that path is a cheap, reliable "has it changed?" signal — no file watching needed.

---

## 9. `CAssimpLoader::Probe`

Reads the scene with **no post-processing** (through the same `ReadScene`, so the `.X` empty-`SkinWeights` fix still applies) and returns `HasBones` and the `aiAnimation` names. No vertex buffers are built. Prefilled clip names look like `00 Anim1`.

---

## 10. `loadSkin` on a failed load

`MeshResourceManager::loadSkin` ignored the `HRESULT` from `LoadSkinned` and called `CSkinMesh::Init` with empty data. It now returns `nullptr`, so a mistyped path fails cleanly. Kept even though the context-menu attempt that motivated it was reverted.

---

## How the work was done / verified

- **Diagnosis** by reading each code path end to end, plus grepping the data files (`AnimationSet` names, `FrameTransformMatrix` offsets in the `.X` files) instead of guessing.
- **Build check** after every step: `MSBuild td_project.vcxproj -p:Configuration=Debug -p:Platform=x64` — no new warnings.
- **Runtime** — you used the editor in the app: it added the `map_nomal` actor and saved `assets.json`. The camera and viewer changes were not run-tested on my side.
- **Encoding** — project sources are UTF-8 with BOM and CRLF (compiled with `/utf-8`). New files were converted with `unix2dos -m`. CR/LF counts were checked after editing with `sed` in Git Bash, which can change line endings.
- **Commit** — single commit `529b142` on `save1` (includes your `isClosed = true` change and the `map_nomal` actor). Not pushed.

> [!warning] `assets.json` diff noise
> When the editor saves, whole numbers become floats (`0` → `0.0`, `8` → `8.0`) because the JSON writer's `Round3` returns `double`. Harmless, but it makes the first save produce a large diff.

---

## Open follow-ups

- [ ] The viewer ignores `ActorAsset::scale` (always draws at 0.1).
- [ ] Per-asset viewer positions are not saved between sessions (could become an `offset` field in `assets.json`).
- [ ] The new `map_nomal` actor has scale `1.0` — the other stage meshes use `0.025`.
- [ ] Changing the **path** of an already-loaded sound/effect name needs a restart; an open cinematic isn't rebuilt after a manifest save.
- [ ] **Detect from model** blocks the app briefly on big files (`ignis.X` is 23 MB).
- [ ] The Skies tab lists every `.dds`, not just cubemaps.
- [ ] `if (ImGui::Bullet)` in `CScene::ShowEntityProperties` is always true (warning C4551) — probably meant `ImGui::Bullet(); ImGui::SameLine();`.
- [ ] `CViewer::Create` does `m_pSkinDemoMesh = new CSkinMesh();`, which is overwritten by `loadSkin` → leak.
