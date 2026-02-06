# haos-on-mac

Run Home Assistant OS on a Mac Mini with a local voice assistant powered by Ollama. No cloud, no subscriptions, no data leaving your house.

This guide walks you from a clean Mac to a fully working voice-controlled home assistant with:

- **HAOS** running in a lightweight VM (UTM/QEMU)
- **Ollama** running natively on Metal GPU for fast LLM inference
- **Voice pipeline** — speech-to-text (Whisper), your LLM, text-to-speech (Piper)
- **Optional:** persistent memory, music control, and more

Each section is a checkpoint — you can stop at any tier and have something working.

## Using This Guide

This guide is written for someone comfortable at a command line. Every command is copy-pasteable with verification steps so you know each piece is working before moving on.

**If you'd prefer a co-pilot:** install [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and paste sections of this guide into it. CC can adapt the instructions to your specific Mac, fill in your IPs, troubleshoot errors, and handle the back-and-forth of setup. This entire environment was originally built with Claude Code over a few evenings.

## Why Not Docker / Colima?

We started there. Our first attempt was running HA Core in a Docker container on Colima (a lightweight Docker VM for macOS). It worked, but:

- **Colima's VM ate 19GB of RAM** just for the Docker runtime — leaving almost nothing for the LLM on a 32GB machine
- **No Supervisor** — HA Core in Docker doesn't have the Supervisor, which means no App Store, no one-click Whisper/Piper/Music Assistant installs, and no easy addon management
- **Networking headaches** — bridging Docker containers to the LAN for Ollama access was fragile

The HAOS VM approach solves all of this: the UTM/QEMU VM uses only 4GB, gives you the full HAOS experience with Supervisor and Apps, and bridged networking Just Works. The LLM runs natively on the Mac outside the VM, getting full Metal GPU access.

## Hardware Requirements

The Mac Mini is the AI compute server *and* the HAOS host. The biggest variable is **unified memory** — it determines which LLM you can run, and the LLM determines how smart your assistant is.

| Unified Memory | Storage | LLM Budget | What You Get |
|----------------|---------|------------|--------------|
| **16GB** ($599 base M4) | 256GB is fine | ~8-9GB | Small model (7-8B params). Basic conversation and device control. Explicit tool calling only — the model does what you ask but won't proactively store facts or make smart decisions. |
| **24GB** | 256GB+ | ~16-17GB | Mid-size model (14-15B params). Noticeably better conversation. Tool calling possible but reliability varies by model — test before relying on it. |
| **32GB** (recommended) | 256GB+ | ~24-25GB | Large model (30B+ params). Reliable proactive tool calling — the model detects personal facts and stores them without being asked. Full voice pipeline with persistent memory. This is what we developed and tested on. |

**Tested environment:** Mac Mini M4, 32GB unified memory, 1TB SSD, macOS Sequoia.

> **Storage note:** 256GB is plenty. HAOS VM (~15GB) + Ollama models (~5-18GB depending on model) + macOS + tools = ~50GB total. Storage is not the bottleneck — memory is.

> **Memory note:** The LLM budget above accounts for macOS (~3-4GB), the HAOS VM (4GB allocated), and the embedding model (~0.5GB if using persistent memory). Your actual headroom may vary.

### Which Mac Mini?

Any **Apple Silicon** Mac Mini works (M1, M2, M4, Pro/Max variants). The guide assumes Apple Silicon throughout — Intel Macs won't have Metal GPU acceleration for Ollama and are not covered here.

The base M4 Mac Mini (16GB, 256GB, $599) gets you a working local voice assistant. The 32GB model ($799) is recommended if you want reliable tool calling and persistent memory.

## What You'll Build

```
Mac Mini (Apple Silicon)
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Ollama (native, Metal GPU)     UTM (VM hypervisor) │
│  ├─ Conversation LLM            ├─ HAOS VM          │
│  │  (GLM-4.7-Flash, etc.)       │  ├─ HA Core       │
│  └─ Embedding model             │  ├─ Whisper (STT) │
│     (nomic-embed-text)          │  ├─ Piper (TTS)   │
│                                 │  └─ Your addons   │
│                                 └───────────────────│
│                                                     │
│  Voice: Mic → Whisper → LLM → Piper → Speaker      │
│         (HAOS VM)   (Mac native)  (HAOS VM)         │
└─────────────────────────────────────────────────────┘
```

The key insight: HAOS runs in a VM for its full addon ecosystem, while Ollama runs natively to use the Metal GPU. Bridged networking lets them talk to each other on your LAN.

---

## Phase 1: HAOS Virtual Machine

**Goal:** Home Assistant running in a VM, accessible from your browser.

### 1.1 Install UTM

[UTM](https://mac.getutm.app/) is a free, open-source VM hypervisor for macOS.

```bash
brew install --cask utm
```

Or download from [mac.getutm.app](https://mac.getutm.app/).

### 1.2 Download HAOS

Download the **HAOS generic-aarch64** image (the ARM64 build for Apple Silicon):

```bash
# Download the latest HAOS qcow2 image
# Check https://www.home-assistant.io/installation/alternative for the latest version
curl -L -o ~/Downloads/haos_generic-aarch64.qcow2.xz \
  https://github.com/home-assistant/operating-system/releases/download/15.0/haos_generic-aarch64-15.0.qcow2.xz

# Decompress
xz -d ~/Downloads/haos_generic-aarch64.qcow2.xz
```

> **Note:** Check the [HAOS releases page](https://github.com/home-assistant/operating-system/releases) for the latest version number and update the URL accordingly.

### 1.3 Create the VM

1. Open UTM
2. Click **Create a New Virtual Machine**
3. Select **Emulate** (not Virtualize)
4. Choose **Other**
5. Skip the ISO — we'll attach the disk directly
6. Configure:
   - **Architecture:** ARM64 (aarch64)
   - **Memory:** 4096 MB (4GB)
   - **CPU Cores:** 2
7. **Storage:** Skip the default disk (we'll import the HAOS image)
8. Save the VM

> **CRITICAL:** You must use the **QEMU backend** (Emulate), not Apple Virtualization (Virtualize). HAOS will not boot with Apple Virtualization — the bootloader is incompatible. If you chose "Virtualize" in step 3, delete the VM and start over with "Emulate."

### 1.4 Import the HAOS Disk

1. Right-click your new VM in UTM → **Edit**
2. Go to the **Drives** section
3. Delete the default empty drive
4. Click **New Drive** → **Import** → select the `haos_generic-aarch64.qcow2` file
5. Set the interface to **VirtIO**

### 1.5 Expand the Disk

The default HAOS disk is only 6GB — it will fill up almost immediately with apps. Expand it before first boot:

```bash
# Install qemu tools if needed
brew install qemu

# Find your VM's disk image
# Usually at: ~/Library/Containers/com.utmapp.UTM/Data/Documents/YOUR_VM_NAME.utm/Data/
qemu-img resize "<path-to-qcow2>" 64G
```

> **Tip:** If you're using Claude Code, tell it your VM name and it will find the exact path for you.

### 1.6 Configure Networking (Bridged)

This step is critical — without bridged networking, the HAOS VM can't reach Ollama on your Mac, and your other devices can't reach Home Assistant directly.

1. Right-click the VM → **Edit**
2. Go to **Network**
3. Change from **Shared Network** to **Bridged (Advanced)**
4. Select your active network interface (usually `en0` for Ethernet or Wi-Fi)

**Why bridged?** Bridged networking puts the VM on your home LAN with its own IP address (e.g., `192.168.1.x` alongside your Mac). This means:
- The VM can reach Ollama on your Mac's LAN IP — required for the LLM integration
- Other devices on your network (phones, tablets) can reach Home Assistant directly
- mDNS discovery works — `homeassistant.local` resolves from your network

The alternative (Shared/NAT networking) puts the VM on an isolated `192.168.64.x` subnet behind NAT. It technically works but requires port forwarding and makes Ollama connectivity harder. Use bridged.

### 1.7 First Boot

1. Start the VM in UTM
2. Wait 2-5 minutes for HAOS to initialize on first boot (it auto-expands the filesystem)
3. The console will eventually show a URL like `http://homeassistant.local:8123`
4. Open that URL in your browser (or find the VM's IP from your router)
5. Complete the HA onboarding wizard — create your user account

**Verify:**
```bash
# From your Mac terminal — replace with your VM's actual IP
# (check your router's DHCP list or the VM console output)
curl -s -o /dev/null -w "%{http_code}" http://YOUR_HAOS_IP:8123/api/
# Should return: 401 (unauthorized — this means HA is running)
```

> **Checkpoint:** You now have Home Assistant running. You can add integrations, control devices, and build automations — all without Ollama. Everything from here builds on this foundation.

---

## Phase 2: Ollama + LLM

**Goal:** A local LLM running on your Mac's GPU, connected to Home Assistant as a conversation agent.

### 2.1 Install Ollama

```bash
brew install ollama
```

### 2.2 Configure Ollama for Network Access

By default, Ollama only listens on localhost. The HAOS VM needs to reach it over your LAN, so Ollama must listen on all interfaces.

Edit the Ollama launchd plist to add environment variables:

```bash
# Find the plist
ls ~/Library/LaunchAgents/homebrew.mxcl.ollama.plist
```

Add these environment variables to the plist (in the `EnvironmentVariables` dict). If no `EnvironmentVariables` key exists, add the entire block:

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>OLLAMA_HOST</key>
    <string>0.0.0.0</string>
    <key>OLLAMA_FLASH_ATTENTION</key>
    <string>1</string>
    <key>OLLAMA_KV_CACHE_TYPE</key>
    <string>q8_0</string>
</dict>
```

| Variable | Value | Why |
|----------|-------|-----|
| `OLLAMA_HOST` | `0.0.0.0` | **Required.** Listen on all interfaces so the VM can reach Ollama. Without this, the HA Ollama integration will fail to connect. |
| `OLLAMA_FLASH_ATTENTION` | `1` | Faster inference on Metal GPU |
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` | Quantized KV cache, reduces memory usage |

Then restart Ollama:

```bash
brew services restart ollama
```

**Verify Ollama is listening on your LAN IP:**
```bash
# Use your Mac's LAN IP, not localhost
curl -s http://YOUR_MAC_IP:11434/api/tags | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin),indent=2))"
```

If this returns a JSON response, Ollama is accessible from the network. If it fails but `curl http://localhost:11434/api/tags` works, `OLLAMA_HOST` isn't set — re-check the plist.

> **Tip:** If you're using Claude Code, tell it to add these env vars to the Ollama plist — it knows the XML format.

### 2.3 Pull a Conversation Model

Choose based on your available memory (see [Hardware Requirements](#hardware-requirements)):

```bash
# 32GB Mac — recommended, reliable tool calling
ollama pull glm-4.7-flash

# 24GB Mac — lighter option, tool calling untested
ollama pull qwen2.5-14b

# 16GB Mac — basic conversation
ollama pull llama3.1:8b
```

### 2.4 Test the Model

```bash
# Quick test — should respond in a few seconds after initial load
curl -s http://localhost:11434/api/generate \
  -d '{"model":"glm-4.7-flash","prompt":"Hello, who are you?","stream":false}' \
  --max-time 120 | python3 -c "import sys,json; print(json.load(sys.stdin)['response'])"
```

> **Note:** The first call after a reboot loads the model into GPU memory. For large models (18GB+), this can take 30-60 seconds. Subsequent calls are fast.

### 2.5 Connect Ollama to Home Assistant

1. In HA, go to **Settings → Devices & Services → + Add Integration**
2. Search for **Ollama**
3. Enter your **Mac's LAN IP** and port: `http://YOUR_MAC_IP:11434`
   - Use the LAN IP (e.g., `192.168.1.100`), **not** `localhost` — HA is in a VM and `localhost` refers to the VM itself
4. Select your model (e.g., `glm-4.7-flash`)
5. Enable **Control Home Assistant** (experimental — lets the LLM control devices)

### 2.6 Set as Conversation Agent

1. Go to **Settings → Voice Assistants**
2. Select your assistant (or create one)
3. Set **Conversation Agent** to your Ollama integration

**Verify:**

In HA, open the Assist dialog (chat bubble icon) and type:

> "What can you help me with?"

You should get a response from your local LLM.

> **Checkpoint:** You now have a local LLM as your HA conversation agent. It can answer questions and control exposed devices. No cloud involved.

---

## Phase 3: Voice Pipeline

**Goal:** Talk to your assistant and hear it respond — speech-to-text and text-to-speech.

### 3.1 Install Whisper (Speech-to-Text)

1. In HA, go to **Settings → Apps → App Store** (previously called "Add-ons")
2. Search for **Whisper**
3. Install and start it
4. Wait for the model to download (this takes a few minutes)

### 3.2 Install Piper (Text-to-Speech)

1. In the App Store, search for **Piper**
2. Install and start it
3. It will download a default voice model

> **Warning:** As of early 2026, Piper v2.2.1 has a known bug (`espeakbridge` ImportError). If you hit this, check the [GitHub issue](https://github.com/home-assistant/addons/issues/4384) for the current status. The workaround is to pin to v2.1.1 — see [Troubleshooting](#piper-espeakbridge-error).

### 3.3 Configure the Voice Pipeline

1. Go to **Settings → Voice Assistants**
2. Select your assistant
3. Set:
   - **Speech-to-text:** Whisper
   - **Text-to-speech:** Piper
   - **Conversation agent:** Your Ollama integration (from Phase 2)

### 3.4 Test Voice

The easiest way to test is with the **Home Assistant Companion App** on your phone:

1. Install the Companion App (iOS or Android)
2. Connect to your HA instance (use the HAOS IP: `http://YOUR_HAOS_IP:8123`)
3. Tap the microphone icon and speak

You should hear your assistant respond with the Piper voice.

> **Why the Companion App?** Browsers block microphone access on non-HTTPS pages. Since HAOS runs on HTTP locally, you need the Companion App for voice. This is the intended HA voice client — not a workaround.

> **Checkpoint:** Full voice pipeline working. You can talk to your Mac Mini and it responds — Whisper transcribes, your LLM thinks, Piper speaks. All local.

---

## Phase 4: Persistent Memory (Optional)

**Goal:** Your assistant remembers facts across conversations.

This requires the **ha-semantic-memory** project, which adds semantic search-based memory to your voice assistant. It's a separate FastAPI service that runs on your Mac alongside Ollama.

**Requirements:** 32GB recommended (the conversation LLM needs to reliably call tools for memory to work).

See the full setup guide: [ha-semantic-memory](https://github.com/iXanadu/ha-semantic-memory)

Quick summary of what it adds:
- Tell your assistant "I live in Portland" → it stores the fact
- Days later, ask "Where do I live?" → it searches by meaning and answers correctly
- Works even when words don't match (semantic search, not keyword matching)

---

## Phase 5: Music Control (Optional)

**Goal:** Voice-controlled music playback through your Mac Mini speakers.

### 5.1 Install Music Assistant

1. In HA, go to **Settings → Apps → App Store**
2. Search for **Music Assistant**
3. Install and start it
4. Configure your music sources (Spotify, SiriusXM, local files, etc.)

### 5.2 Make Mac Speakers Available

Modern macOS doesn't support AirPlay *receiving* (Apple removed RAOP), so Music Assistant can't stream to your Mac directly. The workaround is **Squeezelite** — a lightweight Logitech Media Server client that makes your Mac's speakers appear as a player.

```bash
# Install Squeezelite
brew install squeezelite

# Test it (replace IP with your HAOS IP — Music Assistant runs Slimproto on port 3483)
squeezelite -o default -s YOUR_HAOS_IP
```

You also need to add the **Slimproto** provider in Music Assistant:
1. Open Music Assistant (HA sidebar or `http://YOUR_HAOS_IP:8097`)
2. Go to **Settings → Providers → Add → Slimproto** (port 3483)

The Mac Mini speakers should appear as a player in Music Assistant. For auto-start on boot, create a launchd plist.

> **Tip:** Tell Claude Code to create a launchd service for Squeezelite — it will generate the plist and install it.

### 5.3 Voice Control for Music

Install the [Music Assistant LLM Script Blueprint](https://github.com/music-assistant/voice-support):

```bash
# On HAOS via SSH:
mkdir -p /config/blueprints/script/music_assistant
curl -o /config/blueprints/script/music_assistant/llm_voice_script.yaml \
  https://raw.githubusercontent.com/music-assistant/voice-support/main/llm-script-blueprint/llm_voice_script.yaml
```

Then in HA:
1. **Settings → Automations & Scenes → Scripts → + Add Script → Use a Blueprint**
2. Select the Music Assistant blueprint
3. Save, then expose the script to your voice assistant (**Settings → Voice Assistants → Expose Entities**)

Add instructions to your Ollama system prompt telling the LLM how to use the music script (player name, media types, etc.).

> **Checkpoint:** "Play some jazz" → your Mac Mini plays music. All through voice, all local.

---

## SSH Access to HAOS (Recommended)

For managing files on HAOS (deploying Pyscript, editing configs), install the SSH addon:

1. In HA, go to **Settings → Apps → App Store**
2. Search for **Advanced SSH & Web Terminal**
3. Install, then go to **Configuration**:
   - Disable **Protection Mode** (on the Info tab — allows full filesystem access)
   - Add your SSH public key:
     ```bash
     # Generate a keypair on your Mac if you don't have one
     ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

     # Copy the public key to paste into the SSH addon config
     cat ~/.ssh/id_ed25519.pub
     ```
   - Paste the public key into the addon's Configuration → SSH → Authorized Keys field
4. Restart the addon

```bash
# Test SSH access from your Mac
ssh YOUR_HAOS_USER@YOUR_HAOS_IP

# Copy files to HAOS
cat local_file.py | ssh YOUR_HAOS_USER@YOUR_HAOS_IP 'sudo tee /config/path/file.py > /dev/null'
```

> **Note:** SCP is disabled on HAOS. Use the `cat | ssh tee` pattern shown above, or use the File Editor addon in the HA web UI.

---

## Entity Exposure Cleanup

This is easy to overlook but **critical for tool calling reliability**. When you expose entities to your voice assistant, each one generates a tool in the LLM's context. Too many tools dilute the model's attention and lead to unreliable tool selection.

**The problem:** By default, HA may expose media players, zones, device trackers, update entities, and other things the LLM doesn't need. With 20+ tools and 14+ entities, even good models start making mistakes — calling the wrong tool, or generating text about calling a tool instead of actually calling it.

**The fix:** Only expose what the LLM actually needs to control.

1. Go to **Settings → Voice Assistants → Expose Entities**
2. Toggle **OFF** everything except the entities your LLM should use
3. For a typical setup, you might expose:
   - `script.memory_tool` (if using persistent memory)
   - Your Music Assistant voice script (if using music control)
   - The specific media player target (e.g., `media_player.mac_mini_speakers`)
   - Any smart home devices you want voice-controlled (lights, switches, etc.)

**What to remove:**
- Duplicate media players (browser players, AirPlay instances, etc.)
- `zone.home`, `device_tracker.*`, `button.*`, `update.*`, `todo.*`
- `conversation.home_assistant`, `conversation.ollama_conversation` (meta-entities)

Fewer exposed entities = simpler LLM prompt = more reliable tool calling.

---

## Ollama System Prompt

The system prompt tells your LLM how to behave. This matters more than you'd think — a good prompt makes the difference between an assistant that reliably calls tools and one that just talks about calling them.

Configure it in **Settings → Devices & Services → Ollama → Configure**.

Key principles:
- **"CRITICAL RULE" language at the top** — the HA Assist API injects a preamble into your system prompt that tells the model to "answer questions about the world from internal knowledge." This directly conflicts with tool usage instructions (like "always search memory before answering"). Strong override language at the top of your custom prompt counters this.
- **Explicit tool instructions** — tell the model exactly when and how to call each tool
- **Personality section** — make it conversational, not robotic

If you're using ha-semantic-memory, see the [recommended system prompt](https://github.com/iXanadu/ha-semantic-memory/blob/main/docs/SYSTEM_PROMPT.md) for a tested template.

---

## Troubleshooting

### HAOS Not Loading

1. Check if the VM is running in UTM
2. `ping YOUR_HAOS_IP` — if no response, restart the VM in UTM (Stop → Start)
3. HA takes 1-2 minutes to fully boot after VM start
4. Check: `curl -s -o /dev/null -w "%{http_code}" http://YOUR_HAOS_IP:8123/api/` — should return `401`

### HAOS Won't Boot at All

If the VM shows a blank screen or boot loop, you likely used Apple Virtualization instead of QEMU. Delete the VM and recreate it — in step 3 of VM creation, choose **Emulate**, not Virtualize. See [Phase 1.3](#13-create-the-vm).

### Ollama Not Responding from HAOS

1. Check Ollama is running: `ollama list`
2. Check it's listening on all interfaces: `curl http://YOUR_MAC_IP:11434/api/tags` (use the LAN IP, not localhost)
3. If only localhost works, `OLLAMA_HOST=0.0.0.0` isn't set — check your Ollama launchd plist and restart with `brew services restart ollama`

### Model Loading Slow on First Call

This is normal. Large models (18GB+) take 30-60 seconds to load into GPU memory after a reboot. Once loaded, they stay in memory until Ollama is restarted or the model is unloaded. You can warm up the model:

```bash
curl -s http://localhost:11434/api/generate \
  -d '{"model":"glm-4.7-flash","prompt":"Hello","stream":false}' --max-time 120
```

### Voice Not Working

1. Check Whisper: **Settings → Apps → Whisper → Log** (should say "Ready")
2. Check Piper: **Settings → Apps → Piper → Log** (should say "Ready")
3. Verify pipeline: **Settings → Voice Assistants** — STT, TTS, and conversation agent all set
4. Browser mic requires HTTPS — use the Companion App instead for voice

### Piper espeakbridge Error

Piper v2.2.1 has a known `ImportError: cannot import name 'espeakbridge'` bug. The workaround is to pull the older working image and retag it:

```bash
# From HAOS SSH terminal:
ha addons stop core_piper
docker pull homeassistant/aarch64-addon-piper:2.1.1
docker rmi homeassistant/aarch64-addon-piper:2.2.1
docker tag homeassistant/aarch64-addon-piper:2.1.1 homeassistant/aarch64-addon-piper:2.2.1
ha addons start core_piper
```

Check [GH #4384](https://github.com/home-assistant/addons/issues/4384) — this may be fixed by the time you read this.

### "Add-ons" Missing in HA

Recent HAOS versions renamed "Add-ons" to "Apps." Look under **Settings → Apps → App Store**.

### VM Crash / Unresponsive

Usually caused by an app in a crash loop (Tailscale Serve and broken Piper versions are common culprits):
1. Force stop the VM in UTM
2. Start it again
3. Immediately disable the problem app: from SSH, run `ha addons stop <addon-slug>`
4. Fix the root cause before re-enabling

---

## Tips for Claude Code Users

If you're using Claude Code to set up your Mac:

1. **Start a session** in your home directory and tell CC what Mac you have (model, memory, what you want to achieve)
2. **Paste one phase at a time** from this guide — CC will adapt commands to your system
3. **Let CC troubleshoot** — if something fails, paste the error. CC has context about HAOS, Ollama, and the full stack
4. **Verification steps** are your friend — run them after each section so CC can confirm success before moving on
5. **Store your config** — tell CC to save IPs, model choices, and settings somewhere for future sessions

---

## License

MIT

## Acknowledgments

- [Home Assistant](https://www.home-assistant.io/) — the platform that makes all of this possible
- [Ollama](https://ollama.com/) — local LLM inference with Metal GPU support
- [UTM](https://mac.getutm.app/) — free macOS VM hypervisor
- [ha-semantic-memory](https://github.com/iXanadu/ha-semantic-memory) — persistent semantic memory for HA voice assistants
