# SceneCue

A mobile app that turns a theme-park ride visit into an upload-ready 60-second video.

**The bet:** every competitor treats narration as a track you lay over an already-cut
timeline. SceneCue inverts it. The spoken line's measured length determines how long
the scene holds, so narration drives the edit and not the other way around.

**The goal:** media captured to finished video in under 60 seconds of assembly time.
Realistic competitor time today is 15 to 30 minutes.

## Status

Browser prototype with a working capture-to-video pipeline. No native Android code yet.

The prototype records or uploads real clips, trims them with a live preview, divides the
60 seconds with a budget solver, and renders a downloadable video with music, clip audio
and a ride-info end card. See [`docs/HANDOFF.md`](docs/HANDOFF.md) for what was tested and
what was not.

## Repo contents

```
docs/
  HANDOFF.md              Design decisions, algorithm spec, test results, open questions
prototypes/
  capture-flow.html       Working prototype: capture, trim, plan, render (open in a browser)
  output-animatic.html    Playable animatic of the intended 60s cut
```

Both prototypes are standalone HTML with no build step. Use Chrome and open the file.
Camera recording needs a page served over https or localhost, or the file opened locally.
Uploads and the "sample footage" test mode work everywhere.

## Try it

1. Open `prototypes/capture-flow.html`.
2. Either record or upload clips for each scene, or press "Test mode: fill every unfinished
   scene with sample footage".
3. Confirm the height minimum on the ride info screen. It is safety-critical and gates the render.
4. Press Assemble. The browser records in real time, so it takes about as long as the video.

## The 60-second structure

| # | Scene | Cut target | Class |
|---|---|---|---|
| 1 | Intro Bumper | 3s | Locked |
| 2 | Walking In | 6s | Elastic |
| 3 | Queue | 5s | Elastic |
| 3b | Height Sign | 2s | Elastic |
| 4 | Pre-Ride Check-In | 5s | Narration-driven |
| 5 | **The Ride (payload)** | 25s | Floored, min 18s |
| 6 | Post-Ride Reaction | 6s | Elastic |
| 7 | Sign-Off | 5s | Narration-driven |
| 7b | End Card | 3s | Locked |

Targets are starting points. The solver moves them to fit 60s, for example a longer spoken
line shortens the elastic scenes first and then the ride, down to its floor.

## Riskiest assumptions

1. **Accelerometer highlight detection.** The prototype reads phone motion sensors during
   recording when the browser exposes them, but this has not been tried on a real ride.
2. **Speed.** The browser render is real time. The 60-second goal needs on-device hardware
   encoding in the native app.

## Next tasks

1. Scaffold the Android project (stack undecided: Kotlin and CameraX, React Native, or Flutter).
2. Port the scene data model and the budget solver as pure, unit-tested functions.
3. Spike accelerometer highlight detection against real ride footage.
4. Validate the ride-data sources (Queue-Times, ThemeParks.wiki) before designing around them.
