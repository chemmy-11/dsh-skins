# Agent Note: wallpaper-exclusive meets the dsh 0.2.0 desktop shell

Status: implemented

## Problem

The official DSH desktop moved to 0.2.0-rc.2 (app.asar 2026-09-29). A skin is a
pure asset directory, so when the shell renames or drops an anchor the skin's
rule does not error - it silently matches nothing. Two anchors this skin pins
stopped matching, and one desktop-only surface cannot be addressed at all.

**The right column anchor is gone.** `[class*="detailsCol"]` occurs 0 times in
the whole 0.2.0-rc.2 archive and 0 times in the live DOM; `[data-slot="details"]`
and `[data-pane="details"]` are equally dead. The column now carries
`[data-rightbar-col]` (0-width while collapsed, so painting it costs nothing
when closed). In the details glass group the retired heads were harmless because
the compat adapter stamps `[data-dsh-surface="details"]` on the new column, but
the right-panel INPUT rule had no such sibling, so the right panel's inputs never
received the skin's input material.

**The sidebar-key anchor never matches either.** The rules were keyed on
`[data-dsh-part="sidebar-entry"]:has([data-dsh-panel-entry])`. Measured on the
running shell: `[data-dsh-part="sidebar-entry"]` 0, `[data-dsh-taskboard-entry]`
0, `[data-dsh-ssh-entry]` 0, `[data-dsh-panel-entry]` 1. The identity anchor is
the one the contract actually ships (contracts/semantic-attrs-v1.md L140: the row
identity belongs to the registering panel plugin, emitted on the row glyph); the
part attribute the adapter is supposed to stamp on the row never lands.

**The desktop chrome wash is real but not addressable from a skin.** On win32 the
preload stamps `html[data-windows-titlebar]` and `html[data-platform]` with no
configuration gate, and the shell then paints the window frame with
`--dsw-specific-sidebar-fill` and the conversation column with a second copy of
`--dsw-alias-bg-base` plus the 16px desktop content radius. Measured on the live
shell with the skin applied, the conversation column loses contrast against the
browser build: mean RGB [47,53,62] vs [54,60,69], and the single most common
colour rises from 30.8% to 43.2% of the region. It is a wash, not a cover - the
negative-z background layers stay visible in both builds.

## Decision

Re-anchor the two reachable defects on tested anchors, and record the third as a
decision rather than shipping a rule that cannot match.

- The details glass group and the right-panel input rule gain
  `[data-rightbar-col]` beside the retired heads. Existing heads are kept, so one
  file still serves pre-0.1.7, 0.1.7+ and 0.2.0 hosts.
- The sidebar keys gain `[class*="panelRow"]:has([data-dsh-panel-entry])` beside
  the `sidebar-entry` head - the same head the skin center's own stylesheet uses
  for this row. It matches 1 live (was 0).
- No `html[data-platform]` / `html[data-windows-titlebar]` rule is shipped.
  `css-safety/transform.ts` `scopeSelectorText` rewrites only three root-ish
  heads (:root, a bare `html` head, and `html[data-ds-*]`, which it maps onto
  `body`). Every other selector is appended to the `html[data-dsh-skin="<id>"]`
  scope as a DESCENDANT, and a second `html` element never matches, so such a rule
  compiles into a silently dead selector. This was verified live: the drafted block
  left the frame at rgba(17,25,39,0.16) and the column on the derived base
  unchanged. A comment-only decision record in patches.css carries the host rules,
  the measurements, and the two ways to close the gap.

## Alternatives considered

- **Clear the frame and column unconditionally.** Rejected: `.BynINW_frame` paints
  `var(--dsw-alias-bg-base)` in the browser build too, and that token is derived
  from the skin's own `--dsw-alias-bg-layer-1` by the token audit, so clearing it
  removes an intentional part of the skin's glass look and diverges from the
  committed preview renders.
- **Gate the clear on `body[data-dsh-wallpaper-active]`.** Already present for the
  centre column and it works; it does not cover skin-only backgroundMedia mode, and
  it does not cover the frame.
- **Keep the dead `html[data-windows-titlebar]` block as documentation-in-code.**
  Rejected: a selector that can never match is exactly the failure mode this change
  exists to remove. The record is a comment, not a rule.
- **Re-anchor the sidebar keys on `[data-dsh-taskboard-entry]` / `[data-dsh-ssh-entry]`.**
  Rejected: both are 0 on the running shell.

## Consequences

- The right panel's inputs get the skin's input material again, and the sidebar-key
  rules reach the row they were written for.
- Stated limit: the matched sidebar row (`_2H3hWW_panelRow`) computes
  rgba(0,0,0,0) at rest, on hover and with aria-current set, WITH and WITHOUT the
  skin - the shell does not paint that element in this build. The head restores the
  rule's reach without a visible delta today; it is latent-correctness, not a
  visual fix.
- `[class*="panelRow"]` is a CSS-module hash suffix, so it carries the
  transform's existing `[class*=...]` warning. It is used here only because the
  skin center's own stylesheet addresses the same row the same way; if the shell
  ever gives `sidebar.panellist` a stable per-entry hook, this head should be
  retired in favour of it.
- The desktop chrome wash stays. Closing it needs either the pipeline to give the
  platform markers the rewrite it already applies to `html[data-ds-*]`, or the host
  to stop painting the app base on the frame and the centre column while
  `html[data-dsh-skin]` is set. Until then the skin keeps the stock desktop chrome,
  which keeps it consistent with its preview renders (captured on the browser build).
- `skin.json` 0.2.2 -> 0.2.3. The regenerated `lib/` and
  `src/reviewed-hooks.generated.ts` travel with the manifest change.
