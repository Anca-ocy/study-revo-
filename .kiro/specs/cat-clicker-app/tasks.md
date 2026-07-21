# Implementation Plan: Cat Clicker App

## Overview

Build the app as a single `index.html` file containing all markup, inline styles, and a `<script>` block. Work proceeds in layers: HTML scaffold → CSS animations → static data → logic modules → UI wiring → responsive polish → automated tests.

---

## Tasks

- [x] 1. Create the HTML scaffold and inline CSS
  - [x] 1.1 Create `index.html` with the full HTML5 boilerplate
    - Add `<meta>` viewport tag, page `<title>`, and Tailwind CDN `<script>` tag
    - Add the `<style>` block containing all `@keyframes` rules (`anim-bounce`, `anim-shake`, `anim-spin`, `anim-pulse`, `anim-flip`) and `animation-duration: 600ms` / `animation-fill-mode: forwards` on each class
    - Add `@media (prefers-reduced-motion: reduce)` override block
    - _Requirements: 4.1, 4.2, 7.3, 8.1, 8.2_

  - [x] 1.2 Add the body layout and key DOM elements
    - `<body>` with Tailwind classes `min-h-screen bg-amber-50 flex flex-col items-center justify-center gap-6 p-4`
    - Fixed `#counter-bar` with `#click-count` span, `#sound-toggle` checkbox, and `#reset-btn` button
    - `#cat` div (`role="button"`, `tabindex="0"`, `aria-label="Click the cat"`) containing the 🐱 emoji at `text-[10rem]`
    - `#message-display` div with `min-h-[2rem]`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.4, 5.2, 6.2, 7.1, 7.2, 7.4_

- [ ] 2. Implement static data constants
  - [-] 2.1 Define `ANIMATION_MAP` and `SOUND_MAP` constants
    - `ANIMATION_MAP`: plain object mapping the 5 `AnimationKey` strings (`"bounce"`, `"shake"`, `"spin"`, `"pulse"`, `"flip"`) to CSS class name strings (e.g. `"anim-bounce"`)
    - `SOUND_MAP`: plain object mapping `SoundKey` strings to inline base64 data-URI strings for short audio clips (`meow`, `hiss`, `purr`, `chirp`)
    - Both defined with `const` inside the `<script>` block before any modules reference them
    - _Requirements: 3.2, 4.1, 5.1, 8.3_

  - [-] 2.2 Define the `REACTION_POOL` frozen array
    - Exactly 8 `ReactionItem` objects: `{ id, message, animation, soundKey }` matching the data model in the design
    - Call `Object.freeze()` on the array
    - All `animation` values reference keys in `ANIMATION_MAP`; all non-null `soundKey` values reference keys in `SOUND_MAP`
    - _Requirements: 3.1, 3.2, 3.4, 3.5_

- [ ] 3. Implement `AppState` and `ReactionEngine`
  - [~] 3.1 Define the `AppState` mutable object
    - Fields: `clickCount: 0`, `lastReactionId: null`, `soundEnabled: false`
    - _Requirements: 2.3, 5.5, 6.1_

  - [~] 3.2 Implement `ReactionEngine.pick()`
    - Filter `REACTION_POOL` to exclude `AppState.lastReactionId`
    - If filtered pool is empty (degenerate single-item pool) return the sole item
    - Select uniformly at random using `Math.floor(Math.random() * filtered.length)`
    - Update `AppState.lastReactionId` to the selected reaction's `id`
    - Return the selected `ReactionItem`
    - _Requirements: 2.1, 3.2, 3.3_

  - [ ]* 3.3 Write property test for `ReactionEngine.pick()` — no consecutive repeats
    - **Property 2: Reaction selection never repeats consecutively**
    - **Validates: Requirements 3.3**
    - Use fast-check `fc.integer({ min: 2, max: 200 })` to generate sequence lengths; run `pick()` that many times; assert no two adjacent results share the same `id`

  - [ ]* 3.4 Write property test for `ReactionEngine.pick()` — uniform distribution
    - **Property 3: Reaction selection is uniform over non-previous reactions**
    - **Validates: Requirements 3.2, 3.3**
    - For each reaction as `lastReactionId`, run `pick()` 500 times; assert the previous id never appears and that every other id appears with roughly equal frequency (within ±30% of expected share)

  - [~] 3.5 Implement `ReactionEngine.execute(reaction)`
    - Retrieve the CSS class from `ANIMATION_MAP[reaction.animation]`; emit `console.warn` and return early if not found
    - Remove any existing animation class from `#cat`, force reflow (`cat.offsetWidth`), then add the new class
    - On `animationend`, remove the class; install a `setTimeout(cleanup, 1100)` fallback in case the event never fires
    - Set `#message-display.textContent` to `reaction.message`
    - If `AppState.soundEnabled` is true and `reaction.soundKey` is non-null, call `SoundPlayer.play(reaction.soundKey)`
    - _Requirements: 3.4, 3.5, 4.2, 4.3, 4.4, 5.1_

  - [ ]* 3.6 Write property test for `ReactionEngine.execute()` — correct message and animation
    - **Property 6: Reaction execution applies correct message and animation**
    - **Validates: Requirements 3.4, 3.5**
    - Use fast-check `fc.constantFrom(...REACTION_POOL)` to pick any reaction; call `execute()`; assert `#message-display.textContent === reaction.message` and `#cat.classList.contains(ANIMATION_MAP[reaction.animation])`

  - [~] 3.7 Implement `ReactionEngine.reset()`
    - Set `AppState.lastReactionId = null`
    - _Requirements: 6.3, 6.4_

- [~] 4. Checkpoint — core logic complete
  - Ensure `pick()`, `execute()`, and `reset()` are wired together correctly; call each manually in the browser console to verify no runtime errors before proceeding.

- [ ] 5. Implement `SoundPlayer`
  - [~] 5.1 Implement `SoundPlayer.play(soundKey)`
    - Early-return no-op if `soundKey` is `null` or `undefined`
    - Look up `SOUND_MAP[soundKey]`; if the key is missing emit `console.warn` and return
    - Create `new Audio(dataUri)` and call `.play()`; catch any rejected Promise silently
    - _Requirements: 5.1, 5.3, 5.4_

  - [~] 5.2 Implement `SoundPlayer.isEnabled()` and `SoundPlayer.setEnabled(val)`
    - Delegate to `AppState.soundEnabled`
    - _Requirements: 5.2, 5.3, 5.4, 5.5_

  - [ ]* 5.3 Write property test for `SoundPlayer` — sound gating
    - **Property 5: Sound toggle gates playback**
    - **Validates: Requirements 5.1, 5.3, 5.4, 5.5**
    - Use fast-check `fc.record({ soundEnabled: fc.boolean(), soundKey: fc.oneof(fc.constantFrom("meow","hiss","purr","chirp"), fc.constant(null)) })` to generate inputs; mock `Audio` constructor; assert `.play()` was called iff `soundEnabled && soundKey !== null`

- [ ] 6. Implement `init()` and DOM event wiring
  - [~] 6.1 Implement the cat click handler inside `init()`
    - Query DOM refs: `#cat`, `#message-display`, `#click-count`, `#sound-toggle`, `#reset-btn`
    - On click (and `keydown` Enter/Space for keyboard accessibility): call `ReactionEngine.pick()`, then `ReactionEngine.execute(reaction)`, increment `AppState.clickCount`, update `#click-count.textContent`
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

  - [~] 6.2 Implement the sound toggle and reset handlers inside `init()`
    - Sound toggle `change` event → `SoundPlayer.setEnabled(event.target.checked)`; sync `AppState.soundEnabled`
    - Reset button `click` event → set `AppState.clickCount = 0`, clear `#message-display.textContent`, call `ReactionEngine.reset()`, update `#click-count.textContent`
    - Call `init()` inside a `DOMContentLoaded` listener
    - _Requirements: 5.2, 5.3, 5.4, 6.3, 6.4_

  - [ ]* 6.3 Write property test for counter increment
    - **Property 1: Click counter increments monotonically**
    - **Validates: Requirements 2.3, 2.4, 6.1**
    - Use fast-check `fc.integer({ min: 1, max: 1000 })` for N; simulate N click-handler calls; assert `AppState.clickCount === N`

  - [ ]* 6.4 Write property test for counter reset
    - **Property 4: Counter reset returns to zero**
    - **Validates: Requirements 6.3, 6.4**
    - Use fast-check `fc.integer({ min: 0, max: 10000 })` for C; set `AppState.clickCount = C`; invoke reset handler; assert `AppState.clickCount === 0` and `#message-display.textContent === ""`

- [~] 7. Checkpoint — full interaction loop complete
  - Open `index.html` via `file://` in a browser; click the cat several times; verify counter increments, messages change, animations play, sound toggle works, and reset clears state.

- [ ] 8. Write unit tests for edge cases and data integrity
  - [~] 8.1 Write unit tests for `AppState` and data constants
    - Counter starts at 0 — _Requirements: 6.1_
    - Sound is disabled by default — _Requirements: 5.5_
    - `REACTION_POOL` contains ≥ 8 items with distinct ids — _Requirements: 3.1_
    - `ANIMATION_MAP` contains exactly 5 keys — _Requirements: 4.1_

  - [~] 8.2 Write unit tests for `ReactionEngine` and `SoundPlayer` edge cases
    - Counter is 0 after reset regardless of prior count — _Requirements: 6.3_
    - Message display clears after reset — _Requirements: 6.4_
    - `SoundPlayer.play(null)` does not throw — Error handling
    - Animation class is removed after `animationend` fires — _Requirements: 4.4_
    - Animation class is removed after `setTimeout` fallback when `animationend` never fires — Error handling

  - [ ]* 8.3 Write property test for animation restart on rapid click
    - **Property 7: Animation restart on rapid click**
    - **Validates: Requirements 4.3**
    - Simulate a second click while an animation class is already present; assert the class was removed and re-applied (spy on `classList.remove` and `classList.add`)

- [ ] 9. Responsive layout verification tests
  - [~] 9.1 Write automated layout checks for viewport extremes
    - Use `jsdom` or a lightweight DOM emulation to assert that no element has `scrollWidth > clientWidth` at viewport widths 320px and 2560px
    - Assert `#cat` does not overflow its container at either extreme
    - _Requirements: 7.1, 7.2, 7.4_

- [~] 10. Final checkpoint — all tests pass
  - Run all unit and property-based tests with `node --experimental-vm-modules`; ensure zero failures; ask the user if any questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP build.
- The test suite requires [fast-check](https://github.com/dubzzz/fast-check) as a dev dependency (`npm install --save-dev fast-check`); the production `index.html` has no npm dependencies.
- All property tests must run a **minimum of 100 iterations** and include the comment `// Feature: cat-clicker-app, Property <N>: <property text>`.
- The animation restart test (Property 7 / task 8.3) requires a spy on `element.classList` because the DOM side effect is the observable outcome.
- Checkpoints in tasks 4 and 7 are manual browser checks; task 10 is the automated test gate.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "2.2"] },
    { "id": 2, "tasks": ["3.1"] },
    { "id": 3, "tasks": ["3.2", "3.5", "3.7"] },
    { "id": 4, "tasks": ["3.3", "3.4", "3.6", "5.1", "5.2"] },
    { "id": 5, "tasks": ["5.3", "6.1", "6.2"] },
    { "id": 6, "tasks": ["6.3", "6.4", "8.1", "8.2"] },
    { "id": 7, "tasks": ["8.3", "9.1"] }
  ]
}
```
