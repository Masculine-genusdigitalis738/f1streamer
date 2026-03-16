# f1streamer

![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?style=flat-square&logo=node.js&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS-lightgrey?style=flat-square)

**Fully automated F1 live stream bot for Discord.** Watches your calendar, hunts down the stream, and pushes it into a voice channel with DAVE E2EE encryption. Runs headless all season with zero intervention.

---

## What it does

f1streamer wakes up before each session, queries two stream providers, scores every available stream against the session metadata, and extracts a playable URL using a cascade of three methods. It then intercepts the HLS playlist, runs the video through a frame-accurate analysis pipeline, and pushes it into Discord via screenshare. When the session ends, it stops and goes back to sleep.

The interesting part is what happens in between. Stream CDNs fingerprint TLS handshakes and block Node's native stack, so all HTTP routes through a local Python proxy that impersonates Chrome at the TLS level. Discord's DAVE E2EE protocol requires NUT container format. B-frames break Discord's WebRTC decoder, and many streams lie about their presence in the container header, so a second ffprobe pass reads the actual frames. Hardware encoder detection runs a smoke test before committing to any backend.

None of it is clever for its own sake. Each piece exists because the simpler approach failed.

---

## Architecture

```
F1 Calendar (ICS)
      |
      v
 f1scheduler.js
      |
      +-- 15 min before session: start provider cascade
      |
      |   ppv.to  -----> score streams by GP name + session type + time window
      |                  extract: API iframe field > uri_name construction > page scrape
      |
      +-- fallback if ppv.to fails
      |
      |   streamed.st --> same scoring logic
      |                  extract: API embedUrl > watch page scrape > slug construction
      |
      +-- retry every 2 min, up to 10 attempts
      |
      v
 streamer.js  <-- receives embed URL or direct M3U8
      |
      +-- Embed URL: Puppeteer (stealth) + CDP network intercept --> M3U8
      +-- Direct M3U8: skip browser
      |
      v
 tls_proxy.py  <-- all HTTP routes here (curl_cffi, Chrome TLS impersonation)
      |
      v
 HLS segment fetcher (concurrent, live-edge tracking)
      |
      v
 ffprobe x2  (format/codec pass + frame-level B-frame scan)
      |
      +-- clean H.264, no B-frames, under bitrate ceiling --> remux (zero transcoding)
      +-- needs work --> hardware encode
           VAAPI > NVENC > VideoToolbox > libx264
      |
      v
 FFmpeg  -->  NUT container  -->  Discord voice (DAVE E2EE)
```

---

## Prerequisites

- **Node.js** 18+
- **Python** 3.9+
- **FFmpeg** with libopus support
- A **Discord account** (selfbot -- personal user token, not a bot application)
- An **F1 ICS calendar file** (see [Getting a calendar](#getting-a-calendar))

Hardware acceleration is auto-detected and optional. The bot runs a smoke test for each encoder and falls back gracefully if it fails:

```
VAAPI (Linux Intel/AMD)  >  NVENC (Linux/Windows NVIDIA)  >  VideoToolbox (macOS)  >  libx264 (any)
```

---

## Installation

```bash
git clone https://github.com/fragmentone/f1streamer
cd f1streamer
npm install          # installs deps + downloads Puppeteer's bundled Chromium (~170MB)
pip install -r requirements.txt
```

---

## Configuration

```bash
cp config.example.json config.json
# edit config.json with your values
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `token` | string | **required** | Discord user token. [How to get it](#getting-your-discord-token). |
| `guildId` | string | **required** | Server ID. Right-click server > Copy Server ID (requires Developer Mode). |
| `channelId` | string | **required** | Voice channel ID. Right-click channel > Copy Channel ID. |
| `alwaysOnEmbed` | string | optional | Fallback embed URL used when both providers fail. |
| `preSessionMinutes` | number | `15` | Minutes before session start to begin stream discovery. |
| `postSessionMinutes` | number | `60` | Minutes past the scheduled end to keep streaming (buffer for overruns). |
| `calendarFile` | string | `f1-better-calendar.ics` | Path to the ICS file, relative to the project root. |
| `probeByteTarget` | number | `52428800` | Bytes to buffer for ffprobe analysis (50 MB). |
| `probeTimeoutMs` | number | `90000` | Probe timeout in ms before falling back to safe defaults. |
| `tlsProxyPort` | number | `18888` | Local port for the TLS proxy. Change if 18888 conflicts. |
| `ppvEmbedBase` | string | `pooembed.eu` | Embed domain for ppv.to URL construction. |
| `streamedEmbedBase` | string | `embedsports.top` | Embed domain for streamed.st URL construction. |
| `ppvDomains` | array | `["ppv.to", ...]` | ppv.to mirrors to try in order. Add new domains here when one goes down. |
| `streamedDomains` | array | `["streamed.pk", ...]` | streamed.st mirrors to try in order. |

### Getting your Discord token

Open Discord in a browser, open DevTools (`F12`), go to **Application > Local Storage > https://discord.com**, and find the `token` key. Alternatively, open the **Network** tab, refresh, and check the `Authorization` header on any request to `discord.com/api`.

> **Note:** This bot authenticates as your personal Discord account. Selfbots violate Discord's Terms of Service. Use at your own risk.

### Getting a calendar

[Better F1 Calendar](https://better-f1-calendar.vercel.app/) is the recommended source -- a community-maintained ICS feed synced from official data with clean, compact session names in exactly the format this bot's parser expects. Download the `.ics` file, drop it in the project directory, and set `calendarFile` accordingly.

---

## Running

```bash
# Run the full scheduler -- handles everything automatically
node f1scheduler.js

# Dry-run: find the next stream URL without launching the streamer
node f1scheduler.js --dry-run

# Test stream discovery by keyword (works for any sport, not just F1)
node test_providers.js "australian grand prix"
node test_providers.js "timberwolves celtics" --sport basketball
node test_providers.js "ufc 300" --launch streamed

# Run the streamer directly against a known URL
node streamer.js "https://embed-page-url"
node streamer.js "https://cdn.example.com/stream.m3u8"
```

### Running as a systemd service

```bash
sudo cp f1streamer.service /etc/systemd/system/
# Edit the file: set User and WorkingDirectory to match your setup
sudo systemctl daemon-reload
sudo systemctl enable f1streamer
sudo systemctl start f1streamer

# Follow logs
journalctl -u f1streamer -f
```

---

## Under the hood

A few things in the codebase look unusual. They are all load-bearing.

**All HTTP routes through `tls_proxy.py`, not Node's native stack.**
Cloudflare fingerprints TLS client hellos and blocks requests that don't look like real browsers. Python's `curl_cffi` library impersonates Chrome's TLS fingerprint at the handshake level. This is not optional -- even non-Cloudflare CDNs benefit from it, and removing it breaks segment fetching in practice.

**NUT container format for FFmpeg output.**
Discord's DAVE E2EE protocol (enforced March 2026) requires the `@dank074/discord-video-stream` library's `format: 'nut'` mode. No other container works with it.

**`-bf 0` on every encode path.**
Discord's WebRTC decoder rejects B-frames. The bot runs two parallel ffprobe passes: one for format and codec metadata, one that reads actual frame types. This exists because streams frequently report `has_b_frames: 0` in the container header while containing B-frames in the payload. The frame scan catches them regardless of what the header says.

**Concurrent segment fetching (`Promise.all`).**
Serializing fetches causes the proxy to drift behind the live edge on slow CDNs. A single slow segment fetch shouldn't stall everything else.

**Puppeteer AdblockerPlugin is absent.**
It intercepts and blocks M3U8 playlist requests. This is intentional.

**Remux is always preferred.**
Transcoding only activates when the stream needs it -- B-frames present, or bitrate above Discord's ceiling. Clean H.264 streams pass through with zero CPU/GPU usage.

---

## License

MIT (c) 2026 fragmentone
