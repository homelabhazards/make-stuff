# obs-decklink-headless

Bind Blackmagic **DeckLink Output Filters** to OBS Studio scenes **by scene name**,
for fully headless Linux deployments.

This is a small companion tool for [cg2121's Decklink Output
Filter](https://github.com/cg2121/obs-decklink-output-filter). 
It makes headless deployments slightly less painful. 
The filter stores its target device as an opaque `device_hash` string that OBS generates
from the Blackmagic SDK enumeration. That hash **only exists on the machine with
the DeckLink card installed** — so if you author your scenes on a workstation
(where the card isn't), the filter's device dropdown is empty and you can't bind
outputs.

`obs-decklink-headless` probes the running OBS instance for the real device hashes, 
writes a small mapping file, and then rewrites your scene collection to attach a
correctly-bound filter to each scene whose name matches the map 
(e.g. `io0`, `io1`, `io2`, `io3`).

## Why this exists

The intended deployment is a headless OBS "playout" box:

- OBS Studio runs with **no desktop**, under a headless Wayland compositor
  (`cage`) as a systemd service, getting a hardware GL context on a GPU.
- One scene per physical SDI output. Each scene carries a Decklink Output Filter
  locked to a specific DeckLink channel, so the scene is mirrored continuously to
  that SDI connector.
- Scenes are authored on a normal workstation OBS (with a GUI), then the scene
  collection JSON is copied to the headless box, where this tool binds the
  outputs.
- Live switching of what's on each output is done over
  [obs-websocket](https://github.com/obsproject/obs-websocket).

See [`docs/headless-deployment.md`](docs/headless-deployment.md) for a full,
reproducible walkthrough of that server (Ubuntu, `cage`, systemd, Intel/AMD GPU,
DeckLink driver, plugin build).

## Requirements

- OBS Studio 28+ with obs-websocket enabled (built in since 28).
- The [Decklink Output Filter](https://github.com/cg2121/obs-decklink-output-filter)
  plugin installed and loading.
- Blackmagic Desktop Video driver installed and the card's firmware current.
- Python 3.9+ and `obsws-python`.

## Install

```bash
pip install obs-decklink-headless
# or from a checkout:
pip install .
```

## Usage

The workflow has three steps: **probe**, edit, **inject**.

### 1. Probe the running OBS for real device hashes

Run this **on the headless box** (where the DeckLink card lives), against the
running OBS:

```bash
obs-decklink-headless probe \
  --password "$OBS_WS_PASSWORD" \
  -o decklink-map.json
```

This creates a temporary DeckLink input, reads the device + mode enumeration,
removes the temp input, and writes `decklink-map.json`:

```jsonc
{
  "_devices_seen": [
    { "name": "DeckLink Duo (1)", "device_hash": "3970865216_DeckLink Duo 2" },
    { "name": "DeckLink Duo (2)", "device_hash": "3970865217_DeckLink Duo 2" }
    // ...
  ],
  "_modes_available": [
    { "mode_id": 10, "name": "1080p60" }
    // ...
  ],
  "defaults": {
    "mode_id": 10,           // 1080p60
    "video_connection": 1,   // SDI
    "audio_connection": 1,   // Embedded
    "pixel_format": 846624121, // 8-bit YUV
    "auto_start": true
  },
  "scenes": {
    "io0": { "device_hash": "3970865216_DeckLink Duo 2" },
    "io1": { "device_hash": "3970865217_DeckLink Duo 2" }
    // ...
  }
}
```

The `scenes` block is seeded with `io0`, `io1`, ... in enumeration order.

### 2. Edit the mapping

Rename the keys under `scenes` to match the scene names in your OBS collection,
and confirm each maps to the physical SDI port you intend. **Enumeration order
usually matches the connector labels on the card, but verify with a real feed.**
You can override any default per scene, e.g. a 720p output:

```jsonc
"scenes": {
  "io0": { "device_hash": "3970865216_DeckLink Duo 2" },
  "io3": { "device_hash": "3970865219_DeckLink Duo 2", "mode_id": 16 }
}
```

### 3. Inject filters into the scene collection

Point it at the scene-collection JSON (under
`~/.config/obs-studio/basic/scenes/YourCollection.json`):

```bash
obs-decklink-headless inject \
  ~/.config/obs-studio/basic/scenes/YourCollection.json \
  -m decklink-map.json
```

It writes a `.bak`, then adds a bound `decklink_output_filter` to each matching
scene. Re-running is idempotent — it replaces any prior filter it added rather
than stacking duplicates. Restart OBS (or reload the collection) to apply.

### Verify (optional)

List the DeckLink filters currently live over websocket:

```bash
obs-decklink-headless verify --password "$OBS_WS_PASSWORD"
```

## Notes & known limitations

- **Audio is the master mix only.** The underlying filter outputs the OBS master
  audio to every DeckLink output, not per-scene audio. This is a limitation of
  the filter, not this tool.
- **Hashes are per-card, per-driver.** `device_hash` values come from the
  Blackmagic SDK enumeration on that machine. If you move the card, change slots,
  or the driver renumbers, re-run `probe`.
- **OBS overwrites the collection on save.** Prefer to treat the authored JSON as
  the source of truth and inject on deploy. If OBS re-saves the collection it
  keeps the filters (they are normal filter entries), but authoring elsewhere and
  injecting keeps the pipeline clean.

## License

GPL-3.0-or-later — see [LICENSE](LICENSE).

Not affiliated with Blackmagic Design or the OBS Project. "DeckLink" and
"Blackmagic" are trademarks of Blackmagic Design.