# SelectAir — Research: an "air DJ" webcam controller for Rekordbox

*Research date: 2026-10-14 · Scope agreed with the project owner: **macOS only**, **live webcam**, **fun prototype***

---

## 1. TL;DR (the short version)

| Question | Answer |
|---|---|
| Can we write a real *plugin* for Rekordbox? | **No.** Rekordbox has no public plugin SDK or control API (none found). |
| So how do we control it? | We **pretend to be a MIDI controller**. Our app creates a *virtual MIDI port* on the Mac and Rekordbox's built-in **MIDI LEARN** maps its messages to deck functions. |
| How do we see the hands? | **Google MediaPipe Hand Landmarker** (free, runs locally, 21 points per hand, Python/JS). Apple's **Vision** framework is a native alternative. |
| Which controls are easy? | Tempo, EQs (Hi/Mid/Low), Beat Jump, Play/Cue, track browsing/loading: **all ordinary MIDI LEARN targets**. |
| Which control is hard? | **The jog wheel.** Pioneer officially blocks jog wheels for non-Pioneer controllers. There is a known workaround (impersonating a *DDJ-SX*), but it's unofficial and fragile. |
| Do I need to pay for Rekordbox? | **Yes, probably.** The Free plan blocks MIDI LEARN and external control. A paid plan (Core or higher) is needed. |
| Recommended stack for the prototype | **Python** + OpenCV (camera) + MediaPipe (hands) + `mido`/`python-rtmidi` (virtual MIDI) → Rekordbox MIDI LEARN. |

---

## 2. Background for beginners: the moving parts

Here's the whole chain from your hand to the music, in plain terms:

```
 ┌──────────┐   frames    ┌─────────────────┐  21 points/hand ┌──────────────────┐
 │  Webcam  │ ──────────▶ │ Hand tracking   │ ──────────────▶ │ Gesture engine   │
 └──────────┘  ~30-60fps  │ (MediaPipe)     │                 │ "pinch + drag up │
                          └─────────────────┘                 │  = tempo +"      │
                                                              └────────┬─────────┘
                                                                       │ MIDI messages
                                                                       ▼
 ┌──────────────┐   MIDI LEARN maps    ┌───────────────────────────────────────┐
 │  Rekordbox   │ ◀─────────────────── │ Virtual MIDI port "SelectAir"         │
 │  (decks, EQ) │   message → function │ (CoreMIDI, created by our app)        │
 └──────────────┘                      └───────────────────────────────────────┘
```

- **MIDI** is a 40-year-old, very simple language that music gear uses to talk. A message is ~3 bytes: e.g. *"control #7 on channel 1 is now at value 64"*. Every DJ controller speaks MIDI (or HID).
- **A virtual MIDI port** is a "fake cable" that exists only in software. macOS supports this natively through **CoreMIDI**. Other apps, Rekordbox included, see it as a connected device.
- **MIDI LEARN** is a Rekordbox screen. You pick a function (say *"Deck 1 EQ Low"*), click **LEARN**, wiggle your controller, and Rekordbox remembers *"this message = that function"*.
- **Hand landmarks** are 21 (x, y, z) points per hand (fingertips, knuckles, wrist) that the ML model finds in every video frame. All our "gestures" are just math on these points (distances, angles, movement over time).

**Why this design matters:** Rekordbox never knows a camera is involved. To Rekordbox we are just a MIDI controller, which is the most stable integration point it offers.

---

## 3. Findings: talking to Rekordbox

### 3.1 No plugin/SDK
I found no public Rekordbox plugin system or developer API. The only external-control paths Pioneer/AlphaTheta documents are **hardware integrations**: supported DJ players/mixers, MIDI LEARN, and DMX lighting in 7.0.7+ ([rekordbox link support](https://rekordbox.com/en/support/link/), [DMX in 7.0.7](https://www.digitaldjtips.com/dmx-compatibility-rekordbox/)).
→ **MIDI is our integration point.**

### 3.2 MIDI LEARN: what the official guide says
Source: the official *rekordbox MIDI LEARN Operation Guide* ([PDF, cdn.rekordbox.com](https://cdn.rekordbox.com/files/20200312170611/rekordbox5.3.0_midi_learn_operation_guide_EN.pdf)). I read the full text.

- Available since rekordbox 4.0.6, in **Performance mode**.
- Control **types** you can assign:
  - **Button, Pad**: on/off actions (Play, Cue, Beat Jump, Load…)
  - **Knob/Fader (0h–7Fh)** = 128 steps, or **Knob/Fader (0h–3FFFh)** = 16,384 steps (14-bit). Use **14-bit for tempo** so it moves smoothly.
  - **Rotary**: endless encoder (ideal for **browsing the track list**)
  - **Indicator**: Rekordbox sends MIDI *back* (LED feedback). Useful later for showing state on our screen.
  - **Value**: Needle Search and Velocity Sampler.
- Mappings can be **EXPORTed / IMPORTed** as files. So we can **ship a ready-made mapping file** with the app instead of making users click through LEARN.
- ⚠️ The guide says MIDI LEARN works with equipment *"connected to your computer with a USB cable"*, and that AlphaTheta **does not support configuring third-party products**. A virtual port is not a USB cable. Community tools (see 3.4) show that virtual ports *do* get picked up in practice, but **this is the #1 thing to verify on a real Mac** before writing any vision code.

### 3.3 Licensing / plan
- The official plans FAQ says MIDI LEARN is among features the **Free plan can't use**, and *"you can't control rekordbox via connected DJ equipment when using the Free plan"*. The **Core** plan *"enables you to connect DJ equipment to control PERFORMANCE mode"* ([rekordbox plans FAQ](https://rekordbox.com/en/support/faq/plans/)).
- A *Hardware Unlock* device (a Pioneer controller that ships with Rekordbox) also unlocks Performance mode ([Digital DJ Tips](https://www.digitaldjtips.com/pioneer-dj-rekordbox-6-0-rekordbox-for-ios-3-0-cloud-music-subscriptions/)).
- **Implication:** every user of SelectAir needs a **paid Rekordbox plan** (or a Hardware Unlock device plugged in). Prices change, so check the current plan page.

### 3.4 The jog wheel problem ⚠️
- Pioneer staff have said repeatedly on their forums that **jog wheels can't be MIDI-mapped on non-Pioneer controllers** ([Pioneer DJ community](https://community.pioneerdj.com/hc/en-us/community/posts/22979790097945-MIDI-Learn-Jog-wheel), [older thread](https://forums.pioneerdj.com/hc/en-us/community/posts/115000176646-Control-the-jogwheels-with-3rd-party-controller)).
- **Workaround (unofficial):** name your virtual MIDI port exactly like an official controller, e.g. **`PIONEER DDJ-SX`**, and send the exact jog messages that controller sends. Rekordbox then treats you as that device ([DJ TechTools, 2017](https://djtechtools.com/2017/05/08/hack-rekordbox-use-controllers-jogwheels/)). Rekordbox's built-in mapping CSVs live inside the app bundle (`rekordbox.app → Show Package Contents → Contents/Resources`), and you can read them to learn the expected messages.
- **RekordJog** ([GitHub, MIT license](https://github.com/timkondratev/RekordJog)) is an open-source translator doing exactly this for Rekordbox **6 and 7**: it impersonates a DDJ-SX, and jog values are centered on **64** (roughly 48–80 range). It's a good reference implementation.
- **Risks:** a future Rekordbox update may break it, it may confuse users who own a real DDJ-SX, and impersonating a device is a grey area for a public product. **For a fun prototype it's acceptable. For anything public, re-evaluate.**
- **Fallback if the hack fails:** map the "jog" gesture to **pitch-bend buttons** (temporary nudge faster/slower) or to **Beat Jump** with fine sizes. You lose scratching but keep the "nudge to beat-match" use case.

### 3.5 Control-by-control feasibility

| Control | Rekordbox MIDI type | Feasibility | Notes |
|---|---|---|---|
| Play / Cue | Button | ✅ Easy | |
| Tempo | Knob/Fader 14-bit | ✅ Easy | Use 14-bit for smoothness |
| EQ Hi / Mid / Low | Knob/Fader | ✅ Easy | Commonly cited as MIDI-learnable |
| Beat Jump (back/fwd, size) | Button | 🟡 Likely | Exists as a Rekordbox function. Confirm it shows in the LEARN menu on your version |
| Track browse | Rotary | ✅ Easy | |
| Load to deck | Button | 🟡 Likely | Same as above: confirm in the LEARN list |
| Jog wheel | (impersonation) | 🔴 Hard / unofficial | See 3.4 |

---

## 4. Findings: seeing the hands

### 4.1 Options

| | **MediaPipe Hand Landmarker** (Google) | **Apple Vision** (`VNDetectHumanHandPoseRequest`) |
|---|---|---|
| Language | Python, JS (browser), Android, iOS | Swift / Objective-C |
| Output | 21 landmarks/hand + handedness | 21 joints/hand + confidence |
| Platform | Cross-platform | Apple only |
| Learning curve | Low, lots of tutorials | Medium (Xcode, Swift) |
| Speed | Official: ~17 ms CPU / ~12 ms GPU per frame *on a Pixel 6* ([Google guide](https://ai.google.dev/mediapipe/solutions/vision/hand_landmarker)) | No public hand-pose benchmark found. Body-pose was ~40 ms on an iPhone 11 ([Apple forum](https://developer.apple.com/forums/thread/679909)) |
| Recommendation | ✅ **Prototype with this** | Consider later for a polished native Mac app |

MediaPipe details worth knowing:
- Use the **Tasks API** (`mp.tasks.vision.HandLandmarker`) in **LIVE_STREAM** mode with `detect_async`. Many tutorials use the older `mp.solutions.hands` API, which is legacy.
- In video/live mode it **tracks** hands between frames and only re-runs the slower palm detector when tracking is lost, so steady-state performance is good.
- I found **no independent Mac webcam benchmark**, so we should **measure it ourselves** on your Mac (Phase 0 below).

### 4.2 Latency budget (why this matters for DJing)
Rough estimate, all **to be measured**:

| Stage | Estimate |
|---|---|
| Webcam exposure + USB/driver | 33–50 ms (at 30 fps a frame is 33 ms old already) |
| Hand landmarks | ~15–30 ms |
| Gesture math + smoothing | 1–20 ms (smoothing adds lag) |
| MIDI to Rekordbox | < 1 ms |
| **Total** | **~50–100 ms** |

For continuous controls (EQ, tempo) 50–100 ms is fine. For **jog/scratching** it's noticeable, which is another reason the jog wheel is the "stretch goal". A 60 fps webcam helps.

---

## 5. Prior art (similar projects)

None of these is a webcam-to-Rekordbox DJ controller, so the idea looks open:

- **AirChords** ([GitHub](https://github.com/RonitSachdev/AirChords)): MediaPipe finger counting → MIDI chords. The closest code to borrow from: webcam → MediaPipe → MIDI out.
- **Hand Gesture Music Controller** ([GitHub](https://github.com/Mensorpro/Hand_Gesture_Music_Controller)): OpenCV + MediaPipe, but uses media keys, not MIDI.
- **Handmate** ([paper, Computer Music Journal](https://www.citedrive.com/en/discovery/an-accessible-browser-based-gestural-controller-for-web-audio-midi-and-open-sound-control)): academic, browser-based MediaPipe → MIDI/OSC controller. Good for gesture-design ideas.
- **RekordJog** ([GitHub](https://github.com/timkondratev/RekordJog)): not vision, but the reference for the jog-wheel trick.

---

## 6. Proposed architecture (prototype)

```
selectair/
├── camera.py      # OpenCV VideoCapture, mirror image, timestamps
├── tracker.py     # MediaPipe HandLandmarker (LIVE_STREAM), returns landmarks
├── gestures.py    # pure functions: landmarks → intents ("tempo=0.53", "beatjump_back")
├── smoothing.py   # One-Euro filter / dead-zones / hysteresis to stop jitter
├── midi_out.py    # mido + python-rtmidi, virtual port "SelectAir"
├── mapping.py     # intent → MIDI message table (one place to change mappings)
├── overlay.py     # draws the "virtual controller" over the video so users see zones
└── main.py
mappings/
└── SelectAir.rekordbox-midi   # exported MIDI LEARN file, shipped with the app
```

Libraries:
- `opencv-python`: camera + drawing the overlay
- `mediapipe`: hand landmarks
- `mido` + `python-rtmidi`: python-rtmidi supports **virtual ports on macOS via CoreMIDI** ([PyPI](https://pypi.org/project/python-rtmidi/1.5.8/))

**Principal-engineer design choices (and why):**
1. **Keep `gestures.py` pure** (no camera, no MIDI). Then we can unit-test gestures with recorded landmark data instead of waving at a webcam.
2. **Smoothing is its own module.** Raw landmarks jitter a lot. A [One-Euro filter](https://gery.casiez.net/1euro/) is the standard trade-off between jitter and lag.
3. **"Clutch" gestures.** A control only changes while you *pinch* (thumb + index touching). Without this, every accidental hand movement changes the EQ. This is the most important UX decision.
4. **Draw the controller on screen.** Air controls have no physical feedback, so the overlay *is* the controller. The left half of the screen is Deck 1 and the right half Deck 2. Zones are drawn for EQ, tempo and pads.

### Gesture ideas (to validate in testing)

| Control | Gesture idea |
|---|---|
| Tempo | Pinch inside the tempo zone, move hand up/down (like a fader) |
| EQ Hi/Mid/Low | Pinch inside the EQ column, twist the wrist (knob) **or** move vertically |
| Beat Jump | Hand in pad zone: quick "tap" (fingertip moves toward camera / z-change) on left/right pad |
| Play/Cue | Tap pad zone |
| Browse | Open palm, rotate the hand like a dial (Rotary) |
| Load | Fist in "Load to deck 1/2" zone |
| Jog (stretch) | Index finger draws circles → angle delta per frame → relative jog value around 64 |

---

## 7. Risks & unknowns

| # | Risk | Severity | How we find out |
|---|---|---|---|
| R1 | Rekordbox doesn't list a **virtual** MIDI port in MIDI LEARN (guide mentions USB) | 🔴 Blocker | Phase 0 test, ~30 min |
| R2 | User needs a **paid Rekordbox plan** | 🟡 | Confirm with your own account/plan |
| R3 | Jog wheel only via **DDJ-SX impersonation** (unofficial, may break) | 🟡 | Phase 3 experiment |
| R4 | Beat Jump / Load not in LEARN menu on current version | 🟡 | Phase 0, look at the ADD menu |
| R5 | Latency / jitter makes it feel bad | 🟡 | Phase 0 measurement |
| R6 | Lighting / background affects tracking | 🟢 | Testing; on-screen hint "more light" when confidence is low |
| R7 | macOS camera permission prompts (Terminal/Python needs Camera access) | 🟢 | First run |

---

## 8. Suggested roadmap

**Phase 0: prove the risky bits (½–1 day). No gestures yet.**
1. Small Python script opens virtual port `SelectAir` and sends a CC sweep. Does Rekordbox MIDI LEARN see it and learn it? (**R1**)
2. Look through the ADD menu for Tempo, EQ, Beat Jump, Browse, Load. Write down exact names. (**R4**)
3. MediaPipe webcam demo on your Mac: print FPS and per-frame ms. (**R5**)

**Phase 1: one deck, three controls.** Tempo + EQ Low + Play via pinch gestures, with overlay and smoothing.

**Phase 2: full minimal controller.** Two decks, all EQs, Beat Jump, Browse/Load. Export the MIDI LEARN mapping and commit it to `mappings/`.

**Phase 3: jog wheel experiment.** Try DDJ-SX impersonation (study RekordJog). Fall back to pitch-bend if it's unreliable.

---

## 9. Open questions for you (I won't assume these)

1. **Which Rekordbox version and plan do you have?** That decides whether MIDI LEARN is available at all.
2. **Do you own a Pioneer controller?** Relevant for Hardware Unlock, and for conflicts if it's a DDJ-SX.
3. **Language preference:** Python (fastest to prototype, recommended) or Swift (native Mac app, Apple Vision)?
4. **Is the jog wheel a must-have** for the prototype, or OK as a later stretch goal?
5. **One hand per deck, or both hands on one deck?** This changes the whole gesture layout.

---

## Sources
- [rekordbox MIDI LEARN Operation Guide (official PDF)](https://cdn.rekordbox.com/files/20200312170611/rekordbox5.3.0_midi_learn_operation_guide_EN.pdf)
- [rekordbox plans FAQ (official)](https://rekordbox.com/en/support/faq/plans/)
- [rekordbox: DJ players/turntables support (official)](https://rekordbox.com/en/support/link/)
- [Digital DJ Tips: Rekordbox 6 subscriptions & hardware unlock](https://www.digitaldjtips.com/pioneer-dj-rekordbox-6-0-rekordbox-for-ios-3-0-cloud-music-subscriptions/)
- [Digital DJ Tips: Third-party DMX in Rekordbox 7.0.7](https://www.digitaldjtips.com/dmx-compatibility-rekordbox/)
- [DJ TechTools: Hack Rekordbox to use any controller's jogwheels](https://djtechtools.com/2017/05/08/hack-rekordbox-use-controllers-jogwheels/)
- [Pioneer DJ community: MIDI Learn jog wheel](https://community.pioneerdj.com/hc/en-us/community/posts/22979790097945-MIDI-Learn-Jog-wheel)
- [Pioneer DJ forums: Control the jogwheels with 3rd party controller](https://forums.pioneerdj.com/hc/en-us/community/posts/115000176646-Control-the-jogwheels-with-3rd-party-controller)
- [RekordJog (GitHub)](https://github.com/timkondratev/RekordJog)
- [djworx: rekordbox DJ 4.0.6 MIDI learn](https://djworx.com/rekordbox-dj-4-0-6-now-hardware-mapping-midi-learn/)
- [Google MediaPipe Hand Landmarker guide](https://ai.google.dev/mediapipe/solutions/vision/hand_landmarker)
- [Apple Developer Forums: Vision pose performance](https://developer.apple.com/forums/thread/679909)
- [python-rtmidi on PyPI](https://pypi.org/project/python-rtmidi/1.5.8/)
- [Apple: Audio MIDI Setup / IAC Driver](https://support.apple.com/guide/audio-midi-setup/ams1013/mac)
- [AirChords (GitHub)](https://github.com/RonitSachdev/AirChords)
- [Hand Gesture Music Controller (GitHub)](https://github.com/Mensorpro/Hand_Gesture_Music_Controller)
- [Handmate (Computer Music Journal)](https://www.citedrive.com/en/discovery/an-accessible-browser-based-gestural-controller-for-web-audio-midi-and-open-sound-control)
- [One-Euro filter](https://gery.casiez.net/1euro/)
