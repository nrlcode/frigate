# Frigate Frontend Testing Workflow

This document describes the lightweight workflow used to test frontend changes in a fork before promoting app-only changes to `dev`.

## Goals

- Keep test-only CI and image publishing out of the main `dev` branch.
- Validate frontend changes in a real Frigate instance before merging.
- Avoid touching production recordings/history while testing.

## Branch Strategy

Use two branch lanes:

- `history-ui-test`
  - Lives in your fork.
  - Contains feature work plus test-only workflow changes.
  - Builds and publishes a test image to GHCR.
  - Safe place to iterate quickly.

- `dev`
  - Clean app-code branch.
  - Only receives application commits that are ready to keep.
  - Does not need the fork-only test workflow commits.

## Recommended Development Flow

1. Make frontend changes on `history-ui-test`.
2. Push `history-ui-test` to your fork.
3. Let GitHub Actions build and smoke-test the image.
4. Deploy the test image in your isolated Docker/Portainer Frigate instance.
5. Validate behavior manually in the UI.
6. Cherry-pick only the app commit(s) into `dev`.
7. Push `dev` or open a PR from `dev`.

## Why Not Merge the Whole Testing Branch?

`history-ui-test` contains workflow and image-publishing plumbing that is useful for your forked testing flow, but may not belong in upstream history.

The clean approach is:

- keep CI/image workflow commits on `history-ui-test`
- promote only app commits to `dev`

## Cherry-Picking App Changes into `dev`

When a feature is validated:

```bash
git checkout dev
git pull origin dev
git cherry-pick -x <app-commit-sha>
git push origin dev
```

Use `-x` so the resulting commit records where it came from.

If there are multiple app commits, cherry-pick them in order.

## Test Image Workflow

The test branch includes a workflow that:

1. Generates `frigate/version.py` and `web/.env`
2. Builds an amd64 Frigate image
3. Pushes it to GHCR
4. Runs a startup smoke test in GitHub Actions
5. Uses GitHub Actions cache and concurrency cancellation for faster repeat builds

Published image tags:

- `ghcr.io/nrlcode/frigate-test:latest`
- `ghcr.io/nrlcode/frigate-test:<commit-sha>`

When validating a specific build, prefer the SHA tag over `latest`.

## Isolated Runtime Testing

Use a separate test Frigate instance with:

- separate `/config`
- separate `/media/frigate`
- different host ports
- optional MQTT disabled or isolated

This avoids touching production history or recordings.

Example test directories:

- `/opt/frigate-test/config`
- `/opt/frigate-test/media`

## Portainer / Docker Notes

For Intel hardware acceleration testing:

```yaml
devices:
  - /dev/dri:/dev/dri
```

Optional:

```yaml
group_add:
  - "video"
  - "render"
```

If `detect.enabled: false`, GPU decode activity may remain low. To verify decode, temporarily enable detect on one camera and monitor with `intel_gpu_top`.

## Manual Validation Checklist

- History snapshot button appears and downloads successfully.
- Snapshot filename timestamp matches playback/timeline time.
- History video fits available desktop space cleanly.
- Zoom starts from the fitted baseline.
- Panning works while zoomed.
- Fullscreen keeps surrounding History UI on desktop.
- Camera switching does not visibly wobble or resize incorrectly.
- Mobile portrait playback is usable for ultrawide cameras.
- `Fit` / `Fill` and mobile theater mode behave as expected.
- No obvious console or runtime errors appear during playback actions.

## Current Pattern

In short:

- build and test on `history-ui-test`
- cherry-pick app commits into `dev`
- keep workflow-only commits out of `dev` unless intentionally desired
