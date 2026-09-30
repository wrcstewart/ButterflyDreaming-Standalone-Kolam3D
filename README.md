# ButterflyDreaming — Kolam3D (standalone)

One media module from [ButterflyDreaming](https://butterflydreaming.org),
running on its own. Open `index.html` — there is no build step, no bundler and
no server. The module fetches three.js from a CDN and draws.

**[butterflydreaming.org](https://butterflydreaming.org)** · CC0

---

## What it draws

An L-system kolam in three dimensions. The turtle walks a rewritten string; `+`
turns it by `angle` about its up axis **and** by `pitch` about its left, so a
pitch of zero draws the flat figure exactly and a degree of pitch lifts it into a
shell. Three further steppers place the camera, and one of them can turn it.

Edit the script and the drawing follows. Move a stepper and the script follows
back. **Copy script** puts the whole thing on your clipboard; where it goes after
that is yours to decide.

---

## How the drawing is made

Two directives do all the work. Everything else is colour, camera and pace.

    %%bd_score [
    axiom: F+F+F+F+F+F+F+F
    F: F+F-F-F+F+F+F-F
    %%bd_]

**The rewriting.** Start with the axiom. Then, `depth` times over, replace every
`F` with the rule's right-hand side. The rule has eight `F`s in it, so each pass
multiplies the string by eight: an axiom of 8 becomes 64, then 512, then 4,096,
then 32,768. `depth` is capped at 5 for that reason — 262,144 segments.

**The walking.** A turtle then reads the string one symbol at a time:

| symbol | what the turtle does |
|---|---|
| `F` | draw forward one step |
| `+` | turn by `angle` |
| `-` | turn by `-angle` |

So the string is not a picture of the figure — it is a set of *instructions*, and
the same string draws something completely different at a different `angle`.

`step` is divided by `3^(depth-1)`, which is what keeps the figure roughly the
same size as it grows more intricate rather than exploding off the canvas.

**The symmetry.** The finished curve is then stamped `symmetry` times, each copy
rotated about the vertical axis. One geometry, many draws.

### Only certain angles give a true kolam

A kolam is a **closed looped figure** — the line returns to where it began and
the pattern sits on a lattice of dots. Neither property comes from the rule. Both
come from the angle, and most angles give neither.

Measured on this rule, at depth 2 — the gap is how far the turtle ends from where
it started, as a fraction of the figure's own radius:

| angle | 360 ÷ angle | distinct headings | end-to-start gap | |
|---|---|---|---|---|
| 45° | 8 | 8 | **0.0000** | **closes — a true kolam** |
| 90° | 4 | 4 | **0.0000** | **closes — a true kolam** |
| 60° | 6 | 6 | 0.86 | lattice, but open |
| 72° | 5 | 5 | 0.99 | lattice, but open |
| 30° | 12 | 12 | 0.67 | lattice, but open |
| 68° | 5.29 | 82 | 0.33 | neither — a drifting tangle |
| 50° | 7.2 | 36 | 0.32 | neither |

Two separate things are going on:

- **A lattice** needs `360 ÷ angle` to be a whole number. Then the turtle only
  ever faces a finite set of directions — 4 at 90°, 6 at 60° — and the figure
  has the woven, grid-like quality kolams have. At 68° it faces 82 different
  ways and the weave is gone.
- **Closure** is rarer, and for this rule it happens at **45° and 90° only**.
  Checked at depths 1 to 4 and with axioms of 3, 4, 6, 8 and 12 `F`s: it closes
  every time. It is a property of the rule, not of how far you grow it.

So `angle 90` and `angle 45` are the kolams. Everything else is worth looking at
— the open figures are often beautiful, and `angle_drift` exists precisely to
walk slowly through them — but it is not what a kolam is.

`angle_minutes` matters more than it looks for the same reason. A sixtieth of a
degree is enough to open a closed figure, so the drift setting is how you watch
a kolam come apart and reassemble rather than a way of getting a different one.

### And the third dimension

`pitch` is the second turn. `+` rotates the turtle by `angle` about its up axis
**and** by `pitch` about its left; `-` does both negatives. At `pitch 0` the
turtle never leaves the plane and the figure is exactly the flat kolam — that is
the invariant the module is built around.

Pitch compounds at every one of the hundreds of turns, so it bites harder than
it looks: one degree lifts the default figure 75 world units out of a plane whose
own radius is 97. `step_pitch` is what makes it a dial rather than a switch — the
turtle's step length in the new dimension, in the same units as `step`. Equal is
isotropic; zero is flat whatever the pitch says.

---

## For a developer: what a module has to do

A ButterflyDreaming media module is **an iframe that speaks four messages**.
That is the entire contract.

| direction | message | meaning |
|---|---|---|
| module → host | `BD_READY` | loaded; send me a script |
| host → module | `bd_script_update` `{ script }` | the text to render |
| module → host | `bd_av_state` `{ text, fromDrift }` | its live script, on **every** render |
| module → host | `bd_module_log` `{ level, line }` | its console, so a host can see inside the iframe |

Answer `bd_script_update`, announce `bd_av_state`, and **any** ButterflyDreaming
host can drive your module — this page, or BD itself, or a viewer on another
device. Nothing else is required.

`index.html` is a complete host in about 120 lines of plain JavaScript, written
to be read. Two details in it are worth stealing:

- **Attach the message listener before setting the iframe's `src`.** The module
  announces `BD_READY` the moment it loads, and a listener added afterwards
  misses it.
- **`fromDrift` marks a frame the module's own timer caused**, not a person. A
  host that writes every frame into a text box will fight anyone typing in it.
  This page tracks those frames in a variable so **Copy** is never stale, while
  the box itself only updates on a human change.

### Deliberately absent: deep links

Earlier standalones packed the whole script into a URL. That meant compression,
a wire table of abbreviated keys, and a length ceiling to measure against — a
great deal of apparatus standing between a reader and how a module actually
works. It is gone. Copy the script and paste it wherever you like.

---

## The script format

Lines beginning `%%bd_` are directives; everything else is prose. A block
directive opens with `[` and closes on a line that is exactly `%%bd_]`.

    %%bd_module bd_V_Kolam3D
    %%bd_p_symmetry 8
    %%bd_p_angle 90
    %%bd_score [
    axiom: F+F+F+F+F+F+F+F
    F: F+F-F-F+F+F+F-F
    %%bd_]

**`_p_` asks for a control.** `%%bd_p_symmetry` means "give this one a stepper";
`%%bd_symmetry` means "use this value and offer no control". The mark is
presentation only — it never changes what a directive *means*, and a module
strips it before looking the value up.

Two rules a module must honour:

1. **The writer reconstructs the form it read.** A script carrying
   `%%bd_p_angle` must come back carrying `%%bd_p_angle`, or a value update
   would quietly change which controls appear.
2. **A script with no mark anywhere keeps every control.** Marking is opt-in, so
   nothing written before the convention existed changes behaviour.

---

## Keeping this copy honest

`visual_module.html` is **vendored** — a copy of the module as it stands in
ButterflyDreaming's own repository. That is deliberate: a developer should be
able to open it, read it and break it without a server.

The cost of vendoring is drift, and it has bitten this project before — two
copies of an earlier module diverged and polish landed in only one of them. So
the copy is refreshed by one deliberate command rather than by hand:

    ./sync_from_bd.sh

It overwrites `visual_module.html` from the BD working tree and records which
commit it came from in `MODULE_SOURCE.txt`. **If you have changed the module
here, that command will discard your changes** — it is a copy-down, not a merge.

---

## Licence

CC0. Do what you like with it.
