# HardwareScan

**AI-powered hardware identification and inventory for ITAD / e-waste operations — in a single HTML file.**

Scan a barcode (or photograph a bare, unlabeled part), let an AI provider identify it with live web search, and build an export-ready inventory — grading, lot tracking, data-destruction status, and a local knowledge base that gets smarter every time a lookup fails and a human fills in the gap.

No build step. No server. No install. One file, open it in Chrome, go.

> **[▶ Live demo]((https://swanamakerdev.github.io/hardwarescan/))** — click *"Or try the demo — no key needed"* on first launch. Sample inventory loads instantly and lookups are simulated locally.

<!-- TODO: replace with a real capture — a 10-second GIF of scan → result → add-to-inventory is worth more than everything below it -->
![HardwareScan demo](docs/demo.gif)

---

## Why this exists

Component-level ITAD work has an identification problem. Whole machines flow through ERP systems with service tags and serial databases — but the loose parts bin doesn't. A tub of mixed RAM means typing part numbers into a search engine one stick at a time, and the industrial/embedded parts (Advantech, Innodisk, ATP…) often return nothing at all. Bare sticks with no label don't even get that far.

HardwareScan turns that into: **scan → AI identifies it with web search → press Enter → next part.** And when the AI can't identify something, the manual-entry fallback *teaches the tool* — the next time anyone scans that barcode, it resolves instantly from local data, forever, for free.

## Features

**Identification**
- USB barcode scanner input (keyboard-wedge) with part-number cleaning and a transparent `Scanned: X | Sent to AI: Y` trace
- Four AI providers — OpenAI, Anthropic, Gemini, Groq — each using live web search, with per-hardware-type source steering (industrial RAM vendors, Intel ARK, TechPowerUp, FCC ID…)
- **Photo identification** for unlabeled parts: the vision model reads chip silkscreen codes and web-searches them in the same call
- **Auto-capture** with two trigger models: *handheld* (buffers your sharpest frame over a burst window — hand tremor is irrelevant by design) and *bench* (place part, hand withdraws, hold-still snap). Adaptive crop: the detected object defines its own capture region, so a SODIMM and a 3.5" drive both work untouched

**A tool that learns**
- 30-day cache for AI lookups — repeat scans are instant and free
- **Permanent memory for human-verified data**: failed scan → manual entry → that barcode is remembered forever, including label/serial codes that map to a different orderable part number
- **Perceptual-hash photo memory**: re-photograph a previously identified part and it offers the stored result (confirm-first — visually identical sticks can be different models, so a human always ratifies the match)
- Rapid count: re-scanning a part already in the current lot just bumps quantity — no AI call, no API key needed, no keystroke

**Inventory & workflow**
- Condition grading A–D (keyboard-driven: A/B/C/D + Enter, with a scanner-burst guard), toggleable per-job
- Lot / customer / location session stamping; data-destruction status (pending / wiped / certified) with bulk updates
- Batch queue mode with per-item grade override and quantity steppers; manual entry form; inline cell editing; undo
- Search, filter chips, sortable columns, collapsible stats (totals by type/grade/lot, total RAM GB, total storage TB)
- Export to Excel/CSV (three templates, by-lot filter); CSV/Excel import with column mapping and preview
- Dark mode (default), large-font mode, sound cues designed to be parsed eyes-off

## Quick start

1. Download `HardwareScanGit.html` (or clone and open it). Chrome recommended.
2. Open it. Click **"Or try the demo"** to explore with sample data — or:
3. **Settings → pick a provider → paste an API key.** Groq has a free tier; keys live in your browser's localStorage and are sent only to that provider's official API endpoint, nothing else.
4. Scan tab → scan or type a part number → Enter.

Sample part numbers that work in demo mode: `KVR16N11/8`, `M378A1K43CB2-CRC`, `ST4000DM004`, `SR2L6`, `WS-C2960X-24TS-L`.

## Architecture

The interesting design constraint: the identification logic is built for a future beyond the browser.

```
┌──────────────────────────────────────────────┐
│  UI layer — DOM, localStorage, workflows     │
│  (scan tabs, inventory, caching, settings)   │
├──────────────────────────────────────────────┤
│  LOOKUP CORE  — zero DOM dependencies        │
│  lookupHardwarePart()   pure async           │
│  lookupHardwareImage()  pure async           │
│  parsePartNumber · detectHardwareType        │
│  buildLookupPrompt · normalizeFields         │
│  per-provider request builders + parsers     │
└──────────────────────────────────────────────┘
```

Everything inside the `LOOKUP CORE` markers takes explicit context (provider, key, model, field definitions) and touches neither the DOM nor storage — caching is a UI-layer concern. The core is designed to be lifted verbatim into a Node.js **MCP server**, so the same identification engine can serve every workstation from a shared, learning parts database. See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full writeup and extraction guide.

Other deliberate choices documented there: why photo lookups can't cache like barcodes (and the perceptual-hash + human-confirmation design that works instead), the scanner-burst keyboard guard, and the two auto-capture trigger models.

## Known gotchas (field-tested)

- **OpenAI "quota exceeded" that isn't.** Some org/project configurations restrict spend on legacy-tier models and report it as `insufficient_quota` — a billing-shaped error for an access-shaped problem. Symptom: text and photo calls fail with a quota message while the billing page shows credit. Fix: use a current-generation model (default here is `gpt-5.4-mini`).
- **Anthropic from the browser** requires the `anthropic-dangerous-direct-browser-access` header; server tools (web_search) don't support forced `tool_choice`.
- **Barcode ≠ part number.** RAM labels carry multiple barcodes; the scannable one is often a serial or agency code. The tool's alias memory exists precisely for this — identify it once manually, and that label resolves forever.
- **Webcam optics are the floor for photo ID.** Chip silkscreen is ~1mm text; fixed-focus laptop cameras have a sweet-spot distance. The live sharpness meter in the photo panel tells you when you've found it.

## Privacy & data

Everything lives in your browser's localStorage: inventory, caches, settings, API keys. Nothing is transmitted anywhere except the AI lookup requests, which go directly from your browser to the provider you selected. Clearing browser data clears the tool — export regularly.

## Roadmap

- [ ] MCP server extraction of the lookup core (shared cache across workstations, server-side keys, CORS-free integrations: Icecat, eBay Browse, FCC ID)
- [ ] Cache export/import for backup and team sharing
- [ ] Two-photo (front/back) identification for dense modules

## License

MIT — see [LICENSE](LICENSE).

Built by a working ITAD technician to solve a real bench problem, with AI-assisted development. The best feature requests came from production use.
