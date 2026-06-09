# Blip — the mascot

**Blip** is the recurring character: the skill's IP and the subject of every
image. The name reads off the design — an antenna over a screen is a *blip*, a
little signal, and the dot eyes echo it. The canonical model is
`assets/character-reference.png` — the engine conditions on it (see SKILL.md).
This file is the written spec and the rules.

## Locked design

- **Body**: a rounded-cube with soft corners. Small, stubby arms and legs.
- **Face**: ONE big rounded-rectangle **screen** that holds two simple white/cream
  dot eyes. Completely blank, deadpan, neutral — **no eyebrows, no mouth, ever**.
- **Antenna**: one short antenna with a round accent-colored ball tip. This is
  the only "robot" part and it carries the palette accent.
- Cuteness comes from **proportion and roundness**, never from added parts.

## Anti-complexity guardrails

The fastest way to ruin this character is detail creep. Never add:

- panels, seams, bolts, rivets, vents, gauges
- screen UI, pixels, text, or expressions on the screen
- multiple antennae, ears, hats, clothing, accessories
- shiny/anime eyes, eyebrows, mouth, teeth

If a render adds any of these, re-roll. Simpler is more on-model and more
reproducible.

## Value-follows-palette (critical)

The mascot is built with the same value logic as the rest of the scene, so it
never becomes a foreign dark blob:

- **Dark/bold palettes** (e.g. `ink-punch`): the body may read dark; the screen
  is the deepest value.
- **Light/warm palettes** (e.g. `notes`, derived light palettes): the body is
  **light/cream with the structure-ink outline** (built like the funnel and
  boxes), and the screen is the **structure ink, not pure black** — dark enough
  to hold the eyes, not a heavy black mass.

When in doubt in a light palette: light body, charcoal (not black) screen.

## Personality

An earnest, low-key operator doing something slightly absurd with a straight
face. Calm, deadpan, competent, never zany or cute-for-cute's-sake.

## Blip must be load-bearing

Blip performs the idea's one move — wedged in the neck, cranking the press,
holding the gate, hauling the load. Quick check: mentally **paint Blip out of
the sketch.** If the picture still explains itself, Blip was a sticker — rebuild
the scene so the move can't happen without Blip in it.

## Naming

The character is **Blip**. In generation prompts, describe it by its design
(rounded-cube body, screen face, dot eyes, antenna) rather than by name — image
models render the description, not the proper noun. Use "Blip" in human-facing
copy, captions, and shot lists.
