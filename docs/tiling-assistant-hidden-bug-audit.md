# Tiling Assistant hidden-bug audit

Static audit only. This is not proof that no other bugs exist. It is a ranked list of the most likely user-facing failure modes found by reading the code.

## Scope audited

- keyboard tiling / dynamic tiling state
- drag / move / resize / restore-on-grab
- tiling popup / popup-all-workspaces
- layout activation / app-launch tiling
- tile editing mode
- workspace / monitor / session-lock restore

## Validation done

- `node --check` passed on all JS files
- findings below are based on code inspection, not GNOME end-to-end runtime repro

---

## Priority 1

### [x] P1. Layout app-launch race / async step desync

- **Status:** Fixed. Layout activation now waits for each `openAppTiled()` request to finish before advancing to the next layout item, and successful launched windows are recorded in `_tiledWithLayout`.

- **Files:** `tiling-assistant@leleat-on-github/src/extension/layoutsManager.js`, `tiling-assistant@leleat-on-github/src/extension/tilingWindowManager.js`
- **Path:** `LayoutManager._openAppTiled()` -> `Twm.openAppTiled()`
- **Why risky:**
  - `layoutsManager._openAppTiled()` calls `Twm.openAppTiled(app, this._currRect);`
  - then immediately calls `this._step();`
  - `Twm.openAppTiled()` is asynchronous and tracks only one global pending `window-created` / `first-frame` flow
- **Likely user trouble:**
  - wrong app gets tiled
  - intended app misses its target rect
  - multi-item layouts with `appId` fill incorrectly
- **Suggested repro:**
  - create a layout with multiple `appId` items
  - activate it with apps that open slowly or spawn splash/loading windows
  - verify each launched app lands in the intended rect in order

### [x] P1. Quick move-release may crash when restore-on-grab-end is enabled

- **Status:** Fixed. The move handler now seeds `_lastPointerPos` when the handler is created and again at grab start, so the restore-on-grab-end path no longer depends on a prior `position-changed` event.

- **File:** `tiling-assistant@leleat-on-github/src/extension/moveHandler.js`
- **Path:** `_onMoveFinished()`
- **Why risky:**
  - code uses `this._lastPointerPos.x`
  - `_lastPointerPos` is assigned in `_onMoving()`, not initialized in the constructor
  - a quick grab/release without `position-changed` can leave it undefined
- **Likely user trouble:**
  - move-end crash
  - broken untile / restore on a fast click-drag-release path
- **Suggested repro:**
  - set restore mode to `ON_GRAB_END`
  - start dragging a tiled window
  - release immediately without actual movement

### [x] P1. Resize handler can keep stale state after interrupted resize

- **Status:** Fixed. `_onResizeFinished()` now clears resize bookkeeping before every interrupted return path, including cases where the window is no longer tiled or the pre-grab rect is missing.

- **File:** `tiling-assistant@leleat-on-github/src/extension/resizeHandler.js`
- **Path:** `_onResizeFinished()`
- **Why risky:**
  - function returns early on `!window.isTiled`
  - `_preGrabRects.clear()` and `_resizeOps.clear()` happen later
  - if tiling state changes mid-resize, stale bookkeeping can survive
- **Likely user trouble:**
  - later resizes affect wrong windows
  - corrupt passive resize behavior
- **Suggested repro:**
  - start resizing a tiled window
  - force a tiling-state change before release if possible
  - perform another resize and watch for wrong companion window updates

### [x] P1. `openAppTiled()` uses singleton in-flight state

- **Status:** Already fixed in current code. `openAppTiled()` now uses per-request state (`_openAppTiledRequests`, request-local signal ids, per-request first-frame tracking, and claimed-window bookkeeping) rather than singleton pending fields, and the layout sequencing fix above now consumes that request lifecycle explicitly.

- **File:** `tiling-assistant@leleat-on-github/src/extension/tilingWindowManager.js`
- **Path:** `openAppTiled()`
- **Why risky:**
  - pending actor / signal IDs live in global singleton fields:
    - `_openAppTiledCreateId`
    - `_openAppTiledFirstFrameId`
    - `_openAppTiledFirstFrameActor`
  - multiple `window-created` events can overwrite earlier pending state
- **Likely user trouble:**
  - loading screen wins over real window
  - wrong spawned window gets tiled
  - overlapping app-launch tiling requests interfere with each other
- **Suggested repro:**
  - tile-launch an app that spawns more than one window
  - or trigger two app-launch tiling requests back-to-back

---

## Priority 2

### [x] P2. Dynamic keyboard tiling depends on exact rect equality

- **Status:** Fixed. Dynamic tiling state now matches the canonical tile rectangles with tolerance instead of relying on exact edge equality, so gaps and rounding drift no longer misclassify tiled halves and quarters.

- **File:** `tiling-assistant@leleat-on-github/src/extension/keybindingHandler.js`
- **Path:** `_dynamicTilingState()`
- **Why risky:**
  - left/right/top/bottom classification uses strict equality on `x`, `y`, `x2`, `y2`, width/height
  - rounding, gaps, or scaling can make a visually-correct tile fail the classifier
- **Likely user trouble:**
  - hotkey does the wrong next action
  - unexpected untile instead of moving to another half/quarter
- **Suggested repro:**
  - use dynamic tiling state mode
  - test with gaps, fractional scaling, multi-monitor, and repeated directional hotkeys

### [x] P2. Popup cross-workspace retile ordering looks race-prone

- **Status:** Fixed. Cross-workspace popup tiling now clears stale tiling props before the workspace and monitor move, so workspace-change handlers do not observe old tile metadata during retile.

- **File:** `tiling-assistant@leleat-on-github/src/extension/tilingPopup.js`
- **Path:** `_tileWindow()`
- **Why risky:**
  - code changes workspace and monitor before clearing old tiling props
  - `workspace-changed` can fire while stale tiling metadata still exists
- **Likely user trouble:**
  - flicker
  - temporary restore / untile
  - inconsistent popup retile behavior for windows pulled from other workspaces
- **Suggested repro:**
  - enable popup-all-workspaces
  - choose a tiled window from another workspace
  - observe for temporary geometry jumps or incorrect untiling

### [x] P2. `appId` layout loops stop after the seeded app launch

- **Status:** Fixed. Layout activation now treats `appId` loop items as a seeded loop: the attached app is launched once into the loop rect, existing looped windows are redistributed immediately, and subsequent loop iterations switch to the Tiling Popup instead of skipping the loop or relaunching the app forever.

- **File:** `tiling-assistant@leleat-on-github/src/extension/layoutsManager.js`
- **Path:** `_step()`, `_openAppTiled()`
- **Why risky:**
  - layout items allow both `appId` and `loopType`
  - `_openAppTiled()` always advanced with `this._step()`
  - loop continuation was only implemented for popup-driven items in `_onTilingPopupClosed()`
- **Likely user trouble:**
  - master-and-stack style layouts cannot be seeded with an app launch
  - loop items with an attached app only place the launched app, then jump to the next layout item
- **Suggested repro:**
  - create a layout item with both `appId` and `loopType`
  - activate the layout and let the app launch tile into that rect
  - verify the next step keeps tiling into the same rect via the popup instead of advancing immediately

### [x] P2. Session restore can revive stale tiling metadata

- **Status:** Fixed. Session restore now validates restored tiled state against the current work area and drops stale tiling metadata instead of blindly reattaching pre-lock geometry after topology changes.

- **File:** `tiling-assistant@leleat-on-github/extension.js`
- **Path:** `_saveBeforeSessionLock()`, `_loadAfterSessionLock()`
- **Why risky:**
  - restore reattaches `isTiled`, `tiledRect`, `untiledRect`, and tile groups
  - it does not reconcile those values against actual post-unlock geometry / monitor topology
- **Likely user trouble:**
  - later untile or resize uses stale rects
  - group behavior after unlock feels wrong
- **Suggested repro:**
  - tile windows
  - lock session
  - change monitor setup or workspace context if possible
  - unlock and test raise/untiling/resize behavior

### [x] P2. Session-lock tile-group restore identity mismatch

- **Status:** Fixed. Session-lock persistence now stores tile groups in stable-sequence terms and rebuilds them against the current runtime window IDs on restore, so group raise/resize wiring uses the correct post-unlock identities.

- **File:** `tiling-assistant@leleat-on-github/extension.js`
- **Path:** `_saveBeforeSessionLock()`, `_loadAfterSessionLock()`
- **Why risky:**
  - window geometry/state is restored by `get_stable_sequence()`
  - persisted `tileGroups` come from `_twm.getTileGroups()` and are keyed by `window.get_id()`
- **Likely user trouble:**
  - after unlock, group raise / group resize can fail or reattach incorrectly
- **Suggested repro:**
  - lock/unlock with a visible tile group
  - verify tile-group raise and joint resize behavior still works

### [x] P2. Session-lock restore only persists the active workspace

- **Status:** Fixed. Session-lock save/restore now enumerates windows across all workspaces, so lock/unlock persistence no longer drops tiling metadata for non-active-workspace windows.

- **File:** `tiling-assistant@leleat-on-github/extension.js`
- **Path:** `_saveBeforeSessionLock()`, `_loadAfterSessionLock()`
- **Why risky:**
  - saved/restore window enumeration is limited to the active workspace
  - `disable()` later deletes custom tiling properties for all windows
  - windows on other workspaces can lose their tiling metadata across lock/unlock
- **Likely user trouble:**
  - tiled windows on non-active workspaces stop behaving as tiled after unlock
  - untile / tile-group behavior becomes inconsistent until windows are re-tiled
- **Suggested repro:**
  - tile windows on multiple workspaces
  - switch away from one of those workspaces
  - lock and unlock the session
  - revisit the other workspace and verify untiling / group behavior still works

---

## Priority 3

### [x] P3. Layout manager can remove the wrong remaining window

- **Status:** Fixed. The layout manager now removes the just-tiled window from `_remainingWindows` only when it is actually present, so popup churn no longer splices the wrong entry on a missing match.

- **File:** `tiling-assistant@leleat-on-github/src/extension/layoutsManager.js`
- **Path:** `_onTilingPopupClosed()`
- **Why risky:**
  - does `const i = this._remainingWindows.indexOf(tiledWindow);`
  - then `this._remainingWindows.splice(i, 1);`
  - there is no guard for `i !== -1`
- **Likely user trouble:**
  - wrong window disappears from the remaining layout sequence
- **Suggested repro:**
  - start a layout
  - change the candidate window set while popup is open
  - confirm the next popup still offers the correct remaining windows

### [x] P3. `appId` layout launches stay in `_remainingWindows`

- **Status:** Fixed. Successful `appId` launches are now removed from `_remainingWindows` as soon as they are recorded in `_tiledWithLayout`, so later popup steps no longer offer already-placed windows again.

- **File:** `tiling-assistant@leleat-on-github/src/extension/layoutsManager.js`
- **Path:** `_openAppTiled()`
- **Why risky:**
  - the launched window is tracked as already tiled for the layout
  - later popup steps still build candidates from `_remainingWindows`
  - the same launched window can be offered again and re-tiled into the wrong rect
- **Likely user trouble:**
  - multi-step layouts with `appId` entries can reuse a window that was already placed
  - later popup steps may target the wrong remaining windows
- **Suggested repro:**
  - create a layout with at least one `appId` item followed by a popup-driven item
  - activate the layout and let the app launch tile successfully
  - verify the next popup does not offer the already launched window again

### [x] P3. Wayland drag-restore still has hardcoded fallback width

- **Status:** Fixed. Wayland drag-restore now falls back to the live post-untile frame width instead of a hardcoded `1000`, which keeps pointer anchoring tied to real geometry.

- **File:** `tiling-assistant@leleat-on-github/src/extension/moveHandler.js`
- **Path:** `_restoreSizeAndRestartGrab()`
- **Why risky:**
  - if no reliable untiled geometry is available, fallback width is `1000`
- **Likely user trouble:**
  - jumpy pointer anchoring
  - odd restore geometry during move restart on Wayland
- **Suggested repro:**
  - on Wayland, drag tiled/maximized windows that lack stable untiled geometry
  - watch for jumpy restore/restart behavior

### [x] P3. Tile editing mode may drift from real window state

- **Status:** Fixed. Tile editing mode now synchronizes its local window list against live tiled windows before key handling and hardens delete/restore/popup replacement flows against stale entries.

- **File:** `tiling-assistant@leleat-on-github/src/extension/tileEditingMode.js`
- **Path:** multiple (`handleKeyPress()`, popup replacement flow, restore/delete flows)
- **Why risky:**
  - mode keeps a local `_windows` list while windows can be deleted, replaced, untiled, or retiled asynchronously
- **Likely user trouble:**
  - stale selection
  - wrong replacement target
  - focus mismatch after delete/restore/popup actions
- **Suggested repro:**
  - enter tile editing mode
  - mix delete, restore, popup replace, and move/resize commands quickly
  - verify selection/focus always tracks the right live window

### [x] P3. App switcher tile-group items can keep closed windows cached

- **Status:** Fixed. Tile-group items in the custom app switcher now drop closed-app windows from `cachedWindows` and rebuild their icon/label state when an app stops, so activating the item no longer targets stale windows.

- **File:** `tiling-assistant@leleat-on-github/src/extension/altTab.js`
- **Path:** `AppSwitcherItem.removeApp()`
- **Why risky:**
  - app-stop updates previously removed only the visible icon
  - `cachedWindows` stayed unchanged
  - tile-group activation still uses `cachedWindows[0]`
- **Likely user trouble:**
  - app switcher tile-group entry can try to activate a window that was already closed
  - label/icon state can drift after one app in a tile group exits
- **Suggested repro:**
  - enable tile groups in the app switcher
  - create a tile group with windows from different apps
  - close one app while the other remains open
  - verify the remaining app-switcher item still activates the live window correctly

---

## Lower-confidence behavior smells

These may be intentional, but they are worth reviewing because users may report them as bugs:

- opposite-direction dynamic hotkeys may untile rather than move to the opposite half
- maximize toggle logic may untile on repeated maximize-style hotkeys
- dynamic focus mode uses a tile-group snapshot that can drift under rapid churn

---

## Recommended fix order

1. Layout app-launch race / `openAppTiled()` singleton state
2. Move-end quick-release crash path
3. Resize stale state after interrupted resize
4. Session-lock restore / tile-group restore
5. Popup cross-workspace retile ordering
6. Dynamic keyboard tiling equality assumptions

---

## Recommended manual test order

1. multi-`appId` layout activation
2. quick drag-release with restore-on-end enabled
3. interrupted resize on tiled groups
4. lock/unlock with tile groups
5. popup-all-workspaces retile
6. dynamic keyboard tiling with gaps/scaling
