# Task: build LotM-style Divination for Mysteries of Orbis

Read CLAUDE.md first and follow it (build → always deploy the jar, I restart the server myself; never run the server
yourself; write JSON/assets with the Write tool; check the server log for "[NPC|P] FAIL" / validation errors after
touching assets; say clearly what is untested). Use the APIs and patterns CLAUDE.md lists as proven before inventing
new ones, and respect its dead ends. Add what you learn to CLAUDE.md as you go.

## Goal
Divination as close to Lord of the Mysteries as possible. In the novel it is a **craft**, not a spell: you prepare,
pick a medium, phrase a precise statement, recite it, and **interpret** an answer that comes from the Spirit World
through your own spirituality. Answers can be vague, wrong or dangerous. It is a **shared system** (any player can try,
like in the novel); pathway bonuses plug in later (a Hermit Mystery Pryer and a Fool Seer are planned — the Hermit
pathway does not exist yet, so only leave the hooks).

LotM rules to keep:
1. A **precise statement**, not a question: "The owner of this item is to the north" — the Spirit World confirms or denies.
2. Recited **seven times** while holding the tool.
3. A **medium** linked to the target (blood, belongings, likeness). Closer link = clearer answer.
4. Paid with **spirituality**; a calm mind helps; a biased subconscious bends the answer.
5. Answers are **symbols** to interpret (except yes/no methods).
6. Divining something far above you **hurts**, and they may **notice** you.
7. Some things are **shielded** (rituals, mystical items, powerful Beyonders).
8. Anyone with mysticism knowledge can try; Seers / Mystery Pryers do it far better. **Divining for others** is a
   Seer's acting (Klein digested Seer as a fortune teller) — hook only for now.

## Flow of one divination
1. Hold a **divination tool** (main hand) — the tool decides the method. No new Beyonder hotbar slot needed.
2. Hold a **medium** in the off hand, or the statement uses the place you stand / the creature you look at.
3. **Tap use** on the tool = open the **statement page** (custom page with Buttons, like PartyPage / SpiritWheelPage):
   subject buttons filled from what you hold / look at, then a predicate. Examples:
   - "[The owner of this item | The creature I look at | The one whose blood this is | This place | The path ahead]"
   - "[is within 100 blocks | is to the north/east/south/west | is in danger | is a Beyonder | is hostile to me | is dead |
     is safe | is dangerous | has something buried here | saw a death here]"
   Pick → page closes, the statement is remembered. (Keys don't reach the server while a page is open, so reciting is
   a separate step.)
4. **Hold use** = recite the last statement: ~4 s Charging (pattern: potion drink chain / hold roots). The statement
   appears **seven times** in faint italic letters near the screen center (reuse the advancement letters system), one per
   ~0.5 s. Moving more than a little, taking damage or releasing early **breaks** it: no answer, spirituality spent.
5. The answer shows the method's way (below). Interpretation is the player's job.

## Clarity (hidden, 0–1)
Every divination rolls against **clarity**. Put all numbers as tunable constants in one place
(e.g. `sequence/DivinationCatalog`, logic in `abilities/Divination`).

Raises: pathway bonus (hook: Seer best, Mystery Pryer good, other Beyonders poor, humans worst); medium strength
(blood > personal item > portrait > none); calm (out of combat ≥10 s via CombatMemory, high sanity); a ritual (lit
candles / our Full Moon Essence Oil used within 3 blocks — check what exists); night under open sky ("astrology",
reuse the daylight + `WorldChunk.getHeight` sky check).
Lowers: no/weak medium; the target is a stronger Beyonder (per sequence above you); the target is shielded; the same
statement repeated within ~5 min ("the Spirit World grows impatient", stacking); low sanity.

What clarity does:
- yes/no: P(true) = 0.5 + clarity/2.
- directions: angle noise up to ±90° at clarity 0.
- dreams: low clarity swaps symbols for wrong ones or gray fog.
- The player never sees the number. Show a subtle hint line instead ("The pendulum hesitates…" / "It answers at once.")
  — constant `SHOW_CLARITY_HINTS = true`.

## Methods (each a craftable tool item)
| Tool | Question | How the answer shows |
|---|---|---|
| **Spirit Pendulum** (chain + crystal) | yes/no | HUD **Sprite** pendulum (sprite sheet rendered by our C# tools, like the emblem halo) swings **clockwise = yes**, counter-clockwise = no; low clarity = it barely moves. |
| **Divining Cane** (also any staff/stick?) | which way | "let go": HUD sprite of a cane falling + a faint ground line only you see (`Particles.spawnFor`) pointing the answer. |
| **Coin** | quick yes/no | HUD coin flip, heads = yes. Cheap, fast, clarity capped low. |
| **Dream divination** (no tool: crouch still 10 s while holding a medium) | what / where / who | advancement darkness; 3–5 **symbol images** fade in one by one (white sigil style, drawn by code like our icons), plus their names in faint italic. |
| **Magic Mirror** | what is it doing now | hold use on the mirror: one symbolic line in the whisper popup + a dim per-player weather tint for a few seconds. |
| **Tarot deck** | fortune | draw 1 or 3 of the 22 Major Arcana (upright/reversed): small buff/debuff for 10 min + reading text hinting what's coming. |

### Dream symbol dictionary (in the catalog, real server facts → fixed symbols the player can learn)
zone/biome → leaves / sand / frost / ash …; far / near → a long road / a door; underground → stone closing overhead;
hostile creatures near it → red eyes in the dark; dead → a candle going out; Death Beyonder → a pale bell; Gatekeeper →
a gate; in danger → a cracked glass; many people → many lanterns; water → a black river; direction → the symbol's
road points that way (or a compass word). Example dream: *frost… a long road east… stone closing overhead… a candle
going out* = dead, snowy cave, far east. Low clarity: one wrong symbol / a gray blur.

## Mediums
- **Bloodied Cloth**: use a cloth on a creature/player you have wounded (≤30 s ago) → it carries that target. Strongest.
  `BLOOD_ON_PLAYERS = true` (constant).
- **Belongings**: an item a player dropped/crafted — needs item metadata (probe). Fallback: record mediums in player
  data instead of on the item.
- **Remains**: bones/drops from a dead creature (link to the death record we already keep for Ask the Dead).
- **Place**: no medium → the statement is about where you stand.
- **Portraits**: hook only (the Mystery Pryer's Notebook later).

## Danger, noticing, shielding
- **Backlash**: target 2+ sequences above you → fails, soul damage (`MysteriesOfOrbis_Soul`) + sanity loss + a bleeding-
  eyes ScreenEffect; a demigod (Seq 4) at Seq 9 → knocked out (darkness + rooted 4 s).
- **Noticed**: a divined player gets "You feel someone peering at you" (whisper popup); at Seq ≤5 also the direction and
  the diviner's pathway color.
- **Shielded** → answer is "Gray mist. Nothing.": a craftable **Warding Amulet** (worn/held), being inside a ritual
  barrier, Drag into the Shadows / hidden state. Keep a single `Divination.isShielded(target)` for future abilities.

## Who / acting
- `TOOLS_ANYONE = true`: everyone can craft and use the tools (low clarity for humans).
- Pathway bonus table with Death filled in as "other Beyonder"; Hermit/Fool entries to come.
- Acting hook: **divining for another player** (they hold the medium / are within 4 blocks and accept a prompt, both see
  the answer). Add `ActingCatalog` methods only when a Seer/Pryer sequence exists; leave a clear TODO.

## Items / assets
Pendulum, Divining Cane, Coin, Magic Mirror, Tarot Deck, Bloodied Cloth (+ blank cloth), Warding Amulet. Closest Hytale
items as recipe inputs, placeholder art from game items recolored like our potions, `Categories` incl. a new creative
child "Divination" (careful: recipe BenchRequirement also has "Categories"). Lang lines for names/descriptions in our
LotM tone. HUD sprites (pendulum, coin, cane) and dream symbols: rendered by our C# tools (C# 5!), white sigil style.

## Order of work (stop and report after each phase; I test in game)
1. **Probes** via `/seq test`: item metadata on an ItemStack (store/read a target id); finding an NPC's position when
   its area isn't loaded (players are always findable); whether an item's Charging use can be tap vs hold (tap opens
   the page). Report results before building on them.
2. **Core**: `Divination` + catalog, clarity, statement page, recite (7 lines), Spirit Pendulum + Coin end to end,
   Bloodied Cloth, backlash/noticed/shielded.
3. Cane + Dream (symbol images) + Mirror.
4. Tarot, Warding Amulet, ritual/astrology bonuses, divining for others hook.
5. CLAUDE.md update: what works, what's untested, numbers to tune.
