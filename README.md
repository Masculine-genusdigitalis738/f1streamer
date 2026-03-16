# f1streamer

> Set it up once. Every session, every weekend, all season.

f1streamer watches your F1 calendar and automatically streams each session into a Discord voice channel — no manual stream hunting, no intervention required. Fifteen minutes before lights out, it wakes up, finds a live stream, and starts pushing video. When the chequered flag drops, it goes back to sleep.

Built around a chain of unglamorous but necessary tricks: a Chrome-impersonating TLS proxy to defeat CDN fingerprinting, a headless browser that intercepts HLS playlists at the network layer, a dual-pass frame analyzer that catches B-frames even when the container lies, and Discord's DAVE E2EE protocol for encrypted screenshare. Most of these exist because the straightforward approach gets blocked.

---

## What's in the box

| File | Role |
|------|------|
| `f1scheduler.js` | Calendar parser, session scheduler, stream discovery engine |
| `streamer.js` | HLS interceptor, FFmpeg pipeline, Discord screenshare |
| `tls_proxy.py` | Chrome-impersonating local HTTP proxy (CDN bypass) |
| `test_providers.js` | Manual stream finder CLI, works for any sport |

---

## Stream discovery

The scheduler wakes up `preSessionMinutes` before each session and runs a provider cascade:

**ppv.to** (primary) -- queries `/api/streams`, filters to motorsport, then scores each result against the GP name words, session type keyword, and a 3-hour time window. For the best match it tries three extraction methods in order:

1. `iframe` field directly from the API response (populated when live)
2. Construct an embed URL from the `uri_name` field
3. Fetch the watch page and scrape the iframe src

**streamed.st** (fallback) -- same scoring logic against `/api/matches/{sportId}`, same three-method extraction. If both providers fail, the cascade retries every 2 minutes up to 10 times before giving up.

Once a URL is found, `streamer.js` is spawned and monitored. If it crashes, the scheduler restarts it up to 5 times within the session window.

---

## The pipeline
ICS Calendar
|
v
f1scheduler.js  -->  ppv.to API  -->  scored match  -->  embed URL
-->  streamed.st API (fallback)
embed URL / direct M3U8
|
v
streamer.js
|
+-- Embed URL? --> Puppeteer (stealth) --> CDP Network intercept --> M3U8
+-- Direct M3U8? --> skip browser
|
v
tls_proxy.py (curl_cffi, Chrome TLS impersonation)
|
v
HLS segment fetcher (concurrent, live-edge tracking)
|
v
ffprobe x2 (format + codec analysis, frame-level B-frame scan)
|
+-- Clean H.264, no B-frames, under bitrate ceiling? --> remux (zero transcoding)
+-- B-frames or over ceiling? --> hardware encode
VAAPI (Linux Intel/AMD) > NVENC (NVIDIA) > VideoToolbox (macOS) > libx264
|
v
FFmpeg --> NUT container --> Discord voice (DAVE E2EE)

---

## Why the TLS proxy exists

Node's native TLS stack produces a fingerprint that Cloudflare recognizes and blocks on stream CDNs. `tls_proxy.py` is a local HTTP server that uses `curl_cffi` to make requests while impersonating a real Chrome TLS handshake. All HTTP in `streamer.js` routes through it unconditionally -- not just the CDN requests, everything.

The proxy uses thread-local `curl_cffi` sessions. This matters because `curl_cffi` sessions are not thread-safe, and the segment fetcher runs multiple concurrent requests per poll cycle.

---

## Why NUT container format

Discord's DAVE E2EE implementation (enforced March 2026) requires the `@dank074/discord-video-stream` library's `format: 'nut'` mode. No other container format works with it.

---

## Why `-bf 0` on every encode path

Discord's WebRTC stack rejects B-frames. `streamer.js` runs two parallel ffprobe passes -- one for format/codec metadata and one frame-by-frame scan -- because streams frequently lie in their container headers about B-frame presence. The frame scan catches them regardless.

---

## Prerequisites

- Node.js 18+
- Python 3.9+
- FFmpeg with libopus support
- A Discord account (selfbot -- your personal user token, not a bot)
- An F1 ICS calendar file

Hardware encoding is auto-detected and optional. The bot tests each encoder with a smoke test before committing:
VAAPI (Linux Intel/AMD)  ->  NVENC (NVIDIA)  ->  VideoToolbox (macOS)  ->  libx264

---

## Installation
```bash
git clone https://github.com/fragmentone/f1streamer
cd f1streamer
npm install          # also downloads Puppeteer's bundled Chromium (~170MB, expected)
pip install -r requirements.txt
cp config.example.json config.json
```

---

## Configuration

Edit `config.json` after copying from the example:

| Key | Default | Description |
|-----|---------|-------------|
| `token` | required | Your Discord user token. See below. |
| `guildId` | required | Server ID. Right-click server -> Copy Server ID (needs Developer Mode on). |
| `channelId` | required | Voice channel ID. Right-click channel -> Copy Channel ID. |
| `alwaysOnEmbed` | optional | Fallback embed URL for manual testing. |
| `preSessionMinutes` | `15` | How early to start searching for a stream before session start. |
| `postSessionMinutes` | `60` | How long to keep streaming after the scheduled session end. |
| `calendarFile` | `f1-better-calendar.ics` | Path to your ICS file, relative to the project root. |
| `probeByteTarget` | `52428800` | Bytes to buffer for ffprobe analysis (50 MB). |
| `probeTimeoutMs` | `90000` | Probe collection timeout in ms before falling back to safe defaults. |
| `tlsProxyPort` | `18888` | Local port for the TLS proxy. Change if 18888 is in use. |
| `ppvEmbedBase` | `pooembed.eu` | Embed domain for ppv.to URL construction. |
| `streamedEmbedBase` | `embedsports.top` | Embed domain for streamed.st URL construction. |
| `ppvDomains` | `["ppv.to", ...]` | ppv.to mirrors to try in order. Add new ones here when domains go down. |
| `streamedDomains` | `["streamed.pk", ...]` | streamed.st mirrors to try in order. |

### Getting your Discord token

Open Discord in a browser, open DevTools (`F12`), go to Application -> Local Storage -> `https://discord.com`, and find the `token` key. Alternatively check the `Authorization` header on any request in the Network tab.

> **Selfbot warning:** This authenticates as your personal Discord account, which violates Discord's Terms of Service. Use at your own risk.

### Getting an F1 calendar

[Better F1 Calendar](https://better-f1-calendar.vercel.app/) is the recommended source -- a community-maintained ICS feed synced from official data, with clean session names in exactly the format this bot's parser expects. Download the `.ics` file, drop it in the project directory, and point `calendarFile` at it.

---

## Running
```bash
# Full scheduler -- runs continuously, handles everything
node f1scheduler.js

# Dry-run -- finds and prints the next stream URL without launching the streamer
node f1scheduler.js --dry-run

# Test stream discovery by keyword -- works for any sport
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
# Edit the file -- set User and WorkingDirectory to match your setup
sudo systemctl daemon-reload
sudo systemctl enable f1streamer
sudo systemctl start f1streamer

# Follow logs
journalctl -u f1streamer -f
```

---

## License

MIT (c) 2026 fragmentone
