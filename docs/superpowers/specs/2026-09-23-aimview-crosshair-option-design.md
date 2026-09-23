# AimView Crosshair Option — Design

Date: 2026-09-23
Approach: A (minimal restore, approved)
Status: approved, pending implementation plan

## 1. Background

Upstream 1.0.38 (commit `1930815`) removed `AimViewWidget::DrawCrosshair`
(declaration in `UI/aimview.h`, definition and call site in
`UI/aimview.cpp`), leaving the crosshair available only on the Fuser
overlay via `fuserRender.h::RenderCrosshair`, gated by
`espGlobals::drawCrosshair` ("Crosshair" toggle in ESP Local settings,
`UI/render.cpp`).

Request: restore the crosshair on AimView behind its own setting, so
AimView and Fuser crosshairs are independent ("not fuser only").

## 2. Decisions (user-confirmed)

- Separate toggles: AimView crosshair flag independent of Fuser flag.
- Reuse shared style: same size (`espGlobals::crosshairSize`), type
  (`espGlobals::crosshairType`), and colour
  (`coloursGlobals::crosshair`) drive both views. No AimView-specific
  style storage.
- Setting location: toggle row next to "Aim view" in Settings >
  Settings > Radar Draw (`UI/render.cpp` ~line 1522).
- Default: ON for fresh configs (old configs missing the key also
  default ON).

## 3. Architecture

`AimViewWidget` owns a new `AimViewConfig::drawCrosshair` flag (default
`true`). `AimViewWidget::Render` keeps upstream's structure (early-out
when disabled; background/players/loot/containers inside the clip
rect) plus one restored step: when the flag is on,
`DrawCrosshair(drawList)` draws at `canvasCentre_` inside the same
clip rect. Style inputs are read-only shared state; no new style
storage. Settings UI writes the flag through
`g_AimViewWidget.GetConfig()`; `UI/config.cpp` persists it under
`aimView.drawCrosshair`. The Fuser path is untouched.

## 4. Components (exact file changes)

- `UI/aimview.h`: add `bool drawCrosshair = true;` to `AimViewConfig`;
  redeclare `void DrawCrosshair(ImDrawList* drawList) const;` (private,
  next to `DrawBackground`).
- `UI/aimview.cpp`: restore the deleted `DrawCrosshair` body verbatim
  (null-`drawList` guard, `clamp(crosshairSize, 1, 20)`, cross lines
  when `crosshairType == 1`, else circle, colour from
  `coloursGlobals::crosshair`, centred on `canvasCentre_`); insert the
  call after `DrawContainers(drawList)` inside the `transformValid_`
  clip-rect block, gated on `config_.drawCrosshair`.
- `UI/render.cpp`: add
  `ToggleRow("Aim view crosshair", "aimViewCrosshair",
  &aimviewConfig.drawCrosshair)` directly after the "Aim view" toggle;
  always editable (Render already no-ops when AimView is off).
- `UI/config.cpp`: extend `to_json`/`from_json` for `AimViewConfig`
  with `drawCrosshair` (`j.value(..., true)` so old configs default
  on).
- Explicit non-changes: `UI/fuserRender.h`,
  `espGlobals::drawCrosshair`, Appearance marker styles, any
  native/DMA offsets.

## 5. Data flow

Settings toggle → `g_AimViewWidget.GetConfig().drawCrosshair` →
`saveIfChanged` persists to config file under `aimView.drawCrosshair`.
On load, `from_json` restores it (default `true`). Each frame, Render
reads the flag plus live shared style; no caching, so Appearance
changes reflect immediately in AimView. Fuser reads its own flag
independently; the two flags never interact.

## 6. Error handling / edge cases

- `drawList == nullptr` → `DrawCrosshair` returns immediately.
- `transformValid_ == false` → call site unreachable, nothing drawn.
- AimView `enabled == false` → `Render` early-returns, flag
  irrelevant.
- `crosshairSize` clamped 1–20; out-of-range `crosshairType` falls
  through to the circle branch (same as deleted code).
- Old/missing config key → defaults `true`.
- No scoped-weapon suppression in AimView (matches deleted behavior;
  Fuser keeps its `FrameCameraScoped()` early-out).

## 7. Testing / verification

- Build `Release|x64` with `MSBuild.exe`; expect 0 errors.
- Manual matrix:
  1. AimView on + new toggle on → centre crosshair visible; toggle
     off → gone, Fuser crosshair unaffected.
  2. Fuser crosshair on/off → AimView unaffected.
  3. Change shared size/type/colour → both views update.
  4. Restart app → AimView toggle persists.
  5. AimView disabled → no crosshair regardless of flag.
- Scope limited to `aimview.h/.cpp`, `render.cpp` (one row),
  `config.cpp` (JSON pair). No native/DMA changes.

## 8. Rejected alternatives

- B (full shape parity for X/Square/Triangle/Diamond in AimView):
  deferred; Approach A restores prior circle/cross behavior. Extend
  later if wanted.
- C (shared single flag, no new setting): rejected per requirement
  for a separate AimView option.
