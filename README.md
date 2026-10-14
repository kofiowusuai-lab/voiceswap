# voiceswap

Replace the voice in a video with an ElevenLabs voice, with timing aligned to the original audio.

Two modes, depending on what you want:

| Mode | Command | What it does | When |
|------|---------|--------------|------|
| **Voice swap** | `voiceswap` | ElevenLabs **speech-to-speech**: transforms only the *timbre* of the original audio, keeping its exact delivery. Segments are fitted to source durations. | Same words, same language, just change who it sounds like. |
| **Re-narrate** | `revoice` | **Whisper** transcribes the words → **ElevenLabs TTS** regenerates them → each word is warped onto its original timestamp. | Clean studio voice; the source audio is reused only as the script. |

The paced-clip demo in `examples/demo_revoice_paced.py` compares aligned output against naive TTS. Its measurements depend on the source clip, provider output, and transcription model; they are not general timing guarantees.

## Re-narrate mode (`revoice`)

```
video ─► extract audio ─► Whisper word timestamps (the script + the timing reference)
      ─► group into phrases on natural pauses
      ─► per phrase: ElevenLabs TTS with per-character timestamps
      ─► subsequence-align generated words to original words, warp each into its slot
      ─► reassemble at original offsets ─► mux onto the original video
```

```bash
python -m voiceswap.revoice_cli input.mp4 --voice <VOICE_ID>   # or: revoice input.mp4 --voice ...
# -> input.revoiced.mp4
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--voice` | (required) | ElevenLabs target voice id |
| `--out` | `<name>.revoiced.<ext>` | output path |
| `--model` | `eleven_multilingual_v2` | ElevenLabs TTS model |
| `--whisper-model` | `base` | Whisper size; `small`/`medium` tighten the timing reference |
| `--language` | auto | force transcription language |
| `--max-gap` | `0.6` | pause length (s) that starts a new phrase |
| `--api-key` | `$ELEVENLABS_API_KEY` | override the key |

Live proof, end to end (source → re-voice → measure drift vs naive):

```bash
python examples/demo_revoice_paced.py
```

## Voice-swap mode (`voiceswap`)

Uses ElevenLabs **speech-to-speech** (voice changer), which transforms only the *timbre* of the original audio while keeping its cadence, rhythm, pauses and emotion. Timing is preserved by the model rather than reconstructed, then the pipeline fits segment durations to the source. Mouth movements may still differ; this mode does not perform lip-sync.

## How it works (voice-swap)

```
video ─► extract audio ─► detect silences ─► speech segments (with exact start/end)
      ─► per segment: ElevenLabs speech-to-speech ─► rubberband-fit to the exact slot
      ─► reassemble on a silent canvas at original offsets (gaps preserved)
      ─► mux the new track onto the original video stream (video copied, not re-encoded)
```

The frame-lock comes from two guarantees:

1. Each chunk is **snapped to its original duration** with `rubberband` (pitch-preserving). Speech-to-speech keeps timing very close already, and the correction constrains segment boundaries. Audible quality and within-segment word timing still require review.
2. Each chunk is **placed at its original start offset** on a canvas whose length equals the source duration. Silences between chunks are preserved exactly.

## Requirements

System tools on `PATH`:

- `ffmpeg`, `ffprobe`
- `rubberband`

Python:

```bash
pip install -r requirements.txt   # elevenlabs, numpy, scipy
```

Environment:

```bash
cp .env.example .env              # optional local reference only
export ELEVENLABS_API_KEY=...     # or pass --api-key
```

## Usage

```bash
python -m voiceswap input.mp4 --voice <ELEVENLABS_VOICE_ID>
# -> input.voiceswap.mp4
```

Options:

| Flag | Default | Meaning |
|------|---------|---------|
| `--voice` | (required) | ElevenLabs target voice id |
| `--out` | `<name>.voiceswap.<ext>` | output path |
| `--model` | `eleven_multilingual_sts_v2` | speech-to-speech model |
| `--noise-db` | `-30.0` | silence threshold (lower = stricter) |
| `--min-silence` | `0.5` | minimum silence (s) that splits a chunk |
| `--remove-background-noise` | off | ask ElevenLabs to denoise each chunk |
| `--api-key` | `$ELEVENLABS_API_KEY` | override the key |
| `--sample-rate` | `44100` | working sample rate |

Find a voice id with the ElevenLabs SDK:

```python
from elevenlabs import ElevenLabs
for v in ElevenLabs().voices.get_all().voices:
    print(v.voice_id, v.name)
```

## Programmatic use

```python
from voiceswap.pipeline import process_video
from voiceswap.sts import make_converter

conv = make_converter(voice_id="VOICE_ID")
process_video("input.mp4", "out.mp4", conv, on_progress=lambda i, n: print(i, n))
```

Any object with `convert_file(in_wav, out_path)` works as a converter, so the pipeline is testable without network calls.

## Tests

```bash
python -m pytest -q
```

The suite drives the **entire pipeline end-to-end** (real ffmpeg extract / silence-detect / `rubberband` stretch / mux) using an identity converter in place of ElevenLabs, so it is deterministic, offline and free. The only thing not exercised is the live ElevenLabs call itself, which is isolated behind `voiceswap/sts.py` and unit-tested with a fake SDK client.

## Notes / limits

- **Voice-only audio.** This path replaces the whole audio track. If your video has music or SFX under the voice, add a `demucs --two-stems=vocals` step before conversion and mix the `no_vocals` stem back in. (Left out here by design, the chosen scope was voice-only.)
- Speech-to-speech keeps the **original words and language**. To change the script or translate, that is a different pipeline (transcribe → align → TTS → fit) or the ElevenLabs Dubbing API.
- Long videos are chunked on silence, so very long monologues with no pauses become one large chunk; tune `--min-silence` / `--noise-db` if you want finer splits.

## Agent Quick Start

When another user gives this repo link to an agent, the expected flow is:

```bash
git clone https://github.com/kofiowusuai-lab/voiceswap.git
cd voiceswap
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -q
export ELEVENLABS_API_KEY=replace-with-user-key
python -m voiceswap input.mp4 --voice ELEVENLABS_VOICE_ID --out output.mp4
```

Agent operating rules are in `AGENTS.md`. The short version: use the caller's own provider keys, keep generated media out of git, and run the offline test suite before changing the pipeline.

## Complete local installation

Python **3.9+** is declared by the package. FFmpeg, ffprobe, and Rubber Band must be available on PATH, including for media tests. To install the console scripts and test dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[dev]'
# Optional revoice dependency; downloads a model on first transcription
python -m pip install openai-whisper
# Optional WaveSpeed example dependency, not declared by the base package
python -m pip install requests
python -m voiceswap --help
python -m voiceswap.revoice_cli --help
python -m pytest -q
```

`pip install -r requirements.txt` includes Whisper but does not itself install the `voiceswap`/`revoice` console entry points or pytest. Module entry points work from the repository root. A copied `.env` is not loaded automatically: export `ELEVENLABS_API_KEY` into the current process, or explicitly load your private environment file. `WAVESPEED_API_KEY` is only needed for optional lip-sync examples. Use authorized provider voice IDs and your own account.

## Additional revoice options and outputs

`revoice` also supports `--align word` (default) or `--align phrase`, `--output-format`, and `--sample-rate` (default 44100). Word alignment targets tighter cadence; phrase alignment favors smoother delivery within phrase boundaries. Both depend on transcription accuracy and provider output.

```bash
revoice input.mp4 --voice ELEVENLABS_VOICE_ID --align phrase --out output.mp4
```

The normal pipeline uses temporary extracted WAVs, converted chunks, and assembled audio and cleans them up after processing. Only the output video persists unless an example writes additional artifacts. Defaults are `<input-stem>.voiceswap.<extension>` and `<input-stem>.revoiced.<extension>`. The original video stream is copied during muxing; replacement audio is encoded for the output container.

## Source map and troubleshooting

- `audio.py`, `segment.py`, `stretch.py`, `assemble.py`: extraction/mux, silence detection, duration fitting, and offset assembly.
- `pipeline.py`, `sts.py`, `cli.py`: speech-to-speech orchestration, provider adapter, and CLI.
- `transcribe.py`, `tts.py`, `align.py`, `revoice.py`, `revoice_cli.py`: word-timed transcription, timestamped TTS, text alignment, and re-narration.
- `wavespeed.py`: optional upload/poll/download lip-sync client, separate from both default CLI pipelines.
- `tests/`: fake-provider unit tests and local media integration fixtures.

If console commands are missing, install the editable package or use the documented modules. If extraction or stretching fails, check all three system executables. If revoice cannot import Whisper, install `openai-whisper` in the active environment; initial model loading may require a download. For shifted or unnatural words, correct source transcription and compare word/phrase alignment; silence thresholds control chunk boundaries but cannot fix an incorrect transcript. Provider authentication/quota failures require checking the user's key and account. The examples contain local demo paths and make live billable calls; adapt and review them before use.

There is no bundled source-separation step, translation workflow, web service, or automatic publishing. A successful test suite checks local/fake-provider behavior, not live service availability or perceptual lip-sync quality. See [AGENTS.md](AGENTS.md) for repository operating rules.

## Licensing

This repository does not include a top-level license file. Confirm reuse terms with the owner; dependencies and any bundled media retain their own licenses.
