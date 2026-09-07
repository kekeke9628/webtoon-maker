# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository is a single self-contained HTML file (`index.html`) — a browser-based tool ("AI 귀염 웹툰 스튜디오") that generates a multi-panel webtoon (Korean comic strip) from user-supplied character photos and a scene description. There is no build system, package manager, server, or test suite; all HTML, CSS, and JavaScript live inline in `index.html`.

## Running / developing

There is no build or install step. Open `index.html` directly in a browser (or serve the directory with any static file server, e.g. `python3 -m http.server`) and edit `index.html` in place — changes are visible on reload.

## Architecture

The app runs a two-stage client-side pipeline, all orchestrated from `startWebtoonPipeline()`:

1. **Storyboard planning (Gemini API).** Character reference images (converted to base64 via `fileToGenerativePart`) and their name/trait descriptions are sent to a user-selected Gemini text model (`generateContent` endpoint, called directly with the user's own API key — no backend proxy). The prompt instructs Gemini to split the scene into N panels and return a JSON array of English image-generation prompts, each repeating the characters' visual traits to preserve consistency across panels.
2. **Image rendering (Pollinations.ai).** Each panel prompt from step 1 is appended with the user-selected art style string and sent to `image.pollinations.ai` as a direct GET request. All panels in one episode share a single random `globalSeed` so the same seed is reused across panel requests, which is the main mechanism for keeping character/art consistency across the generated images.

Other notable client-side behavior:
- The Gemini API key is persisted in `localStorage` (`gemini_free_api_key`) and auto-loads (silently) on page load via `loadModels(true)`.
- `loadModels()` fetches the live list of available Gemini models from `v1beta/models` and filters to those supporting `generateContent`, preferring a `flash` model as the default.
- Character cards are added/removed dynamically in the DOM (`addCharacter()`); `getCharactersData()` reads all `.character-card` inputs into an array used by both the episode-suggestion and generation flows.
- `suggestEpisode()` is a separate, lighter Gemini call that proposes a short episode premise from the current character set, without generating images.
- Generated panel images are fetched as blobs and kept as `URL.createObjectURL` references in `window.currentImageUrls`; `downloadWebtoon()` triggers sequential downloads (staggered by 400ms) to avoid browser popup/download blocking.
- All UI strings, prompts sent to Gemini, and status messages are in Korean; prompts sent to the image model are required to be in English.
