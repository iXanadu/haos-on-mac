# DORMANT — haos-on-mac

**Status:** Sidelined (2026-06-13). The HAOS projects are mostly parked.
**Why parked:** Not under active development right now.

## What this is
A **public-facing install guide** (has a LICENSE): run Home Assistant OS on a Mac Mini
with a fully local voice assistant powered by Ollama — no cloud, no subscriptions. Walks
from a clean Mac to voice control: HAOS in a VM (UTM/QEMU), Ollama on Metal, voice pipeline
(Whisper STT → LLM → Piper TTS), plus optional persistent memory and music control. Tiered
— each section is a working checkpoint.

## What exists
- `README.md` — the tiered, checkpoint-based guide (the deliverable).
- `LICENSE`, `claude/` notes. Has `.engram.cfg` (added 2026-04-16, 8ce99fb).

## What's missing / open
- POSSIBLY STALE: the optional "persistent memory" tier likely describes
  ha-semantic-memory + Ollama embeddings, both RETIRED (now engram, no Ollama). Verify and
  refresh before publishing.
- OPEN (Rob): is this guide meant for publication/sharing, and is it being kept current?

## How to resume
Read `README.md`. It's documentation, not code — the work is verifying each tier still
matches current tooling and rewriting the memory section against engram.

## Related
`haos-macmini-management` (the ops/management companion), ha-mcp, engram.

_Last touched: 2026-04-16 (8ce99fb). Update this brief when you next put the project down._
