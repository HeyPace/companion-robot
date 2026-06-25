# Companion Robot — Communication Protocol

The robot is three pieces that talk over two links. This doc is the **contract**:
firmware, the iOS head app, and the Mac brain (Pace) all build against it. Keep
it the single source of truth — if a link changes, change it here first.

```
  ESP32 firmware  ──Link A (BLE)──  iPhone head app  ──Link B (Wi-Fi/Bonjour)──  Mac brain (Pace)
  drive + sensors                   camera/voice/face                            planner + VLM
```

Two repos, one contract:
- **`companion-robot`** (this repo) — ESP32 firmware + iOS head app. Owns Link A
  fully; owns the **client** half of Link B.
- **`pace`** — owns the **server** half of Link B (a new inbound listener; see
  "Pace-side bridge" below). Pace's planner/VLM/voice pipeline is reused unchanged.

---

## Link A — iPhone ↔ ESP32 (BLE)

iPhone is **central**, ESP32 is **peripheral**. Binary, little-endian, fixed-size
structs (no JSON over BLE — too slow). One custom GATT service.

Service UUID: `to be generated once, pinned here` (use a fixed UUID, not a kit default).

| Characteristic | Props | Payload | Meaning |
|----------------|-------|---------|---------|
| `drive`     | write (no response) | `int8 left, int8 right` (−100..100, % power) | Differential drive command. |
| `heartbeat` | write (no response) | `uint8 seq` | Watchdog ping, sent ~5 Hz by the phone. |
| `sensors`   | notify | `uint16 frontDistanceMm, uint8 flags, uint8 batteryPct` | Pushed ~20 Hz. `flags` bit0 = obstacle-stop active. |
| `command`   | write | `uint8 opcode, ...` | Out-of-band: LED, beep, calibrate. Extensible. |

**Reflex rules (firmware-side, non-negotiable — the LLM never overrides these):**
1. **Obstacle stop:** if `frontDistanceMm < STOP_THRESHOLD`, zero the motors and set
   `flags` bit0, regardless of the last `drive` command.
2. **Link watchdog:** if no `heartbeat` write arrives within `HEARTBEAT_TIMEOUT_MS`
   (e.g. 500 ms), zero the motors. A dropped BLE link must stop the robot, not
   leave the last command running.

Both reflexes run on the ESP32 at firmware speed so they never wait on the phone
or the Mac. The phone's follow/steer logic only ever *requests* motion; the
firmware is allowed to refuse it.

---

## Link B — iPhone ↔ Mac brain (Wi-Fi)

Reuses the inbound transport scoped for Pace's iPhone-remote work. The robot's
head app is just another client of the same listener.

- **Discovery:** Bonjour `_pace-remote._tcp` on the LAN. Phone browses, finds the
  Mac, no manual IP.
- **Pairing:** one-time shared secret (QR shown on the Mac, scanned by the phone),
  stored in iOS Keychain. Every request carries it. Listener is **default-OFF** and
  rejects unpaired connections.
- **Scope:** LAN-only. Out of Wi-Fi range = no brain (acceptable for a home
  companion; see PROJECT_STATUS "out of scope").

### Request envelope (phone → Mac), JSON over the framed connection

```json
{
  "type": "chat" | "frame" | "approvalDecision",
  "auth": "<paired-secret>",
  "sessionId": "<robot-uuid>",
  "text": "what do you see?",          // type=chat (≤500 chars, matches Pace's cap)
  "imageJpegBase64": "...",            // type=frame: a camera still for the VLM
  "decision": "allow" | "cancel"       // type=approvalDecision (see below)
}
```

`type:chat` maps directly onto Pace's existing `submitChatTranscriptFromDeepLink(_:)`
entry point. `type:frame` is the new bit: a real-world camera frame takes the place
of the screenshot Pace's VLM path normally consumes.

### Response stream (Mac → phone), JSON deltas

```json
{
  "state": "idle" | "listening" | "processing" | "responding",
  "spokenText": "I see a kitchen and a person waving.",  // Pace's lastSpokenReplyText
  "failureNarration": null,                               // Pace's PaceFailureNarration, verbatim
  "motionIntent": { "kind": "turnToward" | "approach" | "stop" | "none", "bearingDeg": -15 }
}
```

`spokenText` is the post-processed string Pace already produces (action tags and
`<think>` stripped) — the phone speaks it via `AVSpeechSynthesizer`. `motionIntent`
is **advisory only**: the phone decides whether/how to act on it, and the firmware
reflexes still get the final say. The brain sets goals; it does not drive wheels.

### Approve-on-phone round-trip

Action turns keep Pace's approval gate, but the prompt is forwarded to the phone
(the earlier decision: approve remotely). Mac sends an out-of-band:

```json
{ "type": "approvalRequest", "actionSummary": "drive into the next room", "expiresInMs": 8000 }
```

Phone shows a sheet, replies with `type:approvalDecision`. Default on timeout =
**cancel**, matching Pace's local NSAlert default.

---

## Pace-side bridge (the one change in the `pace` repo)

To make the two repos talk, Pace gains a **`PaceRemoteListener`** (new, additive):
`NWListener` + Bonjour advert, paired-secret auth, every command logged to the
existing `PaceAPIAuditLog`, gated behind a new default-OFF `enableRemoteControl`
toggle in Settings. It wires incoming `chat`/`frame` to the existing
`submitChatTranscriptFromDeepLink` / VLM path and streams `lastSpokenReplyText` +
`voiceState` + `lastFailureNarration` back out. Nothing in the planner, VLM, voice,
or approval pipeline is rewritten — the listener is a thin inbound wrapper.

This is the only edit Pace needs. It's deferred until Link B work begins (build
phase 3); phases 0–2 don't touch Pace at all.

---

## Shared types

Link B's envelopes are the only types both repos must agree on. Keep them as a
small `RobotBrainProtocol` Swift file **vendored identically** in both repos (or a
tiny shared Swift package if that proves worth the overhead). Link A's binary
structs live only in this repo (firmware ↔ iOS).
