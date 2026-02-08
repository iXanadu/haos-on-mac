# haos-on-mac — Codebase State

**Last Updated:** 2026-02-07
**Status:** Complete — Published

---

## Project Overview

Documentation repository providing a comprehensive guide for setting up Home Assistant OS on a Mac Mini with a fully local (no cloud) voice assistant powered by Ollama.

---

## Current State

- **README.md**: ~560 lines, 5 phases covering the full setup
- **2 commits**: Initial guide + real-world refinements
- **Public on GitHub**: iXanadu/haos-on-mac

### Guide Structure

| Phase | Topic | Status |
|-------|-------|--------|
| 1 | HAOS VM Setup (UTM) | Complete |
| 2 | Ollama + LLM Installation | Complete |
| 3 | Voice Pipeline (Whisper STT, Piper TTS) | Complete |
| 4 | Persistent Memory (optional) | Complete |
| 5 | Music Control (optional) | Complete |

Additional sections: SSH access, entity exposure cleanup, system prompt best practices, troubleshooting, Claude Code tips.

---

## Recent Work

- Added real-world setup details from build experience
- Hardware tier recommendations (16GB → 32GB unified memory)
- Troubleshooting section with known issues and workarounds

---

## Next Planned Work

No immediate updates planned. Potential future updates:
- Update for newer HAOS versions as they release
- Add xAI Grok cloud LLM option alongside local Ollama
- Add Kokoro TTS as alternative to Piper
- Update model recommendations as new models release
