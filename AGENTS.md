# AGENTS.md

## Purpose

This repo is a local video voice replacement harness. Agents use it to convert a video voice while preserving timing, or to re-narrate a video by transcribing, regenerating, aligning, and muxing a new voice track.

## Start

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m pytest -q
```

Required system tools:

- `ffmpeg`
- `ffprobe`
- `rubberband`

## Provider Keys

Use `.env.example` as a reference. Keep real keys in the shell, `.env`, or a private agent vault. Never commit provider keys.

Required for normal voice work:

- `ELEVENLABS_API_KEY`

Required only for the optional WaveSpeed lip-sync example:

- `WAVESPEED_API_KEY`

## Agent Usage

Voice-swap the original words and cadence:

```bash
voiceswap input.mp4 --voice ELEVENLABS_VOICE_ID --out output.mp4
```

Re-narrate from transcript while preserving source timing:

```bash
revoice input.mp4 --voice ELEVENLABS_VOICE_ID --out output.mp4
```

If console scripts are not installed, use module entrypoints:

```bash
python -m voiceswap input.mp4 --voice ELEVENLABS_VOICE_ID --out output.mp4
python -m voiceswap.revoice_cli input.mp4 --voice ELEVENLABS_VOICE_ID --out output.mp4
```

## Safety

- Do not commit generated videos, extracted audio, `.env*`, or provider keys.
- Run `python -m pytest -q` before publishing changes.
- Treat examples as local demos; replace hard-coded input/output paths before running them in an agent workflow.
- The deterministic test suite uses fake converters and should not call ElevenLabs.
