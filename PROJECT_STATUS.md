# Companion Robot — Project Status

## Why / What

A "phone on wheels" home companion robot. An iPhone is the robot's **head**
(camera, mic, speaker, face-screen, on-device Vision); a wheeled ESP32 base is the
**body**; a Mac running Pace is the off-board **brain** it reaches over Wi-Fi. v1
goal: rolls around one space, shows a face, holds a voice conversation, answers
"what do you see?", and can follow a person.

**Thesis:** reuse Pace's local LLM brain + voice loop instead of rebuilding them on
the robot, and keep all heavy reasoning on the Mac so the phone stays a thin head.

**In scope (v1):** teleop + voice + camera-to-VLM + follow-me, on a home LAN.
**Out of scope (v1):** SLAM / mapped navigation, roaming off Wi-Fi, autonomous
errands, manipulation/arms, multi-room goal-seeking. (The brain dies off-LAN — a
known structural limit; a hybrid on-phone Apple-FM fallback is a later option.)

## Dependencies

- **External:** none cloud. On-device only (Apple FM / LM Studio on the Mac).
- **Internal (fleet):** `pace` — provides the Mac brain via a new inbound
  `PaceRemoteListener` (Link B server). See `docs/protocol.md`.
- **Hardware (to buy, ~$80–120):** ESP32 dev board (BLE+Wi-Fi); 4WD chassis kit w/
  TT motors; TB6612FNG motor driver; VL53L0X ToF (front obstacle); 2× 18650 +
  holder; tall/gooseneck phone mount (face at eye height, not the floor); optional
  power bank to charge the phone.

## Timeline

- **2026-06-25** — Project scoped. Decisions: home telepresence/companion v1;
  iPhone head (reuse Pace Swift/voice/avatar); building from zero. Communication
  contract written (`docs/protocol.md`). No hardware ordered yet.

## Products

Nothing built yet. Planned surfaces:
- ESP32 firmware (C++/Arduino) — Link A peripheral + reflex safety loop.
- iOS head app (Swift) — BLE central, camera, voice loop, robot face, Link B client.
- Pace `PaceRemoteListener` — Link B server (lives in the `pace` repo).

## Features (shipped)

None yet.

## Todo / Planned / Deferred / Blocked

Build phases (each is a standalone demo):
1. **Phase 0 — Bench:** ESP32 + driver + wheels; throwaway iOS test app drives them
   over BLE. Proves a phone can move wheels. No brain.
2. **Phase 1 — Reflex firmware:** ToF obstacle stop + heartbeat watchdog.
3. **Phase 2 — The head:** iOS app with BLE link, robot face (from Pace's
   `PaceAvatarOverlay`), voice loop, teleop joystick.
4. **Phase 3 — Brain link:** Link B + `PaceRemoteListener` in Pace; "what do you
   see?" sends a camera frame to the VLM; conversation flows through Pace.
5. **Phase 4 — Follow-me:** on-phone Apple Vision person tracking → autonomous
   steering (advisory `motionIntent` from the brain, reflexes still final).

**Deferred:** on-phone Apple-FM brain fallback for off-LAN use; SLAM/navigation;
multi-room goals; charging dock.

**Blocked:** phases 0–1 blocked on ordering hardware (owner: user).
