# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pagecaster streams web pages to RTMP servers using Puppeteer (headless Chromium) and FFmpeg. The application captures browser content via X11 screen grabbing and supports three audio modes: browser audio capture via PulseAudio, external Icecast streams, or silent audio.

## Commands

### Development
```bash
npm install              # Install dependencies
npm run dev             # Run with nodemon (auto-restart on changes)
npm start               # Run application directly
```

### Docker
```bash
docker build -t pagecaster .                      # Build image
docker run --shm-size=256m \                      # Run container (requires env vars)
  -e WEB_URL="..." \
  -e RTMP_URL="..." \
  -e AUDIO_SOURCE="browser|icecast|silent" \
  pagecaster
```

## Architecture

### Core Components

**PageCaster Class** (`src/index.js`): Single-class architecture managing the entire streaming pipeline.

**Audio Pipeline**: Three distinct audio sources:
- `browser`: Captures webpage audio via PulseAudio's virtual-audio.monitor device
- `icecast`: Streams from external Icecast URL
- `silent`: Generates silent audio track (anullsrc)

Audio source auto-detection logic (src/index.js:12):
- If `AUDIO_SOURCE` is explicitly set, use that value
- Else if `ICE_URL` is provided, default to `icecast`
- Otherwise, default to `silent`

**Video Capture**: FFmpeg X11 screen grabbing from virtual display :99.0 (no screenshot loop needed)

### Key Integration Points

**Puppeteer → X11 → FFmpeg Flow**:
1. `entrypoint.sh` starts Xvfb virtual display (:99) with specified dimensions
2. Puppeteer launches Chromium in kiosk mode on :99.0 display
3. FFmpeg captures X11 display directly using `-f x11grab -i :99.0`

**Browser Audio Capture** (src/index.js:103-163):
- PulseAudio virtual sink (`virtual-audio`) configured in entrypoint.sh
- Chromium routes audio to virtual sink via `--enable-features=PulseAudio`
- FFmpeg captures from `virtual-audio.monitor` using `-f pulse`
- Page evaluation checks for AudioContext and audio/video elements before enabling capture

**FFmpeg Arguments** (src/index.js:219-261):
- Video input: X11 grab at specified framerate
- Audio input: Dynamic based on audioConfig (PulseAudio/URL/silent)
- Output: libx264 with configurable preset, AAC audio, FLV format to RTMP

### Container Environment

**entrypoint.sh** orchestrates initialization:
1. Validates required environment variables
2. Starts Xvfb virtual display
3. Configures PulseAudio daemon with virtual sink and Unix socket
4. Launches Node.js application

**Docker shm requirements**: `--shm-size=256m` required for Chromium shared memory

## Environment Variables

**Required**: `WEB_URL`, `RTMP_URL`

**Optional**:
- `AUDIO_SOURCE`: `browser`, `icecast`, or `silent` (auto-detected)
- `ICE_URL`: Icecast stream URL (required if AUDIO_SOURCE=icecast)
- `SCREEN_WIDTH`: Browser width (default: 854)
- `SCREEN_HEIGHT`: Browser height (default: 480)
- `FFMPEG_PRESET`: Encoding preset (default: veryfast)
- `FRAMERATE`: Video framerate (default: 30)

## Implementation Notes

- Process cleanup handled via SIGINT/SIGTERM handlers (src/index.js:303-317)
- Browser launched with `--autoplay-policy=no-user-gesture-required` to enable immediate audio playback
- FFmpeg uses constant frame rate (`-vsync cfr`) for RTMP stability
- PulseAudio runs in daemon mode with infinite exit timeout
