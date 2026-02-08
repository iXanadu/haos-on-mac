# haos-on-mac — Context Memory

**Last Updated:** 2026-02-07
**Status:** Stable — No Active Development

---

## Current Focus

Documentation is complete and published. Updates only as needed for accuracy.

---

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| Phased approach (5 phases) | Each phase builds on the previous; users can stop at any level |
| Command-line audience | Target users comfortable with terminal; no GUI-only instructions |
| Bridged networking for VM | Allows direct device discovery on LAN, critical for HA |
| Local-only architecture | Privacy-first, no cloud dependency, commodity hardware |
| Hardware tier recommendations | Real-world tested configurations at 16/24/32GB levels |

---

## Content Guidelines

- Keep instructions precise and testable — every command should work as written
- Include expected output or verification steps after key commands
- Hardware recommendations must reflect tested configurations only
- No personal IPs, hostnames, or paths with usernames in committed content
- Troubleshooting section should cover real issues encountered during setup

---

## Notes

- The guide references ha-semantic-memory (Phase 4) and Music Assistant (Phase 5) as optional
- Piper TTS section may need updating when Kokoro TTS becomes a recommended alternative
- Model recommendations should be reviewed periodically as new models release
