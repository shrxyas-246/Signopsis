# Signopsis

A sign language interpreter that runs entirely on your own machine. Speak, and it signs
what you said with a 3D avatar. Sign at the camera, and it turns that into text and speech.
No cloud calls, no accounts — audio and video never leave the laptop.

It's built around one idea: when the system isn't sure what it heard or saw, it should say
so instead of guessing silently. Every caption carries a confidence badge, and anything
genuinely ambiguous (like the Hindi word "kal", which means both yesterday and tomorrow)
gets asked about rather than assumed.

This isn't a certified interpreter and shouldn't be treated as one for anything medical,
legal, or otherwise high-stakes. It's a working prototype.

## What's actually in here

- **Speech → text → sign, live.** Talk into the mic, a local Parakeet model transcribes
  you, and a 3D avatar signs each sentence as it finishes. This is the main demo — open
  `/avatar` and hit "Start speaking".
- **Sign → text.** Point a webcam at it (MediaPipe does the hand/pose tracking right in
  the browser, only landmarks get sent to the server) and it captions what you sign.
  There's also a simulator mode if you don't have a camera handy.
- **A web app** with Converse (both directions at once), a Compose screen for typing text
  and previewing the sign output, a place to teach it your own signs, and a diagnostics
  page showing the actual trust/calibration numbers.

## The honest limitations

The sign vocabulary is about 70 hand-built motions on a placeholder avatar — not real ISL,
just something visually distinct enough to test the pipeline with. Anything outside that
vocabulary gets a *generated* sign (deterministic per word, so the same word always gets
the same motion) rather than just spelling everything out letter by letter. It's still not
real sign language. If you're building on this for actual Deaf users, that lexicon needs to
be replaced with recorded ISL and reviewed by Deaf signers before it's usable for anything
beyond a demo.

The camera-based sign recognizer matches against those same placeholder motions, so it
recognizes the simulated signer reliably but real human signing needs to be taught first
(four takes per sign, no retraining needed) or it'll come back mostly "unknown."

Parakeet only does English. Say something in Hindi and the web app falls back to your
browser's speech recognition, which — worth knowing — sends that audio to Google's servers.
The local pipeline stays fully offline; the fallback doesn't.

## Running it

Fastest way, if you're on Windows with WSL2 set up:

```powershell
powershell -ExecutionPolicy Bypass -File backend\scripts\start_demo.ps1
```

That builds the web app if it hasn't been built yet, boots the backend inside WSL, waits
for the models to warm up, and opens the avatar page.

If you want to do it by hand — backend runs in WSL2 (needs the GPU stack), the web app
build works from Windows or WSL:

```bash
cd /mnt/c/Users/shrey/Downloads/setu
cp .env.example .env
make setup            # pulls in the GPU speech stack, ~4-5 GB
make models           # downloads Parakeet + Sortformer, asks first (~2.9 GB)
make vendor           # MediaPipe's web assets, for offline camera use (~40 MB)
make check            # sanity check that torch/onnxruntime actually see the GPU
bash backend/scripts/run_backend.sh
```

```powershell
powershell -ExecutionPolicy Bypass -File backend\scripts\build_web.ps1
```

No GPU, or just want to poke at it without downloading anything? `make backend-fake` runs
the whole thing with a scripted fake speech recognizer. `SETU_FALLBACK=cpu` switches to
faster-whisper if you have a CPU but no matching GPU.

Once it's up:

| | |
|---|---|
| `/avatar` | speak or type, watch the 3D avatar sign it |
| `/app/#/live` | the same thing, but through the full web app |
| `/sign-to-text` | camera → captions |
| `/text-to-sign` | type something, see the ISL gloss and a round-trip check |
| `/` | links to everything, plus the VRAM/mode planner |

## Testing

```bash
python -m pytest -q -m "not gpu"           # runs in ~35s, no models needed
python -m pytest -m gpu                     # needs real models + data/clips/*.wav
python backend/scripts/e2e_check.py         # hits a running backend end to end
```

The e2e script is the one I actually trust — it streams real audio through the whole
speech→sign pipeline against a live server and checks the timing budgets hold up, not
just that the code runs.

For the web app: `cd app/web && npm test`.

## Config

Everything lives in `.env` (copy `.env.example` to start). The two settings that matter
most for how the demo behaves:

- `SETU_TRUST_POLICY` — `always` (default) means low-confidence results still get shown
  and signed, just with a lower badge. Set it to `gated` to get the stricter emit/repair/hold
  behavior instead, where uncertain stuff gets held back.
- `SETU_SIGN_UNKNOWN` — `generate` (default) signs every word it can, using generated
  placeholder motions for anything not in the curated lexicon. `fingerspell` spells
  unrecognized words out instead.

Full list of endpoints, contracts, and the reasoning behind the architecture is in
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). The original design spec is in `CLAUDE.md`
one level up.

## Where things stand, latency-wise

Measured on an RTX 5070 laptop (your numbers will vary): first partial caption shows up in
roughly 200-350ms, the sign starts playing 300-700ms after you stop talking, and typed
text→sign resolves in under 100ms. Simulated sign→text takes about half a second round trip.

## Privacy, briefly

Nothing gets written to disk — audio and camera frames are decoded in memory and thrown
away right after. There's a test for this (`backend/tests/test_privacy.py`) so it's not
just a claim. Trace logs only keep timings and IDs unless you explicitly turn on
`SETU_STORE_TRANSCRIPTS`. Any signs you teach it are stored locally under `data/users/`
and you can delete them from the Library page whenever.
