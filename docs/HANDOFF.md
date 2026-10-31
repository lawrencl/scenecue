# SceneCue — Project Handoff

A mobile app that turns a theme-park ride visit into an upload-ready 60-second video.
Target: media captured → finished video in **under 60 seconds of assembly time**.

## Files in this folder

| Path | What it is |
|---|---|
| `prototypes/capture-flow.html` | Working prototype: capture, trim, plan, and real video render |
| `prototypes/output-animatic.html` | Playable animatic of the intended 60-second output |
| `docs/HANDOFF.md` | This doc, the decisions that live nowhere else |

Both HTML files are design references. The parts worth porting are the
**scene data model**, the **elasticity classes**, and the **budget solver**.

## Prototype build state

`capture-flow.html` is no longer a mockup. It does the following for real:

- **Capture:** camera video and photos (scene 3 takes up to 3 photos, minimum 2), plus upload.
  Video-only on the payload scene. Defer and resurface logic for scene 4.
- **Trim:** shown for every video clip. A live preview plays only the kept window. The
  window slides (slip edit) or resizes from either edge, with the budget impact shown.
- **Analysis:** loudness per half second from the microphone (measured live while recording,
  or decoded from uploads), speech start and end, and phone motion sensors when available.
  Narration scenes set their hold to 0.4s lead + speech + 0.6s tail.
- **Plan:** the budget solver divides 60s across scenes and shows each result.
- **Render:** the browser draws the cut to a canvas and records it with a synthesized
  music bed that ducks under speech, clip audio, intro bumper (4 styles), stat chips,
  and an end card built from the confirmed ride info.
- **Safety gate:** the render is blocked until the height minimum is confirmed.

### Solver behavior

Locked scenes reserve first, narration scenes claim what the speech needs, the payload
keeps its floor, and elastic scenes absorb the rest. Over budget: elastic scenes shrink,
then the payload shrinks to its floor. Under budget: the payload grows, then elastic
scenes, each up to the footage available. If footage runs out the cut is shorter than 60s
and the screen says so.

### What was tested

- Solver: 2000 randomized cases. Total reached 60s within 0.05s and every scene stayed
  inside its limits whenever the fixed scenes fit.
- Speech detection: a clip with speech at 3.0s to 6.2s was found at 3.0s to 6.25s, giving
  a 4.3s hold. Silence and a single click were correctly ignored.
- Render from uploaded clips: clip audio appeared in the expected scenes of the final mix.
- Music ducking, measured in an isolated offline render: about 15 dB lower under speech and
  about 7 dB lower under the ride.
- Capture flow: photo counter, finish-early button, video-only enforcement, trim drag and
  resize, and the preview all worked against uploaded clips.

### What was not tested

- **Recording from a camera.** The simulated camera used for testing returned zero bytes
  to the recorder even in a bare test page, so camera recording is unverified. The app
  shows a clear message if a recording comes back empty.
- **Full-resolution render quality and frame rate.** The test machine had one CPU core
  and could not keep up with encoding 720x1280 video in real time.
- **Motion sensors on a real ride.** No device was available.
- **Safari and Firefox.**

### Known limits

- The browser render runs in real time and needs the tab to stay open.
- Output is mp4 only when the browser can write H.264, otherwise WebM.
- Ride data values are sample data. Only the Space Mountain height (40 in at Disneyland)
  was checked against published sources. An earlier placeholder of 44 in was the Walt
  Disney World figure and was wrong for Disneyland.

---

## The core bet

Every competitor (CapCut, GoPro Quik, InShot, VN, PicPlayPost) treats narration as a
track you lay over an already-cut timeline. SceneCue inverts this: **the spoken line's
measured length determines how long the scene holds.** That's the differentiator, and
it's what makes "hold key moments for narrator fillers" automatic rather than manual.

Secondary bet: **speed**. Realistic competitor time from footage to upload is 15–30 min
(CapCut best-case with templates ≈ 2 min). A true 60-second path is 2–20x faster than
anything shipping today.

Speed requires: no cloud round-trip (on-device processing), zero template-browsing
decision points, and background rendering while the creator is still previewing.

---

## Scene structure (7 scenes + 2 sub-scenes)

| # | Scene | Capture spec | Cut target | Class |
|---|---|---|---|---|
| 1 | Intro Bumper | none (pre-built, user picks style) | 3s | Locked |
| 2 | Walking In | selfie video, entrance sign in frame | 6s | Elastic |
| 3 | Queue | 3 photos (2 minimum) | 5s | Elastic |
| 3b | Height Sign | 1 photo, or use ride database | 2s | Elastic |
| 4 | Pre-Ride Check-In | selfie video, app-fed prompt | 5s | Narration-driven |
| 5 | **The Ride — PAYLOAD** | ~40s raw video, **video only** | 25s | Floored (min 18s) |
| 6 | Post-Ride Reaction | selfie video, unscripted, no prompt | 6s | Elastic |
| 7 | Sign-Off | selfie video, short goodbye | 5s | Narration-driven |
| 7b | End Card | none (ride data burned in) | 3s | Locked |

Total: **60s**

### Rules
- **Payload scene (5) is video-only.** No photo fallback — the whole video is built
  around it. Any scene with this property gets `allowTypes: ["video"]` + a PAYLOAD tag.
- **Every other scene accepts photo OR video**, plus upload-from-library.
- **Capture spec ≠ cut spec.** Scene 5 captures ~40s but only ~22s survives. Every
  scene needs both numbers in the data model.

---

## Reusable vs. live content

Built once, reused across all videos:
- Intro bumper (user picks from a style library — must stay editable even when "done")
- End card graphic frame
- Narrator question bank
- Overlay motion graphics (queue, stats, transitions)

Built once **per ride**, reused by every creator at that ride:
- Height minimum, park hours, locker location
- Ride stats (speed, drop height, year opened)

Genuinely live every time: scenes 2, 3, 5, 6, and the narration in 4 and 7.

---

## The scene-4 branch (record now vs. record after)

Scene 4 has two valid capture paths and the shot list must fork:

- **Branch A — Record now:** captured live in the queue, marked complete, move on.
- **Branch B — Record after:** marked **deferred** (a third state, distinct from
  done/pending). Resurfaces automatically **immediately after scene 6** with a reworded
  prompt ("Earlier you were about to ride — how nervous were you actually?").

Checklist needs three states: `done` / `deferred` / `pending`.

**Key architectural point:** *when* a clip was captured and *where it appears in the
final cut* are two separate properties. A deferred scene-4 answer still slots into
timeline position 4.

Resurface trigger is anchored to scene 6 completion, not a timer — reflective narration
degrades fast if you let it drift to hours later.

---

## Assembly algorithm

### Stage A — Auto-highlight detection

Pick the best window from raw footage. **Use motion sensor data, not pixel analysis** —
a phone recording a coaster is also recording accelerometer/gyro telemetry, and the drop
is literally a spike in vertical acceleration. Orders of magnitude cheaper than optical
flow, runs on-device, critical for the 60s goal.

| Scene type | Primary signal | Fallback |
|---|---|---|
| Ride (payload) | accelerometer spike + audio peak | optical flow magnitude |
| Reaction / check-in | voice activity detection (VAD) | face-motion energy |
| Walk-in | VAD + stability (reject blur/swing) | first stable N seconds |
| Queue photos | sharpness + face presence, rank top 3 | chronological |

### Stage B — Edge trim (creator override)

Auto-detection proposes a window; creator gets handles on both edges.
**Default drag = slip edit** — window keeps its budgeted length, so moving one edge
moves the other. Preserves the 60s total automatically. Expanding past budget is allowed
but must show live impact ("+3s → other scenes tighten").

### Stage C — Narration-driven pacing

```
scene_hold = lead_in(0.4s) + VAD_speech_length + tail_pad(0.6s)
```

Music ducks across `[narration_start - 0.4s, narration_end + 0.6s]`.

### Stage D — 60-second budget solver

Elasticity classes and resolve order:

1. **Locked** (intro, end card) — reserve fixed duration first
2. **Narration-driven** (4, 7) — measure actual speech, claim what's needed
3. **Floored** (payload, min 18s) — take floor
4. **Elastic** (2, 3, 3b, 6) — divide the remainder proportionally

If elastic scenes would compress below legibility minimum (~1.5s per photo), *then*
the payload gives up seconds above its floor.

---

## Ride database

Hybrid: scrape/API public data → suggest → creator overrides → correction feeds back.

### Source tiers

| Tier | Source | Good for | Caveat |
|---|---|---|---|
| 1 | Queue-Times / ThemeParks.wiki APIs | wait times, hours, ride metadata | coverage varies — **validate before designing around** |
| 2 | Official park sites | height minimums, official hours | scraping ToS varies per operator |
| 3 | Community DBs (RCDB etc.) | ride stats, year, speed | not authoritative — never for safety fields |
| 4 | Crowdsourced corrections | everything, esp. volatile fields | self-healing once you have users |

### Field handling by cost-of-being-wrong

- **Safety-critical (height minimum):** never auto-publish. Show as suggestion with
  source citation, require explicit tap-to-confirm, flag visually if sources disagree.
- **Volatile (hours, wait times):** show staleness stamp ("as of 9:38 AM").
- **Low-stakes (speed, year opened):** auto-populate silently, pre-confirmed.

### MVP bootstrap sequence
1. Start with Queue-Times/ThemeParks.wiki for names, hours, wait context (zero scrape risk)
2. Hand-verify height minimums for a seed set (~15–20 rides at one park)
3. Turn on the override/correction loop from day one
4. Only add park-site scraping after per-operator ToS review

---

## Naming

**SceneCue** — cleared casual search. Reads as a verb ("SceneCue it"), encodes the core
mechanic (the app cues each scene), no collision with CapCut.

Rejected after search: **Director's Cut** (multiple existing apps + generic film term),
**Shot List** (Shot Lister, StudioBinder Shotlist, Shot Designer all active),
**CutScene** (CutScene AI is a live direct competitor + universal gaming term),
**SceneCut** (two active "SceneCut Ai" apps in the exact category).

⚠️ **Still needs authoritative USPTO search + App Store / Play Store handle check.**
"Shot List" survives as an in-app *feature* name, just not the brand.

---

## Platform context (why 60s)

| Platform | Max length | Engagement sweet spot |
|---|---|---|
| TikTok | 10 min in-app | 21–34s |
| Instagram Reels | 90s–3 min | 15–30s (loops reward shorter) |
| YouTube Shorts | 3 min | 30–45s |

60s sits at the edge of "narrative territory" — long enough for a story arc with
narration beats. Completion rate beats raw length everywhere; **first 3 seconds decide
whether the full 60 gets watched.** Protect the hook.

---

## Open questions (decide before/while building)

1. **Is the re-solve visible?** If scene 4's answer runs 7s instead of 4s, elastic scenes
   silently compress. Does the creator see that happen, or is it invisible?
2. **Is photo count elastic?** Partly decided. Capture accepts 2 or 3 photos. The solver keeps
   each photo at least 1.5s, so 3 photos need 4.5s. It does not yet drop a photo to buy time,
   so a squeezed queue stays at 3 photos.
3. **Do ride-data corrections publish instantly or queue for moderation?** Prototype
   implies instant. Safety-critical fields may need N independent confirmations before
   overwriting an official-site value.
4. **Which scenes force video?** Only scene 5 is locked to video today. Reactions (6) may
   deserve the same treatment.
5. **Does the 60-second goal include narration recording time, or only assembly?**
   Very different engineering problems.

---

## Suggested first tasks in Claude Code

1. Scaffold the Android project (Kotlin + CameraX, or React Native / Flutter — undecided)
2. Port the scene data model and elasticity classes from `scenecue_prototype.html`
3. Implement the budget solver as a pure function, unit-tested against the animatic's
   numbers (should output the exact 60s split in `scenecue_animatic.html`)
4. Spike accelerometer-based highlight detection on a real ride clip — this is the
   riskiest technical assumption in the whole plan
