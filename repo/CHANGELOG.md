# Changelog

## 1.0.0 — 2026-06

First public release. Built iteratively against daily production use at a working ITAD bench.

**Identification**
- Four-provider AI lookup (OpenAI / Anthropic / Gemini / Groq) with live web search and per-hardware-type source steering
- Photo identification for unlabeled parts (vision reads chip silkscreen + web-searches the codes in one call)
- Auto-capture with handheld (sharpest-frame burst) and bench (hold-still) trigger models; adaptive crop from the detection bounding box
- Demo mode: simulated lookups + sample inventory, zero-key evaluation

**Learning systems**
- 30-day AI lookup cache; permanent memory for human-verified manual entries, including barcode→part-number aliases
- Perceptual-hash photo memory with confirm-first reuse
- Rapid count: re-scans of known parts bump quantity with no API call

**Inventory**
- Grading A–D (keyboard flow with scanner-burst guard, per-job toggle), lots/customers/locations, data-destruction status with bulk ops
- Batch queue, manual entry with smart field mapping, inline editing, quantity steppers, undo
- Excel/CSV export (three templates, by-lot) and import (column mapping + preview); search/filters/sorting/stats

**Engineering**
- LOOKUP CORE isolated with zero DOM/storage dependencies, designed for extraction into a Node.js MCP server
- Self-contained single file (SheetJS inlined); headless Node test suites over a DOM shim
