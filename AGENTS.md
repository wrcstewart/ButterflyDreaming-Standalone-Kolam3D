# AGENTS.md — picking up development here

*For an AI agent, or a person arriving cold. Named `AGENTS.md` because that is
the filename the common coding agents look for without being told.*

**Read §1 before editing anything.** It is the one rule in this file that will
cost you work if you skip it.

---

## 1. The module file is VENDORED. Do not edit it here.

`visual_module.html` is a **copy**. The original lives in the ButterflyDreaming
working tree at `V_Kolam3D/visual_module.html`, and the copy here is refreshed by:

    ./sync_from_bd.sh

That is a **copy-down, not a merge**. It overwrites `visual_module.html` and
discards anything you changed in it. `MODULE_SOURCE.txt` records which BD commit
the current copy came from.

So the workflow for a module change is:

1. edit `V_Kolam3D/visual_module.html` in the BD repo,
2. commit and push there,
3. `./sync_from_bd.sh` here,
4. commit the refreshed copy here.

**What you MAY edit freely here:** `index.html` (the host page), `README.md`,
this file, `sync_from_bd.sh`. Those are this repository's own.

This is not bureaucracy. Two copies of an earlier module diverged in exactly
this way and polish landed in only one of them; the project has paid for the
lesson twice.

---

## 2. What this repo is

    index.html          the host page — ~145 lines of plain JS (88 code, 46 comment)
    visual_module.html  the module, vendored (see §1)
    README.md           what the module does and how the figure is made
    sync_from_bd.sh     refresh the vendored copy from BD
    MODULE_SOURCE.txt   which BD commit that copy came from

No build step, no bundler, no server, no dependencies to install. **Open
`index.html` in a browser.** If you want a server for cache reasons,
`python3 -m http.server` in this directory is enough.

Published at <https://wrcstewart.github.io/ButterflyDreaming-Standalone-Kolam3D/> from `main`, root path.
Pushing to `main` redeploys; allow a minute or so.

---

## 3. The contract, and where it lives in the code

A ButterflyDreaming media module is **an iframe that speaks a handful of
`postMessage`s**. That is the whole integration surface — there is no SDK and
nothing to import.

| direction | message | meaning |
|---|---|---|
| module → host | `BD_READY` | loaded; send me a script |
| host → module | `bd_script_update` `{ script }` | the text to render |
| module → host | `bd_av_state` `{ text, fromDrift }` | its live script, on **every** render |
| module → host | `bd_module_log` `{ level, line }` | its console, forwarded out of the iframe |
| host → module | `bd_ui_config` `{ hideControls, hostChrome }` | optional; see §4 |

Two details in `index.html` are load-bearing and easy to undo by accident:

- **The message listener is attached BEFORE `frame.src` is set.** The module
  announces `BD_READY` the moment it loads; a listener added afterwards misses
  it and the page sits there empty. The `frame.src = ...` line is deliberately
  the last statement in the file.
- **`fromDrift` marks a frame a timer produced, not a person.** The host keeps
  a `live` variable that follows every frame, but only writes the textarea on
  non-drift frames — otherwise it fights anyone typing and destroys the undo
  history. **Copy reads `live`, not the textarea**, so it is never stale.

---

## 4. `bd_ui_config` — two flags that are NOT the same flag

    hideControls   the HOST supplies the stepper column; hide mine
    hostChrome     the host draws things AROUND this iframe (default TRUE)

BD reserves layout for furniture only BD draws — arrows beside the stepper column, an Extension strip under the canvas, for which it holds 88px to the left and 52px below. A host
that stamps nothing into that reserve gets it as lost picture.

These were one flag until 2026-09-30 and had to be split, because the two cases
pull apart exactly here:

| | hideControls | hostChrome |
|---|---|---|
| BD itself | false | true |
| an Ancillary Viewer | **true** | false |
| **this page** | **false** | **false** |

A standalone wants the chrome released and the steppers **kept**. That
combination was unreachable before the split. `hostChrome` defaults to `true`
so a host that says nothing keeps BD's behaviour; `hideControls` implies it.

---

## 5. The script format, and the two rules that bind a renderer

Lines beginning `%%bd_` are directives; everything else is prose. A block
directive opens with `[` and closes on a line that is exactly `%%bd_]`.

**`_p_` asks for a control.** `%%bd_p_angle` means *give this one a stepper*;
`%%bd_angle` means *use this value and offer no control*. The mark is
**presentation only** — it never changes what a directive means, and a module
strips it before looking the value up.

- **RULE 1 — the writer reconstructs the form it read.** A script that arrived
  carrying `%%bd_p_angle` must be written back carrying `%%bd_p_angle`. Get
  this wrong and merely moving a stepper silently changes which controls exist.
- **RULE 2 — a script with no mark anywhere keeps every control.** Marking is
  opt-in, so nothing written before the convention changes behaviour.
- **RULE 9 — a module's default script carries no directive it cannot act on.**
  A directive in a default script is a promise the module keeps.

The full set (RULES 1–9) is in BD's `CollagePlanStarted_2026-09-22.md`.

---

## 6. Preparing something for ButterflyDreaming

If you build a new module, or change this one in a way BD must see, this is the
checklist. **A new module needs FOUR registries and nobody has ever remembered
all four unprompted.**

### 6.1 The four registries (all in the BD repo)

| what | where |
|---|---|
| `MODULES` — embedded + standalone URLs | `viewer.js` (~line 2783) |
| `AV_VIEWER_MODULES` — which modules have a viewer page | `viewer.js` (~line 13297) |
| `AV_RENDERERS` — module id → renderer path | `AV/kolam.html` (~line 149) |
| `express.static` route — `/bd_X/` → the source dir | `server.js` (~lines 163–205) |

Note the route name and the directory name differ on purpose: `/bd_M_ABC/`
serves `./M_Music/`. Do not "fix" that.

### 6.2 The database side

Memgraph, reached through `node bd_tool.js` at the BD repo root.

    node bd_tool.js cypher "MATCH (n) WHERE n.name STARTS WITH 'bd_V_Kolam3D' RETURN n.name, n.seq, n.hasModuleScript"

A module occupies three kinds of node: a **Cluster**, a **gateway** TextNode
carrying `seq = -1`, and one or more **content** TextNodes carrying
`hasModuleScript = '<module id>'` and `seq >= 1`. The content node's text IS the
default script.

**Every DB-mutating `bd_tool.js` subcommand takes a pre-flight backup by
default. Do not pass `--no-backup`.**

**Editing stored text does not reach a running BD.** Node text loads with the
graph at boot, so re-tapping a node shows the stale copy. After a DB edit, say
so: *reload BD*.

### 6.3 Cache-busters and canaries

- `AV/kolam.html` loads renderers as `AV_RENDERERS[m] + '?v=NN'`. **Bump `NN`
  on any change to a module.** iOS Safari caches an iframe `src` hard, and a
  stale renderer means layout fixes silently never arrive.
- BD, the AV and `sr_editor` each host their **own** canary — a visible border
  rotating red → green → blue on every change to a cached file. They are
  independent; one says nothing about another. **Media modules have no canary
  host**, so a module-only change rotates nothing — but it still needs the
  `?v=` bump above.

### 6.4 Two-phase deploy

**Receivers before writers.** A renderer must be able to READ a new form before
anything WRITES one. A new module ships as a receiver from day one.

---

## 7. BDX / AVX / RX — the next stage

There is a second, independent harness, and it is where this repo's ideas grow
up. Repo <https://github.com/wrcstewart/bdx-demo>, local checkout
`~/bdx_demo`, pages <https://wrcstewart.github.io/bdx-demo/>.

| | what it is | where it runs |
|---|---|---|
| **BDX** | the controller — script panel, steppers, renderer | GitHub Pages (static) |
| **AVX** | the viewer — renders what it is told, no controls | GitHub Pages (static) |
| **RX** | the relay — a rendezvous for two browsers on different devices | a Node host |

RX is **live** at <https://rx.virtualfictions.uk/health>, on the existing
Discourse VPS behind a Cloudflare tunnel.

**The claim it makes:** the module architecture needs nothing of BD — no
Memgraph, no corpus, no graph, no pairing, no curation, no speech. And the
dependency ladder is worth stating precisely, because it is easy to overclaim
in either direction:

1. **Same machine → no server at all.** If the controller *opened* the viewer it
   holds a window handle, and `postMessage` reaches it, cross-origin included.
   Measured at **~1 ms, against ~30 ms through a socket**.
2. **Cross-device → a rendezvous is unavoidable.** Two browsers on two devices
   cannot reach each other; no window handle exists. That is the *only* reason
   the socket path exists in BD at all.
3. **Whose rendezvous is a free choice.** Run RX yourself (~10 lines around
   `bd_relay.js`, no account), or point at ours.

**How this page relates to it.** This repo is a BDX with the relay left out:
script panel, steppers, renderer, on a URL, no corpus. The next step for any of
these four pages is the same one — **add a View button** that opens an AVX and
drives it. Same machine needs no relay at all (tier 1), which makes it a genuinely
small change: claim the window *inside the click* (see §8), then `postMessage`
the script to it on every `bd_av_state`.

`AV/bd_av_client.js` in the BD repo (414 lines) is the viewer shim and **is the
third-party contract** — the artifact someone else would use.

---

## 8. Traps already paid for

Each of these cost a debugging round. They are not hypothetical.

- **A gesture does not survive an `await`.** `window.open` and clipboard writes
  are both refused after one. Claim the resource *inside* the click — open
  `about:blank` first and navigate later; hand the clipboard the *promise*.
  Safari refuses what Chrome allows. `index.html`'s Copy button has a
  select-the-text fallback for exactly this.
- **A missing asset fails silently.** Not a live risk here — this module's only
  external resources are CDN scripts, and it reports their absence explicitly
  (`HAVE_THREE`, `HAVE_LSYSTEM`). It bit the two music modules hard: four
  sample files 404'd, `Tone.loaded()` never resolved, and Play stayed disabled
  with **no error anywhere**. Keep the explicit availability check when you add
  a dependency; a silent `undefined` is how that one hid.
- **Copying a file copies its claims.** Three of these four hosts were spliced
  from the fourth, and shipped six comments true only of the origin — one of
  which was a layout bug wearing a comment's clothes. After generating siblings
  from a template, read each against *the thing it now describes*. Grep the
  copies for the origin's proper nouns.
- **`node --check` parses as CommonJS** and silently passes ES-module errors.
  For an HTML file, extract the `<script>` body and check that.
- **A check whose filter excludes the failing pattern proves nothing.** Said
  after a "clean" grep reported four broken call sites as fine.
- **Test the path, not a stage of it.** A regex bug survived a test that ran on
  raw text and so never reached the normaliser that caused it.
- **iOS inputs below 16px auto-zoom on focus.** Every focusable input needs
  `font-size: 16px` or larger.

---

## 9. Developments worth trying

Ordered roughly by ratio of interest to effort. None is started.

### Intermingled grammars for pitch and angle  *(the one to do first)*

**Today pitch is welded to yaw.** Look at `buildTurtle`: `+` calls
`turn(yawRad, pitchRad)` and `-` calls `turn(-yawRad, -pitchRad)`. One symbol
applies **both** rotations, always, in fixed proportion. The grammar has no way
to say *turn here but do not climb*, or *climb twice here without turning*.
`pitch` is therefore not a second dimension the grammar can compose in — it is a
global tilt applied to a 2D grammar.

The fix is the standard 3D turtle vocabulary (Prusinkiewicz), which the module's
`H`/`U`/`L` basis already supports and simply never uses:

    &   pitch down        ^   pitch up
    \   roll left         /   roll right

Roll is **entirely absent today** — there is no roll anywhere in the module —
and it is what makes the basis fully general rather than a tilted plane.

With those, two grammars genuinely interleave in one string:

    F: F+F&F-F^F        turning and climbing driven independently

**Three things to get right, in this order:**

1. **Unknown symbols currently fall through the `if`-chain in silence.** A
   script using `&` today renders as though it were not there. That is the
   failure mode RULE 9 exists to prevent, and it must be fixed *before* anything
   writes a `&` — receivers before writers (§6.4).
2. **Give the pitch symbols their own magnitude directive.** If `&` and `+`
   share `angle`, you have one grammar with decoration, not two grammars. A
   separate `%%bd_pitch_step` (defaulting to `pitch`, which defaults to 0) is
   what makes them independent.
3. **The invariant survives this for free, and you must check that it did.** At
   `%%bd_pitch 0` the new symbols rotate by zero, so every existing script
   renders identically. Which brings us to:

### THE INVARIANT — read before changing any sign in the turtle

At `%%bd_pitch 0` and `%%bd_cam_elevation 90` this module draws **exactly** what
the 2D Kolam draws. Verified numerically, not by eye: worst deviation
**6.1e-5 world units on radius 1362**, which is `Float32Array` precision.

Three sign conventions hold it up, and **each is easy to "fix" wrongly**:

- yaw is applied as **-a** about the up axis (rotating +X about +Y by theta gives
  `(cos, 0, -sin)`, and the 2D heading is `(cos a, 0, sin a)`);
- the bezier bulge is along **-L (left), NOT U** — at pitch 0 the drawing plane
  is XZ, `L` lies in it and `U` is its normal, so bulging along `U` would push
  the curve out of the plane;
- camera up is the sphere's **north tangent** `(-sinEl*sinAz, cosEl, -sinEl*cosAz)`,
  not world +Y — because +Y is *parallel to the view direction* at elevation 90,
  which is the very view the invariant is stated at.

If you change the turtle, re-run the comparison against the 2D module before
committing. A figure that still "looks right" is not evidence.

### Smoothly altering the grammar

The steppers move continuously; **the grammar does not move at all**. Editing a
rule is a discrete jump, and `depth` is worse — each pass multiplies the string
by the rule's `F`-count (8 for the default rule: 8 → 64 → 512 → 4,096), so the
figure leaps by 8x in detail with nothing in between. Everything continuous
about this module today lives in the *turn magnitudes*, not in the string.

Three routes, and they are not equally good:

**(a) Fractional depth — the strongest of the three.** Rewrite only *part* of
the string one extra pass. `%%bd_depth 2.4` = depth 2 with 40% of its symbols
rewritten again. It is deterministic, it reproduces exactly from the script, it
needs no new directive vocabulary, and it makes `depth` a stepper like any
other. Two selection orders are worth building and comparing, because they will
not look alike: **prefix** (the first 40% of symbols) morphs in from one end
like a wave crossing the figure; **interleaved** (every 5th of every 2) thickens
evenly everywhere. Expect to want the second and be surprised by the first.

**(b) Weighted productions.** `lindenmayer` supports stochastic successors, so a
slider could move probability mass from rule A to rule B. Attractive, and it
**breaks a rule you must not break**: the script is the source of truth, and the
same script must give the same figure on two devices. It would need a seed
directive, and a seed is a thing a user has to understand. Do (a) first.

**(c) Interpolate the magnitudes, not the string.** This already exists —
`angle_minutes` and `angle_drift` are exactly this, and they are the evidence
that continuous motion through figure-space is the interesting thing. Cheapest
possible win: extend the same treatment to anything else that is currently
integer-only.

### A grammar editor rather than a textarea

The score is free text with **no feedback until it renders**. A malformed rule
is indistinguishable from a boring one. A small inspector beside the box —
`F`-count, resulting string length at each depth, whether the figure closes —
would turn blind editing into informed editing, and it is all computable
without drawing anything.

### A closure and lattice finder

The README records a measurement: for the default rule, **45 deg and 90 deg
close exactly** (end-to-start gap 0.0000, at every depth and for axioms of 3, 4,
6, 8 and 12 `F`s), and a **lattice** needs `360 / angle` to be a whole number —
4 headings at 90 deg, 82 at 68 deg where the weave disappears.

That was measured by hand, for one rule. **Any rule a user writes has its own
answer and no way to discover it.** A panel that sweeps the angle and reports
closure gap and distinct-heading count would tell you the thing you actually
want to know: *is what I have a kolam?* It needs no rendering — walk the turtle,
compare start to end — so it is cheap, and it is the only feature here that
teaches the user something about their own input.

### WebXR on Quest

**This is why the module is three.js at all.** The standing decision is that no
new visual module goes on a 2D canvas, precisely to keep this open.

The plumbing is small: `renderer.xr.enabled = true`, swap `requestAnimationFrame`
for `renderer.setAnimationLoop`, and add `VRButton.js` (4.5 KB, no imports).

**The design question is the whole job, and it is not plumbing.** In VR the
steppers must move the **world**, not the eye. `cam_azimuth`, `cam_elevation`
and `cam_distance` currently place a camera; in a headset the *headset* owns the
camera, and moving it under the user is how you make people ill. Those three
become a transform on the world group instead. Get that wrong and the module is
technically working and physically unusable.

Budget, for reference: the scene is **one geometry shared by N `LineSegments`
children** — so `symmetry` costs N draw calls (8 by default) and one buffer, not
N buffers. That sharing is the reason symmetry is affordable here at all, and
the reason a headset's 72–90 Hz is plausible.


---

## 10. Where the rest of the knowledge is

The BD repo is the source of truth for everything above.

| | |
|---|---|
| `DOCS_INDEX.md` | what each of ~40 docs *is* |
| `PLANNING_REGISTER.md` | how far each design is *built*, evidence-based, ending in every unbuilt item in one table |
| `CHANGELOG.md` | newest-first narrative log; the friendly read |
| `CollagePlanStarted_2026-09-22.md` | the `%%bd_p_` convention and RULES 1–9 |
| `BDX_DEMO_PLAN.md` | §7 above, in full |
| `AV/README.md` | the Ancillary Viewer, and the 1 ms vs 30 ms measurement |

**Start with the two indexes.** They exist so that an arriving agent does not
have to grep.

### Known inconsistency, not yet fixed

BD's `MODULES` table still points `standalone` at the **old, retired** repos
(`bd_V_Kolam`, `bd_M_ABC`, `bd_M_Fractal` — the `preview.html` pages), not at
the four `ButterflyDreaming-Standalone-*` repos these files live in. Kolam3D has
no `standalone` entry at all. Updating that table is a small, safe change that
nobody has made yet.

---

*Licence CC0. Do what you like with it.*
