# Deterministic Story Pipeline

A research prototype that turns a short story request into a narrated, multi-scene MP4.

The application coordinates five stages:

1. A language model converts the request into a structured storyboard.
2. A text-to-image model creates one keyframe per scene.
3. A text-to-speech model generates scene narration.
4. An image-to-video model animates each keyframe.
5. FFmpeg combines the scene clips and narration into one MP4.

The demo supports four to eight scenes, English narration, two configured speakers, automatic stage progression, bounded retries, media-integrity checks, full-screen image inspection, individual scene-video playback, and local generation history.

## Important status

This is a pipeline and evaluation prototype, not a production-ready generator. The current text-to-image interface creates scenes independently and does not accept an approved character image as conditioning. Strong prompts reduce visual drift but cannot guarantee identical character anatomy, clothing, objects, or art direction across every scene. See [RESEARCH_REPORT.md](RESEARCH_REPORT.md) for findings and production recommendations.

## Requirements

- Node.js 20 or newer
- FFmpeg available on `PATH`, or configured with `FFMPEG_PATH`
- Access to compatible storyboard, image, TTS, and image-to-video endpoints
- A public webhook URL that forwards completed API callbacks to `/webhook`

## Configuration

Copy `.env.example` to `.env` and supply your own values. Do not commit `.env`.

The server reads these environment variables:

| Variable | Purpose |
| --- | --- |
| `DEMO_PORT` | Local web-server port; defaults to `7861` |
| `API_BASE_URL` | Default API URL shown in the interface |
| `WEBHOOK_URL` | Public callback URL ending in `/webhook` |
| `FFMPEG_PATH` | FFmpeg executable or absolute binary path |

The API key is entered in the browser and stored only in that tab's session storage. It is removed from job-status responses and is not intentionally written to disk.

## Run locally

PowerShell:

```powershell
$env:API_BASE_URL = "https://your-api.example"
$env:WEBHOOK_URL = "https://your-tunnel.example/webhook"
$env:FFMPEG_PATH = "ffmpeg"
npm start
```

Then open `http://127.0.0.1:7861`.

The callback provider must be able to reach:

```text
POST https://your-tunnel.example/webhook
```

## Security and repository hygiene

- Never commit API keys, webhook secrets, internal credentials, customer data, or generated private media.
- Generated outputs and common media formats are excluded by `.gitignore`.
- Replace organization-specific endpoint routes before using the project with another provider.
- Review API licensing and company code-ownership rules before making a repository public.
- No open-source license is included because ownership and redistribution permission must be confirmed first.

## Validation

```powershell
npm run check
```

This validates JavaScript syntax. A full end-to-end test additionally requires the configured external APIs, webhook delivery, and FFmpeg.

