# AGENTS.md — Pokémon Red 3D: Single Source of Truth

**This file is the constitution for this project.** Any AI agent (or human) making changes must read this file in full before touching code. If something in a prompt conflicts with this file, this file wins. If this file conflicts with reality (the actual code), fix the file in the same session you discover the mismatch — a stale SSOT is worse than no SSOT.

Last verified against the live repo: **2026-07-23**, via full clone + `tsc --noEmit` + `npm run build` + `npm audit`. Everything under "Ground Truth" below is measured, not assumed.

---

## 1. Prime Directive

Build a solo, browser-native 3D remake of the original Gen 1 **Pokémon Red**, with:
- **Pinpoint accuracy** to the original game — real species, real stats, real maps, real trainers, real story beats.
- **Better visuals** — original 3D low-poly assets per `ASSET_PIPELINE.md`'s budgets, not the original's 2D sprites.
- Built primarily through AI coding agents, one scoped session at a time.

Everything in this document exists to make that achievable by a solo developer without the codebase collapsing under its own weight — which is exactly what started happening before this document existed (see Section 3).

---

## 2. Ground Truth — Current State

Do not trust prior chat summaries, prior agent claims, or this section's own memory of itself. Verify with the commands in Section 8 before relying on any status below.

| Area | Status | Evidence |
|---|---|---|
| Engine | React 19 + React Three Fiber + Three.js + Zustand + Vite 6 + TS 5.8 | `package.json` |
| Species data | 14/151 on `main` as of last audit. **A 151-species patch was generated and delivered to the project owner but its merge status into `main` is unconfirmed — verify before assuming either state.** | `src/game/pokemonData.ts`, `PHASE1_SESSION_REPORT.md` |
| Maps | 1 (Pallet Town), hardcoded inline in `World.tsx` and `MapData.ts`. A multi-map `MapDefinition`/`MapId` schema was designed (Section 6) but **implementation was interrupted mid-session and is not delivered.** Treat Phase 2 as not started until verified otherwise. | `src/game/MapData.ts`, `src/components/3d/World.tsx` |
| TypeScript strictness | **`strict` mode is OFF** in `tsconfig.json`. `npm run lint` is just `tsc --noEmit` against a non-strict config — passing it means much less than it looks like. | `tsconfig.json` |
| Tests | **Zero.** No `*.test.*` / `*.spec.*` files anywhere. | full repo search |
| CI | **None.** No `.github/workflows`. | repo root listing |
| Dependencies | `express`, `dotenv`, `@google/genai` are declared but **never imported anywhere in `src/`** — dead weight from the original AI Studio template. | `grep -rl "express\|@google/genai" src/` → empty |
| Known vulnerabilities | 2 (1 low, 1 moderate) via `npm audit` — `body-parser` and `protobufjs`, both **transitive dependencies of the dead packages above**. Removing the dead deps removes both CVEs. | `npm audit` |
| `any` usage | 14 instances, worst offender `src/game/soundManager.ts` repeating `(state as any).soundEnabled ?? true` six times instead of typing the field once. | `grep -rn ": any\|as any" src/` |
| God files | `src/components/3d/World.tsx` (1,454 lines, single component, one map hardcoded inline) and `src/store/gameStore.ts` (1,044 lines, 5 unrelated domains in one store) | `wc -l` |
| Bundle | Single unsplit JS chunk, 1,519.98 kB (416.47 kB gzip) — Vite itself warns about this. | `npm run build` output |
| License | None present. | repo root listing |
| Documentation | `README.md` is unedited AI Studio boilerplate (542 bytes, doesn't describe the game). `ASSET_PIPELINE.md` is genuinely good — specific, followed conventions, worth preserving as-is. | file contents |
| Git history | 4 commits, all one day (2026-06-22), single author. Bus factor of 1 — expected and fine for a solo project, just don't pretend otherwise. | `git log` |

**Any agent's first action on this repo should be re-running the verification commands in Section 8 and updating this table if reality has moved.** Do not add features on top of an assumed state you haven't checked.

---

## 3. Why This Document Exists

Two real incidents already happened on this project without it:
1. An agent session overwrote `App.tsx`/entry files with unrelated audit-report content, with no commit checkpoint to recover from cleanly.
2. The dependency tree accumulated dead, vulnerable packages (`express`, `@google/genai`) from a template that were never pruned, sitting unnoticed until an audit caught them.

Both are process failures, not one-off bad luck. Section 8's session protocol exists specifically to prevent both from recurring.

---

## 4. The 10/10 Definition

"10/10" is not a vibe, it's every row below being true simultaneously, re-verified by the commands in Section 8:

| Category | 10/10 bar |
|---|---|
| Code Quality | `strict: true` in `tsconfig.json`, zero `any` outside justified/commented exceptions, no duplicated unsafe casts. |
| Architecture | No file over ~400 lines doing more than one job. Maps, species, moves, trainers, items all data-driven (Section 6), not hardcoded JSX/if-chains. |
| Technical Debt | Zero unused dependencies. Zero dead code paths (`server.js`-style phantom references). Zero TODO/FIXME left unresolved without a linked tracking note. |
| Testing | Unit tests covering: battle damage formula, type effectiveness, evolution logic, warp/collision logic. CI runs them on every push and fails the build if red. |
| Security & Supply Chain | `npm audit` clean. No secrets in source (already true — keep it true). LICENSE present and accurate. |
| Build & DevEx | Bundle code-split (`manualChunks` or dynamic `import()`) so no single chunk trips Vite's 500kB warning. Clean `npm install && npm run dev` with no manual steps beyond `.env` setup. |
| Documentation | `README.md` accurately describes the actual game, setup, and controls — not template boilerplate. This file (`AGENTS.md`) kept current every session. |
| Performance | Stable frame rate with the full 151-species roster and full map set loaded (verify visually — build passing is not sufficient evidence here). |
| Maintainability | Any single agent session can find, scope, and complete one task using this file alone, without needing prior chat context. |
| Content Completeness | All 151 species, all ~40 Kanto locations, all 8 gyms + Elite Four, real Gen 1 battle formula. This is the biggest bar and the one earlier sections of this doc exist to make achievable. |
| Production Readiness | Everything above, plus a save system that persists full game state, and a full playthrough (starter → Champion) completable without a crash. |

Nothing here is aspirational fluff — every bar traces directly to a specific finding in the audit (Section 2 / audit report). If a future audit finds something this list doesn't cover, add a row; don't let the list go stale.

---

## 5. Non-Negotiable Rules For Any Agent

These are hard constraints, not suggestions. Violating them is grounds for reverting the session's work wholesale.

1. **Never touch `App.tsx`, `main.tsx`, or `index.html`** unless the task is explicitly about entry/routing. This is the exact failure mode from Section 3, incident 1.
2. **Commit after every scoped session, before starting the next one.** A session with no commit checkpoint is a session that can't be safely reverted if it goes wrong.
3. **One scoped task per session.** "Add species #30–45 to `pokemon.json`" is a task. "Improve the game" is not. If a request is vague, narrow it before starting, don't guess big.
4. **Never mix data changes and engine changes in one commit.** "Added 15 Pokémon" and "refactored battle store" are two commits, always.
5. **Never claim a task is done without running the verification commands in Section 8.** Self-reported completion without `tsc --noEmit` + `npm run build` passing is not "done" — this is exactly how the dead-dependency and non-strict-mode debt accumulated silently.
6. **No verbatim copyrighted flavor text.** Original Pokédex descriptions are Nintendo/Game Freak's copyrighted text — write original descriptions (see the existing `genericDescription()` pattern in the Phase 1 data pipeline), never copy game text verbatim.
7. **No new `any` casts.** If the type system is fighting you, that's signal to fix the underlying type, not suppress it. Existing `any` usages in `soundManager.ts` are tracked debt (Section 4), not precedent.
8. **Stone/trade/friendship evolutions are not auto-triggered.** The current `checkEvolution()` design deliberately skips them until an item/trade system exists (Phase 4). Don't silently force them by level as a shortcut.
9. **Don't hand-author a new map's content by guessing.** If real Gen 1 map layouts, trainer rosters, or encounter tables are needed, source them the way Phase 1 sourced species data — from an authoritative dataset (e.g. `@pkmn/dex`-adjacent sources), not from memory alone. Document the source in the commit message.
10. **If you can't visually verify a 3D/rendering change, say so explicitly** in your handoff instead of claiming it's confirmed working. `tsc`/`vite build` passing proves the code compiles, not that the scene renders or plays correctly.

---

## 6. Architecture Reference

### 6.1 Current (as of Section 2's ground truth)
```
src/
  App.tsx, main.tsx          — entry, DO NOT touch casually (Rule 1)
  game/
    pokemonData.ts           — species/move/evolution/wild-encounter data + logic (target: 151 entries)
    MapData.ts                — single hardcoded Pallet Town map (target: replaced by maps/ below)
    soundManager.ts           — singleton audio; has the `any`-cast debt (Section 2)
    usePlayerControls.ts      — input handling
  store/
    gameStore.ts               — single Zustand store, 5 domains in one file (target: split, Section 6.2)
  components/
    3d/World.tsx               — single-map 3D scene, 1,454 lines (target: split, Section 6.2)
    3d/Player.tsx, PokemonModel.tsx, etc.
    ui/BattleUI.tsx, PokedexModal.tsx, InventoryModal.tsx, PcBoxModal.tsx, QuestLog.tsx, Minimap.tsx, GameMenuController.tsx
```

### 6.2 Target architecture (what Phases 2–6 below build toward)
```
src/
  data/
    pokemon.json / pokemon.ts       — canonical 151-species data (Section 6.3 schema)
    moves.json                       — canonical move data
    maps/                            — one MapDefinition per location (Section 6.4 schema)
      palletTown.ts, route1.ts, viridianCity.ts, ...
      types.ts, index.ts (registry)
    trainers.json                    — trainer battle rosters keyed by map id
    items.json
  store/
    battleSlice.ts, inventorySlice.ts, questSlice.ts, pcBoxSlice.ts, playerSlice.ts
    index.ts                         — composes slices into the single useGameStore hook consumers already use
  components/3d/
    World.tsx                        — thin shell: reads active map from store, delegates rendering
    map-props/                       — extracted building/NPC/terrain components, reusable across maps
```
Nothing here changes the public shape consumers already import (`useGameStore`, `POKEDEX_CATALOG`, `checkEvolution`, etc.) — the split is internal, so it can happen incrementally without breaking `BattleUI.tsx`/`PokedexModal.tsx`/etc.

### 6.3 Canonical species/move schema
This is the schema Phase 1's data pipeline already outputs (`PokedexEntry` in `pokemonData.ts`). Keep new entries conforming to it:
```ts
interface PokedexEntry {
  name: string; num: string; types: string[];
  description: string;        // original text, never copied game flavor text (Rule 6)
  color: string;
  evolutionChain: string[];
  baseHp: number;              // arcade-scaled (real HP / 2) — used by current damage system
  moves: { name: string; power: number; type: string }[];  // arcade-scaled power (real power / 8, min 1)
  baseStats: { hp: number; attack: number; defense: number; special: number; speed: number }; // REAL Gen 1 numbers, reserved for Phase 3
  evoLevel: number | null; evoType: string | null; evoItem: string | null;
}
```

### 6.4 Canonical map schema
Designed but not yet implemented (Section 2) — the next agent picking up Phase 2 should build to this exact shape rather than re-deriving it:
```ts
enum TileType { GRASS, PATH, WATER, SOLID, TALL_GRASS }
interface Warp { toMap: MapId; spawnX: number; spawnZ: number; }
interface MapDefinition {
  id: MapId; name: string; width: number; height: number;
  grid: number[][];                              // grid[z][x]
  interactions: Record<string, string>;           // "x,z" -> dialogue
  extraColliders: Array<[number, number]>;         // props that block movement but aren't grid tiles
  warps: Record<string, Warp>;                     // "x,z" -> where stepping there sends you
  encounterMap: boolean;
}
```

---

## 7. Roadmap

Each phase lists: scope, acceptance criteria (how an agent proves it's actually done — ties to Section 8), and current status. **Update the status column every session that touches a phase.**

| Phase | Scope | Acceptance Criteria | Status |
|---|---|---|---|
| **0 — Stabilize** | Remove dead deps (`express`,`dotenv`,`@google/genai`); enable `strict: true` and fix resulting errors; add Vitest + first tests (damage formula, type effectiveness, evolution); add a one-file GitHub Actions CI (`tsc --noEmit && npm test && npm run build`); add LICENSE; rewrite `README.md` to describe the actual game. | `npm audit` clean; `tsc --noEmit` clean under strict mode; CI green on a pushed commit; README no longer mentions "AI Studio". | **Not started.** Highest priority — do this before piling more content on top, per the audit's own recommendation. |
| **1 — Data Foundation** | All 151 species with real Gen 1 types/stats/evolutions/movesets, sourced from authoritative data (not memory). | `tsc --noEmit` clean, `npm run build` clean, `Object.keys(POKEDEX_CATALOG).length === 151`. | **Patch generated, delivery/merge status unconfirmed** — verify `pokemonData.ts` species count before starting Phase 2 work that assumes it's merged. |
| **2 — World Skeleton** | Multi-map system per Section 6.4 schema. Migrate Pallet Town losslessly, add Route 1 / Viridian City / Route 22 minimum, wire warps, update `World.tsx` to read the active map reactively instead of a static import. | Player can walk from Pallet Town → Route 1 → Viridian City and back without a build error, and (critically) without a visual regression — **requires actual browser verification, not just `tsc`/`build`** (Rule 10). | **Designed, not implemented.** Interrupted mid-session — see Section 6.4 for the exact schema to resume from. |
| **3 — Battle Parity** | Real Gen 1 damage formula using the `baseStats` already present in every species entry; full Gen 1 status/stat-stage rules; decide and document Psychic/Ghost bug behavior (faithful bug vs. corrected — pick one). | Battle test suite (Phase 0) covers the real formula, not just arcade-scaled numbers. | Not started. `baseStats` field already reserved for this in every species entry (Section 6.3). |
| **4 — Trainers & Story** | All 8 gyms, Elite Four, 5 rival battles, Team Rocket arc, item/trade system (unblocks the stone/trade evolutions deliberately skipped since Phase 1 — Rule 8). | Trainer data-driven per Section 6.2's `trainers.json`, not hardcoded. | Not started. |
| **5 — Visual Pass** | Apply `ASSET_PIPELINE.md` budgets across all 151 species consistently; lighting per biome. | Do this *after* 1–4 are content-complete, not before — re-skinning twice is wasted effort. | Not started. |
| **6 — QA & Save Integrity** | Full save/load of game state; full playthrough test script; code-splitting to fix the bundle warning (Section 2). | Bundle no longer trips Vite's 500kB chunk warning; a full starter→Champion run completes without a crash. | Not started. |
| **7 — Ship** | Production build, hosting, version-tagged release. | First real git tag in the project's history. | Not started. |

---

## 8. Agent Session Protocol

Every session, in order:

1. **Read this file in full.** Not skimmed — the rules in Section 5 exist because skipping this step is what caused the incidents in Section 3.
2. **Re-verify Section 2's ground truth** with these commands, and update the table if anything's drifted:
   ```bash
   npm install --no-audit --no-fund
   npx tsc --noEmit
   npm run build
   npm audit
   git log --oneline | head -5
   grep -rc "POKEDEX_CATALOG\[" src/game/pokemonData.ts  # sanity, not a real species count check
   node -e "console.log(Object.keys(require('./src/game/pokemonData.ts')).length)"  # adapt per actual export shape
   ```
3. **Pick exactly one task** from the current Roadmap phase (Section 7) whose prerequisites are already met. Don't jump ahead — e.g. don't start Phase 3's real damage formula before Phase 1's species data is confirmed merged.
4. **Implement it**, following Section 5's rules and Section 6's target schemas.
5. **Verify**: `tsc --noEmit` clean, `npm run build` clean, relevant tests passing (once Phase 0 adds them). For anything touching rendering, state explicitly in your handoff whether you could visually confirm it (Rule 10).
6. **Commit** with a message describing exactly what changed and what was verified — future sessions (and this file) depend on that trail being accurate.
7. **Update this file**: the Roadmap status column at minimum; Section 2's ground truth table if it drifted; Section 4 if a new debt category was discovered that isn't already listed.

---

## 9. Debt Ledger

Living record — add a row when new debt is discovered, check it off (don't delete the row) when resolved, so the history of what's been fixed stays visible.

| Debt | Discovered | Status |
|---|---|---|
| Dead deps (`express`,`dotenv`,`@google/genai`) causing 2 CVEs | Audit, 2026-07-23 | Open — Phase 0 |
| `tsconfig.json` missing `strict: true` | Audit, 2026-07-23 | Open — Phase 0 |
| Zero tests, zero CI | Audit, 2026-07-23 | Open — Phase 0 |
| `soundManager.ts` repeated `(state as any)` cast (6x) | Audit, 2026-07-23 | Open — Phase 0 |
| `World.tsx` god component (1,454 lines) | Audit, 2026-07-23 | Open — Phase 2/6 (split while doing the multi-map work, don't do it twice) |
| `gameStore.ts` god object (1,044 lines) | Audit, 2026-07-23 | Open — Phase 6 |
| Unsplit 1.5MB JS bundle | Audit, 2026-07-23 | Open — Phase 6 |
| No LICENSE | Audit, 2026-07-23 | Open — Phase 0 |
| `README.md` is unedited template boilerplate | Audit, 2026-07-23 | Open — Phase 0 |
| Pidgeot "Hurricane" / Butterfree "Bug Buzz" (post-Gen1 moves, inaccurate) | Phase 1 session | **Fixed in the Phase 1 patch** — confirm merged |
