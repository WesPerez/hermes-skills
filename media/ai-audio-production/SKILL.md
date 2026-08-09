---
name: ai-audio-production
description: "Use for AI/audio production workflows: songwriting, Suno/HeartMuLa-style generation, AudioCraft MusicGen/AudioGen, and audio feature/spectrogram analysis."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [audio, music, songwriting, ai-music, audiocraft, heartmula, spectrogram]
    related_skills: []
---

# AI Audio Production

## Overview

Use this umbrella for music/audio tasks from creative songwriting through generation and analysis. Keep craft decisions (lyrics, structure, tags) separate from model/runtime decisions (AudioCraft, HeartMuLa, CLI setup) and verification (listening, metadata, spectrograms).

## Routing Table

| Need | Section |
|---|---|
| Write lyrics, genre tags, Suno-style prompts | Songwriting and prompt craft |
| Generate songs with HeartMuLa | HeartMuLa generation |
| Generate music/sounds with MusicGen/AudioGen | AudioCraft |
| Inspect audio features or spectrograms | songsee analysis |

## Songwriting and Prompt Craft

- Clarify genre, mood, point of view, language, structure, and explicit constraints.
- Produce lyrics with sections (`[Verse]`, `[Chorus]`, `[Bridge]`) when the generator benefits from structure.
- Tags should be concise and musical: genre, instrumentation, tempo/energy, vocal style, production era.
- Avoid copyrighted imitation of living artists; describe attributes instead.

## HeartMuLa Generation

- HeartMuLa is useful for Suno-like local/open generation from lyrics + tags.
- Treat `assets/lyrics.txt`, `assets/tags.txt`, and `assets/output.mp3` in old examples as placeholders; create real task files in the working directory.
- Verify output file existence, duration, and playability before returning.

## AudioCraft

- Use MusicGen for music continuation/text-to-music and AudioGen for sound effects/soundscapes.
- Start small for smoke tests, then scale duration/model size.
- Reference material migrated from `audiocraft-audio-generation` lives under `references/audiocraft-audio-generation/`.

## songsee Analysis

- Use for spectrograms/features such as mel, chroma, MFCC, waveform, and quick audio diagnostics.
- Report concrete file paths for generated visualizations and any notable audio properties.

## Verification Checklist

- [ ] Creative brief or audio analysis question is explicit
- [ ] Runtime/model chosen and dependencies checked
- [ ] Output file exists and metadata/duration inspected
- [ ] Final answer includes path(s), parameters, and caveats
