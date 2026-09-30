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
