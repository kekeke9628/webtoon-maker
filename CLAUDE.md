# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository is a single self-contained HTML file (`index.html`) — a browser-based tool ("AI 귀염 웹툰 스튜디오") that generates a multi-panel webtoon (Korean comic strip) from user-supplied character photos and a scene description. There is no build system, package manager, server, or test suite; all HTML, CSS, and JavaScript live inline in `index.html`.

## Running / developing

There is no build or install step. Open `index.html` directly in a browser (or serve the directory with any static file server, e.g. `python3 -m http.server`) and edit `index.html` in place — changes are visible on reload.

## Architecture

The app runs a two-stage client-side pipeline, all orchestrated from `startWebtoonPipeline()`:

1. **Storyboard planning (Gemini API).** Character reference images are downscaled client-side to at most `MAX_IMAGE_DIMENSION` (1024px) via an offscreen `<canvas>` and re-encoded as JPEG before being converted to base64 (`fileToGenerativePart`), to keep request payloads small. Images and name/trait descriptions are sent to a user-selected Gemini text model (`generateContent` endpoint, called directly with the user's own API key — no backend proxy). Gemini is asked for a JSON **object** with two separate string arrays: `characters` (one locked, scene-free appearance sentence per character) and `panels` (scene/action/framing only, no appearance text). The response is validated (`extractGeminiText`, plus an all-strings check on both arrays) before use, so a safety-filtered or malformed Gemini response surfaces a clear error instead of a raw `TypeError`.
2. **Image rendering (Pollinations.ai).** Each request is assembled **in code** as `characterSheet + panel + artStyle`, where `characterSheet` is `plan.characters.join(', ')` — the identical string is mechanically prepended to every panel. This split is the app's primary consistency mechanism: the model is never trusted to re-describe the characters per panel, so appearance cannot drift between cuts. A shared random `globalSeed` across the episode's requests is a secondary nudge for art-style stability. Note that Pollinations takes only a text prompt (no reference-image input on this path), so the character sheet is what carries identity — not the uploaded photo itself.

   Requests carry a `referrer` query param (`POLLINATIONS_REFERRER`, the GitHub Pages host this app is served from) rather than a Pollinations token. This is deliberate and must stay that way: the repo is public and this is a static browser-only page, and Pollinations' own docs say *"Never put Bearer tokens in frontend code — use `referrer` authentication for web apps."* The referrer must match a domain registered on the Pollinations account for the Seed tier to apply; otherwise requests silently fall back to the anonymous tier.

   Requests are issued via `Promise.allSettled`, but their **start times are staggered** by `IMAGE_REQUEST_INTERVAL_MS` (5s, matching the registered Seed tier's one-request-per-5s limit; the anonymous tier is one per 15s, so this constant must be raised if the referrer stops resolving to a registered account). Generation still overlaps, so this is faster than fully sequential rendering while staying under the rate limit — firing all panels at once trips HTTP 429. Placeholder slots are inserted in panel order up front and swapped for `<img>` elements as each request resolves, so panels render progressively and stay correctly ordered regardless of completion order. A per-panel failure doesn't abort the batch: failed slots show an inline error and the episode completes if at least one panel succeeded.

Other notable client-side behavior:
- The Gemini API key is persisted in `localStorage` (`gemini_free_api_key`) and auto-loads (silently) on page load via `loadModels(true)`.
- `loadModels()` fetches the live list of available Gemini models from `v1beta/models` and filters to those supporting `generateContent`, preferring a `flash` model as the default; if that filtered list is empty, it's treated as a load failure rather than a silent success.
- Character cards are added/removed dynamically in the DOM (`addCharacter()`); `getCharactersData()` reads all `.character-card` inputs into an array used by both the episode-suggestion and generation flows.
- `suggestEpisode()` is a separate, lighter Gemini call that proposes a short episode premise from the current character set, without generating images.
- All Gemini/Pollinations requests go through `fetchWithTimeout` (`GEMINI_TIMEOUT_MS` / `IMAGE_TIMEOUT_MS`), so a hung request fails with a message instead of leaving the UI stuck loading forever.
- Generated panel images are fetched as blobs and kept as `URL.createObjectURL` references in `window.currentImageUrls`; `downloadWebtoon()` triggers sequential downloads (staggered by 400ms) to avoid browser popup/download blocking. `revokeCurrentImageUrls()` releases the previous episode's blob URLs before a new one starts generating, to avoid leaking memory across runs.
- Both the "generate" and "suggest episode" buttons are disabled for the duration of their async pipeline to prevent duplicate concurrent submissions.
- All UI strings, prompts sent to Gemini, and status messages are in Korean; prompts sent to the image model are required to be in English.
