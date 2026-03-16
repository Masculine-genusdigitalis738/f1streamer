<div align="center">

# f1streamer

**Fully automated F1 live stream bot for Discord.**
Watches your calendar. Finds the stream. Pushes it into a voice channel. Repeats all season.

[![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?style=flat-square&logo=node.js&logoColor=white)](https://nodejs.org)
[![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![FFmpeg](https://img.shields.io/badge/FFmpeg-required-007808?style=flat-square&logo=ffmpeg&logoColor=white)](https://ffmpeg.org)
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

</div>

---

Configure it once. Every practice, qualifying, sprint and race for the rest of the season streams automatically into Discord with DAVE E2EE encryption, hardware-accelerated transcoding, and zero manual intervention.

---

## Table of Contents

- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running](#running)
- [Under the hood](#under-the-hood)

---

## How it works

f1streamer has two jobs: **finding** the stream and **delivering** it.

**Finding:** `f1scheduler.js` parses your F1 calendar and sleeps until `preSessionMinutes` before each session. At wake time, it runs a scored provider cascade across two sources. Every available stream gets scored against the GP name, session type keyword, and a 3-hour time window. The highest-scoring match goes through three extraction methods in sequence until one produces a playable URL:

| Step | ppv.to | streamed.st |
|------|--------|-------------|
| 1 | `iframe` field from API response | `embedUrl` from stream API |
| 2 | Construct URL from `uri_name` | Scrape the watch page iframe |
| 3 | Fetch watch page and scrape | Construct URL from match ID slug |

If both providers come up empty, the cascade retries every 2 minutes up to 10 times. On success, the streamer is spawned and monitored until the session window closes, restarting automatically on crash.

**Delivering:** `streamer.js` accepts an embed page URL or a direct `.m3u8` link. For embed URLs, a headless Puppeteer browser intercepts the HLS master playlist via Chrome DevTools Protocol. For direct M3U8s, the browser is skipped. Either way, segments are fetched concurrently through a local TLS proxy, analyzed by a dual-pass ffprobe pipeline, then fed into FFmpeg and pushed into Discord.

---

## Prerequisites

- **Node.js** 18+
- **Python** 3.9+
- **FFmpeg** with libopus support
- A **Discord account** — selfbot mode, personal user token (not a bot application)
- An **F1 ICS calendar file** — see [Getting a calendar](#getting-a-calendar)

### Hardware acceleration

Auto-detected at runtime. The bot runs a quick smoke test before committing to any encoder, and falls back gracefully if the test fails:

```
VAAPI  (Linux Intel/AMD)
  ↓ if unavailable
NVENC  (Linux/Windows NVIDIA — requires h264_nvenc in your FFmpeg build)
  ↓ if unavailable
VideoToolbox  (macOS — built-in, no extras needed)
  ↓ if unavailable
libx264  (CPU fallback, works everywhere)
```

---

## Installation

```bash
git clone https://github.com/fragmentone/f1streamer
cd f1streamer
npm install
pip install -r requirements.txt
```

> The `npm install` step also downloads Puppeteer's bundled Chromium (~170MB). This is expected.

---

## Configuration

```bash
cp config.example.json config.json
```

Open `config.json` and fill in your values. The three required keys are `token`, `guildId`, and `channelId`. Everything else has a sensible default.

### Required

| Key | Description |
|-----|-------------|
| `token` | Your Discord user token. [How to get it](#getting-your-discord-token). |
| `guildId` | Server ID. Enable Developer Mode in Discord settings, then right-click the server and copy. |
| `channelId` | Voice channel ID. Right-click the channel and copy. |

### Optional / tunable

| Key | Default | Description |
|-----|---------|-------------|
| `preSessionMinutes` | `15` | How early to wake up and start searching before session start. |
| `postSessionMinutes` | `60` | How long to keep streaming past the scheduled end. Buffer for overruns. |
| `calendarFile` | `f1-better-calendar.ics` | Path to your ICS file, relative to the project root. |
| `alwaysOnEmbed` | — | Optional fallback embed URL when both providers fail. |
| `probeByteTarget` | `52428800` | Bytes to collect for ffprobe analysis. 50MB gives accurate bitrate measurements. |
| `probeTimeoutMs` | `90000` | How long to wait for probe data before falling back to safe defaults. |
| `tlsProxyPort` | `18888` | Local port for the TLS sidecar proxy. Change if 18888 is already in use. |
| `ppvEmbedBase` | `pooembed.eu` | Embed domain used when constructing ppv.to URLs from `uri_name`. |
| `streamedEmbedBase` | `embedsports.top` | Embed domain used when constructing streamed.st URLs from match slugs. |
| `ppvDomains` | `["ppv.to", ...]` | ppv.to mirror list. Add new domains here when one goes down. |
| `streamedDomains` | `["streamed.pk", ...]` | streamed.st mirror list. |

### Getting your Discord token

Open Discord in a browser. Open DevTools (`F12`), go to **Application > Local Storage > https://discord.com**, and find the `token` key. Alternatively open the **Network** tab, refresh, and look for the `Authorization` header on any request to `discord.com/api`.

> **Selfbot warning.** This bot authenticates as your personal Discord account. Selfbots violate Discord's Terms of Service. Use at your own risk.

### Getting a calendar

[Better F1 Calendar](https://better-f1-calendar.vercel.app/) is the recommended source. It's a community-maintained ICS feed synced from official data, with clean and compact session names in exactly the format this bot's parser expects. Download the `.ics` file, put it in the project directory, and point `calendarFile` at it.

---

## Running

```bash
# Full scheduler -- runs continuously, handles the whole season
node f1scheduler.js

# Dry-run -- finds and prints the next stream URL, does not launch the streamer
node f1scheduler.js --dry-run
```

### Manual stream discovery

`test_providers.js` is a standalone CLI that queries ppv.to and streamed.st by keyword. It works for any sport, not just F1.

```bash
# Search without launching
node test_providers.js "australian grand prix"
node test_providers.js "timberwolves celtics" --sport basketball

# Search and launch the streamer with the result
node test_providers.js "australian grand prix" --launch ppv
node test_providers.js "ufc 300" --launch streamed
```

### Direct streamer launch

```bash
# Pass an embed page URL -- streamer will intercept the M3U8 via CDP
node streamer.js "https://embed-page-url"

# Pass a direct M3U8 -- skips the browser entirely
node streamer.js "https://cdn.example.com/stream.m3u8"
```

### Running as a systemd service

```bash
sudo cp f1streamer.service /etc/systemd/system/

# Edit the file and set User and WorkingDirectory to match your system
sudo nano /etc/systemd/system/f1streamer.service

sudo systemctl daemon-reload
sudo systemctl enable f1streamer
sudo systemctl start f1streamer

# Live logs
journalctl -u f1streamer -f
```

---

## Under the hood

Several parts of this codebase look like they're doing things the hard way. They are. Here is why each one exists.

<details>
<summary><strong>Why all HTTP routes through a Python proxy instead of Node's native stack</strong></summary>

<br>

Cloudflare and major stream CDNs fingerprint TLS client hellos. Node's native TLS stack produces a fingerprint that gets recognized and blocked. `tls_proxy.py` is a local HTTP server built on Python's `curl_cffi` library, which impersonates Chrome's TLS handshake at the cryptographic level.

This is not optional. Removing it breaks segment fetching even on CDNs that don't explicitly require it, because the fingerprint check happens at the connection level before any content is served.

The proxy also uses thread-local `curl_cffi` sessions. This matters because `curl_cffi` sessions are not thread-safe, and the HLS segment fetcher fires multiple concurrent requests per poll cycle.

</details>

<details>
<summary><strong>Why NUT container format instead of something more standard</strong></summary>

<br>

Discord enforced DAVE E2EE on all screenshare streams in March 2026. The `@dank074/discord-video-stream` library supports DAVE E2EE only in `format: 'nut'` mode. No other container format is compatible with it. This is not a configuration choice.

</details>

<details>
<summary><strong>Why two parallel ffprobe passes instead of one</strong></summary>

<br>

The first pass reads format and codec metadata from the container. The second pass reads actual frame types from the payload.

Both are necessary because streams frequently report `has_b_frames: 0` in the container header while actually containing B-frames in the encoded data. Discord's WebRTC decoder rejects B-frames at the hardware level. A stream that the container claims is clean will cause decoder failures in Discord if it isn't. The frame-level scan catches this regardless of what the header claims.

</details>

<details>
<summary><strong>Why segment fetching is concurrent instead of sequential</strong></summary>

<br>

HLS segments on live streams have tight timing. A slow CDN response on one segment shouldn't block every other segment from being fetched. Serializing fetches causes the proxy to fall behind the live edge on any CDN hiccup, which causes buffering on the Discord side. `Promise.all` keeps all pending fetches in flight simultaneously.

</details>

<details>
<summary><strong>Why Puppeteer's AdblockerPlugin is absent</strong></summary>

<br>

It was intercepting and blocking M3U8 playlist requests. Removed intentionally.

</details>

<details>
<summary><strong>Why remux is preferred over transcoding</strong></summary>

<br>

Transcoding burns CPU or GPU cycles, introduces latency, and can degrade quality. When a stream is already H.264 with no B-frames and within Discord's bitrate ceiling, it can be passed through directly with zero re-encoding. The hardware encoder cascade only activates when the stream actually requires it: B-frames detected, or measured bitrate above the Discord ingest limit.

</details>

---

## License

MIT (c) 2026 fragmentone
