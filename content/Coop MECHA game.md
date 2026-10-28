
# Main Idea

A group of friends ( 2 - 4 players) have to coordinate together to drive a mecha

## Core Gameplay Ideas

- Coordinate to walk , run, jump etc (physics system and procedural animations used to move the mech)
- Coordinate to attack and use the best out of each person abilities 
- Traverse a long distance while encoutering certain enemies objects structures and different situations
## Additional Ideas 

Each player can detach and control their own mini mecha to do certain actions

---

# Concept

## Control Model: Hybrid (Stations + Combo Moments)

Each player runs a **station** (a system of the mech). Most of the time stations work on their own, but some big actions need **two or more players at the same time** (combo moments).

### Stations
- **Pilot (Legs)**: walk, turn, run, balance
- **Left Arm / Right Arm**: melee, grab, block, guns
- **Engineer (Core)**: sends a limited **power pool** to legs / arms / shields, handles heat, repairs damage

### Combo Moments (examples)
- **Jump** = Pilot jumps + Engineer vents power at the same time
- **Heavy punch** = both arms together
- **Climb / pull up** = Pilot + arms

### Shared Resources = Coordination
Stations share **Power**, **Heat** and **Balance**, so one player's choice affects everyone (e.g. Pilot runs → arms get weaker).

### Scaling by Player Count
| Players | Setup |
|---|---|
| 4 | Pilot, Left Arm, Right Arm, Engineer |
| 3 | Pilot, Arms (both), Engineer |
| 2 | Pilot + Engineer, Arms |

→ In code, stations are separate modules and one player can own several.

## Mini Mechs
- A player can **detach** into a small mini mech
- Their station is left **empty** → the big mech gets weaker while they are away (real risk/reward)
- Uses: scouting, repairing the outside of the mech, small tunnels, opening doors, hitting enemy weak points

## The Journey
The long trip is built from **segments**:
**Traversal → Encounter → Set Piece → Rest / Upgrade**
Each segment tests a different mix of roles:
- Narrow bridge → Pilot balance
- Enemy swarm → Gunners
- Boss → Everyone

## Open Questions
- [ ] Do players **walk around inside** the mech (physical interior) or **pick stations from a menu**?
- [ ] Setting / art style?
- [ ] How does the mech get upgraded between segments?

---

# Technical Plan (Unity 6.6, URP)

## Architecture
```
Player (human)
  └─ PlayerInputSource ──► intents ("move", "fire", "route power")
                              │
MechController (one per mech, runs on the host)
  ├─ MechCore         → health, power pool, heat, balance
  ├─ Stations[]       → LegsStation, ArmStation(L/R), EngineStation
  ├─ MechLocomotion   → Rigidbody body + procedural legs
  └─ ComboResolver    → checks for moves that need two players at once
MiniMech (separate pawn a player can possess)
```

**Key rule:** stations never read the keyboard directly. They read an **intent struct** from whichever player sits in them → makes local co-op, networking and bots easy.

## Key Tech Choices
- **Movement**: Rigidbody body + simulated balance value + procedural IK legs (Animation Rigging). NOT a full active ragdoll.
- **Interior**: if players walk inside, put the interior in a **fixed spot** in the scene (not parented to the moving mech) and show the outside through screens/windows.
- **Networking**: Netcode for GameObjects + Unity Multiplayer Services (Relay/Lobby). **Host authoritative**: clients only send their inputs.
- **Data**: weapons, arm modules, enemies, segments as **ScriptableObjects**.
- **Local co-op first** (PlayerInputManager + gamepads) → networking after the fun is proven.

## Folder Layout
```
Assets/_Project/
  Scripts/ Core/ Mech/ Stations/ Locomotion/ Players/ MiniMech/ Enemies/ Net/
  Data/  Prefabs/  Scenes/  Art/
```

---

# Milestones

- [ ] **1. Mech moves**: block body + procedural IK legs, one gamepad, feels heavy
- [ ] **2. Split control**: 2 gamepads locally: Pilot + Arms, one punch, shared power pool
- [ ] **3. First combat**: simple NavMesh enemy, damage, one combo move
- [ ] **4. Networking**: move to NGO (host authoritative) + Relay/Lobby
- [ ] **5. Engineer + scaling**: Engineer station, works with 2 / 3 / 4 players
- [ ] **6. Mini mechs**: detach / re-dock, one task that needs the mini mech
- [ ] **7. Journey**: 2 or 3 segments linked together

> Milestones 1–2 decide if the game is fun. Don't start networking before they feel good.
