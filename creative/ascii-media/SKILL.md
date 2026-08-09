---
name: ascii-media
description: "Use for ASCII visual media: terminal/banner art, boxes/cowsay, image-to-ASCII stills, and colored ASCII video/GIF production pipelines."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [ascii, art, video, gif, terminal, pyfiglet, ffmpeg]
    related_skills: []
---

# ASCII Media

## Overview

Use this umbrella when the user wants text-mode visuals: quick ASCII banners, terminal decorations, image-to-ASCII conversions, or full colored ASCII video/GIF outputs.

## Routing Table

| Output | Workflow |
|---|---|
| Banner text | `pyfiglet`, `toilet`, fonts, width checks |
| Decorative boxes | `boxes`, markdown/code fences, monospace-safe output |
| Fun terminal art | `cowsay`, `lolcat`, ANSI color if supported |
| Still image to ASCII | sample luminance/color, choose width, preserve aspect ratio |
| Video/GIF to ASCII | frame extraction + ASCII shader/effects + audio/video recomposition |

## Still ASCII Art

- Ask for target width or infer from delivery medium.
- Preserve aspect ratio; terminal glyphs are taller than wide.
- For chat, prefer plain monospace blocks over fragile ANSI unless the platform supports it.
- For image conversion, verify the file exists and inspect output legibility before returning.

## ASCII Video Pipeline

High-level pipeline:

1. Decode input with `ffmpeg`.
2. Convert frames to glyph/color representation.
3. Apply scene/effect/composition choices.
4. Re-encode to MP4/GIF and preserve or remix audio when requested.
5. Verify duration, resolution, and output file size.

Reference material migrated from `ascii-video` lives under `references/ascii-video/`.

## Common Pitfalls

- Chat clients collapse whitespace unless fenced.
- ANSI color often fails in mobile messengers.
- GIFs balloon quickly; cap FPS/resolution for delivery.
- Dense glyph sets can reduce recognizability for low-contrast images.

## Verification Checklist

- [ ] Output medium and width chosen
- [ ] Monospace/whitespace preserved
- [ ] For files, output exists and is readable/playable
- [ ] For video/GIF, duration and size checked
