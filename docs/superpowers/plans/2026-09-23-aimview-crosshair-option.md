# AimView Crosshair Option Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore the AimView centre crosshair behind its own `AimViewConfig::drawCrosshair` setting, independent of the Fuser crosshair toggle.

**Architecture:** Add one boolean to `AimViewConfig` (default `true`), restore the deleted `AimViewWidget::DrawCrosshair` verbatim, gate its call on the new flag, add one settings toggle row next to "Aim view", and persist the flag in the `aimView` JSON object. Style (size/type/colour) stays shared read-only state.

**Tech Stack:** C++ (MSBuild, `Release|x64`), ImGui `ImDrawList`, nlohmann::json config.

## Global Constraints

- Do NOT push to `origin` (https://github.com/paul01784/MeatyEFTRelease.git) — work locally only.
- Build with `C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe`, project `MeatyEFTRelease\MeatyEFTRelease.vcxproj`, flags `-p:Configuration=Release -p:Platform=x64`.
- Build output goes to `D:\MeatyEFTRelease-master\x64\Release\` (set via `OutDir` in the `.vcxproj`); do NOT create a `MeatyEFTRelease\x64\` output folder.
- Reuse shared style state (`espGlobals::crosshairSize`, `espGlobals::crosshairType`, `coloursGlobals::crosshair`) — no new style storage.
- Do NOT modify `UI/fuserRender.h`, `espGlobals::drawCrosshair`, Appearance styles, or any native/DMA offsets.

---

## File Structure

- `MeatyEFTRelease/UI/aimview.h` — owns the new flag (`AimViewConfig::drawCrosshair`, default `true`) and the `DrawCrosshair` declaration. One responsibility: widget config + interface.
- `MeatyEFTRelease/UI/aimview.cpp` — restores the `DrawCrosshair` definition and its gated call site inside the clip-rect block. One responsibility: AimView drawing.
- `MeatyEFTRelease/UI/render.cpp` — adds the "Aim view crosshair" toggle row next to "Aim view". One responsibility: settings UI wiring.
- `MeatyEFTRelease/UI/config.cpp` — persists `drawCrosshair` in the `aimView` JSON object (default `true` for old configs). One responsibility: config serialization.

---

### Task 1: AimView widget flag + crosshair drawing

**Files:**
- Modify: `MeatyEFTRelease/UI/aimview.h` (struct lines 7-15, private decls lines 31-34)
- Modify: `MeatyEFTRelease/UI/aimview.cpp` (call site lines 149-155, definition anchor lines 534-538)
- Test: Release x64 build, 0 errors

**Interfaces:**
- Consumes: `espGlobals::crosshairSize` (int), `espGlobals::crosshairType` (int, `1` = cross), `coloursGlobals::crosshair` (`glm::vec4`) — all already visible in `aimview.cpp` (used at lines 559, 949); `<algorithm>` for `std::clamp` already included (line 9). No new includes.
- Produces: `AimViewConfig::drawCrosshair` (`bool`, default `true`); `AimViewWidget::DrawCrosshair(ImDrawList*) const` — used by Task 2 (settings toggle writes the flag) and Task 3 (JSON persists it).

- [ ] **Step 1: Add the flag and declaration in `aimview.h`**

```cpp
struct AimViewConfig {
    bool enabled = true;
    bool drawCrosshair = true;

    float zoom = 2.5f;
    float minimumZoom = 1.0f;
    float maximumZoom = 6.0f;

    ImVec2 defaultWindowSize = ImVec2(420.0f, 240.0f);
};
```

```cpp
    void DrawBackground(ImDrawList* drawList) const;
    void DrawCrosshair(ImDrawList* drawList) const;
    void DrawPlayers(ImDrawList* drawList);
```

- [ ] **Step 2: Restore the `DrawCrosshair` definition in `aimview.cpp`, inserted between `DrawBackground` (ends line 536) and `DrawPlayers` (starts line 538)**

```cpp
void AimViewWidget::DrawCrosshair(ImDrawList* drawList) const {
    if (!drawList)
        return;

    const float size = static_cast<float>(std::clamp(espGlobals::crosshairSize, 1, 20));
    const ImU32 colour = ImGui::ColorConvertFloat4ToU32(ImVec4(
        coloursGlobals::crosshair.r,
        coloursGlobals::crosshair.g,
        coloursGlobals::crosshair.b,
        coloursGlobals::crosshair.a
    ));

    if (espGlobals::crosshairType == 1) {
        drawList->AddLine(
            ImVec2(canvasCentre_.x - size, canvasCentre_.y),
            ImVec2(canvasCentre_.x + size, canvasCentre_.y),
            colour,
            1.0f
        );
        drawList->AddLine(
            ImVec2(canvasCentre_.x, canvasCentre_.y - size),
            ImVec2(canvasCentre_.x, canvasCentre_.y + size),
            colour,
            1.0f
        );
        return;
    }

    drawList->AddCircle(canvasCentre_, size, colour, 0, 1.0f);
}
```

- [ ] **Step 3: Gate the call site in `AimViewWidget::Render` (currently lines 149-155)**

Replace:
```cpp
        DrawPlayers(drawList);

        DrawLoot(drawList);

        DrawContainers(drawList);

        drawList->PopClipRect();
```
With:
```cpp
        DrawPlayers(drawList);

        DrawLoot(drawList);

        DrawContainers(drawList);

        if (config_.drawCrosshair)
            DrawCrosshair(drawList);

        drawList->PopClipRect();
```

- [ ] **Step 4: Build to verify it compiles**

Run (via `cmd.exe`, e.g. a temporary `.bat` file to avoid PowerShell quoting issues with the spaced path):
```bat
"C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe" "D:\MeatyEFTRelease-master\MeatyEFTRelease\MeatyEFTRelease.vcxproj" -p:Configuration=Release -p:Platform=x64
```
Expected: `Build succeeded. 0 Warning(s) 0 Error(s)` (the pre-existing `xutility` C4244 warning may appear; that is not a failure).

- [ ] **Step 5: Commit**

```bash
git add MeatyEFTRelease/UI/aimview.h MeatyEFTRelease/UI/aimview.cpp
git commit -m "feat(aimview): restore crosshair drawing behind AimViewConfig::drawCrosshair"
```

### Task 2: Settings toggle next to "Aim view"

**Files:**
- Modify: `MeatyEFTRelease/UI/render.cpp` (line 1522)
- Test: Release x64 build, 0 errors; manual UI check

**Interfaces:**
- Consumes: `g_AimViewWidget.GetConfig().drawCrosshair` (`bool&`) produced in Task 1; existing helpers `menuLayout::ToggleRow(const char*, const char*, bool*)` and `saveIfChanged(bool)` already used on the adjacent line.
- Produces: settings UI wiring — Task 3 persists the value this toggle writes.

- [ ] **Step 1: Add the toggle row directly after the "Aim view" row (line 1522)**

Replace:
```cpp
                 saveIfChanged(menuLayout::ToggleRow("Aim view", "aimView", &aimviewConfig.enabled));
```
With:
```cpp
                 saveIfChanged(menuLayout::ToggleRow("Aim view", "aimView", &aimviewConfig.enabled));
                 saveIfChanged(menuLayout::ToggleRow("Aim view crosshair", "aimViewCrosshair", &aimviewConfig.drawCrosshair));
```
Keep the 17-space indentation to match the surrounding rows. The key `"aimViewCrosshair"` is new (verified absent from `render.cpp`).

- [ ] **Step 2: Build to verify it compiles**

Run: same MSBuild command as Task 1, Step 4.
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 3: Commit**

```bash
git add MeatyEFTRelease/UI/render.cpp
git commit -m "feat(aimview): add Aim view crosshair toggle next to Aim view"
```

### Task 3: Persist the flag in config JSON

**Files:**
- Modify: `MeatyEFTRelease/UI/config.cpp` (`to_json` lines 559-565, `from_json` lines 567-574)
- Test: Release x64 build, 0 errors; restart-persistence manual check

**Interfaces:**
- Consumes: `AimViewConfig::drawCrosshair` (`bool`, default `true`) from Task 1. `std::clamp` already used in this file; no new includes.
- Produces: persisted `aimView.drawCrosshair` key — nothing consumes it downstream; this completes the feature.

- [ ] **Step 1: Extend `to_json` for `AimViewConfig`**

Replace:
```cpp
void to_json(nlohmann::json& j, const AimViewConfig& a)
{
    j = nlohmann::json{
        {"enabled", a.enabled},
        {"zoom", a.zoom}
    };
}
```
With:
```cpp
void to_json(nlohmann::json& j, const AimViewConfig& a)
{
    j = nlohmann::json{
        {"enabled", a.enabled},
        {"drawCrosshair", a.drawCrosshair},
        {"zoom", a.zoom}
    };
}
```

- [ ] **Step 2: Extend `from_json` for `AimViewConfig`**

Replace:
```cpp
void from_json(const nlohmann::json& j, AimViewConfig& a)
{
    a.enabled = j.value("enabled", a.enabled);
    a.zoom = std::clamp(
        j.value("zoom", a.zoom),
        a.minimumZoom,
        a.maximumZoom);
}
```
With:
```cpp
void from_json(const nlohmann::json& j, AimViewConfig& a)
{
    a.enabled = j.value("enabled", a.enabled);
    a.drawCrosshair = j.value("drawCrosshair", a.drawCrosshair);
    a.zoom = std::clamp(
        j.value("zoom", a.zoom),
        a.minimumZoom,
        a.maximumZoom);
}
```
`a.drawCrosshair` defaults to `true` via the member initializer, so old configs missing the key get crosshair ON, per the approved design. The nested `aimView.drawCrosshair` key cannot clash with ESP's top-level `drawCrosshair` key (config.cpp lines 485/541).

- [ ] **Step 3: Build to verify it compiles**

Run: same MSBuild command as Task 1, Step 4.
Expected: `Build succeeded. 0 Error(s)`.

- [ ] **Step 4: Run the manual verification matrix**

1. AimView on + new toggle on → centre crosshair visible in Aim View; toggle off → gone, Fuser crosshair unaffected.
2. Fuser "Crosshair" toggle on/off → AimView crosshair unaffected.
3. Change shared crosshair size/type/colour (Appearance) → both views update.
4. Restart app → AimView toggle persists (config file contains `aimView.drawCrosshair`).
5. AimView disabled → no AimView crosshair regardless of the new flag.

- [ ] **Step 5: Commit**

```bash
git add MeatyEFTRelease/UI/config.cpp
git commit -m "feat(aimview): persist Aim view crosshair flag in config"
```
