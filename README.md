# medkit

hands-free first-aid coaching through meta ray-ban smart glasses. say "hey medkit" — the ai sees through your glasses camera, asks clarifying questions, and guides you step-by-step until ems arrives.

built at devfest cu 2025.

## what it does

medkit follows established red cross and aha first-aid playbooks. it never diagnoses — it coaches. every response opens with "call 911" for life-threatening situations, and a persistent banner reads: *decision support only — call emergency services.*

supported scenarios:
- **cpr** — 110 bpm metronome through the glasses speakers, compression rhythm guidance
- **severe bleeding** — pressure hold timers, wound care steps
- **choking** — heimlich maneuver walkthrough
- **burns, fractures, allergic reactions** — playbook-based guidance

if you go quiet during an active emergency, medkit checks in — "still pressing on the wound?" — because silence might mean you need encouragement, not that you're done.

## architecture

```
iOS App (SwiftUI) ──WebSocket──▶ Modal Backend ──▶ OpenAI Realtime API
                                       │
                               Dedalus (GPT-4o Vision)
                                       │
                          Tool commands ──▶ iOS App (local execution)
```

the backend runs four concurrent async loops per session:

| loop | role |
|---|---|
| ios → realtime | streams pcm16 audio and blurred video frames |
| realtime → ios | routes ai voice, transcripts, and tool call commands |
| scene analysis | gpt-4o vision analyzes frames every 8 seconds, injects visual context |
| proactive follow-ups | checks in after 30s of silence, then every 45s (max 3 without response) |

## stack

| layer | technology |
|---|---|
| glasses | meta ray-ban + wearables dat sdk (`MWDATCore`, `MWDATCamera`) |
| ios app | swiftui, avfoundation, scenekit, mapkit |
| voice | openai realtime api (pcm16, 24khz, `alloy` voice) |
| vision | gpt-4o vision via dedalus (dauth credential isolation in hardware enclaves) |
| backend | modal (serverless asgi), fastapi websockets |
| privacy | mediapipe face blurring — on-device, before any frame leaves the phone |

## ios components

| component | role |
|---|---|
| `AudioManager` | bidirectional audio: wake word detection via `SFSpeechRecognizer`, glasses mic capture, ai voice playback through glasses speakers |
| `ToolExecutor` | runs metronome, countdown timers, and ui checklist cards locally on device |
| `SessionLogger` | records 30 fps h.264 video via `AVAssetWriter`, timestamped transcripts, pdf and ems report export |
| `WebSocketManager` | backend communication — audio/frame streaming, tool command ingestion |
| 3d wireframe | scenekit body model with animated sphere overlays highlighting the active region (chest, arm, etc.) |
| mapkit | nearby aed, hospital, and pharmacy search |

## privacy

face blurring runs on every frame on-device before any data is sent to the cloud. no raw video reaches the backend. session recordings stay local — video, transcripts, and ems reports are only exported when the user explicitly chooses to share them.

## session export

at the end of every session:

| export | format | contents |
|---|---|---|
| video | mp4, h.264, 30 fps | full session recording |
| transcript | pdf | timestamped conversation with session metadata |
| ems report | txt | structured briefing for paramedics: timeline, scene observations, actions taken |

## quick start

### backend

```bash
pip install modal
modal deploy app.py
```

set secrets in the modal dashboard: `OPENAI_API_KEY`, `DEDALUS_API_KEY`.

### ios app

1. open `MedKit.xcodeproj` in xcode
2. set your team in signing & capabilities
3. in `Info.plist`: set `MetaAppID = ""` and `TeamID = <your-team-id>`
4. build and run on a physical iphone with the ray-ban glasses paired
