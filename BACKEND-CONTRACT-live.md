# ProofCast Live — backend contract to build in one pass

This is the running list of server endpoints and behaviours the front-end
now expects, captured as we build so the backend can be written once for the
whole surface (the agreed strategy). Nothing here is live yet; the front-end
runs browser-only and degrades honestly until these exist.

## 1. Control room — make the desk server-authoritative
The front-end already calls these best-effort (mirrors the LPM discipline):

- `POST /api/live/shows` — upsert a show + its run-of-show.
  Body: `{id, channel, name, program, presenter, status, planned_start, cues:[{id,type,label,asset,campaign,pos,status}]}`
  Semantics: replace-the-cue-set per show (like the LPM windows endpoint);
  protect fired/aired/skipped cues as history — never overwrite from a later push.

- `POST /api/live/events` — append one immutable event.
  Body: `{id, show, ts(ISO), kind(show|fire|clear|skip), text, operator, live(bool), cue, confirm(null|"stream"), cert, command, encoder}`
  Semantics: append-only ledger. This is the certificate's source of truth.
  The `operator` field is the attribution — keep it server-verified once auth exists.

- `GET /api/live/shows`, `GET /api/live/shows/:id/events` — read back for the certificate.

## 2. Encoder relay — the honesty seam
A browser on GitHub Pages cannot reach a studio PC's encoder socket (and
shouldn't). The relay is a small authenticated server component the studio
runs (or the VPS reaches over a tunnel) that replays neutral commands.

- `POST /api/live/encoder/test` — ping the configured encoder, return reachability.
  Body: `{driver(obs|vmix), host, port, password}` → `{ok, msg, latencyMs?}`

- `POST /api/live/encoder/command` — replay ONE neutral command to the encoder.
  Body (the contract emitted by cueToCommand/cueClearCommand):
    `{cueId, action(showSource|hideSource|cutScene), target, scene?, channel?}`
  The relay translates via the driver (OBS obs-websocket v5 / vMix HTTP API)
  and — critically — reports back success so the event/cert can be upgraded
  from `confirm:"desk"` to `confirm:"stream"`. THIS is what lets a certificate
  honestly claim stream-confirmation.

- Driver wire formats (already implemented client-side in ENCODER_DRIVERS):
  - OBS: `SetSceneItemEnabled {sceneName,sceneItemId,sceneItemEnabled}`,
          `SetCurrentProgramScene {sceneName}`
  - vMix: `GET /api/?Function=OverlayInput{ch}In&Input=...`,
           `...OverlayInput{ch}Out...`, `...CutToInput&Input=...`

## 3. Overlay live channel — drive the browser-source graphic
The overlay page (`index.html?overlay=break`) currently reads state from the
URL hash. To update it LIVE without a reload, it needs a push channel:

- `GET /api/live/overlay/:channelId` — Server-Sent Events (or WebSocket).
  Emits `{show(bool), advertiser}` whenever a break cue fires/clears.
  The overlay page already has the wiring point marked:
    `const es=new EventSource('/api/live/overlay/'+channelId); es.onmessage=...`

## 4. Campaign upsert (already noted from the LPM phase)
- `POST /api/campaigns` — so server-side certificates know inline-created
  campaigns. Front-end already calls this on lpmPush.

## Confirm-state model (the product's spine)
Every fired cue carries `confirm`:
- `"desk"`  — ProofCast fired it from the control room (intent-to-air). Phase 1.
- `"stream"`— the relay confirmed the encoder acted on it. Phase 3.
Certificates must render which one they hold and NEVER claim "stream" without
a relay confirmation. This is the ProofCast integrity principle applied to live.
