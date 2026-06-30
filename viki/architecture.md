# V.I.K.I — Architecture Reference

**Virtual Intelligent Kinetic Interface | Phase 3**

> This document is a structural reference for the VIKI Phase 3 codebase. It covers
> component responsibilities, system topology, data flows, and the architectural
> boundaries that govern how the system evolves. It does not cover implementation
> detail — it covers the map.

---

## System Overview

VIKI is a persistent AI agent with distributed physical presence. She is not a
chatbot, a home automation controller, or a voice assistant in the conventional
sense. She is a continuously running intelligence whose identity, memory, and
awareness exist in one place — the soul — and surface through whatever interface
is closest and most natural: a desk unit, a wall display, a wearable, a mobile
device.

The soul holds everything that makes VIKI who she is: conversational memory,
extracted facts, a rolling profile of the people she interacts with, a trust
model governing what she can act on, and the personality that carries across
every form she inhabits. The soul does not live on a device that gets closed,
carried away, or interrupted. It lives on always-on server infrastructure and is
continuously available.

Physical form factors — desk unit, wall display, future biped — are called
**extensions**. They are not different versions of VIKI. They are windows into
the same VIKI.

**Current phase:** Phase 3 — LLM brain, persistent memory, tool use, and physical
extension integration are complete. Soul-node separation (soul on dedicated Linux
server, presence nodes connecting via WebSocket) is in active development.

---

## Engineering Philosophy

Five principles govern every architectural decision in VIKI. They are not
aspirational — they are enforced in code.

**Abstraction over dependency.** Every external provider sits behind an abstract
base class. The LLM is swappable. The STT engine is swappable. The TTS voice is
swappable. The transport layer is swappable. VIKI does not notice when the
implementation behind an abstraction changes. Provider swap is a config change,
not a rewrite.

**Python owns all state. Hardware executes only.** No intelligence lives in
Arduino or any peripheral device. The soul makes all decisions and sends commands.
Hardware carries them out. This rule holds from the desk unit's single-char
serial protocol through to the future biped.

**Trust is earned, then codified.** VIKI observes and reports before she acts.
Acting privileges are granted one domain at a time, based on demonstrated
judgment. Trust enforcement is architectural — it lives at the registry boundary,
not as a policy document.

**Squeeze versus juice.** Every architectural decision is weighed by cost against
return. Low-squeeze, high-juice decisions get built early without hesitation.
High-squeeze decisions get broken into smaller steps until the squeeze is
understood.

**Soul continuity is non-negotiable.** Memory, personality, and the trust model
are backed up, versioned, and recoverable. Loss of soul is not treated as a data
loss event. It is treated accordingly.

---

## System Topology

```
┌─────────────────────────────────────────────────────────────┐
│                        SOUL SERVER                          │
│                   (Linux box, always-on)                    │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │  Brain   │   │  Memory  │   │   Tools  │   │Extensions│  │
│  │(brain.py)│   │(SQLite)  │   │(Registry)│   │(Registry)│  │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘  │
│       └──────────────┴──────────────┴──────────────┘        │
│                          │                                  │
│                   soul_server.py                            │
│                   (WebSocket server)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │ WebSocket (LAN)
          ┌─────────────────┼─────────────────┐
          │                 │                 │
   ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
   │  Mac Node   │   │ Wall Display│   │   Wearable  │
   │  (primary   │   │   (future)  │   │   (future)  │
   │  dev node)  │   └─────────────┘   └─────────────┘
   │             │
   │ InputRouter │ ◄── mic + console
   │ Desk Unit   │ ◄── Arduino (serial)
   │ Camera Unit │ ◄── OpenCV (local camera)
   │ Presence    │ ◄── orb renderer (subprocess)
   │ Visualizer  │
   └─────────────┘
```

**Soul server** — Runs brain, memory, LLM calls, and tool execution. Has no
presence capabilities. No microphone, no camera, no display. Configured via
`config/linux_node.yaml`.

**Presence nodes** — Declare input and output capabilities. Route input to the
soul server. Receive responses back. The Mac is the current primary node and
development platform. It will transition to a pure presence node once the Linux
soul server is fully operational. Configured via `config/mac_node.yaml`.

**Extensions** — Physical hardware or local processes that either embody VIKI's
presence (desk unit OLED face, servo head, orb visualizer) or provide environmental
awareness (PIR sensor, camera-based person detection). Each extension connects
to its presence node via serial, network, or inter-process transport. The bridge
translates between soul-level events and the extension's native protocol.

---

## Component Reference

### Brain — `core/brain.py`

The central orchestrator. Brain owns all runtime state and coordinates every
other component. Nothing in the system communicates peer-to-peer — everything
flows through brain.

**Owns:**
- Conversation history (live session context passed to LLM)
- Full session transcript (written to memory on conversation close)
- Conversation state machine (open → active → closing)
- Extension registry (live collection of connected extensions)
- Ambient event queue (environmental events held during active conversation)

**Responsibilities:**
- Receive input from InputRouter (mic or console)
- Inject rolling profile and relevant facts into system prompt context
- Dispatch to LLM via abstraction layer
- Route on response type: speech path or tool call path
- Execute tool call loop (up to 5 chained calls before graceful fallback)
- Detect conversation closing signals: `[END]` (goodbye), `[PIVOT]` (redirect),
  inactivity timeout
- Trigger memory write on conversation close (facts extraction, profile update)
- Manage extension lifecycle (bridge threads, ambient queue drain)
- Sync talking face animation and presence visualizer state to all speech output

**Note on provider imports:** Brain imports concrete provider classes directly
(`AnthropicLLM`, `WhisperSTT`, `SamanthaTTS`, `KokoroTTS`) at the top of the
module. The abstraction is enforced at the interface level — provider swap is
still a config change — but brain.py does not remain provider-agnostic at the
import level.

**Does not:**
- Touch SQLite directly
- Know provider wire formats

**Conversation lifecycle — three closing signals:**

| Signal | Source | Behavior |
|--------|--------|----------|
| `[END]` prefix in LLM response | LLM detects goodbye intent | Store facts, update profile, clean shutdown |
| `[PIVOT]` prefix in LLM response | LLM detects topic redirect | Store facts, update profile, open new conversation immediately |
| Inactivity timeout (10 min default) | Brain timer | Same as `[END]` — memory is never lost because user walked away |

---

### Memory — `memory/`

Persistent storage for VIKI's soul. Three files, three responsibilities.

**`memory/db.py`** — Connection, PRAGMA configuration, schema initialization,
trusted entity seed. Safe to call on every startup — idempotent. Nothing outside
`memory/` imports SQLite directly.

**`memory/store.py`** — All write operations: open conversation, close
conversation, store extracted facts, update rolling profile. Returns clean Python
types. No SQLite specifics leak outside the module.

**`memory/recall.py`** — All read operations: get profile, get facts, get
transcript, get recent conversations. Profile retrieval returns `None` on first
run — brain handles this gracefully.

**Schema — five tables:**

| Table | Purpose | Key fields |
|-------|---------|-----------|
| `entities` | Trusted and interactor entity registry | id, name, role, created_at |
| `conversations` | Full session transcripts | entity_id, started_at, closed_at, close_reason, transcript |
| `facts` | LLM-extracted discrete facts | entity_id, conversation_id, content, extracted_at |
| `profile` | Rolling per-entity summary | entity_id, summary, updated_at |
| `voice_prints` | Per-node voice embeddings for speaker verification | entity_id, node, embedding, enrolled_at |

**Entity model:** The schema is built around `entity_id` as the foreign key
across all tables. An `entities` table holds the authoritative record — the
trusted entity (user) is seeded on first run by `db.py`. In Phase 3 Step 2A
there is one trusted entity. The Step 2B entity model (people, animals, devices,
spaces each with their own profile) requires no schema migration — the structure
already speaks that language.

**Memory recall pattern:** On conversation open, brain retrieves the rolling
profile and recent facts for the active entity and injects them into the system
prompt. The LLM has continuity without holding any state itself. At conversation
close, brain makes one additional LLM call to extract discrete facts from the
session transcript, then updates the rolling profile. The LLM is the extraction
engine — memory lives in Python.

**Abstraction note:** Memory does not have a provider abstraction layer. SQLite
is not a vendor dependency — it is a standard that migrates cleanly to any
platform VIKI will run on. The `memory/` module boundary provides containment:
a future rewrite of db.py, store.py, and recall.py would not touch brain.py or
anything outside the module. The abstraction layer revisit is triggered when a
concrete swap scenario emerges — cloud sync, distributed nodes, or a specific
performance requirement SQLite cannot meet.

---

### LLM Abstraction — `llm/`

**`llm/base.py` — `LLMBase`**

Abstract base class enforcing the provider contract. Three methods:

- `chat(messages, tools)` → `dict` — A stateless conversation turn. Returns
  either `{"type": "speech", "content": str}` or
  `{"type": "tool_call", "content": {"id": str, "name": str, "arguments": dict}}`.
- `chat_with_result(messages, tool_call, tool_result, tools)` → `dict` — A
  single-step follow-up: brain passes the original `tool_call` dict and a
  provider-agnostic `tool_result` dict. The provider reconstructs the wire-format
  exchange internally. Returns same shape as `chat()`.
- `chat_with_chain(messages, exchanges, tools)` → `dict` — Multi-step follow-up:
  brain passes the full accumulated list of `exchanges` (each a `{tool_call,
  tool_result}` pair). The provider translates all pairs to wire format
  internally. Brain calls this method at every step of a chained tool call sequence.

Nothing outside `llm/` knows the provider wire format. Tool definition
translation (neutral → provider schema) and tool result translation (neutral →
provider wire format) both happen inside the provider implementation.

**`llm/anthropic_llm.py` — `AnthropicLLM`** *(active)*
Default model: `claude-sonnet-4-20250514` (overridden to `claude-sonnet-4-5`
in both `mac_node.yaml` and `linux_node.yaml`). Handles Anthropic's `tool_use` /
`tool_result` content block exchange internally. `_extract_conversation()`
separates the system message from conversation history — required by Anthropic's
API, which takes system as its own parameter.

**`llm/openai_llm.py` — `OpenAILLM`** *(available)*
Preserved as a valid provider. Swappable by config change.

---

### STT Abstraction — `stt/`

**`stt/base.py` — `STTBase`**
Enforces `transcribe(audio: np.ndarray) → str`. Audio is a `float32` numpy
array at 16,000 Hz — Whisper native format.

**`stt/whisper_stt.py` — `WhisperSTT`** *(active)*
Uses `faster-whisper` (pre-built binaries, no compiler toolchain required).
Model loaded once on init — not per-transcription. Model size is configurable
at instantiation. Privacy-first: audio never leaves the local machine on
nodes that run Whisper locally. Remote nodes without the compute to run Whisper
stream audio to the soul server where Whisper runs centrally.

---

### TTS Abstraction — `tts/`

**`tts/base.py` — `TTSBase`**
Enforces `speak(text: str) → None`. Synchronous by design in Phase 3 — mic
does not open while VIKI is speaking. Abstraction layer is designed to support
streaming TTS when that upgrade is warranted.

Three implementations exist; the active provider is selected via `tts.provider`
in the node YAML:

**`tts/samantha_tts.py` — `SamanthaTTS`** *(Mac node)*
macOS Samantha voice via `subprocess.run()`. Blocks until speech is complete.
Zero cost, zero dependency.

**`tts/kokoro_tts.py` — `KokoroTTS`** *(Linux node)*
Local neural TTS via `kokoro-onnx`. Generates audio entirely on-device and plays
via `sounddevice`. Model loaded once on init. Voice declared in `linux_node.yaml`.
No network required.

**`tts/none_tts.py` — `NoneTTS`** *(soul server / headless)*
No-op implementation. Used when the node has no audio output — the soul server
does not speak locally; it routes response text back to the originating presence
node via WebSocket for that node to speak.

---

### Tool Framework — `tools/`

Three components with one responsibility each.

**`tools/base.py` — `ToolBase` + exception hierarchy**

Abstract base class for all tools. Every tool declares:
- `name` — the identifier the LLM uses to call it
- `description` — what the tool does (LLM-facing)
- `input_schema` — provider-agnostic parameter definition
- `execute(input: dict) → str` — pure execution, returns a plain string

Three exception types cover all failure cases:
- `ToolNotFoundError` — raised by registry when tool name is unknown
- `ToolPermissionError` — raised by registry when entity role lacks access
- `ToolExecutionError` — raised by tools when execution fails

Brain catches all three. Every failure path results in a spoken LLM response.
No silent failures.

**`tools/registry.py` — `ToolRegistry`**

The policy enforcement point. Trust is enforced here, before `execute()` is
called. Untrusted entities never reach tool code. Tools are pure — they contain
no trust logic.

Entity role flows from the node YAML (user identity declaration) into every
`registry.dispatch()` call. Access policy is per-tool. Revoking access to a
tool is a registry change only.

**Active tools:**

| Tool | Capability | Trust required |
|------|-----------|----------------|
| `SystemInfoTool` | Platform, Python version, uptime, memory usage | trusted |
| `DateTimeTool` | Current local date and time, speakable format | trusted |
| `ShoppingListTool` | Add, remove, read, clear, print shopping list | trusted |
| `PrintContentTool` | Print any text content to default printer | trusted |
| `ListFilesTool` | List the project file tree | trusted |
| `ReadFileTool` | Read a specific file from the project | trusted |
| `SearchFilesTool` | Search file contents by keyword | trusted |
| `NotesTool` | Create and list notes | trusted |
| `DeleteNoteTool` | Delete a note by name | trusted |
| `WriteFileTool` | Write or overwrite a file in the project | trusted |
| `GetSnapshotTool` | Request a camera snapshot description from camera_unit | trusted |
| `SelfUtilizationTool` | Report CPU and memory utilization of the VIKI process | trusted |
| `EnrollVoiceTool` | Capture and store a voice embedding for speaker verification | trusted |

**Tool call loop — bounded chain:**
VIKI does not speak and call a tool in the same turn. Tool calls are exclusive.
The LLM either speaks or acts. Multiple tool calls may be chained within a
single turn — up to a maximum of 5. Each step appends a provider-agnostic
`{tool_call, tool_result}` exchange pair to a list in brain. The full accumulated
list is passed to `chat_with_chain()` on each step. If the chain reaches 5 calls
without resolving to speech, a graceful fallback is spoken.

---

### Input Router — `input/router.py`

Manages all input sources for a presence node. On the Mac node: microphone and
console run simultaneously in separate daemon threads. Neither is a fallback for
the other — both are active input paths.

**Mic input:** Continuous audio capture via `sounddevice`. Opens at device native
sample rate and resamples to 16,000 Hz for Whisper. RMS energy detection
identifies speech start and silence boundaries. Complete utterance is fired as a
`numpy` array to the STT layer. The mic stream is closed entirely during VIKI's
speech — when `mute_event` is set, the outer loop sleeps and the `InputStream`
context is not entered. This eliminates feedback loops from buffered audio.

**Console input:** Reads from stdin in a blocking daemon thread. Routes to the
same pipeline from that point forward. In muted mode, checks for the wake phrase
before passing through.

**Brain lock:** A `threading.Lock` in brain prevents simultaneous input
handling. An `is_speaking` flag blocks new input while VIKI speaks.

**Wake-word mute mode:** On explicit sleep request (LLM emits `[SLEEP]` tag),
brain and router both enter muted mode. In muted mode, mic audio is transcribed
by Whisper and checked against the configured wake phrase — no new dependencies.
Router clears muted mode and fires the `on_wake` callback (and `send_wake()` to
the soul server in node mode) on detection.

**Voice verification:** Optional per-node speaker verification using stored voice
embeddings (see `voice_prints` table). When enabled, mic audio is scored against
the enrolled embedding before dispatching to brain. Unenrolled nodes pass through.
Disabled by default — enabled via `input.voice_verification.enabled` in the node
YAML.

---

### Extension Architecture — `extensions/`

Extensions are physical manifestations of VIKI's presence or environmental
awareness. Each extension is self-contained: its own electronics (or subprocess),
its own Python bridge, its own capability declaration. Three extensions are
currently active.

**`extensions/transport/` — Transport abstraction**

`TransportBase` enforces four methods: `connect()`, `send()`, `receive()`,
`disconnect()`. `SerialTransport` implements `TransportBase` for USB serial
connections. Send and receive operate on plain strings — not dicts. The bridge
owns translation both directions.

The presence visualizer does not use `SerialTransport` — it communicates via
`multiprocessing.Queue` since the renderer runs as a subprocess on the same
machine.

---

**`extensions/desk_unit/`**

The desk unit is the first extension and the permanent development sandbox.
New sensors and protocols are validated here first; a production extension is
built second.

*`capabilities.yaml`* — Declares what the desk unit can perceive and express.
Soul discovers capabilities at runtime via the registration handshake — never
via hardcoded knowledge.

*`bridge.py`* — Runs in two daemon threads (read loop and debounce loop). Owns:
- Registration handshake on connect (`EVENT:register:desk_unit`)
- Presence debounce (2-minute continuous absence required before `presence_cleared`
  reaches soul, with 30-second cooldown before next `presence_detected` fires —
  prevents PIR noise from flooding the ambient event queue)
- Translation soul → Arduino: JSON command → single-char serial
- Translation Arduino → soul: `EVENT:*` string → structured event dict

**Arduino sketch — `viki_desk_unit.ino`**

Built on the proven Phase 2 architecture with two surgical changes: registration
message on boot and named presence events replacing raw sensor telemetry.

*Events (Arduino → bridge):*

| Event | Wire format | Class |
|-------|-------------|-------|
| Register | `EVENT:register:desk_unit` | Transactional |
| Presence detected | `EVENT:presence_detected` | Ambient |
| Presence cleared | `EVENT:presence_cleared` | Ambient |

*Commands (bridge → Arduino):*

| Command | Wire | Effect |
|---------|------|--------|
| Neutral face | `S` | OLED: neutral expression |
| Talking face | `T` | OLED: animated talking mouth |
| Mad face | `M` | OLED: mad expression |
| Unimpressed face | `U` | OLED: half-closed eyelids |
| Sleeping face | `D` | OLED: sleeping expression |
| Nod | `N` | Servo: nod gesture |
| Look center | `F` | Servo: head to center |
| Look left | `L` | Servo: head left |
| Look right | `R` | Servo: head right |
| Look up | `^` | Servo: head up |
| Look down | `v` | Servo: head down |

---

**`extensions/camera_unit/`**

Software extension — no Arduino, no serial. Runs OpenCV-based person detection
against one or more connected camera sources. Each source runs its own capture
thread; a shared debounce loop controls event forwarding to soul.

*Events (bridge → soul):*

| Event | Class | Condition |
|-------|-------|-----------|
| `person_detected` | Ambient | Person visible; cooldown elapsed |
| `person_cleared` | Ambient | 2-minute continuous absence confirmed |
| `snapshot_result` | Transactional | Response to `get_snapshot` command |

*Commands (soul → bridge):*

| Command | Effect |
|---------|--------|
| `get_snapshot` | Describe what the named camera source currently sees |

The `GetSnapshotTool` dispatches `get_snapshot` to camera_unit, which returns a
plain-English description as a transactional event. That result resolves the tool
call and lets the LLM speak the description. In soul-node mode, `get_snapshot` is
declared as a node tool — the soul proxies the call to the presence node that has
the camera.

---

**`extensions/presence_visualizer/`**

Software extension — no serial transport. Spawns an orb renderer as a child
process on the same machine and communicates via `multiprocessing.Queue`. Output
only in Phase 3 — emits no events to soul.

*Commands (soul → bridge → queue):*

| Command | Effect |
|---------|--------|
| `set_state: speaking` | Orb enters speaking visual state; audio-reactive energy stream starts via loopback device |
| `set_state: listening` | Orb enters listening visual state; audio stream stops |

When the speaking state activates, the bridge opens a loopback audio input
(BlackHole on Mac, What U Hear on Linux) and streams real-time RMS energy to the
renderer, producing an audio-reactive orb animation synchronized to VIKI's voice.

---

**Two message classes — transactional and ambient:**

Transactional messages require acknowledgment and response. Failure is
detectable and consequential (registration, security events, auth requests,
snapshot results). Ambient messages are fire-and-forget with debounce — a missed
PIR event is inconvenient, not catastrophic. The registration handshake is itself
transactional — an extension that believes it is registered but is not would be
silently ignored.

**Environmental events and the ambient queue:**

Environmental events do not interrupt an active human conversation. Ambient
events received during a conversation turn are held in the ambient queue and
drained when the conversation is idle. Transactional events bypass the queue
and are handled immediately. When an environmental event does reach brain, it
triggers a prompted LLM call — not a static phrase list. VIKI's response is
grounded in her current context and conversation history, never repeated, and
genuinely alive.

---

## Data Flow

### Conversational Turn

```
User speaks
    │
    ▼
InputRouter (mic thread)
  RMS energy → speech boundary detection
  mute_event blocks mic during VIKI speech
    │
    ▼
WhisperSTT.transcribe(audio: np.ndarray)
  → transcribed text
    │
    ▼
Brain.handle_input(text)
  acquire lock
  append to conversation history
  inject rolling profile + facts into system prompt
    │
    ▼
AnthropicLLM.chat(messages, tools)
  translate neutral tool definitions → Anthropic schema
  → {"type": "speech", "content": "..."}
    │
    ▼
Brain: speech path
  append response to conversation history + transcript
  send set_state:speaking → presence_visualizer
  send express:talking → desk_unit
    │
    ▼
SamanthaTTS.speak(text)  (or KokoroTTS / NoneTTS per node config)
  blocks until complete
    │
    ▼
  send express:neutral → desk_unit
  send set_state:listening → presence_visualizer
  release lock
  open mic
```

### Tool Call Sequence

```
AnthropicLLM.chat() → {"type": "tool_call", "content": {"id": "...", "name": "...", "arguments": {...}}}
    │
    ▼
Brain: tool call path
  extract tool_id, tool_name, tool_args from response["content"]
  record in transcript
    │
    ▼
ToolRegistry.dispatch(name, arguments, entity_role)
  check access policy → ToolPermissionError if denied
  look up tool → ToolNotFoundError if missing
  tool.execute(arguments) → plain string  (or ToolExecutionError)
    │
    ▼
Brain: append exchange pair to exchanges list:
  { "tool_call": {...}, "tool_result": {"tool_use_id": ..., "content": ...} }
    │
    ▼
AnthropicLLM.chat_with_chain(messages, exchanges, tools)
  translate all accumulated exchange pairs to tool_use / tool_result wire format internally
  → {"type": "speech", "content": "..."} or next tool_call
    │
    ▼
  (repeat up to 5 times, accumulating exchanges each step, then graceful fallback)
    │
    ▼
SamanthaTTS.speak(result)
```

### Conversation Close

```
LLM response begins with [END]  (or timeout fires)
    │
    ▼
Brain strips tag, speaks response
    │
    ▼
One additional LLM call: extract discrete facts from session transcript
    │
    ▼
memory/store.py: store_facts(conversation_id, facts)
    │
    ▼
One additional LLM call: update rolling profile from current profile + new facts
    │
    ▼
memory/store.py: update_profile(entity_id, new_summary)
memory/store.py: close_conversation(conversation_id, transcript)
    │
    ▼
shutdown_event.set() → main.py exits cleanly
  (in soul_mode: new conversation opened instead — soul stands by)
```

---

## Soul-Node Architecture

**Current state:** Mac runs the full soul. Linux soul server is in active
development.

**Target state (ADR-038, ADR-039):**

The soul — brain, memory, LLM calls, tool execution — runs permanently on a
dedicated Linux box at `192.168.1.110` (ethernet, reserved at router). It has
no presence capabilities. No microphone, no camera, no display. Configured via
`config/linux_node.yaml`.

The Mac, wall display, wearable, and any future form factor are presence nodes.
Each declares its own capabilities in its node YAML. They route input to the
soul server and receive responses back.

**Transport:** WebSocket over LAN. `NodeTransportBase` enforces the contract.
`WebSocketTransport` is the first implementation. Soul runs `soul_server.py`
which accepts node connections, routes to brain, and returns responses.
Nothing outside `core/transport/` imports a transport library directly.

**Why WebSockets over MQTT:** VIKI's topology is few nodes to one soul — not
many devices to a broker. WebSockets are a direct fit: bidirectional, persistent
connection, low latency, no broker dependency, no additional infrastructure.
MQTT's strengths are fan-out to many devices and guaranteed delivery at scale —
neither of which VIKI needs at this stage. If the topology grows to dozens of
nodes, MQTT becomes a reasonable revisit candidate.

**Node capability model (ADR-002):** Each presence node declares its own
capabilities in its node YAML — input modes (mic, console), STT provider, LLM
provider, TTS provider, extensions. The soul does not assume what any node
supports. A wearable with no microphone is a valid node. A maintenance terminal
with no display is a valid node.

**Node-side tools:** Certain tools require hardware only available on a specific
node (e.g., `GetSnapshotTool` requires the camera_unit extension). The node
declares which tools it can handle via `node_tools` in its soul config block. The
soul server proxies those tool calls back to the originating node over WebSocket
rather than executing them locally.

---

## External Dependencies

| Dependency | Role | Abstraction layer | Swappable? |
|-----------|------|-------------------|-----------|
| `anthropic` | LLM provider | `LLMBase` | Yes — config change |
| `openai` | LLM provider (alt) | `LLMBase` | Yes — config change |
| `faster-whisper` | Speech-to-text | `STTBase` | Yes — config change |
| macOS Samantha | Text-to-speech (Mac) | `TTSBase` | Yes — config change |
| `kokoro-onnx` | Text-to-speech (Linux) | `TTSBase` | Yes — config change |
| `sounddevice` | Audio capture and playback | InputRouter / KokoroTTS | Tied to platform |
| SQLite (stdlib) | Memory storage | `memory/` boundary | Revisit on concrete swap scenario |
| `opencv-python` | Person detection in camera_unit | `VisionBase` | Yes — new VisionBase implementation |
| `pyyaml` | Node config | Config loader | Not abstracted |
| `python-dotenv` | Secret injection | Startup script | Not abstracted |
| `psutil` | System metrics | SystemInfoTool | Tool-level only |
| `websockets` | Soul-node transport | `NodeTransportBase` | Yes — config change |
| Arduino / serial | Extension hardware | `TransportBase` | Yes — SerialTransport |

API keys are never committed to the repo. Environment variables or a gitignored
`.env` file only. This is enforced by convention and `.gitignore` — not a
runtime check — and holds from day one of Phase 3.

---

## Architectural Boundaries

These boundaries are the load-bearing walls of the system. Violating them
requires architectural justification, not just a convenient shortcut.

**`llm/` boundary** — Nothing outside `llm/` knows the provider wire format for
tool definitions or tool results. Provider swap is a new `LLMBase` implementation
and a config change. Brain and tools are unchanged.

**`memory/` boundary** — Nothing outside `memory/` imports SQLite directly.
All database access is contained within the module. A future memory storage
swap touches only the three files inside `memory/`.

**`tools/` trust boundary** — Trust is enforced at the registry before
`execute()` is called. Untrusted entities never reach tool code. Tools contain
no trust logic. Revoking access is a registry change only.

**`extensions/transport/` boundary** — Nothing outside `extensions/transport/`
imports a serial transport library directly. Bridge code receives its transport
via dependency injection. New transport is a new `TransportBase` implementation.

**Hardware protocol boundary** — The bridge owns translation in both directions.
The soul speaks JSON. The Arduino speaks single characters. Neither side knows
the other's protocol. A hardware upgrade that changes the wire protocol requires
only a bridge change — soul and Arduino sketch are independent.

**Soul-node boundary** — Presence nodes own interfaces. The soul owns state.
No intelligence lives on a node. A node that goes offline does not affect the
soul's continuity. Memory and personality survive any node failure.

---

## Decision Log

39 architectural decisions are recorded in `docs/phase3_architecture_decision_record.md`.
Key decisions and their current status:

| ADR | Decision | Status |
|-----|---------|--------|
| ADR-002 | Node capability model | Active |
| ADR-004 | Local Whisper, cloud STT for remote nodes | Active |
| ADR-005 | Anthropic as LLM provider, OpenAI as valid alternative | Active |
| ADR-008 | Python owns all state, hardware executes only | Active — foundational |
| ADR-011 | SQLite for memory storage | Active |
| ADR-013 | Rolling profile injected into system prompt | Active |
| ADR-016 | `[END]`/`[PIVOT]`/`[SLEEP]` prefix tags for conversation signals | Active |
| ADR-017 | Memory abstraction layer deferred | Deferred — revisit on concrete swap scenario |
| ADR-018 | Step 2B entity model deferred | Deferred — revisit when real usage reveals need |
| ADR-019 | Tool framework: ToolBase, registry, pure tools | Active |
| ADR-020 | Provider-agnostic tool contract both directions | Active — foundational |
| ADR-023 | Tool calls exclusive per step, max 5-call chain | Active |
| ADR-024 | Trust enforced at registry boundary | Active — foundational |
| ADR-027 | Physical manifestations called extensions | Active |
| ADR-030 | Two message classes: transactional and ambient | Active |
| ADR-032 | Environmental events trigger LLM-generated responses | Active |
| ADR-038 | Soul server on dedicated Linux infrastructure | In progress |
| ADR-039 | WebSocket transport between soul and presence nodes | In progress |

---

*Produced by [Redline Software Recovery](https://redlinesoftwarerecovery.com)*
*Architecture reference — produced from source documentation and codebase review.*
*Private codebase. Public deliverable.*
