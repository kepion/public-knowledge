---
name: voice-clone-privacy-boundary
description: Cloned-voice narration ships with the repo; the operator's voice biometrics never do — capability in-repo, identity in the operator plane
type: reference
classification: public
scope: public
repo: public-knowledge
---

The make-video skill supports narration in the operator's own cloned voice
(`tts_engine: "local-clone"`), and the **capability is shared while the voice is
private**. The boundary:

- **In the repo (shared with every adopter):** the engine code (`core/capture/tts_worker.py`,
  `core/capture/tts_local.py`), the driver hooks, the tool recipes in `core/tools/catalog.md`,
  and the SKILL.md instructions — including the reading-script method, tuning procedure
  (A/B batteries over exaggeration × cfg_weight), and pronunciation-steering tricks.
- **In the operator plane only (`~/.aos/profile/speaker/`, gitignored by location, never
  committed):** the reference recordings, the enhanced reference WAV, `profile.json`
  (chosen parameters + pronunciation rewrites), and every generated tuning clip. Each
  adopter records their own reference and builds their own profile; the renderer resolves
  it from `~/.aos` at runtime, so nothing identity-bearing exists in the tree.

**Why:** voice is biometric data. A shared repo must never carry any operator's voiceprint,
but the machinery to build one locally is exactly the kind of sanitized, reusable capability
shared knowledge repos exist to spread. Everything runs on-device (Chatterbox +
faster-whisper) — narration text and audio never leave the machine, and generated audio
carries Resemble's Perth watermark, so it is self-disclosing as AI-made.

**How to apply:** when adding any personalization to a shared skill (voice, signature,
avatar, style model), split it the same way — capability and method in the repo,
identity artifacts in `~/.aos/`, resolved at runtime by convention path. Never accept a
"just commit the sample" shortcut, and disclose synthetic media when sharing it.

Related: [[aos-memory-schema-quickstart]]
