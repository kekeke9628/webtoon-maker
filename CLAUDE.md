# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository is a single self-contained HTML file (`index.html`) — a browser-based tool ("AI 귀염 웹툰 스튜디오") that generates a multi-panel webtoon (Korean comic strip) from user-supplied character photos and a scene description. There is no build system, package manager, server, or test suite; all HTML, CSS, and JavaScript live inline in `index.html`.

## Running / developing

There is no build or install step. Open `index.html` directly in a browser (or serve the directory with any static file server, e.g. `python3 -m http.server`) and edit `index.html` in place — changes are visible on reload.

## Architecture

The app runs a two-stage client-side pipeline, all orchestrated from `startWebtoonPipeline()`:

1. **Storyboard planning (Gemini API).** Character reference images are downscaled client-side to at most `MAX_IMAGE_DIMENSION` (1024px) via an offscreen `<canvas>` and re-encoded as JPEG before being converted to base64 (`fileToGenerativePart`), to keep request payloads small. Images and name/trait descriptions are sent to a user-selected Gemini text model (`generateContent` endpoint, called directly with the user's own API key — no backend proxy). The prompt instructs Gemini to split the scene into N panels and return a JSON array of English image-generation prompts, each repeating the characters' visual traits to preserve consistency across panels. The response is validated (`extractGeminiText`, plus an `Array.isArray`/all-strings check on the parsed JSON) before use, so a safety-filtered or malformed Gemini response surfaces a clear error instead of a raw `TypeError`.
2. **Image rendering (Pollinations.ai).** All panel prompts from step 1 are appended with the user-selected art style string and requested from `image.pollinations.ai` **in parallel** (`Promise.allSettled`), each as a direct GET request. Placeholder slots are inserted into the DOM up front (in panel order) and swapped for `<img>` elements as each request resolves, so panels render progressively and stay in the correct order regardless of completion order. All panels in one episode share a single random `globalSeed` so the same seed is reused across panel requests, which is the main mechanism for keeping character/art consistency across the generated images. A per-panel failure doesn't abort the whole batch — failed slots show an inline error and the episode still completes if at least one panel succeeded.

Other notable client-side behavior:
- The Gemini API key is persisted in `localStorage` (`gemini_free_api_key`) and auto-loads (silently) on page load via `loadModels(true)`.
- `loadModels()` fetches the live list of available Gemini models from `v1beta/models` and filters to those supporting `generateContent`, preferring a `flash` model as the default; if that filtered list is empty, it's treated as a load failure rather than a silent success.
- Character cards are added/removed dynamically in the DOM (`addCharacter()`); `getCharactersData()` reads all `.character-card` inputs into an array used by both the episode-suggestion and generation flows.
- `suggestEpisode()` is a separate, lighter Gemini call that proposes a short episode premise from the current character set, without generating images.
- All Gemini/Pollinations requests go through `fetchWithTimeout` (`GEMINI_TIMEOUT_MS` / `IMAGE_TIMEOUT_MS`), so a hung request fails with a message instead of leaving the UI stuck loading forever.
- Generated panel images are fetched as blobs and kept as `URL.createObjectURL` references in `window.currentImageUrls`; `downloadWebtoon()` triggers sequential downloads (staggered by 400ms) to avoid browser popup/download blocking. `revokeCurrentImageUrls()` releases the previous episode's blob URLs before a new one starts generating, to avoid leaking memory across runs.
- Both the "generate" and "suggest episode" buttons are disabled for the duration of their async pipeline to prevent duplicate concurrent submissions.
- All UI strings, prompts sent to Gemini, and status messages are in Korean; prompts sent to the image model are required to be in English.
