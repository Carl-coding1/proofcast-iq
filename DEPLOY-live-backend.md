# Live backend — deploy & test runbook

Two components. The **API** (control room, events, certs, overlay channel)
goes on the VPS. The **relay** runs on the studio machine near OBS/vMix.

## A. VPS — deploy the API (fully testable from your desk)

From your PC:
    scp proofcast-backend.zip root@76.13.63.65:/root/

On the VPS:
    cd /root && unzip -o proofcast-backend.zip && cd proofcast-backend
    npm install
    pm2 restart pciq-api && pm2 logs pciq-api --lines 8 --nostream

Check the live routes are up:
    curl -s https://api.proofcastiq.tech/api/live/shows
    curl -s https://api.proofcastiq.tech/api/live/certs
Both should return `[]` (empty arrays) — that means the tables exist and the
routes answer.

### Test the desk→server→certificate loop (no studio needed)
1. Open the app, go to a show desk, add cues, go on air, fire a proof cue.
2. On the VPS: `curl -s https://api.proofcastiq.tech/api/live/certs`
   You should see the certificate the fire minted, `confirm:"desk"`.
3. `curl -s https://api.proofcastiq.tech/api/live/shows/SHOW-ID/events`
   shows the append-only ledger with the fire event and its command.

That loop proves the control room is now server-authoritative.

## B. Studio — the relay (drives OBS, upgrades certs to stream-confirmed)

On the studio machine (same LAN as OBS), with Node 18+:
1. Copy the `proofcast-backend` folder there (or just `src/relay.js` + a config).
2. `npm install` (installs `ws` for the OBS driver; vMix needs no extra deps).
3. Make `src/relay.config.json` from the example, fill in:
   - `apiBase`  = https://api.proofcastiq.tech
   - `apiKey`   = your API key
   - `encoder`  = obs   (switch to vmix later)
   - `host/port`= OBS host + obs-websocket port (Tools → WebSocket Server Settings)
   - `password` = obs-websocket password if you set one
4. In OBS: add each overlay as a **Browser source**, URL from the app's
   Encoder & Graphics → Copy URL (it includes `?overlay=KIND&ch=...&api=...`).
5. Run it:  `node src/relay.js`
   Banner shows API, encoder, and show. Leave it running.

### Test the full stream-confirmed loop
1. Relay running, OBS open with the overlay browser-sources added.
2. On the desk (on air), fire a break cue bound to the OBS source name.
3. Watch: relay logs `fire showSource … -> done`, the overlay appears on the
   OBS canvas, then relay logs `cert … -> stream-confirmed`.
4. In the app's Live Certificates view, that cert flips from amber
   **Desk-confirmed** to green **Stream-confirmed**.

That is the whole system proven end to end.

## Notes
- If `ws` isn't installed, the OBS driver runs in **dry-run**: it logs the
  command it would send and does NOT confirm the cert. Good for wiring checks.
- vMix: set `encoder:"vmix"`, `port:8088`. No `ws` needed — it's HTTP.
- The relay only acts on shows that are **On air**; rehearsal fires are ignored
  by design, so you can drill safely.
- Back up first: `tar -czf /root/pciq-data-$(date +%F).tar.gz data/`
