# HardwareScan — Architecture Notes

This document explains the non-obvious design decisions. The code is one HTML file by deliberate constraint (corporate Windows, no install rights, no build tooling), which makes internal architecture discipline matter *more*, not less.

## 1. The LOOKUP CORE boundary

The file is split by a hard, commented boundary:

- **LOOKUP CORE** — identification logic. Pure functions, explicit context, zero DOM access, zero storage access.
- **UI layer** — everything else: app state, localStorage persistence, rendering, workflow orchestration, caching.

### Why caching lives *outside* the core

`lookupHardwarePart()` doesn't know caches exist. The UI wraps it (`runLookupCached`) and implements localStorage caching with a 30-day TTL. This is deliberate: the core's future home is a Node.js MCP server, where the right cache is a shared database with different invalidation semantics. Baking browser caching into the core would mean unbaking it later.

### Extraction guide (MCP server)

1. Copy everything between the `LOOKUP CORE` markers into a module.
2. `lookupHardwarePart(partNumber, opts)` and `lookupHardwareImage(dataUrl, opts)` become MCP tool handlers — they already take `{provider, apiKey, model, fields, prompt, hardwareType}` explicitly.
3. Replace the two `fetch` wrappers (`postJson`, `postAnthropic`) with Node's fetch — signatures are identical. The Anthropic browser-CORS header becomes unnecessary server-side and is harmless if left.
4. Implement server-side caching at the wrapper level, exactly as the browser does — the shared parts database (including human-verified alias entries) is the server's killer feature.

## 2. The learning loop

Three memory systems, layered by confidence:

| Layer | Trigger | Lifetime | Why |
|---|---|---|---|
| AI lookup cache | successful AI lookup | 30 days | results can go stale; APIs cost money |
| Verified-part memory | human confirms a manual entry or photo ID | permanent | a human checked it; this is ground truth |
| Photo fingerprint memory | human confirms a photo ID | permanent (200-entry FIFO) | visual dedupe for unlabeled parts |

The key insight is the **alias mechanism**: barcodes on hardware labels are frequently *not* the orderable part number (serials, agency file numbers, internal codes). When a scan fails and the operator manually enters the real part data, the tool stores the result under *both* the real PN *and* the scanned barcode. The unscannable label becomes scannable forever. Over months of normal work this builds a private database of exactly the industrial/embedded parts that generic web search handles worst.

## 3. Photo identification

### Why photos can't cache like barcodes

A barcode scan yields an identical string every time — a perfect cache key. Two photos of the same part are never byte-identical. The substitute is a **perceptual hash** (dHash: 17×16 grayscale, adjacent-pixel gradients → 256-bit fingerprint, Hamming distance for similarity).

### Why matches confirm instead of auto-applying

Two bare green SODIMMs of the *same model* look alike at fingerprint resolution → reusing the stored result is exactly right. Two bare green SODIMMs of *different models* also look alike → a silent auto-match would assign wrong specs with full confidence. Same mechanism, opposite outcomes — so a match always *asks* ("This looks like a part you've identified before — use that result, or run a fresh lookup?") and the human is the tiebreaker. Exception: in batch mode the queue review before "Add all" *is* the confirmation step, so matches auto-apply there, marked.

### Why learning happens at confirm time, not identify time

The operator can correct the AI's part number on the result card before adding. Storing at confirm time means the memory holds the *corrected* data — human-validated ground truth, consistent with the alias mechanism.

## 4. Auto-capture: two trigger models

The detector diffs a downscaled feed (120×90 @ ~7fps) against an empty-scene baseline. Three metrics drive a state machine: **presence** (fraction of pixels differing from baseline), **motion** (vs. previous frame), **sharpness** (Laplacian variance). All thresholds are named constants with a live on-screen readout, because they are scene-dependent and the operator must be able to tune them.

- **Bench trigger** (`settle`): part placed → hand withdraws → stillness for ~900ms + sharpness floor → snap. The classic ID-scanner model. Re-arms only when the scene returns to baseline, so it can't double-fire on the same part.
- **Handheld trigger** (`burst`, default): a hand holding a 5-gram part in the air *never* passes a stillness gate — tremor is physiology, not a tuning problem. So the model inverts: once a part is present, buffer the **sharpest frame** seen over a 2.5s window and snap that. The operator's steadiest instant wins automatically. An all-blurry window error-tones and retries instead of wasting an API call.

The state machine (`acStep`) is a pure function of metrics — fully unit-tested without a camera.

### Adaptive crop

The presence diff's bounding box *is* the part's location, so the capture crops to it (+10% padding). No fixed capture zone to configure: a SODIMM gets a tight crop, a 3.5" drive a large one, and the silkscreen gets the full upload resolution instead of sharing it with the background. Degenerate boxes fall back to full frame.

## 5. Scanner-burst keyboard guard

Grading is keyboard-driven (A/B/C/D set grade on a pending result). But a USB scanner is a very fast keyboard — a barcode beginning with "A" would set a grade and corrupt the scan. The guard: if a grade key is followed by another printable key within 120ms, it wasn't a human — revert the grade and replay the character into the scan input. Humans can't type that fast; scanners can't type slower.

## 6. Provider abstraction

Each provider entry owns its request builder and response parser. Notable per-provider handling:

- **OpenAI**: Responses API with strict JSON schema; web search tool with automatic retry on the legacy `web_search_preview` tool type.
- **Anthropic**: direct-browser-access header; `pause_turn` continuation loop (server-tool turns can pause mid-search); `tool_choice` stays on auto — server tools don't support forced selection.
- **Gemini**: `google_search` grounding, with sources extracted from `groundingMetadata`.
- **Groq** (compound): no structured-output or image support — a line-based text protocol with its own parser; excluded from photo ID with a clear operator message.

A regex spec-mining layer (`expandFieldsFromDescription`) backfills structured fields from free-text descriptions when a provider returns prose instead of clean JSON — vendor lists, capacity/speed/form-factor extraction per hardware type.

## 7. Testing without a browser

The app is exercised headlessly in Node: the inline script is extracted and run against a minimal DOM/localStorage/canvas/AudioContext shim, with `fetch` mocked per-provider. The suites cover the inventory engine (merge/dedupe/undo/bulk ops), the cache layers, demo mode, the photo-memory flow, and the full auto-capture state machine driven with synthetic metrics. Pure-function design (acStep, the core, the hash/Hamming utilities) is what makes this possible — the parts that can't be simulated (live APIs, real optics, scanner timing) are exactly and only the parts that need a human at a bench.
