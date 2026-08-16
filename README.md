# SceneCue

A mobile app that turns a theme-park ride visit into an upload-ready 60-second video.

**The bet:** every competitor treats narration as a track you lay over an already-cut
timeline. SceneCue inverts it — the spoken line's measured length determines how long
the scene holds. Narration drives the edit, not the other way around.

**The goal:** media captured → finished video in under 60 seconds of assembly time.
Realistic competitor time today is 15–30 minutes.

## Status

Pre-development. Capture flow and assembly logic are designed and prototyped;
no production code yet.

## Repo contents

```
docs/
  HANDOFF.md              Full design decisions, algorithm spec, open questions
prototypes/
  capture-flow.html       Clickable shot-list prototype (open in a browser)
  output-animatic.html    Playable animatic of the assembled 60s cut
```

Both prototypes are standalone HTML — no build step, just open them in a browser.

## Start here

Read [`docs/HANDOFF.md`](docs/HANDOFF.md). It covers the scene structure, the
narration-driven assembly algorithm, the ride-database bootstrap plan, naming research,
and the open questions that still need decisions.

## The 60-second structure

| # | Scene | Cut target | Class |
|---|---|---|---|
| 1 | Intro Bumper | 3s | Locked |
| 2 | Walking In | 6s | Elastic |
| 3 | Queue | 8s | Elastic |
| 3b | Height Sign | 2s | Elastic |
| 4 | Pre-Ride Check-In | 5s | Narration-driven |
| 5 | **The Ride (payload)** | 22s | Floored, min 18s |
| 6 | Post-Ride Reaction | 6s | Elastic |
| 7 | Sign-Off | 5s | Narration-driven |
| 7b | End Card | 3s | Locked |

## Riskiest assumption

Accelerometer-based highlight detection — using the phone's motion telemetry rather than
pixel analysis to find the best window of ride footage. The whole speed claim rests on
this working on-device. Spike it early.

## Next tasks

1. Scaffold the Android project (stack undecided — Kotlin/CameraX vs. React Native vs. Flutter)
2. Port the scene data model and elasticity classes from `prototypes/capture-flow.html`
3. Implement the budget solver as a pure, unit-tested function — it should reproduce the
   exact 60s split shown in `prototypes/output-animatic.html`
4. Spike accelerometer highlight detection against real ride footage
