---
id: history-view-spec
title: History View Redesign Spec
---

## Why We Are Resetting

The previous History View iteration achieved some desired behavior (notably snapshots), but playback reliability and layout consistency regressed across camera switching, desktop playback, and mobile ultrawide handling.

This spec defines a **fresh implementation approach**. It keeps the intended user outcomes but does not assume the previous architecture or state flow.

## Scope

- Applies to History/Recording playback UX and layout.
- Keeps existing snapshot behavior as-is (capture current playback frame + recording timestamp filename).
- No backend API changes.
- No new frontend dependencies.

## Product Outcomes

1. Camera switching in History is as reliable as baseline `dev`.
2. Desktop History does not get stuck on black video after initial load.
3. Mobile supports ultrawide cameras without persistent wasted space.
4. Mobile timeline and video share space predictably while zooming.
5. Fullscreen behavior is robust and independent from timeline constraints.

## Non-Goals

- Rewriting Timeline components unrelated to viewport behavior.
- Replacing HLS playback library.
- Shipping new visual controls like `Fit/Fill`.

## Engineering Principles

1. Separate media readiness from viewport/layout state.
2. Use explicit state transitions instead of implicit booleans spread across components.
3. Keep all geometry math in pure functions with unit tests.
4. Use camera/session tokens so late async events cannot corrupt current state.
5. Ship in small milestones with rollback points.

## Proposed Architecture (Fresh)

### 1. Playback Session Layer

Introduce a dedicated session hook for History playback (name can vary, e.g. `useHistoryPlaybackSession`):

- Owns source lifecycle and readiness state:
  - `idle`
  - `loadingSource`
  - `awaitingMetadata`
  - `ready`
  - `buffering`
  - `error`
- Produces a `sessionId` token on camera/timeRange/source changes.
- Ignores late events (`loadedmetadata`, `playing`, `error`) from stale sessions.

This avoids race conditions where one source update leaks into another camera state.

### 2. Viewport Engine Layer

Create a pure geometry module (e.g. `historyViewportEngine.ts`) responsible only for:

- base fit rectangle for current container + media dimensions
- max/min zoom
- pan clamping
- mode-aware constraints:
  - `embeddedWithTimeline`
  - `fullscreen`

Inputs:
- container size
- media intrinsic size
- mode
- timeline share (mobile embedded)
- user transform state

Outputs:
- render rect
- scale bounds
- pan bounds
- next transform

### 3. Timeline Space Allocator (Mobile Embedded)

Create a dedicated allocator state for vertical split between video and timeline:

- `timelineShare` in `[0.40, defaultShare]`
- Zoom gestures in embedded mode have two phases:
  - Phase A: reduce timeline share until `0.40` floor
  - Phase B: keep timeline at `0.40` and apply content zoom/pan
- Zoom-out reverses Phase B first, then increases timeline share back toward default.

This prevents paradoxical behavior where zoom-out appears to crop more.

### 4. Render Surface Layer

Render rules:

- Never commit transform layout until media dimensions are valid (`videoWidth > 0`, `videoHeight > 0`).
- While awaiting dimensions, show deterministic loading state or preview fallback.
- Once dimensions are known, initialize fit exactly once per session.

This mirrors baseline reliability where sizing settles after metadata is known.

## Behavioral Requirements

### Camera Switching

- On camera change:
  - start new playback session
  - reset transform state for that session
  - keep old session visually isolated
- No black persistent surface after switch when playable media exists.

### Desktop Embedded

- Timeline remains visible and unaffected by content zoom.
- Video remains visible when ready and respects viewport bounds.
- Fullscreen toggle preserves current timestamp and re-computes bounds safely.

### Mobile Embedded

- No `Fit/Fill` toggle.
- Video area can grow by shrinking timeline down to 40% of viewport.
- After timeline floor is reached, additional pinch only zooms video content.
- Pan uses updated clamp bounds at every zoom level.

### Fullscreen (Desktop + Mobile)

- Timeline constraints are removed.
- Full container is available to video fit and subsequent zoom/pan.
- Exiting fullscreen restores embedded constraints without jumps.

## Acceptance Criteria

1. Desktop camera switching across mixed aspect ratios does not produce persistent black video.
2. Mobile ultrawide camera in embedded mode can grow video area until timeline reaches 40% floor.
3. After timeline floor, further zoom only changes content zoom/pan.
4. Fullscreen is stable with correct zoom/pan and no stale bounds.
5. Snapshot behavior remains unchanged and functional.

## Test Strategy

### Unit Tests

- `historyViewportEngine` math:
  - fit scale
  - clamp bounds
  - phase transitions (embedded timeline-share vs content zoom)
- session reducer/state machine transitions, including stale event rejection by `sessionId`.

### Component Tests (Vitest)

- camera switch lifecycle (new session, old events ignored).
- metadata-gated first-fit initialization.
- fullscreen enter/exit with constraint recalculation.

### Manual Matrix

- Desktop: Chrome, Firefox, Safari.
- Mobile: iOS Safari, Android Chrome.
- Aspect ratios: 16:9 and 32:9.
- Scenarios: switch camera repeatedly, scrub, fullscreen toggle, zoom/pan.

## Rollout Plan

1. Milestone A: Implement playback session state machine + metadata-gated first-fit (no mobile split changes yet).
2. Milestone B: Implement mobile timeline share allocator and two-phase zoom model.
3. Milestone C: Final polish and regressions, keep snapshot integration untouched.

Each milestone should be merged only after targeted tests pass and manual matrix is checked.

## Open Questions

1. Should per-camera zoom/position be remembered during a single History session, or always reset on camera switch?  
Recommendation: reset on switch for reliability v1.
2. Should timeline 40% floor be configurable later?  
Recommendation: fixed constant in v1, evaluate telemetry/feedback before exposing a setting.
