# TouchDesigner integration

Cleanest path: a **Script CHOP** that imports `openepoc` directly and emits
14 channels of live EEG at 128 Hz. No OSC bridge, no separate process — TD's
own Python interpreter runs the reader.

The script is in [`openepoc_chop.py`](openepoc_chop.py). Background thread
drains samples from the dongle into a ring buffer; each TD cook flushes the
buffer into the CHOP's time-sliced samples.

## Setup

### 1. Match Python versions

TouchDesigner ships its own Python. Whatever package you use has to be ABI-
compatible with that interpreter (because `hidapi` is a C extension).

Check TD's version: open TD, `Help > About TouchDesigner` shows Python X.Y. As
of TD 2023.1+, that's **Python 3.11**.

### 2. Install openepoc into TD's Python

Two options. Option A (recommended) keeps your project isolated; option B is a
one-shot global install.

#### A. Project-local venv referenced from TD

Build a venv with TD's Python version, install openepoc into it, then point
TD at the venv's `site-packages`:

```bash
python3.11 -m venv /Users/<you>/code/openepoc-td/.venv
source /Users/<you>/code/openepoc-td/.venv/bin/activate
pip install git+https://github.com/fbarretto/openepoc.git
deactivate
```

In TD: `Edit > Preferences > Python > Python 64-bit Module Path`. Add:

```
/Users/<you>/code/openepoc-td/.venv/lib/python3.11/site-packages
```

Restart TD.

#### B. Pip-install into TD's bundled Python

```bash
/Applications/TouchDesigner.app/Contents/Frameworks/Python.framework/Versions/3.11/bin/python3 \
    -m pip install git+https://github.com/fbarretto/openepoc.git
```

Every TD project on this machine then has openepoc available. Cleaner if you
don't want to touch Preferences; messier if you want isolation between
projects.

### 3. macOS permissions

TouchDesigner needs the same permissions any other HID consumer needs on
Apple Silicon Macs:

- `System Settings > Privacy & Security > Allow accessories to connect`:
  ensure it's `Ask for new accessories` or `Always`. (See the main
  [`../../README.md`](../../README.md) for the full story — this is the
  hardest blocker on M-series Macs.)
- `Input Monitoring`: add `TouchDesigner.app` and toggle on.

Quit any other Emotiv apps that might hold the HID interface (EmotivPRO,
Xavier, Launcher).

### 4. Wire it up in TD

**The operator must be a Script CHOP, not a Script DAT.** TD has multiple
"Script" operators across families (CHOP, DAT, SOP, TOP) — they share a name
but have totally different APIs. We're generating channel data, so it's a
CHOP.

1. Right-click in the network → `Add Operator` → `CHOP` tab → find **Script**
   (the icon is `script1` with a CHOP outline). Or press Tab and type
   `Script CHOP`.
2. TD auto-creates a docked Text DAT named `script1_callbacks` next to it.
   Open that DAT.
3. Replace its contents with [`openepoc_chop.py`](openepoc_chop.py).
4. **Click the "Setup Parameters" button** on the Script CHOP. It's in the
   operator's parameter panel — `Script` tab. This triggers the
   `onSetupParameters` callback, which adds a custom `Tick` parameter
   bound to `absTime.seconds`. That expression re-evaluates every frame
   and forces the Script CHOP to cook every frame, even with no inputs.

   You only need to click this once per Script CHOP. After it's clicked,
   the `Tick` page appears on the operator and the CHOP cooks continuously.

   *Why this is needed*: a Script CHOP with no inputs cooks once at startup
   on TD's default scheduler. The `absTime.seconds` parameter dependency
   is the documented workaround. The script also implements
   `onGetCookLevel` returning `CookLevel.ALWAYS` for newer TD builds, but
   the `Tick` parameter is the bulletproof fallback.

5. Channels (`AF3`, `F7`, ..., `AF4`) should now update live at ~128 Hz.

   If you still see all-zero values after clicking Setup Parameters,
   either the `Tick` page didn't get created (paste was wrong, or
   `ParMode.EXPRESSION` failed) or the reader thread hasn't started.
   Drop a `Trail CHOP` after `script1` as a manual override — Trail
   CHOPs cook every frame and force their inputs to cook too.

**If you accidentally created a Script DAT**: errors will mention
`td.scriptDAT` and look like `'td.scriptDAT' object has no attribute 'rate'`.
Delete it, create a Script CHOP instead, paste the script there.

## Extending the script

The default CHOP outputs only the 14 EEG channels. To add gyro / counter /
battery / contact quality, edit the cook callback. The `Sample` TypedDict you
get from `openepoc` carries:

```python
sample["values"]   # list[float] of 14 channel values
sample["counter"]  # int 0..127 packet sequence
sample["gyro"]     # (gx, gy)
sample["battery"]  # int 0..100 or None
sample["quality"]  # dict[str, int] per-channel contact quality
```

To add `gyro_x`, `gyro_y`, `counter`:

```python
EXTRA = ("counter", "gyro_x", "gyro_y")
ALL_CHANS = CHANNELS + EXTRA

# in onCook, replace the channel-setup and fill block:
if scriptOp.numChans != len(ALL_CHANS):
    scriptOp.clear()
    for name in ALL_CHANS:
        scriptOp.appendChan(name)

scriptOp.numSamples = len(samples)
for j, name in enumerate(CHANNELS):
    chan = scriptOp[name]
    for i, s in enumerate(samples):
        chan[i] = s["values"][j]
counter_chan = scriptOp["counter"]
gx_chan = scriptOp["gyro_x"]
gy_chan = scriptOp["gyro_y"]
for i, s in enumerate(samples):
    counter_chan[i] = s["counter"]
    gx_chan[i] = s["gyro"][0]
    gy_chan[i] = s["gyro"][1]
```

Same pattern for battery (note: only present in some packets — guard with
`s["battery"] or 0`).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `'td.scriptDAT' object has no attribute 'rate'` | You created a Script **DAT**, not a Script **CHOP**. Different operator families. Delete and add `CHOP > Script` instead. |
| `numChans is unavailable for this CHOP while it is cooking` | Old version of the script that queried `scriptOp.numChans` mid-cook. Pull the latest `openepoc_chop.py` — current code does `clear()` + rebuild each cook. |
| All 14 channels show value `0` and never change | Script CHOP isn't cooking past its first cook. Pull the latest `openepoc_chop.py` — it adds an `onGetCookLevel` callback and a hidden `Tick` parameter bound to `absTime.seconds` that together force per-frame cooking. As a manual fallback, connect a `Trail CHOP` downstream. |
| `openepoc not importable inside TouchDesigner's Python` | Module Path not set in Preferences, or pointing at the wrong site-packages, or Python version mismatch (TD is 3.11, your venv is 3.12+) |
| All-zero values forever | Headset off, contact pads dry, or wrong AES schema. Run `openepoc wizard` from a terminal to confirm signal is arriving |
| Reader thread crashes (operator shows error) | Most often dongle was unplugged. Re-plug, then right-click the Script CHOP and `Reset` |
| 14 channels show but signal looks junky / clipping | EEG is microvolt-scale, raw values can be ±8000. Use a `Math CHOP` to scale, or `Limit CHOP` for hard bounds |
| TD frame rate drops when CHOP cooks | Reduce buffer drain frequency: insert a `CHOP Execute` that triggers cooks at 60 Hz, or increase `_buffer.maxlen` so you don't lose samples between cooks |

## Why not OSC?

OSC works (and we have [`02_stream_osc.py`](../02_stream_osc.py) for it) but
adds:
- A separate Python process to launch and babysit
- UDP serialization roundtrip per packet
- One more thing that can silently fail (port conflicts, firewalls,
  loopback weirdness)

The Script CHOP path keeps everything in one process. Use OSC only if TD is on
a different machine than the dongle, or if TD's Python won't load `hidapi` for
some platform reason.
