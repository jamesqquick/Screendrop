# Design: "Save button writes to folder without prompting"

**Date:** 2026-06-15
**Status:** Approved (pending spec review)

## Problem

Today, two distinct behaviors are welded to a single preference (`autoSave`,
exposed as the "Save to folder" checkbox in Screenshots → After Capture):

1. **Auto-save on capture** — captures land in the export folder with no
   interaction.
2. **Preview card Save button behavior** — when `autoSave` is ON, clicking the
   preview card's "Save" button silently writes to the export folder; when OFF,
   it shows an `NSSavePanel` prompt.

Because both behaviors share one toggle, a valid combination is impossible:
"don't auto-save everything on capture (I want to review first), but when I *do*
click Save, just put it in my configured folder without prompting."

## Goal

Split these into two independent settings so the manual Save button can write
silently to the configured export folder regardless of the auto-save-on-capture
setting.

## Non-Goals

- The annotation editor's "Save As" button is out of scope; it continues to
  always show an `NSSavePanel`.
- Auto-save-on-capture behavior is unchanged.
- Recording-specific after-capture save (`afterCapture.recording.save`) is
  unchanged.

## Design

### 1. New preference

Add to `ScreendropPreferences` (`Screendrop/ScreendropPreferences.swift`):

```swift
static let saveButtonUsesFolderKey = "saveButtonUsesConfiguredFolder"

static var saveButtonUsesConfiguredFolder: Bool {
    if UserDefaults.standard.object(forKey: saveButtonUsesFolderKey) == nil {
        return autoSave            // inherit from auto-save when unset
    }
    return UserDefaults.standard.bool(forKey: saveButtonUsesFolderKey)
}
```

**Migration / default behavior:** when the key is unset, the accessor falls back
to the current `autoSave` value. Existing users see zero behavior change. The
two settings become fully independent the moment the new toggle is changed.

### 2. Decouple the preview Save button

In `ScreenshotPreviewStack.save(id:)`
(`Screendrop/ScreenshotPreviewStack.swift`, currently line 312), change the gate
from:

```swift
if ScreendropPreferences.autoSave {
```

to:

```swift
if ScreendropPreferences.saveButtonUsesConfiguredFolder {
```

Everything inside the branch (silent write to `exportDirectory` via
`saveToDefaultLocation` / `saveVideoToDefaultLocation`, then `dismiss`) and the
`NSSavePanel` fallback below it stay the same.

**Deliberate non-changes:**

- `republishLatestVersion` (currently line 377) keeps checking `autoSave`. It
  updates a file that was *already auto-saved on capture*, which is genuinely the
  capture setting, not the button setting.
- The Save button routes **both screenshots and videos** through this single
  gate. This is preserved: the new setting governs the preview Save button for
  both kinds, matching today's behavior (which used `autoSave` for both).

### 3. Settings UI

Add a toggle in `Screendrop/SettingsGeneralPane.swift`, inside the existing
`Section("Save Location")`, directly below the export folder picker buttons:

```swift
@AppStorage(ScreendropPreferences.saveButtonUsesFolderKey)
private var saveButtonUsesFolder = false
```

> **Note on default binding:** `@AppStorage` with a `false` default does not
> replicate the "inherit from auto-save when unset" fallback. To keep the UI in
> sync with the accessor's effective value, initialize the toggle's displayed
> state from `ScreendropPreferences.saveButtonUsesConfiguredFolder` on appear (or
> use a `Binding` whose getter reads the accessor and whose setter writes the
> key). The runtime behavior in `save(id:)` always reads the accessor, so the
> source of truth is correct regardless; this note only concerns the toggle's
> initial displayed position.

Toggle copy:

> **Save without choosing a location**
> When you click Save, write straight to the export folder instead of asking
> where to put it.

The existing "Save to folder" checkbox in Screenshots → After Capture is
untouched and remains the auto-save-on-capture control.

## Files Touched

- `Screendrop/ScreendropPreferences.swift` — new key + accessor.
- `Screendrop/ScreenshotPreviewStack.swift` — one-line gate change.
- `Screendrop/SettingsGeneralPane.swift` — new toggle in Save Location section.

No new files. No new dependencies.

## Verification

- Build succeeds via `xcodebuild` (the only automated verification available;
  no test target exists).
- Manual checks:
  - Fresh install (no keys set): preview Save behaves exactly as `autoSave`
    dictates (prompt when off, silent when on).
  - Auto-save OFF + new toggle ON: capture is not auto-saved, but clicking Save
    writes silently to the export folder.
  - Auto-save ON + new toggle OFF: capture auto-saves, but clicking Save shows
    the panel.
  - Annotation editor "Save As" still always prompts.
