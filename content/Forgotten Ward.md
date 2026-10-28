---
tags: [game-idea, team-project]
---
# Forgotten Ward (忘却病棟)

> [!info] Team project with friends — proposal originally written by Yan (Oct 27, 2026)

*Forgotten Ward* is a 15–30 minute psychological horror adventure about a patient who wakes in a hospital made of their own mind, and the stranger who came to save them.

| Item | Detail |
| --- | --- |
| Title | Forgotten Ward (working title; JP: 忘却病棟) |
| Team | Team project with friends |
| Genre | Psychological horror / exploration adventure, BL elements |
| Platform | PC (Windows) |
| Engine | Godot, with the Dialogic plugin for dialogue |
| Play time | 15–30 minutes, 2 endings |
| Scope | 4 rooms, 3 characters, 1 event CG |
| Perspective | **3D environments with 2D sprite characters** (flat billboarded sprites in a 3D world, like *Despelote*) |
| Target players | Fans of story-driven indie horror such as *Ib* and *The Witch's House* |
| Purpose | Course assignment for Project 2 (KCG) |

## Visual Style
- The hospital is built as real 3D rooms; characters (protagonist, B, the patients) are 2D sprites standing in that space, like *Despelote*'s flat cutout people in a 3D city.
- The flat characters against solid 3D rooms adds to the unease — they look slightly "out of place", which fits a hospital that is not real.
- Godot: characters as `Sprite3D` / `AnimatedSprite3D` with billboard mode; dialogue via Dialogic as a UI overlay works the same as in 2D.
- Lighting and fog in the 3D rooms carry the horror mood (flickering lights, long corridors, darkness beyond the camera).

## Concept and Theme
The game asks one question: does a person in pain actually want to be saved?

- **Facing trauma**: every room is a part of the protagonist they have avoided looking at.
- **Self-acceptance**: progress comes from understanding each fragment, not defeating it.
- **The cost of salvation**: being saved means waking up, and waking up means forgetting the one who saved you.

Horror here comes from unease rather than jump scares. Familiar hospital objects turn wrong on a second look, and the player slowly realises the place is not real.

## Story
**Setting**: a silent hospital inside the protagonist's dream. Its other patients are fragments of the protagonist's own psyche.

1. **Awakening**: the protagonist wakes in a ward. B is already there, calls himself "another patient", and offers to help find the exit.
2. **The patients**: each patient embodies one emotion, such as fear, guilt and loneliness. Resolving each one's knot yields a memory fragment.
3. **The truth**: the fragments reveal the protagonist's trauma, and show that B never appears anywhere in the protagonist's real memories.
4. **The choice**: B confesses he is an angel who erased his own memories to live as a human, and entered this dream to save the protagonist. If the protagonist wakes, everyone will forget B.

**Endings**
- **Wake up**: the protagonist returns to reality. On the title screen, B's name is corrupted or gone, so the player feels the forgetting firsthand.
- **Stay**: the protagonist remains in the dream with B, refusing to be saved.

## Characters
| Character | Role | Key trait |
| --- | --- | --- |
| Protagonist | Player character; wakes in the ward with gaps in memory | Avoids the past; must choose whether to be saved |
| B | Companion who follows the player | Gentle and calm; an angel who erased his own memories, revealed at the end |
| The patients | Fragments of the protagonist's psyche (e.g. fear, guilt, loneliness) | Each guards one memory fragment |

## Gameplay Systems
No combat: the game is exploration, puzzles and one short chase.

- **Investigation**: objects can be examined. Some change description on a second look, turning unsettling.
- **Fragment puzzles**: each patient's area has one core puzzle tied to its emotion.
  - Fear: a chase through the 3D corridors where the player must remember the safe route.
  - Guilt: find a lost object and return it to its owner.
  - Loneliness: piece together a code from torn diary pages.
- **Talking to B**: B follows the player and can be spoken to anytime. His lines shift as the story advances and carry the foreshadowing.
- **Final choice**: a single two-option choice decides the ending.

**Controls**: WASD to move, mouse to look/turn the camera, E or left click to examine and advance text, Esc for the menu.

## Open Questions
- How the patients fit the 3-character scope (one shared design, or one patient on screen at a time).
