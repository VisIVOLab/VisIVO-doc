> **SUPERSEDED.** Use `HANDOFF_cube_viewer.md` — it consolidates this file, the parts
> of it that were never implemented, and the turn-12 layout work into one ordered list.

# Handoff — Stage 4 addendum: layout defaults and collapsible docks

Read `CUBE_VIEWER_SPEC.md` in full before starting; it is normative and supersedes my
earlier instructions where they disagree. Open `VisIVO Workspace.dc.html` and look at
`#12a`, `#12b`, `#12c`, `#11a`, `#11b`.

This addendum is **independent of the Stage 4 inline-form migration** and can be done
before, after, or alongside it. Do it in the order below; stop and rebuild between 1 and 2.

---

## 1. One pane by default, and a view picker

Currently the cube viewer opens with 3D and 2D side by side. In a 1440 px window that
gives two portrait slivers of about 454 × 735.

- `paneCount` defaults to **1**.
- With one pane, add a second segmented control to the centre toolbar — **3D / 2D** —
  choosing which view fills it. The other render widget is hidden, not destroyed
  (`QStackedWidget`, or `setVisible(false)` on the splitter child).
- `Linked` is **disabled and dimmed** (opacity 0.4) while `paneCount == 1`.
### What the 2 and 4 layouts do — see `#12d`, `#12e`, `#12f`

Two invariants: **nothing is created or destroyed by switching layout** (show/hide
widgets, never rebuild — round-tripping ▣ → ⊞ → ▣ must lose no camera, no zoom, no
loaded slice), and **the active pane is the unit of everything** (one filled dot at a
time, 1 px `kPrimary()` border; clicking in a pane makes it active).

Fill order for a newly appearing pane: the first of
**2D Slice → Moment 0 → Spectrum** not already on screen; session products extend the
list after Moment 0.

| Switch | Filled with | Side effects |
|---|---|---|
| ▣ → ▥ | left keeps current view; right takes first unused | toolbar 3D/2D picker hides; per-pane title dropdowns appear; `Linked` enables, on by default, qualifier "camera · channel · LUT" |
| ▥ → ⊞ | existing two keep top-left / top-right; new two continue the list | `Linked` qualifier narrows to "channel · LUT"; headers compact (padding 6/9, LUT strip 24 × 8) |
| ⊞ → ▣ | the **active** pane survives, not necessarily the first | picker returns set to the survivor; `Linked` disables + dims, remembers state |
| a slot has nothing to show | an **empty pane**: dark bg + centred "Pick a view" dropdown | never a bare black rectangle; never silently fall back to fewer panes |
| window < 1600 px | ▥ splits top/bottom; ⊞ stays 2 × 2 | direction derived from width on every resize, not persisted |

With `paneCount > 1` each pane header title is a dropdown. Picking a view already shown
elsewhere **swaps** the two panes instead of duplicating it.
- Optional, if cheap: when the user selects 2 panes and the window is narrower than
  1600 px, split the splitter **vertically** (top/bottom) instead of horizontally. A
  1112 × 347 landscape pane is more useful for a channel map than a 550 × 735 sliver.

**Do not** force a square aspect on the panes. See rule 7 in the spec — at this window size
it would letterbox away the space this change recovers.

## 2. Collapsible docks

Both docks become collapsible to a **36 px rail**. A rail is never 0 px wide.

Rail contents, top to bottom:
- expand glyph — "◧" on the left rail, "◨" on the right — in `kPrimary()`;
- a 20 × 1 px `kOutlineVariant()` rule;
- the panel name, `writing-mode: vertical-rl` equivalent (in Qt: a `QLabel` with a
  rotated paint, or a custom widget overriding `paintEvent` with
  `painter.rotate(90)`), Lato 10 px / 700, letter-spacing 0.14em,
  `kOnSurfaceVariant()`;
- stretch;
- **right rail only**: the three tab names rotated the same way, active one in
  `kPrimary()` / 600. Clicking one expands the dock *and* selects that tab;
- a Mono 9 px count of what is behind the rail — products on the left, running tasks on
  the right.

Defaults: **left open (272), right collapsed (36).** The left panel holds threshold, blend
mode and LUT, which the user touches constantly; the Inspector is only needed when starting
a task. Give the left dock a "◧" collapse button in its `SESSION DATA` header.

The Inspector auto-expands when: a chip is pressed in Analysis, a task starts, or a
vertical tab name is clicked. It never auto-collapses.

## 3. Focus mode

`⌥⇧F` (`Ctrl+Shift+F` on Linux/Windows) toggles both rails collapsed, and thins the
status rail to health + queue only. A toggle, not a mode — every other control keeps
working. Add it to the View menu as "Focus mode".

## 4. Persist the layout

Store `paneCount`, `leftCollapsed`, `rightCollapsed` and the 3D/2D selection per viewer
type in `QSettings` and restore them on open.

**Use a new key prefix** (`layout_v2/cube/...`, `layout_v2/image/...`). If anything is
still being restored from an older key, existing installs will come back with the stage-3
arrangement and this work will look like it did not land.

## 5. Same treatment in the image viewer

The image viewer has a single view, so `paneCount` and the 3D/2D picker do not apply — but
the two collapsible rails, focus mode and the persisted state all do. Its left panel holds
LAYERS and DISPLAY; its Inspector defaults to collapsed the same way.

---

## Acceptance check

Open `hi_cube.fits` in a 1440 × 900 window. You should see:

- one pane, viewport about **1112 × 735** (not two panes at ~454 × 735). Note the height is
  unchanged by design — vertical chrome is identical in every state, so this change buys
  width only; do not go looking for 27 px that were never there;
- the left panel open at 272 px with SESSION DATA, DISPLAY — 3D, LAYERS, and two collapsed
  sections;
- a 36 px rail on the right reading "INSPECTOR" vertically, with the three tab names and a
  "0" at the bottom;
- clicking "Analysis" on the rail expands the dock to 320 px with that tab selected;
- `⌥⇧F` collapsing both sides to a viewport about 1348 px wide, and restoring;
- ▥ giving two 552 × 735 panes (3D + 2D Slice) with `Linked` enabled, ⊞ giving four
  552 × 363 panes (3D, 2D Slice, Moment 0, Spectrum), and ⊞ → ▣ keeping whichever pane was
  active — then ▣ → ⊞ → ▣ again with the camera and zoom untouched;
- quitting and reopening restores whatever state you left.
