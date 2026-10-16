# Voice Workout Timer — Implementation Spec v26

> Implementation contract only. This is the final-version document: there is no changelog and no decisions file; measured facts and owner decisions are recorded in §20.

# 0. Read This First

**Verification rule (every milestone):** before writing code that uses an external dependency, inspect the actually resolvable package version and its official examples. Never invent an API, model file, keyword format, or config format. If something cannot be verified, STOP and report the blocker.

Build a small Flutter Android app: a hands-free workout timer (voice Go/Stop within a set; between sets requires manual press or `Listen for Go` re-arm). Follow this document literally. Do not replace technical components without recording the reason in §20.

- **Coding AI:** code, tests, project setup, Voice Test screen, recorded facts (versions, hashes, test output).
- **Human:** live-device voice tests, threshold tuning, decisions, listening to audio assets, fixing real-world faults.
- The AI may propose a decision but never records one as made. The AI never claims a real-device test passed unless it was run and recorded.

**Who the app is for, in order:** (1) the owner, (2) friends, (3) the store if it succeeds. Test levels in §14 match this order. The design goal is *simple and debuggable*: when something breaks, the owner hears an alert, the workout continues, manual controls work, and the owner finds the cause later from the local diagnostics.

**Precedence when two passages disagree.** (1) A rule marked "single definition" or "single rule" wins over any passage that merely repeats or refers to it. (2) Otherwise the owning section wins: §6 for audio and the microphone gate, §8 and its tables for failures, alerts and fault codes, §11 for the `START VOICE SESSION` press and its refusals, §16 for clock and scheduling. (3) The §17 tests describe the rules and never override them. (4) If this still does not settle it, STOP and report the exact passages; never pick one silently and never edit this document to resolve it (the AI may only propose, §0 above).

**Core invariants.** *Audio is feedback. Clock is truth. Speech is command input.* Responsibilities stay separate: Speech detects a command; Audio gives feedback; Clock defines elapsed time; Scheduler evaluates due events; TimerController owns workout behavior; UI displays state and fallback controls. Only real Android hardware can prove voice accuracy, latency, false-positive behavior, battery use, and audio routing — and only for the devices actually tested. Never claim a result that was not measured and recorded (wording rules: Appendix B).

**Release map.** Each release is a complete app; improvements continue in every release.

| Version | Audience | Test level (§14, §18) | Diagnostics | What it must deliver |
|---|---|---|---|---|
| 0.0.1 | the owner | Level 0 | `DIAG_DETAIL=true`: detailed faults, Fault panel, `Check voice`, local `diagnostics_log.txt` | the timer and voice features of §1–§13 working on the reference device, with every real fault captured in enough detail to be fixed correctly |
| 0.0.2 (and any later friends' build below 0.9.0) | friends | Level 1 | `DIAG_DETAIL=false`: no log file | the 0.0.1 faults fixed; the Level 1 items of Appendix B (disclosure, claims proportional to the devices tested) |
| 0.9.0 | store (public) | Level 2 | `DIAG_DETAIL=false` | no known unresolved faults; the Level 2 requirements of Appendix B (license, offline verification, release checks) |
| after 0.9.0 | public | — | — | improvements, one release at a time (0.9.x); `1.0.0` is reserved for a later owner decision |

**Extensions.** Sessions and rounds (Extension S) and saved Count-up results (Extension P) are specified in §21. They are owner-activated, not assigned to a version here, and not part of 0.0.1.

**Not required for 0.0.1:** Appendix A (activated only if Sherpa fails its gates), the Level 1 and Level 2 work of Appendix B and the release-only items of §15 (offline release verification, claims), extra languages (the §12 extension points stay documented but unused), `After a set: Listen for Go` (no setting, no re-arm path, no `rearmDelay`; after `REST` the app always goes to `IDLE` with the microphone closed), and any non-zero `startDelay` (the constant stays 0 ms, the `PREPARING` state stays as specified with zero length, and no non-zero value is built or tested). They stay in this document for later releases; do not implement them early. A rule, constant, or test whose subject is one of these deferred items is skipped in 0.0.1; a test that only mentions one of them among other behavior runs with that mention ignored. The Gate 0A license check of the chosen model still applies from the start. Also not required for 0.0.1: Extensions S and P of §21; they stay in this document and must not be implemented until the owner records their activation in §20.

**Reading map.** Read the whole file once for context, then implement only what the map assigns to the current milestone. Never implement a later milestone's items early.

| Milestone | Read and implement | Not yet |
|---|---|---|
| A-pre | §6 and the A-pre paragraph in §16 (Milestone A2 items 1–2 only) | Any Timer Core code; §7–§13 |
| A | §0–§13 (except §8 *Voice health and Check voice*); §14 *Interface*; §16 (Clock, Riverpod, Structure, Milestone A); §17 groups: Timer and modes, Identities and lifecycle, Audio, Voice failure, Fault codes, Result and cancel, Language/UI/assets, Fake behaviors | §14 gates/levels, §15, Check voice, Appendices |
| A2 | §6 and the A2 protocol in §16 | — |
| B | §14 (Gates, Levels, Targets), §16 Voice Test, §8 *Voice health and Check voice*, §17 *Voice health* tests | Sherpa code |
| B0a, B2 | §15, §14 *Engine of record* | — |
| B0b, C, C2 | §14 (human live tests) | — |
| D, E | §12, §13, §18 | — |
| Before friends / store | Appendix B (Appendix A only if the owner activates it) | — |
| Extensions S, P (only when activated in §20) | §21 | — |

The Fault panel's `[ Check voice ]` button and the `Voice:` status line are added in Milestone B; Milestone A builds the panel with `[ Copy details ]` and `[ Dismiss ]` only.

**The coding AI must not:** build rep counting or any Phase 2 feature other than Count-up (the only exceptions are Extensions S and P of §21, and only after the owner records their activation in §20); add iOS, Bluetooth routing, TTS, cloud speech, a backend, analytics, Crashlytics, camera/pose detection, AI coach, workout plans, calorie counting, social features, or arbitrary custom voice training; ship Arabic/French content in v1 (the architecture is required, only English ships); hard-code user-visible strings, spoken words, or keywords outside the language layer; invent a Sherpa keyword format or API; ship more than one KWS engine; treat a README claim, star rating, or another project's values as evidence; assume model accuracy without real-device testing; add retry, recovery, replacement-instance, or episode-tracking machinery beyond §8; proceed to the next milestone without explicit instruction.

---

# 1. Product Goal

> The workout timer should start when the athlete is actually ready, not when they first touch the Start button.

```text
Choose mode (Countdown 20/30/45/60 s, or Count-up)
  → take workout position
  → say "Go"
  → START DETECTED (visual) + start beep, and the workout clock starts with it (no spoken countdown)
  → Countdown: halfway tone at 50%, finish beep
  → Count-up: elapsed counts up; say "Stop" (or tap STOP); finish beep
  → the result stays on screen; the app returns to IDLE with the microphone closed
  → the athlete rests as long as needed, then presses START VOICE SESSION (or START MANUALLY)
    (deferred, not in 0.0.1: the optional setting "After a set: Listen for Go" re-arms listening automatically)
```

Two first-release differentiators: **Countdown + Count-up**, and a **multi-language architecture** (English only shipped). Voice Stop in Count-up is part of the full MVP; manual Stop is the emergency fallback and never counts as meeting the voice-Stop requirement. **Hands-free operation applies within a set (voice Go to start, voice Stop to end); between sets the default `Wait for me` requires a manual press or the optional `Listen for Go` re-arm.**

No optional work (E2E, Bluetooth, other languages) may delay or destabilize these two differentiators.

---

# 2. Scope

- Android only. `minSdk 26`. API 26 is verified on an emulator only; voice accuracy is claimed only for physically tested devices (Appendix B).
- **Quiet environment only**: no music, no TV, no other speakers; phone 1–2 m away. Music robustness is deferred and never claimed.
- **English only shipped.** Nothing language-specific is hard-coded in Timer logic, widgets, or engine code (§12).
- **Accessibility is out of scope for v1 (known limitation):** no TalkBack semantics audit and no large-font layout guarantee. The UI uses large high-contrast digits (§13), but v1 does not promise usability with screen readers or system font scaling.
- **Offline:** everything works without Internet from first launch. On-device streaming Keyword Spotting (KWS), not cloud speech, not general speech-to-text. No backend, no analytics SDK, no crash-reporting SDK, no TTS at runtime, no upload of audio.
- Do not call a streaming KeywordSpotter an `OfflineRecognizer`.
- **Product constraints:** free, no ads, no purchases or donations, no Internet use. Any change to this requires re-reviewing every model's license first (§15 Model license record).
- **Nothing is stored.** The app writes nothing to the device: no workout record, no result, no settings, no log file, no first-run flag. (Static model files that the engine integration may copy once to the app's private storage are not user data and not a record of use; §13, §15.) **The single exception is `diagnostics_log.txt`, written only in `DIAG_DETAIL` builds (version 0.0.1, the owner's build; §8) and never in 0.0.2 or later (a non-diagnostic build deletes a leftover file at startup, §8); it holds no audio and no workout result.** The result and the settings live in memory only and are gone when the app closes. Every launch starts from the defaults of §13. **Extension P (§21), if the owner activates it, is the only planned exception: it stores only the Count-up results the athlete explicitly saves, and §21.5 then replaces this paragraph and the matching privacy text. Until it is activated, this paragraph holds without exception.**
- **Known limitation (0.0.1): the screen must stay on and the app must stay in the foreground during a session.** Locking the phone, pressing Home, switching apps, or answering a call (which sends the app to the background) cancels the session; a ringing or declined call that leaves the app in the foreground does **not** cancel (§13 Lifecycle): the clock continues, voice is off (`MIC-206`), and manual STOP works. Background and screen-off operation are not supported in this version (§13 Lifecycle).

---

# 3. Modes and Commands

Terminology: "Fixed Timer" means Countdown.

## Countdown
Durations 20 / 30 / 45 / 60 s, default 30. No custom duration. Halfway threshold computed exactly at 50% of the duration in milliseconds; the tone is emitted at the first `tick()` at or after it, so it may lag by up to one scheduler interval (§16). Ends at 100%.

## Count-up
Elapsed time (mm:ss) counts upward from `startTime` (T+`startDelay`, §5) until Stop. No halfway, no periodic cues.

- **Stop reasons:** voice `Stop`, manual STOP button, or safety cap `countUpMaxDuration` (default 30 min, injectable).
- **Result:** the instant a stop is accepted, `finalElapsed = Clock.now() - startTime` (`startTime` per §16: logical T+`startDelay`) is captured and frozen synchronously, shown as the badge `STOPPED` with the line `TIME mm:ss` (with `~` for an approximate voice-Stop time, §13 Result rule; no latency correction — see the bias note below), and kept visible through `REST` and `IDLE` until a newer session enters `RUNNING`, the athlete leaves the screen, or the app goes to the background (§9 Result lifetime). The result is never stored.
- **Known systematic bias (deliberate):** `finalElapsed` is captured at the instant the stop is accepted, so it includes the utterance duration and the recognition latency (target p95 ≤ 1.5 s, §14); a voice-Stop result is therefore systematically later than a manual STOP for the same moment. The Clock stays the truth: no correction is applied. The keep-vs-correct decision is recorded explicitly in §20 by the owner, after the recognition-latency measurements of Gate 0H on the reference device (§14) and the B0a timestamp finding (§15, which records only whether results expose keyword timestamps, not latency).
- **Stop is final** (no Undo). Continuing requires a new Start; the counter restarts from 0.
- **Distinct tones:** start and finish tones differ by pattern, not only pitch.
- **Manual STOP** is always visible. **CANCEL** is a distinct secondary action: it aborts without recording a completed result (§9).

## Voice commands

- `VoiceCommand.start` → English phrase `Go`. Starts preparation in both modes.
- `VoiceCommand.stop` → English phrase `Stop`. **Production use only while `RUNNING` in Count-up**, only when `voiceStopCapability` is ON, and only while `voiceStopAvailable` is true (§10). Ignored everywhere else, always ignored in Countdown (Countdown uses manual Cancel).
- **`voiceStopCapability`** is a build-time flag, not a user setting: ON for the owner's builds after B0b, for friends' builds only after Gate 0H Level 1, for store builds only after Level 2 (§14). When OFF, the Count-up screen shows only the manual STOP button and the hint says `Go` only. Its default in code is OFF; the owner turns it on with `--dart-define=VOICE_STOP=true` for the builds described here. In code it is an injectable boolean (default OFF) so tests can cover both branches without a rebuild; the `--dart-define` only sets its value for release/test builds. **`voiceStopAvailable`** is the per-session runtime health flag (§10); it can only go from true to false during a session. When `voiceStopCapability` is OFF, `voiceStopAvailable` is false for every session from session start, and §10 (b) ("STOP being or becoming unavailable") never fires an alert for that reason alone.
- `VoiceCommand.unknown` may be emitted by diagnostic engines; production ignores it.
- The phrases `Go` and `Stop` are fixed. No alternative phrase is tested or adopted without an approved spec revision and a full rerun of the gates (**except the conditional alternative-phrase branch in §14 Fallback chain, for STOP or START, which requires a recorded human decision and a full gate rerun; if adopted, the hint strings and the `LanguagePack` command phrases change as data only**). Timer logic never sees literal words, only `VoiceCommand`.
- START and STOP are never listened for at the same time. A `START` while Count-up `RUNNING` is ignored. A command executes at most once per recognition event.
- No custom voice training in the first release. Keep the architecture ready: command ID → phrase via `LanguagePack`.

---

# 4. Start Sequence

Mandatory order for a recognized voice Start:

```text
1. Engine emits VoiceCommand.start (accept it once)
2. speechEpoch++ (§7); cancel the listening subscriptions; disable command handling
3. Request stopListening(); wait at most stopListeningTimeout (300 ms)
4. Enter PREPARING (the single transition that defines T+0, §5)
```

The wait in step 3 is bounded. If the stop has not finished, log it, keep ignoring speech events, issue no other engine call while that exact Future is pending, and continue (§8 covers a stop that later fails or never returns). Nothing in this sequence waits for audio.

**Manual Start** is accepted synchronously and never blocked by the engine:
- From `LISTENING_FOR_START`: `speechEpoch++`, cancel the listening timeout, request one bounded `stopListening()` (unless an engine call is already in flight — then do not issue a second one and do not wait), enter `PREPARING`.
- If a `startListening()` call is still in flight, enter `PREPARING` immediately; when that call settles successfully, issue one bounded `stopListening()` as the next engine call; if it fails or times out, apply §8.

Manual and voice Start call the same preparation routine. Manual Start is not offered in `REST`; a second Start in `PREPARING` or `RUNNING` is ignored.

**Microphone policy.** The app listens only: (a) in `LISTENING_FOR_START` for `Go`, (b) in Count-up `RUNNING` for `Stop` while `voiceStopAvailable`, and (c) in the `Check voice` diagnostic (`DIAG_DETAIL` builds only, §8) while a visible listening indicator is shown. It never listens during `PREPARING`, `REST`, or Countdown `RUNNING`.

**Honest limit.** After an accepted Start or a cancel, the capture may stay open for a short time while `stopListening()` finishes (normally under 300 ms). `stopListeningHardTimeout` (2000 ms) is the deadline at which a stop is **declared failed**, not a guarantee that the capture has closed: a stop that fails or never returns leaves `captureMayBeOpen == true` until a later stop succeeds or the process restarts (§8). The timer does not wait for it, speech events are ignored (`speechEpoch`), and no audio is stored. "The app does not listen" therefore means "the app ignores the microphone", not "the capture is already closed". The listening label disappears when commands are disabled even if the capture is still closing; `DIAG_DETAIL` builds log `capture closing` and show it as a diagnostic line (never as a listening label) until the stop completes, and a stop that fails is handled by §8 (`ENG-106`).

---

# 5. Timeline and Audio Timing

```text
T+0 ms                        start_confirm beep (Accepted Start)
T+startDelay (default 0 ms)   workout clock starts (RUNNING)
```

T+0 is the single transition into `PREPARING`. **`startTime` is the logical workout start: `startTime = T+0 + startDelay`**, fixed when `PREPARING` begins, never the time at which `tick()` happens to notice the transition (§16 Clock). `startDelay` is an injectable constant, **default 0 ms**: there is no spoken countdown, the start beep is the signal and the clock starts with it. With 0, `PREPARING` is entered and left in the same synchronous step and has no observable duration; `start_confirm` is requested in that step and plays while `RUNNING`. A non-zero `startDelay` is an owner tuning option (decided after Gate 0F and recorded in §20) and the way tests exercise `PREPARING`. The scheduler and `Clock` are the source of truth; **never wait for an audio completion callback to advance the timeline.**

| Event | Audio |
|---|---|
| T+0 | `start_confirm` |
| Countdown 50% (once) | `halfway` (only if the Halfway cue setting is ON) |
| Countdown 100% → COMPLETED | `finish_beep` requested (plays during `REST`) |
| Count-up stop → COMPLETED | `finish_beep` requested (plays during `REST`) |

All cues are language-neutral tones with distinct patterns (§6). The Halfway tone is audio only; there is no visual `HALFWAY` marker. **Clock versus audible start.** The workout clock starts with the *request* for `start_confirm`, not with its audible start; the request-to-audible latency (p50/p95, §16 A-pre and A2) is the known offset between the clock and the beep the athlete hears. It is recorded as a fact in §20, and whether that offset is acceptable is an owner decision `[owner]` recorded there; the text "the clock starts with the beep" never promises exact audio-visual synchrony. In 0.0.1 `startDelay` stays 0 ms.

A visual `START DETECTED` badge is shown from T+0 for `startDetectedDisplay` (1000 ms, injectable) on the current screen, even if audio fails; it never changes the clock.

---

# 6. Audio

## Assets

```text
assets/audio/common/   start_confirm.wav  halfway.wav  finish_beep.wav  voice_unavailable.wav
```

- All cues are **language-neutral tones** in `assets/audio/common/`; there are no spoken files and no per-language audio folder (a later language needs no audio work, §12). Patterns differ, not only pitch: `start_confirm` one short high beep; `halfway` two short, lower, softer beeps; `finish_beep` one long beep or three falling beeps; `voice_unavailable` a clearly different low falling tone.
- List nested asset directories explicitly in `pubspec.yaml`.
- Format: WAV PCM, mono, 16-bit, 22050 Hz is fine for tones (all four cues are WAV); trim leading/trailing silence; preload short sounds.
- `start_confirm.wav`, `halfway.wav` and `finish_beep.wav` are generated by a deterministic script (committed with its parameters; nothing is generated at runtime). `voice_unavailable.wav` is also a tone: generated, or human-supplied (own recording or any offline tool; this does not violate "no TTS", which applies at runtime only). Mark any placeholder clearly. A missing file logs an error and is skipped; it never crashes the app.
- `voice_unavailable` is a short tone (`.wav`). Played only by `AudioFeedbackEngine`; never generated at runtime, never routed through `SpeechEngine`. The owner approves it after listening at 1–2 m on the reference device and records source and approval in §20. A test asserts it exists, is declared in `pubspec.yaml`, and differs by content hash from the other three cues.
- `finished.wav` does not exist in the MVP.

## Duration limits

| File | Max duration |
|---|---|
| `start_confirm` | 900 ms |
| `finish_beep` | 1500 ms |
| `voice_unavailable.*` | 1200 ms |
| `halfway` | 900 ms |

A test fails if a limit is exceeded: by reading the WAV header. Limits are defaults; the human may change them in §20.

## Playback rules

Use `audioplayers`. Prefer transient ducking if the verified API supports it (best effort only). Verify behavior from the installed package, never from memory.

**One owner of app audio** — `AudioFeedbackEngine` is the single owner of every app sound: it alone decides play requests, keeps `quietAt`, and cancels. It is **not** a serial queue: it never makes a sound wait for another, and overlapping sounds are permitted (rule 1 and §8 rule 3). Ownership (one place decides) and playback (each request may use its own player) are deliberately separate. **Overlap contract:** two sounds that overlap must both be audible, and neither a later request nor an earlier `stopAll()` may cut a sound requested after it (rule 6). The implementation uses one player per play (or an equivalent generation guard) unless Milestone A-pre proves another strategy meets this contract; if the installed `audioplayers` cannot meet it, STOP and report (§19):

1. **Mandatory workout cues** (`start_confirm`, `halfway`, `finish_beep`) are **always requested at their due time**, immediately: no preceding `stopAll()`, no wait on any earlier sound, and never dropped because of playback state. (A cue whose asset is missing or whose play call fails is skipped and logged as `AUD-301`/`AUD-302`: that is a playback fault, not a scheduling decision.) **Overlap is possible but rare:** cues are seconds apart (`start_confirm` at T+0, `halfway` at least about 10 s later since the shortest Countdown is 20 s, `finish_beep` at the end; in Count-up a very early manual STOP can bring `finish_beep` close to `start_confirm`). The guarantee is the cue's timing, not silence of a previous sound. Real devices promise only "cue on time".
2. The scheduler and workout clock never await audio.
3. **App-audio quiet estimate (the only audio-state model).** The engine keeps one instant, `quietAt` (Clock time; `null` until the first play request). When a play's player API call is actually invoked at `requestedAt` (not when it was scheduled or queued): `quietAt = max(quietAt, requestedAt + hardMax(asset) + audioTail)`. `hardMax(asset)` is the hard maximum for that asset in the table above (an asset longer than its hard maximum fails the asset test); the gate never uses a nominal duration. `audioTail` must cover, with a conservative margin, the p95 request-to-audible-start latency (§16 A2 item 2), the audio-output/scheduling latency, and a safety margin; its value and derivation are recorded in §20. The provisional 500 ms is a placeholder: it cannot be approved without those measurements on a physical device. Because `quietAt` takes the maximum, a later short sound never shortens the estimate of an earlier longer one. There is no per-sound bookkeeping: no `playId`, no outstanding set, no completion callback, no stop result. The estimate is never proof: the app never claims that a sound ended or that silence was proven.
4. **App audio that may still be audible (`audioGateSatisfied == false`) only blocks opening the microphone, and only through `microphoneOpenGateSatisfied` (rule 4a)** — it never blocks a cue.
4a. **Microphone open gate (single definition, two owners).** `AudioFeedbackEngine` owns only `audioGateSatisfied` (audio state and its end estimate, below). `SpeechController` is the **sole owner** of the composite `microphoneOpenGateSatisfied = audioGateSatisfied && !alertPending && engineHealthy && captureMayBeOpen == false`, evaluated in one function that the Count-up STOP listener (§10), the START re-arm (§11), a press of `START VOICE SESSION`, the `Check voice` microphone step (§8), and the Voice Test screen listener (§16) all call. The single function is `canOpenMicrophone(path)` with `path` in `stopListener | rearm | button | check | voicetest`, returning exactly one of: `blocked`; `mayStartEngineRetry` (button and check paths only: the engine is broken but retry-eligible per §8, `captureMayBeOpen == false`, `alertPending == false`, no engine call pending — it permits only the `dispose()`/`initialize()` retry and **never** `startListening()`); or `mayOpen` (all of `engineHealthy`, `audioGateSatisfied`, `!alertPending`, `captureMayBeOpen == false`, which is exactly `microphoneOpenGateSatisfied`). `startListening()` is called only on `mayOpen`, evaluated again after any retry has completed (rule 4b). Each path then adds only its own prerequisites (`RUNNING` and `settleAnchor + stopListenSettleDelay` for STOP, `rearmDelay` for re-arm, `IDLE` for the button, `IDLE` for check and voicetest). The `check` and `voicetest` paths are defined here from Milestone A, but their only callers (the `Check voice` flow and the Voice Test screen) are Milestone B features (§16). No path reads `quietAt`, `alertPending`, or engine health on its own to decide an opening, and the audio component never reads or guesses `SpeechController` state. A pending alert has not yet become a play and must never be bypassed (§8 rule 8). `audioGateSatisfied` is true exactly when `playRequestPending` is false and (`quietAt` is `null` or `Clock.now() >= quietAt`), where `quietAt` is the quiet estimate of rule 3. **`audioGateOpenedAt` is the instant `audioGateSatisfied` last became true**: `max(quietAt, the moment playRequestPending last became false)`; it is `null` only before the first play request. The gate is an estimate, never proof: no log, UI, or diagnostic text says that silence was proven or that a sound ended. **No play yet:** if no play has been requested in this process, `audioGateSatisfied` is true immediately (nothing app-made can be audible), `audioGateOpenedAt` is `null`, and `null` counts as already elapsed for `rearmDelay` and `stopListenSettleDelay` (§10, §11). Requesting a new play closes `audioGateSatisfied` from the moment the request is decided, not only once `quietAt` is updated. **`playRequestPending`** (engine-owned, counted not flagged, so overlapping requests are covered): true from the **synchronous** moment a play request is decided until its player API call has been invoked and `quietAt` updated, or the request is abandoned (asset missing, `AUD-301`; or `playInvokeTimeout` below). **Bounded:** if the player API call has not been invoked within `playInvokeTimeout` (injectable, Clock-based) of the decision, the request is **abandoned**: the pending count is decremented, `AUD-302` is recorded with the detail `invoke timeout` and the elapsed time, `quietAt = max(quietAt, abandonedAt + hardMax(asset) + audioTail)` (conservative, as rule 8b: a start cannot be ruled out), the cue is skipped and nothing waits for it. **Generation guard:** every play request carries a generation token; abandonment invalidates it, and the engine checks the token in the same synchronous step that calls the player API (no `await` between the check and the call), so an abandoned request never calls the player API later: no late sound, and no later change to `quietAt` from it. A request whose player call was already invoked is not pending and is never abandoned. No state can keep `playRequestPending` true beyond `playInvokeTimeout`. While it is true, `audioGateSatisfied` is false; `audioGateOpenedAt` is never earlier than the moment it last became false.
4b. **Fresh re-evaluation (no stale open decisions).** No microphone-opening path stores an open decision across an asynchronous wait. After **every** asynchronous wait on the path to `startListening()` (`initialize()`, `dispose()`, `stopListening()`, a delay timer, the grace wait), the path re-evaluates `canOpenMicrophone(path)` fresh with all path prerequisites from the current state; any result other than `mayOpen` aborts that open attempt without listening, and the attempt re-evaluates from the current state. The final evaluation and the `startListening()` call happen in one synchronous step with no `await` between them, so no state change can slip in between. A timer that fires late, or a stale event delivered late, is re-checked at the moment of the actual open and never executes an earlier decision. A change caused by the path's own operation — its own completed engine retry (button path, §8) — is not special: after that operation ends the path re-evaluates **every** condition once from the current state (after a successful retry with the now-healthy engine, via `canOpenMicrophone(button)` → `mayOpen`); it never loops and opens nothing if the retry failed. If the gate is still closed the press follows the §11 grace-wait and refusal rules (single definition). A play requested meanwhile closes the gate and is seen by the fresh re-evaluation.
5. **Play methods and REST.** Each `play…()` method completes when its player API call has been invoked (or the request was abandoned or failed), never when the sound ends. No completion callback, completion fallback, or stop result is used for any decision. `REST` after a set ends at the finish beep's quiet estimate (§11).
6. **`stopAll()` is cancellation only.** It is called on explicit Cancel, lifecycle cancel, and an unrecoverable Timer/core error (§10), nowhere else: not by a cue, an alert, a press, a re-arm, or `REST`. It returns no result, nothing depends on it, and it does not change `quietAt` (a stopped sound is not proof of silence, so the gate stays conservative). It must not cut a sound requested after it: the engine stops only the players that exist when it is called (one player per play, or a generation guard). Milestone A-pre verifies this on the installed `audioplayers`; if it cannot be achieved, the AI reports and the human decides (never invent a workaround).
7. **Test support.** `FakeAudioFeedbackEngine` takes the injected `Clock` and `audioTail`, models rule 3 exactly, and has a strict serial mode and an overlap mode (§17).
8. **Play failure.** (a) Asset missing → it was never requested: no `quietAt` change, record `AUD-301`. (b) The play call fails → record `AUD-302`; because a start cannot be ruled out, `quietAt` is still extended (conservative), the cue is skipped, and nothing waits for it.
9. **Audio engine initialization.** `initialize()` is called once at startup and never opens the microphone or blocks the Timer or any UI. If it fails, or does not complete within `audioInitTimeout` (injectable, provisional 3000 ms), record **one** fault `AUD-303` (stage `playback`; detail: the timeout and elapsed time, or the raw message), mark the audio engine `unavailable` for the rest of the process, and make **no** further attempt (no recovery machinery, §0; restarting the app tries again). While `unavailable`: every play request is skipped silently (no per-cue fault, no `quietAt` change, no pending state), so `audioGateSatisfied` stays true; the Timer, `REST` (which ends at once, as with a missing asset), manual controls and the visual `START DETECTED` / `STOPPED` states are unaffected; the voice-unavailable alert cannot play (it is audio), so a voice failure shows `Voice unavailable` and the Fault panel only. A late successful `initialize()` after the timeout is ignored. A play request decided before `initialize()` completes follows rule 4a (its player call waits for the initialization and is abandoned at `playInvokeTimeout`).

```dart
abstract interface class AudioFeedbackEngine {
  Future<void> initialize();
  /// Each play method completes when its player API call has been invoked
  /// (or the request was abandoned or failed), never when the sound ends (rule 5).
  Future<void> playStartConfirm();
  Future<void> playHalfway();
  Future<void> playFinishBeep();
  Future<void> playVoiceUnavailable();
  /// Cancellation only (rule 6). No result; no decision depends on it.
  Future<void> stopAll();
  /// Quiet estimate (rule 3): the instant after which app audio is no longer expected
  /// to be audible; null before any play request. An estimate, never proof.
  Duration? get quietAt;
  /// App-audio-only gate (rule 4a). Knows nothing about alerts or speech-engine health;
  /// the composite microphone gate is owned by SpeechController, not by this interface.
  bool get audioGateSatisfied;
  Stream<bool> get audioGateSatisfiedChanges;
  /// Moment audioGateSatisfied last became true (rule 4a); null before any play has been requested.
  Duration? get audioGateOpenedAt;
  /// True from the synchronous decision of a play request until its player call has been
  /// invoked and quietAt updated (rule 4a).
  bool get playRequestPending;
}
```

Audio is always on. The only audio setting is the Halfway cue (§13). All cues are language-neutral tones.

Feedback uses the media/playback channel. If the media volume is zero and a verified API can read it, show a visual notice; never claim audio was delivered. The visual `START DETECTED` and `STOPPED` states are the guaranteed feedback.

**Known limit of the alert:** if media volume is zero or output is routed elsewhere (e.g. Bluetooth), the voice-unavailable alert can be inaudible. Milestone A2 measures this; record the result in §20. Do not rely on the alert alone during early testing.

**Constants (injectable):** `audioTail` 500 ms (provisional placeholder; derived from the A2 item 2 measurements on a physical device, §16), `playInvokeTimeout` 1000 ms (provisional; §10 timing constants), `rearmDelay` 500 ms (provisional; measured value set at Gate 0F, §14). The gate and `REST` both use `hardMax(asset) + audioTail` from rule 3; there are no completion fallbacks.

No haptics or vibration in the MVP.

---

# 7. State Machine and Identities

States (exactly one model; events are separate):

```text
IDLE  LISTENING_FOR_START  PREPARING  RUNNING  REST  COMPLETED  CANCELLED  ERROR
```

`STOPPED` is a UI label, not a state.

There is **no** audio-completion event: playback completion never drives a state change (§6 rule 5). Events: `StartCommand`, `ManualStartCommand`, `CancelCommand`, `StopCommand` (voice), `ManualStopCommand`, `HalfwayReached`, `TimerCompleted`, `SpeechError`, `AppBackgrounded`, `InterruptionBegan(permanent | transient)`.

## Two identities (controller-owned)

- **`sessionId`** — created for each accepted Start (voice or manual); stable through `PREPARING`, `RUNNING`, `COMPLETED`, `REST`, and **invalidated when `REST` ends** (entering `IDLE` or `LISTENING_FOR_START`), as well as on cancel, session end, or a new session. A pending voice-unavailable alert carries the `sessionId` of the session that requested it (or null when requested sessionless in `IDLE`); it is not a workout continuation. An Accepted Start discards any pending alert (kind `entry` or `session`) carried from a previous session — the new session starts with no pending alert. Scheduler and audio continuations capture `sessionId` and ignore themselves if it changed (cancel, session end, new session). **Exception (single rule): a pending alert is not a continuation.** It is owned by `SpeechController` (§8 rule 8), is not subject to this generic `sessionId` check, and survives the end of `REST` and `sessionId` invalidation; only §8 rules 4 and 5 discard it. Every other stale scheduler or audio continuation keeps being ignored, and no stale continuation may change the state of a newer session. Invalidating `sessionId` stops stale callbacks from starting or replacing newer session state; a late result from a superseded session never replaces or updates the displayed `lastCompletedResult` of a newer session.
- **Accepted Start** (a defined term used throughout) — the single transition into `PREPARING` (T+0), caused by a recognized `Go` or a manual start button. Only an Accepted Start creates a `sessionId`, discards any pending alert carried from a previous session (kind `entry` or `session`), and resets `sessionAlertPlayed`. It clears no previous result: the previous completed result stays visible until the new session enters `RUNNING` (§9). Pressing `START VOICE SESSION` (entering `LISTENING_FOR_START`), a re-arm, or an ignored `Go` is **not** an Accepted Start and clears nothing.
- **`speechEpoch`** — an integer incremented whenever speech input must stop mattering: before stopping/cancelling a listener, when START or STOP is accepted, on lifecycle cleanup, and when voice becomes unavailable. Every `commandStream` / `errorStream` subscription captures the current epoch; events from an older epoch are ignored. Listening timeouts use this epoch too.

A speech failure increments **only** `speechEpoch`. It never changes `sessionId`, never stops the clock, the safety cap, or any milestone. Stale callbacks are ignored silently and are never reported as new errors.

## Flow

```text
IDLE
  ├─ START VOICE SESSION → ensure mic permission, engine healthy → LISTENING_FOR_START
  │                         (failure → §8 voice-entry failure; stay in IDLE)
  └─ START MANUALLY      → PREPARING

LISTENING_FOR_START
  ├─ StartCommand   → PREPARING
  ├─ START MANUALLY → bounded stop (§4) → PREPARING
  └─ timeout        → IDLE

PREPARING → (T+`startDelay`, default 0 ms = the same synchronous step) → RUNNING
RUNNING (Countdown): halfway (once) → duration reached → COMPLETED
RUNNING (Count-up):  voice STOP | manual STOP | safety cap → COMPLETED
COMPLETED → finish feedback requested → REST (same synchronous step; the feedback plays during REST)
REST → ends at the finish beep's quiet estimate `quietAt` (§6 rule 3), or at once if no play request was made (asset missing); never on a completion callback; setting "After a set" = Wait for me (default) → IDLE (microphone closed, result visible)
REST → same, setting = Listen for Go, rearmDelay elapsed, engine healthy → LISTENING_FOR_START (§11)
REST → cannot confirm audio inactive / engine broken → IDLE (manual controls)

LISTENING_FOR_START / PREPARING / RUNNING / REST
  └─ manual Cancel or lifecycle cancellation → CANCELLED → cleanup → IDLE
Any active state → unrecoverable Timer/core error → ERROR → cleanup → IDLE
```

A speech or audio failure alone **never** cancels a workout once `PREPARING` has begun. A permission-only or speech-entry failure in `LISTENING_FOR_START` ends voice entry and returns to `IDLE`; it does not enter `ERROR`. Reserve `ERROR` for genuine Timer/core faults: an unhandled exception in `TimerController`, `Scheduler`, or `Clock` code. It is logged with its stack trace, cleanup runs, and the app returns to `IDLE` with the message `Something went wrong. The set was cancelled.` (`DIAG_DETAIL` builds also show the exception text). There is no dedicated error screen.

`REST` is a short post-finish bridge, **not** the athlete's rest. The athlete's real rest happens in `IDLE` (default: the microphone is closed and nothing listens) or, with `After a set: Listen for Go`, in `LISTENING_FOR_START`. In `REST` only Cancel is available; Cancel there returns to `IDLE` and **keeps** the completed result (§9).

`COMPLETED` is transient: `speechEpoch++` and `sessionId` stays through `REST`; command handling is disabled; the result is immutable. Finish feedback is requested once, immediately, under the §6 mandatory-cue policy — it does not wait for the engine.

**Inputs during `COMPLETED`.** `COMPLETED` is a bridge inside a single synchronous step (unified stop, §9): finish feedback is requested and the controller enters `REST` in the same step. `COMPLETED` has no observable duration and dispatches no inputs; an input arriving around the transition is handled by the state the controller is actually in when it is processed (`REST`, or `IDLE` after it):

| Input around `COMPLETED` | Handling |
|---|---|
| Voice `Go` / `Stop`, manual Start, manual STOP | ignored — `COMPLETED` is not `RUNNING` (Stop is accepted only in `RUNNING`, §16) and a Start outside `IDLE` / `LISTENING_FOR_START` is not an Accepted Start |
| Voice entry press (`START VOICE SESSION`) | not offered — the UI is on the Finish screen, which offers only `[ CANCEL ]` in `REST` (§13) |
| `[ CANCEL ]` / Back / lifecycle cancel | take effect in `REST`: return to `IDLE`, keep the completed result (§7/§9) |
| `AppBackgrounded` / permanent interruption | cleanup per §9, then `IDLE`; `AppBackgrounded` also clears the result (§9 Result lifetime), a permanent interruption alone keeps it |

---

# 8. Voice Failure Policy (single canonical rule)

A *broken engine* is never recovered automatically in the MVP; a retry happens only on an explicit button press from `IDLE` (§8 Engine call discipline). Two things are not recoveries: a late successful `initialize()` while idle, and the optional automatic re-arm in `Listen for Go` mode. Keep it simple: **any engine failure disables voice until the next explicit retry from `IDLE` (below) or an app restart, plays the alert once, and keeps manual controls; permission denials and environmental microphone faults are the retryable exceptions below.** Restarting the app always resets.

## What counts as a voice failure

Any of: `initialize()` fails or times out; `startListening()` fails or times out; `stopListening()` fails, or is still pending after `stopListeningHardTimeout` (2000 ms from its invocation); `dispose()` fails or times out; any `errorStream` event other than a permission denial, an environmental microphone fault (`MIC-2xx`, below), or a `commandUnsupported` event (`ENG-109`, a capability limit that sets `voiceStopAvailable = false` only and never `engineBroken`); an ambiguous or unclassified error.

Effect: set `engineBroken = true` (until a successful retry or restart), `speechEpoch++`, disable command handling, record a diagnostic (§13), and then apply the alert rule below.

**Closing the microphone is the one engine call always allowed.** If a listener may be open (`captureMayBeOpen`) when any failure is detected, and no earlier call on that instance is still pending, request exactly one best-effort `stopListening()` (bounded by `stopListeningHardTimeout`, never awaited by the Timer). The same single call is allowed at Cancel/cleanup and at Count-up stop. If it fails or hangs, record `ENG-106`; the microphone may then still be open, so show `Voice unavailable` (and the code in `DIAG_DETAIL` builds) and restart the app before further use. "Restart" means terminating the app process (swipe it away or force-stop), not Hot Restart; the operating system then releases the microphone. The app opens no microphone at launch: only `START VOICE SESSION` (or `Check voice`) does.

**Re-evaluate when a pending call completes.** If a `startListening()` that timed out (or was in flight at cancel/background) completes later and no other call is pending, a listener may have opened: issue the same single bounded `stopListening()` immediately and handle its result as above. This is the only late completion on a broken engine that triggers a call.

**Permission denial is the one exception.** A permission preflight denial before any engine call, or a `startListening()` that fails with a clearly classified `permissionDenied` / `permissionPermanentlyDenied` and nothing else, does **not** break the engine. End the voice attempt, show the label `Voice unavailable` and the §8 user message (`MIC-201` with `[ Allow ]`, or `MIC-202` with `[ Open Settings ]`), play the alert once, keep manual controls, and allow a later voice attempt after permission is granted (`MIC-201` ask again; `MIC-202` permanently denied or revoked). In this case no capture was ever opened, so set `captureMayBeOpen = false` (otherwise voice could never be retried). If the stack cannot tell *denied* from *permanently denied* (this needs a verified API or package; record the finding in B0a), record `MIC-201` with the detail `state unknown` and show `[ Open Settings ]` only (never invent a permanent denial). Any conflicting evidence (timeout, another error category, a pending call) is a normal voice failure.

**Environmental microphone faults (`MIC-2xx`) are retryable too.** For `MIC-203`, `MIC-204`, `MIC-205`, `MIC-206`, `MIC-210`, `MIC-211`, and `MIC-202` when it arrives through `errorStream` while a listener is open (permission revoked mid-session): `speechEpoch++`, disable commands, request one bounded `stopListening()`. If it completes successfully (capture closed), the engine stays healthy, `captureMayBeOpen = false`, the alert plays once, the fault is shown (§8 Fault codes), and a later `START VOICE SESSION` may try again without restarting the app. If the stop fails or hangs, it is a normal voice failure. In Count-up `RUNNING` the fault sets `voiceStopAvailable = false` for the rest of that workout.

## Fault codes and detail

Every voice failure, permission denial, and playback fault produces one `AppFault` record in every build: `code`, `stage` (`permission | initialize | startListening | listening | stopListening | dispose | playback | core`), `detail` (human-readable: file path, expected vs actual size, ABI list, timeout and elapsed time, route name), `rawMessage` (exact platform/native text), `stackTrace` (only for stage `core`, `DIAG_DETAIL` builds only; empty otherwise), `timestamp`, `appVersion`, `engineId`, `modelId`, `androidApi`, `deviceModel`, `state`, `mode`. It is appended to the diagnostics ring buffer (§13). **Persisted fault history (`DIAG_DETAIL` builds only; public builds write nothing).** Each new `AppFault` is also appended, best-effort and asynchronously through **one serialized writer** (one write at a time, later faults queue behind it, the last state wins; never blocking the Timer, never awaited by any state change), to `diagnostics_log.txt` in the app's documents directory, which keeps a rolling window of the **last 5 faults across runs**: a sixth fault drops the oldest. **Nothing is written or truncated at startup by a `DIAG_DETAIL` build**, so the evidence of an `ENG-106` survives restarts once its write has completed, until five newer faults push it out; a write still in flight when the process is killed can be lost (the write is best-effort, and no text claims otherwise). At startup the file is read and shown in the Fault panel and Settings → Diagnostics under the heading `Previous session` as **history only**: it never sets `engineBroken`, never plays an alert, never changes the `Voice:` status line (which starts as `not checked`, §8 Voice health), and is not an unresolved fault. An unreadable or malformed file is treated as empty and logged; the next append rewrites the file atomically (temporary file, then rename) from the valid records only, so the problem does not repeat at every launch. **One-time cleanup in builds without `DIAG_DETAIL`:** at startup such a build deletes `diagnostics_log.txt` if it exists (that exact file name only, never any other file), so a log left by 0.0.1 on the owner's phone does not outlive the upgrade; it writes nothing; a failed delete never blocks startup, is retried at the next startup, and no text ever claims the file was removed unless the app verified it. The file is deleted by `[ Delete log ]` (Settings → Diagnostics, `DIAG_DETAIL` builds only, enabled only in `IDLE`) and is never sent anywhere. The in-memory ring buffer (§13) is separate and is lost when the app closes.

**Size and sanitising (provisional, injectable).** Every text field of an `AppFault` is capped: `detail` 1000 characters, `rawMessage` 2000, `stackTrace` 4000; longer text ends with `…[truncated]`. Control characters other than a newline are replaced before the record is stored, shown, copied or written to the file, so the log file stays valid. `[ Copy details ]` puts diagnostic text on the clipboard and the panel says so. The log is designed to hold no audio and no workout data; `rawMessage` is native text copied as received and can include file paths or other environment details the app cannot filter, so review any copied text before sharing it, and the copy button says so.

**Event snapshot (`DIAG_DETAIL` builds only).** Each fault written to `diagnostics_log.txt` carries a snapshot of the last 20 ring-buffer events (§13) before it: for each, the timestamp, event, state, engine call stage, `speechEpoch` and any timeout or elapsed value, each capped at 200 characters, so a race or a late completion can be reconstructed after a restart. It holds no audio, no workout result and no `rawMessage` of the earlier events. The file still keeps the last 5 faults (with their snapshots) and `[ Delete log ]` removes all of it.

**Build mode.** `--dart-define=DIAG_DETAIL=true` (default false) is used only for the owner's diagnostic build (0.0.1). Every build given to friends (0.0.2 and later) and every public build (0.9.0 and later) is built with `false`. The Android `versionCode` is a separate integer that must increase with every upload.
- `DIAG_DETAIL=true`: a persistent **Fault panel** on the Home and Timer screens shows `CODE · stage`, the `detail`, the `rawMessage`, and `[ Copy details ]` / `[ Check voice ]` / `[ Dismiss ]`; `(+N earlier)` if more exist; Settings → Diagnostics lists the last faults with copy and `[ Delete log ]`. The panel never blocks manual controls and never auto-dismisses.
- `DIAG_DETAIL=false`: no codes, details, or raw text. The user sees the short label `Voice unavailable`, the alert, and — for the fixable microphone problems in the next table only — one plain sentence.
- In both modes the alert and manual controls behave exactly as §8 says.

**User messages (all builds).** Fixable microphone problems show one plain, localized sentence in a dismissible card next to the voice button (and on the Timer screen if it happens in Count-up `RUNNING`), together with the alert tone. The card clears on dismissal or after the next successful voice session. In `DIAG_DETAIL` builds the code and details are shown in addition. The action buttons (`[ Allow ]`, `[ Open Settings ]`) are shown only in `IDLE`; on the Timer screen during a workout the card shows the sentence with no action button, and no permission request or Settings navigation may start from a workout state (§10: no permission prompt appears during a workout; opening Settings backgrounds the app and cancels the session).

| Code | Message | Button |
|---|---|---|
| `MIC-201` | Microphone permission is needed for voice commands. | `[ Allow ]` (ask again); **exception:** when the detail is `state unknown` (below) only `[ Open Settings ]` |
| `MIC-202` | Microphone permission is needed for voice commands. | `[ Open Settings ]` |
| `MIC-203` | Another app is using the microphone. Close it and try again. | — |
| `MIC-204`, `MIC-210`, `MIC-211` | No sound from the microphone. Check the microphone privacy switch and make sure no other app is using it. | — |
| `MIC-205` | The microphone input changed. Disconnect or reconnect your headset, then try again. | — |
| `MIC-206` | Voice listening was interrupted (call or other audio). Try again when it ends. | — |

In Count-up `RUNNING` there is no in-workout retry ("never attach a late STOP listener in this workout", §10); for faults happening during a workout, "try again" in the `MIC-203`–`MIC-211` messages therefore means the next voice session (the next set), never the current workout.

Every `ENG-*` and `AUD-*` code (including `ENG-199` and `AUD-399`) shows only the label `Voice unavailable` in public builds. One tone serves all faults in this version; a second, microphone-specific tone is a possible later change (§12 Extension points). The same tone and the card never block manual controls.

**Classification.** Map a platform or package error to a code only from error codes/exceptions verified in the installed package, its official examples, or the Android documentation; never guess. Anything unmapped becomes the group's unclassified code (`ENG-199`, `MIC-299`, `AUD-399`) with the full `rawMessage`. A code's meaning never changes once used. `docs/ERROR_CODES.md` lists every code with meaning, typical cause, and fix; a test fails if a code exists in code but not in the catalog, or the reverse.

| Code | Meaning | Detail to record | Typical fix |
|---|---|---|---|
| `ENG-101` | Model file missing or unreadable | path, expected/actual size | check `pubspec.yaml` assets and the copy-to-local-path step |
| `ENG-102` | Keyword/tokens file missing or does not match the model | file names, model id | regenerate keywords with the model's own tooling |
| `ENG-103` | Native library failed to load / ABI mismatch | device ABIs, APK ABIs, exception | build for the device ABI; avoid Hot Restart |
| `ENG-104` | `initialize()` failed or timed out | timeout, elapsed | prewarm in `IDLE`; check memory; re-measure cold start |
| `ENG-105` | `startListening()` failed or timed out | elapsed, calls in flight | check call order and microphone ownership |
| `ENG-106` | `stopListening()` failed or exceeded 2000 ms | elapsed | check the engine stop path |
| `ENG-107` | `dispose()` failed or timed out | elapsed | check resource release order |
| `ENG-108` | Error from the engine stream while decoding | raw message | read the raw text; compare with the official example |
| `ENG-109` | Command unsupported by the model (e.g. `Stop`) — a capability limit, not an engine failure: no alert, status stays OK, not resolved by `Check voice` | command, model id | different model/keywords; record as unsupported |
| `ENG-110` | Invalid engine configuration (other than keyword or model files) | parameter name, value | fix the configuration; compare with the official example |
| `ENG-199` | Unclassified engine error | raw message | classify after investigating, then add a code |
| `MIC-201` | Permission denied (can be asked again). **Intentional exception:** the stack cannot tell *denied* from *permanently denied* → recorded as `MIC-201` with the detail `state unknown`, shown with `[ Open Settings ]` only (§8 Permission denial) | permission state (`state unknown` in the exception) | ask again on the next voice request; in the `state unknown` case open Settings |
| `MIC-202` | Permission permanently denied or revoked | permission state | `[ Open Settings ]` |
| `MIC-203` | Microphone in use by another app or unavailable | raw platform error | close the other app (call, recorder, assistant) |
| `MIC-204` | Microphone blocked by the system privacy setting (when the platform reports it) | raw platform error | enable microphone access in Quick Settings / Privacy |
| `MIC-205` | Audio input route lost or changed | route name before/after | disconnect or reconnect the headset |
| `MIC-206` | Interrupted: audio focus lost or phone call | focus event | retry after the interruption |
| `MIC-210` | **Silent input:** every sample exactly zero (or no frames) for `micSilenceTimeout` after the listener opened | timeout, frames seen | privacy switch off, another app holding the mic, hardware mute |
| `MIC-211` | Input stalled mid-session (frames stopped for `micSilenceTimeout`) | last frame time | check route/focus; read the log before it |
| `MIC-299` | Unclassified microphone error | raw message | classify after investigating, then add a code |
| `AUD-301` | Audio asset missing | asset path | check `pubspec.yaml` and file names |
| `AUD-302` | Playback failed | asset, raw message | check the player state and the file format |
| `AUD-303` | Audio engine initialization failed or timed out (§6 rule 9); the audio engine is unavailable until the app restarts | timeout, elapsed, or raw message | check the player package, the audio output and the device audio session; restart the app |
| `AUD-305` | Microphone opening refused: gate still closed at the end of `pressGraceWindow`, or refused at once because an alert is pending | elapsed, `alertPending` | policy refusal, not a player fault |
| `AUD-399` | Unclassified playback error | raw message | classify after investigating |
| `CORE-101` | Unhandled exception in `TimerController`, `Scheduler` or `Clock` code (§7 `ERROR`); stage `core`; no alert; the generic §7 message is the only user-visible text in every build | exception type; the stack trace in the `stackTrace` field (`DIAG_DETAIL` builds only) | read the stack trace; fix the Timer/core code |

**Error mapping (one table; `SpeechController` implements it in `fault_catalog.dart`).**

| `SpeechErrorCode` | Fault code | Breaks engine | Retry | Alert |
|---|---|---|---|---|
| `permissionDenied` | `MIC-201` | no | yes (ask again) | once |
| `permissionPermanentlyDenied` | `MIC-202` | no | after Settings | once |
| `microphoneInUse` | `MIC-203` | no\* | yes | once |
| `microphoneBlocked` | `MIC-204` | no\* | yes | once |
| `routeLost` | `MIC-205` | no\* | yes | once |
| `focusLost` | `MIC-206` | no\* | yes | once |
| `silentInput` | `MIC-210` | no\* | yes | once |
| `inputStalled` | `MIC-211` | no\* | yes | once |
| `modelFileMissing` | `ENG-101` | yes | §8 retry rules | once |
| `keywordFileInvalid` | `ENG-102` | yes | §8 retry rules | once |
| `nativeLibraryFailed` | `ENG-103` | yes | §8 retry rules | once |
| `invalidConfiguration` | `ENG-110` | yes | §8 retry rules | once |
| `lifecycleTimeout` | by `stage`: `initialize` → `ENG-104`, `startListening` → `ENG-105`, `stopListening` → `ENG-106`, `dispose` → `ENG-107` | yes | §8 (never after a pending call) | once |
| `engineFailure` | `ENG-108` | yes | §8 retry rules | once |
| `commandUnsupported` | `ENG-109` | no (capability limit) | n/a | none |
| `unknown` | `ENG-199` (`MIC-299` only if raised while the microphone was open and the evidence points to the microphone) | yes | §8 retry rules | once |

\* no only if the closing `stopListening()` completes successfully; otherwise yes (§8). "Once" means the §8 alert rules (one per entry attempt, one per session).

**Silent-input detection.** `micSilenceTimeout` 3000 ms (provisional, injectable): while a listener is open, if the capture delivers only all-zero samples (or no frames) for the whole timeout → `MIC-210`; if frames had been arriving and then stop for that long → `MIC-211`. "Exactly zero" is deliberate: a real quiet room has non-zero noise, whereas Android returns digital silence when the system blocks the microphone, usually **without raising any error**. B0a/B0b record whether the integrated stack lets the app observe frames or a PCM peak; if not, record that `MIC-210/211` cannot be detected and rely on the diagnostics log. Active only while a listener is open.

## Voice health and Check voice (`DIAG_DETAIL` builds only)

**Status line** on the Home screen, always text, never color alone:
- `Voice: not checked` (gray) before any voice step has run;
- `Voice: OK` (green) while the engine is healthy and the last voice step succeeded; a capability limit (`ENG-109`) does not turn it red — the line reads `Voice: OK (Stop unsupported)`;
- `Voice: <CODE>` (red, e.g. `Voice: ENG-101`) while a fault is unresolved.

If several faults are unresolved, `Voice:` shows the most recent and the Fault panel lists all of them (`+N earlier`). It updates the moment the state changes. A fault is **resolved** — the Fault panel clears, the line turns `OK`, and the diagnostics buffer records `fault resolved <CODE>` with a timestamp — on any of: a successful `Check voice`, a voice session whose listener opened successfully, or a late successful `initialize()` (which resolves only `ENG-104`). **Scope of resolution (single rule):** a successful `Check voice` or an opened listener resolves only unresolved `ENG-*` (except `ENG-109`) and `MIC-*` faults; an `AUD-*` fault is resolved only by a later successful playback of the same cue (`AUD-303`: it is not resolved within the process, §6 rule 9); `CORE-101` is never resolved by these events and stays listed until `[ Dismiss ]` or the next app start.

**`Check voice`** is a button in Settings → Diagnostics and on the Fault panel. It runs in `IDLE` only, never during a workout or while listening, holds the wakelock while it runs, plays no alert even on a failed retry (you are looking at the screen), and uses the normal `SpeechController` rules (serialized calls, bounded stops, faults recorded with their codes; a run while the engine is broken counts as the explicit retry of §8, without any alert). Its microphone steps open only through `canOpenMicrophone(check)`: while `alertPending == true` the microphone step is refused and reported as refused in the step list (no `stopAll()`, no alert). The Voice Test screen listener likewise opens only through `canOpenMicrophone(voicetest)` and never past a pending alert. It shows each step with ✅ / ❌, its code, and its time, and the list stays until dismissed and can be copied:

```text
✅ Permission granted
✅ Engine initialized (840 ms)
✅ Microphone opened — frames: 187, peak level: 0.12, route: phone mic
✅ Heard "Go" (optional step) — or ⚠ not heard within 8 s (not a fault)
✅ Microphone closed
❌ MIC-210: all samples zero for 3000 ms          (example of a failing step)
```

1. **Permission:** request if needed; failure → `MIC-201` / `MIC-202`.
2. **Engine:** apply the §8 retry rules, then `initialize()`; failure → `ENG-1xx`. On success list the supported commands (e.g. `Go, Stop`), so you can see whether `Stop` is supported.
3. **Microphone:** when `canOpenMicrophone(check)` returns `mayOpen`, open the listener for `micCheckDuration` (4000 ms) and report frames received, the peak level, and the route name when observable (if the stack cannot expose them, show "not observable on this stack" and record it); a refused gate is reported as refused, never a fault; an opened listener that fails → `MIC-2xx`.
4. **Optional "Say Go":** wait up to `checkVoiceSayGoWindow` (8000 ms) for a `Go`; "not heard" is a warning, never a fault and never breaks the engine.
5. **Close:** `stopListening()`; failure → `ENG-106` (normal §8 consequences).

If steps 1–3 and 5 pass, the status becomes `Voice: OK` — or `Voice: OK (input level not observable)` when step 3 could not observe frames or a level (the engine is healthy but the signal was not measured) — and every unresolved `ENG-*` (except `ENG-109`, a capability limit that a check cannot resolve) and `MIC-*` fault is resolved (`AUD-*` faults are resolved only by a later successful playback of the same cue). Public builds have no status line and no `Check voice`.

**Check voice lifecycle.**
- A visible indicator (`Checking microphone…`, or `Listening for Go` during the optional step) is shown for every moment the microphone is open.
- While it runs: `START VOICE SESSION`, `START MANUALLY`, and Settings changes are disabled; the Fault panel's `[ Check voice ]` is hidden outside `IDLE`. It cannot be started while `alertPending == true` (let the pending alert play first).
- If the app goes to the background or the user leaves the screen, the check is cancelled: one bounded `stopListening()`, wakelock released, the result list marked `cancelled`.
- The wakelock is held only while the check runs (an explicit exception to §13) and is released on success, failure, and cancellation.
- A `Go` heard during the check is shown in the result list and is never a workout command.
- Each run while the engine is broken counts as the explicit retry (`engineRetryCount`). A prewarm failure at launch shows `Voice: <CODE>` immediately and is not a retry; a prewarm success alone leaves the line `not checked` (only a completed voice step turns it `OK`).

## Engine call discipline

- The `SpeechController` is the sole owner of the `SpeechEngine` lifecycle. No other component calls `initialize()`, `startListening()`, `stopListening()`, or `dispose()`.
- Calls are serialized: a new call starts only after the previous one completes. **Never start an engine call behind a pending one.** A Dart timeout does not cancel native work.
- When the engine is broken: make **no engine call during a workout** other than the single bounded `stopListening()` above, and none while any earlier call on it is still pending. Ignore its late events and completions, **except** a late *successful* `initialize()` (next bullet) and the late completion of a pending `startListening()` (which triggers the one closing stop, above).
- **Late successful `initialize()`.** If `initialize()` completes successfully after its timeout, the app is in `IDLE`, and no other engine call has started on that instance, accept it: `engineBroken = false`, `captureMayBeOpen = false`, `engineRetryCount = 0`, clear the `ENG-104` fault (record `fault resolved ENG-104` and the real elapsed time in the diagnostics buffer). During a workout, or if another call has started, ignore it as before. After the timeout, a press of `START VOICE SESSION` while that `initialize()` is still pending is refused with the alert (the fault stays visible); pressing again after it completes works. Use the logged elapsed time to adjust `initializeTimeout`.
- **Press during a normal (not yet timed-out) initialization.** The press waits for the same `initialize()` Future up to `initializeTimeout`, starts no second initialization, shows the label `Preparing voice…` while waiting, and continues into `LISTENING_FOR_START` if it succeeds; if it fails or times out, the normal failure rules apply.
- **Retry on an explicit voice request (`IDLE` only).** Each press of `START VOICE SESSION` in `IDLE` may re-initialize a broken engine once. **All** of these must hold: (1) the app is in `IDLE` (never during `PREPARING`, `RUNNING`, `REST`, or a Count-up workout); (2) the earlier failure was a *completed* error — an exception, a failed result, or an error event with no call pending — not a timeout with a call still pending (native work may still be running; the engine then stays broken until that call completes, where a late successful `initialize()` is accepted per the bullet above, or until the app restarts); (3) `captureMayBeOpen == false` (a closing stop that failed or hung means the capture may still be open: no new instance until the process restarts, so two captures can never exist); (4) `engineRetryCount < 3`. The retry: `dispose()` the old instance (bounded by `disposeTimeout`; if it does not complete, stay broken), create a new instance, `initialize()`. On success the press continues into `LISTENING_FOR_START` and `engineRetryCount` resets. Every failed retry increments `engineRetryCount`, records its fault code, and — except retries run inside `Check voice`, which never plays an alert (§8 Voice health) — plays the entry alert once. After the 3rd consecutive failure a press of `START VOICE SESSION` makes no engine call: it plays the entry alert once, shows `Voice unavailable` (and the last code in `DIAG_DETAIL` builds), and manual controls remain; `Check voice` then only reports the existing fault without calling the engine.
- `stopListening()` has two deadlines: `stopListeningTimeout` (300 ms) ends only the *caller's* wait; the controller keeps observing the same Future until it completes or reaches `stopListeningHardTimeout`. While it is pending, start no other engine call.
- Track one boolean `captureMayBeOpen`: true from a `startListening()` request until a `stopListening()` completes successfully (a `startListening()` that *completed with an error*, with nothing pending, also sets it false, because no capture was opened). It stays true if the closing stop failed or hung, or while a `startListening()` is still pending, until that call completes late and the single closing stop succeeds (§8) or the process restarts. **A retry from `IDLE` is allowed only while `captureMayBeOpen == false`** (§8). It only blocks opening the microphone and re-arming; it never blocks a cue or the timer.
- Show the label `Voice unavailable` — never claim the microphone is off. Listening labels appear only while a healthy listener is active (§12).
- Prewarm the model asynchronously in `IDLE`. `initialize()` must be verified to never open the microphone. A prewarm failure alone sounds no alert; the next explicit voice request does.
- Model/native late events after lifecycle shutdown (background, cancel) are stale and do not independently break the engine; `AppBackgrounded` takes precedence over a shutdown-generated `SpeechError` arriving in the same dispatch turn.

## The alert

`playVoiceUnavailable()` plays the human-supplied file. It has no dependency on `SpeechEngine` and no setting.

**At most one alert per attempt, tracked by two plain booleans on the controller:**
- `entryAlertPlayed` — reset on each explicit `START VOICE SESSION` press. A failed voice entry plays the alert once (every press while voice is broken plays it once).
- `sessionAlertPlayed` — reset at each accepted Start. **One alert per session in total:** whichever happens first — (a) any voice failure detected after Start is accepted, in either mode (including a `stopListening()` that fails or hangs during `PREPARING`/`RUNNING`/`REST`), or (b) Count-up STOP being or becoming unavailable (§10; the known `ENG-109` capability limit is excluded and plays no alert) — plays it; the other does not play a second alert. **Exception:** a session started manually (`START MANUALLY`) with no microphone permission does not play a session alert; a permission alert is played only by a failed voice entry attempt (pressing `START VOICE SESSION`, under `entryAlertPlayed`); other alert causes, such as an engine failure, are unchanged. This is the single definition of the manual-start exception.

Rules:
1. Play when the failure is detected and commands are disabled. It does not wait for capture closure: the engine is already disabled and its events ignored, so overlap is harmless.
2. If app audio may still be audible (`audioGateSatisfied == false`, §6 rule 3), or a mandatory cue is due within the alert's own length (max 1200 ms), the alert is **pending**. In practice only `start_confirm` (at T+0), `halfway`, and `finish_beep` can delay it; with `startDelay` > 0 an alert raised during `PREPARING` plays as soon as `start_confirm` has ended. A pending alert is not marked played until its play request has been invoked. A pending alert is never discarded by the normal end of a set: it plays once the gate opens (the finish beep's estimated end), and then the app follows §11 (`IDLE`, or the `Listen for Go` re-arm if its conditions hold — after an engine failure the re-arm is refused anyway).
3. If the alert is already playing when a mandatory cue becomes due, the cue is requested at its due time as usual (§6 rule 1), the two may briefly overlap, and the alert is marked played and is not replayed.
4. Accepting START dismisses a pending *entry* alert from the voice-entry attempt that led to this Start (never replayed), and it also discards any pending alert (kind `entry` or `session`) carried from a previous session (§7). It does not suppress a *session* alert requested after this Start was accepted.
5. Explicit Cancel, lifecycle cancellation, or a Timer/core error clears any pending alert. Normal completion (Countdown end, accepted Count-up stop) does **not**: see rule 2.
6. The alert is never played while the Count-up STOP listener is actively listening (it is only requested after the failure has disabled that listener).
7. A missing/failing/hung alert file is logged and never crashes or changes Timer state. Never wait for the alert before updating Timer state.
8. **Pending-alert state.** `alertPending` (bool) and `alertPendingSince` (when it became pending; diagnostics only) belong to the `SpeechController`, not to the workout `sessionId`: ending `REST`, invalidating `sessionId`, or entering `IDLE` never cancels a pending alert requested by the finished session; only rules 4 and 5 do (rule 4: an accepted Start dismisses a pending `entry` alert from its own entry attempt and any alert carried from a previous session; rule 5: Cancel, lifecycle cancel, or a Timer/core error). A pending alert is never pending forever: it waits only for app audio and cues, and both are bounded (the quiet estimate always expires, §6 rule 3). It plays at the first instant at which `audioGateSatisfied` is true and no mandatory cue is due within the alert's own length (rule 2); there is no separate timeout. Every microphone opening is governed only by `microphoneOpenGateSatisfied` (§6 rules 4a and 4b), which is false while `alertPending` is true; no path may open the microphone past a pending alert, nor cancel it to do so (only rules 4 and 5 cancel). When the pending alert's turn comes it is requested as an ordinary play and extends `quietAt` by its own estimate (§6 rule 3). **Atomic hand-off:** `alertPending` stays true until the alert's play request has been recorded. In one synchronous step with no `await`, the controller decides the alert's play request (which starts `playRequestPending`, §6 rule 4a) and only then clears `alertPending`; the reverse order is forbidden, so no evaluation can observe `alertPending == false` before the request exists. A press of `START VOICE SESSION` while `alertPending == true` adds no alert (merged, rule 9); it is refused with a single `AUD-305` and stays `IDLE`.
9. **One pending alert at a time.** `alertPending` holds at most one alert, with a kind (`entry` or `session`) and the `sessionId` of the session that requested it (or null when sessionless). A new request while one is pending is merged, never queued twice. Accepting START discards a pending `entry` alert from its own entry attempt and any alert carried from a previous session (so a stale alert never consumes the new session's one-alert budget, §7); Cancel, lifecycle cancel, or a Timer/core error discard any pending alert; normal set completion discards nothing (rule 2).
10. **Playback faults and the alert (single definition; refines rule 2).** A playback fault never triggers an alert by itself; the alert is only for voice failures and an unavailable STOP (§8 above). Whatever the outcome of the **alert's own** play request (invoked, asset missing, play call failed, abandoned by `playInvokeTimeout`), the alert counts as played: it consumes its entry or session budget, is never retried or replayed, and plays no second tone. Every such outcome is a visible fault record.

| Playback fault | Alert | Fault record | `quietAt` | Retry |
|---|---|---|---|---|
| Mandatory cue, asset missing | none | `AUD-301` | unchanged | none |
| Mandatory cue, play call failed | none | `AUD-302` | extended (conservative) | none |
| Invoke timeout (any sound) | none | `AUD-302`, detail `invoke timeout` | extended from the abandonment | none |
| Alert's own asset missing or play failed | none (never a second alert) | `AUD-301` / `AUD-302` | per §6 rule 8 | none; the alert counts as played |
| Unclassified playback error | none | `AUD-399` | extended (conservative) | none |
| Audio engine unavailable (`AUD-303`) | none (no audio is possible) | `AUD-303`, once | unchanged | none (restart the app) |

## During a workout

- Countdown `RUNNING`: no microphone is open; nothing to do. The timer continues.
- Count-up `RUNNING`: set `voiceStopAvailable = false`, keep the timer running, keep manual STOP prominent, play the session alert once (except for the `ENG-109` capability limit, which plays no alert; also except the manual-start-without-permission case, §8 `sessionAlertPlayed`). No recovery is attempted.
- `PREPARING` / `REST`: skip any failed cue and log it; the workout continues; the session alert follows rule 2 (during `PREPARING`, when `startDelay` > 0, it follows rule 2).

---

# 9. Count-up Stop, Cancel, Back, and Result Lifetime

## Unified Count-up stop operation

For voice STOP, manual STOP, and the safety cap alike. At the instant the first stop reason wins, **synchronously**:

1. `finalElapsed = Clock.now() - startTime` (immutable); freeze the counter; show the badge `STOPPED` and the line `TIME mm:ss` (§13); record `stopSource` = `voice` | `manual` | `safetyCap`.
2. State → `COMPLETED`; `speechEpoch++`; disable command handling; keep `sessionId` through `REST`.
3. Store the result in memory (`lastCompletedResult` = `finalElapsed` and `stopSource`); nothing is written to disk (§9 Result lifetime).
4. If the microphone may be open, request `stopListening()` — allowed even when the engine is broken (§8, the one always-allowed call); never awaited before step 5; keep observing it per §8.
5. **Play `finish_beep` immediately** under the §6 mandatory-cue policy. Do not defer it for the engine. A later stop reason (a second voice STOP, a manual STOP after voice) is ignored because the identities have changed.
6. Enter silent `REST`, then follow §11 (default: `IDLE`, microphone closed; re-arm only in `Listen for Go` mode and only under the §11 conditions).

If the safety cap fires while the engine is broken or a stop is pending, the timer still stops and records `finalElapsed` from `Clock`.

## Cancel and Back

- **Cancel** aborts the session without recording a completed result.
  - Count-up in `PREPARING`: the previous completed result, if any, stays displayed (it is cleared only when the new session enters `RUNNING`).
  - Count-up in `RUNNING`: the session is discarded; there is no result to show (the previous one was cleared on entering `RUNNING`).
  - `LISTENING_FOR_START`, or `REST` (either mode): return to `IDLE`; the displayed result is kept.
  - Countdown: nothing to store; the `Set done` marker follows the same lifetime rule.
- **Back** during an active session is never a silent Cancel. In every active state and in both modes: confirmation with `Cancel session` / `Continue` (no keep option: leaving the screen ends the session and nothing is stored). **If the session ends (Countdown completion, accepted voice Stop, manual Stop, or safety cap) while the Back confirmation dialog is showing, the dialog is dismissed without applying any choice, and the completed result is displayed normally.**
- Cleanup (explicit Cancel, lifecycle cancel, or unrecoverable Timer/core error): cancel every scheduled event; call `AudioFeedbackEngine.stopAll()`; invalidate `sessionId` and `speechEpoch`; no stale cue or resume after cancellation; ignore a late `REST`-end timer; close the microphone with the single bounded `stopListening()` that §8 allows (no other engine call if the engine is broken); release the wakelock; → `IDLE`. A voice failure is **not** a cleanup trigger (it follows §8 and does not cancel its own alert).

## Result lifetime (nothing is stored)

The app stores nothing: no workout record, no result, no settings, no log file (the single exception: the `DIAG_DETAIL` fault log of §8, which holds no workout data). The completed result (`lastCompletedResult` = `finalElapsed` and `stopSource`; for Countdown, the `Set done` marker) lives in memory only.

- **Created** at the accepted stop or safety cap (Count-up) or at Countdown completion, in the same synchronous step; nothing waits for any write.
- **Cleared** when: (1) a newer session enters `RUNNING` in either mode (an Accepted Start itself clears nothing; with the default `startDelay` = 0 the two instants coincide); (2) the athlete leaves the screen that shows it for another page (Settings, About testing, Voice Test, any other route); (3) `AppBackgrounded` (§13 Lifecycle); (4) the process ends. Nothing is restored after a restart, a crash, or a process kill.
- **Cancel** in `REST` or `LISTENING_FOR_START` returns to `IDLE` and keeps the result; Cancel in `PREPARING` (only when `startDelay` > 0) keeps the previous result; Cancel in `RUNNING` leaves no result.
- **A completed result is not an interrupted session:** a result already shown (in `REST`, `LISTENING_FOR_START` or `IDLE`) is kept by a permanent interruption alone (§7 table); only `AppBackgrounded` clears it (clear condition 3).
- **Interrupted sessions are simply lost:** background, a permanent interruption, answering a phone call, a crash, or a swiped-away app ends the session with no prompt, no `Keep Result`, no `Discard`, and nothing to resume.
- **Honest limit:** with the default `startDelay` = 0 an accidental `Go` replaces the previous result at once; a non-zero `startDelay` gives a cancel window.
- Extensions S and P (§21) leave this section unchanged for the timer's own `lastCompletedResult`: a session summary and saved results are separate data owned by their own controllers (§21.3). Until they are activated, none of them exists.
- There is no `session_repository`, no snapshot, no write queue, and no versioned record. The monotonic `Clock` stays the only source of time.
---

# 10. Microphone / Speech Lifecycle

## Before Start
Listen only in `LISTENING_FOR_START`. Request `RECORD_AUDIO` only after the user presses `START VOICE SESSION`, never during a session. If permission is not granted when a Count-up session starts with `voiceStopCapability` ON (for example after `START MANUALLY`), `voiceStopAvailable = false` for that session: the single code that matches the verified permission state is recorded (`MIC-201` if it can still be requested, `MIC-202` if permanently denied; if the state cannot be verified, `MIC-201` with the detail `state unknown`, the single definition in §8 Permission denial; never a permanent denial invented from an unknown state), the §8 message card is shown without action buttons during the workout; no session alert plays (manual-start exception, single definition in §8 `sessionAlertPlayed`); no permission prompt appears during a workout.

## Count-up STOP listening (voice Stop capability ON and `voiceStopAvailable`)

- The workout clock enters `RUNNING` at `startTime` (T+`startDelay`, default T+0) **regardless** of audio or microphone readiness.
- **One definition of the settle anchor:** `settleAnchor = max(startTime, audioGateOpenedAt)` (`startTime` = logical T+`startDelay`, §16; a `null` `audioGateOpenedAt` counts as `startTime`). The controller opens the STOP listener at `settleAnchor + stopListenSettleDelay` once all prerequisites hold: engine healthy, `microphoneOpenGateSatisfied` (§6 rule 4a), `captureMayBeOpen == false`. `audioGateOpenedAt` is the instant `audioGateSatisfied` last became true (§6 rule 4a); it is an estimate and is never described as confirmed inactivity. If the gate opens before `RUNNING`, the delay runs from `startTime`; if after, from the gate. Text, code, and Gate 0H all use this anchor and no other. Prerequisites may be confirmed late, **within `stopListenerOpenGrace`** (5000 ms from `RUNNING`, injectable). Initial `stopListenSettleDelay` is 500 ms (a conservative start; tune it from Gate 0H evidence). With the default `startDelay` = 0, `start_confirm` is still audible when `RUNNING` begins, so the anchor is normally the gate opening (the beep's estimated end) plus `stopListenSettleDelay`; `stopListenerOpenGrace` covers a late gate opening.
- **Single rule for "STOP unavailable":** if the engine is already broken when `RUNNING` begins, STOP is unavailable immediately. Otherwise STOP becomes unavailable only when `stopListenerOpenGrace` expires without the prerequisites, or when a later failure occurs. Then: `voiceStopAvailable = false`, show manual STOP prominently, play the session alert once (§8), (subject to the manual-start exception in §8 `sessionAlertPlayed`), and never attach a late STOP listener in this workout.
- **Honest deaf window.** A `Stop` spoken after logical `RUNNING` begins but before the STOP listener has opened (normally `settleAnchor + stopListenSettleDelay`, at the latest `RUNNING + stopListenerOpenGrace`) is not heard; it is a miss by construction, not a recognition failure. Gate 0H counts only utterances spoken while the `Listening for Stop` label is shown.
- Before requesting the STOP listener, check `supportedCommands`: if `stop` is not supported, set `voiceStopAvailable = false` immediately, record `ENG-109` in the diagnostics, show the manual STOP prominently, and do not call `startListening({stop})`. This is a capability limit, not an engine failure: it never sets `engineBroken` and needs no alert.
- Request `startListening({stop})` only. STOP sensitivity is tuned independently of START, expected to be more conservative; no numeric value is fixed here. Do not assume a confidence value exists.
- No application audio plays while the STOP listener is active (Count-up has no mid-run cues). "Active" means *accepting commands*: from the moment the listener opens until a stop is accepted or a failure disables commands. After that, `finish_beep` or the alert may play while the capture is still closing; this is harmless because commands are ignored (§4 Honest limit). The only exception is the alert, after the failure has disabled the listener.
- Stop reasons follow §9. A `START` event during Count-up `RUNNING` is ignored.

## Screen exit
Leaving the timer screen is lifecycle cancellation after the Back confirmation (§9) or on a forced exit (the result is cleared, §9 Result lifetime). The microphone never stays active behind the screen.

## Timeouts for listening
`firstSetListeningTimeout` 90 s applies after a press of `START VOICE SESSION`; `afterRestListeningTimeout` 180 s applies to the automatic re-arm in `Listen for Go` mode (both injectable; separate from engine-call timeouts). On expiry: stop the microphone → `IDLE`; show the label `Stopped listening` (no tone); manual Start remains available; a shown result or a stored `completed` record (together called a *protected result* in this document) is never cleared.

## Timing constants (injectable, named; provisional until measured)

```text
stopListeningTimeout        300 ms   caller wait only (§4)
stopListeningHardTimeout    2000 ms  from the original invocation; pending/failed → voice failure (§8)
initializeTimeout           10000 ms provisional; replace with a value derived from ≥20 cold-start measurements
startListeningCallTimeout   1000 ms  provisional
disposeTimeout              1000 ms  provisional
stopListenSettleDelay       500 ms    (timing delay, not a lifecycle timeout)
stopListenerOpenGrace       5000 ms
micSilenceTimeout           3000 ms  provisional (§8 MIC-210 / MIC-211)
audioInitTimeout            3000 ms  provisional (§6 rule 9: `AudioFeedbackEngine.initialize()`; on expiry `AUD-303`, audio unavailable, no retry)
playInvokeTimeout           1000 ms  provisional (§6 rule 4a: decision of a play request → player API call invoked; on expiry the request is abandoned, `AUD-302`)
audioTail                   500 ms   provisional placeholder (§6 rule 3; ≥ p95 request→audible start + output latency + margin, §16 A2 item 2)
micCheckDuration            4000 ms  (§8 Check voice, microphone step)
checkVoiceSayGoWindow       8000 ms  (§8 Check voice, optional Go step)
resultVisibilityTimeout     45000 ms (§13 wakelock hold after set; injectable, ≥ 30 s ≤ 60 s recommended)
pressGraceWindow            3000 ms  (§11 START VOICE SESSION press waiting for the audio gate; injectable; at least `playInvokeTimeout` + the longest asset hardMax + audioTail = 1000 + 1500 + 500 = 3000 ms, the worst case of an abandoned request, §6 rule 4a)
startDelay                  0 ms     (§5 T+0 → RUNNING; injectable, owner tuning after Gate 0F)
startDetectedDisplay        1000 ms  (§5 visual START DETECTED badge; injectable)
```

Milestone A defines them all with defaults. B2 sets engine-specific values (and any Appendix A engine would do the same). Gate 0E confirms them on the reference device. Expose the values in diagnostics so tests can assert them.

---

# 11. Rest and Next Set

The athlete's rest is not the app's business: after a set the app finishes its audio, shows the result, and **waits with the microphone closed**. This keeps the privacy promise, avoids false `Go` activations while the athlete talks or breathes during rest, and saves battery.

## After a set (both modes)

Both modes: `RUNNING → COMPLETED → REST`, with `finish_beep` requested at `COMPLETED` and playing during `REST` (the same silent bridge in Countdown and Count-up); `REST` ends at the finish beep's quiet estimate: `quietAt` as read right after `playFinishBeep()` returned (`requestedAt + 1500 + audioTail`, §6 rule 3), or at once if no play request was made (asset missing). It never waits for a completion callback.

No completion callback, fallback, or `stopAll()` is involved: `REST` always ends at that instant, and `audioGateOpenedAt` is the same instant. The app then goes to `IDLE` or re-arms as below.


Then, by the setting `After a set`:
- **`Wait for me` (default):** go to `IDLE`. The result (`TIME mm:ss` or `Set done`) stays visible until a newer session enters `RUNNING` or completes. No microphone is open; the wakelock follows the result hold defined once in §13 (`resultVisibilityTimeout`, default 45 s); after it the screen may sleep.
- **`Listen for Go`:** if the re-arm conditions below hold, wait `rearmDelay` and enter `LISTENING_FOR_START` with `afterRestListeningTimeout`; otherwise go to `IDLE`.

## Starting voice from IDLE

Pressing `START VOICE SESSION` applies the same conditions as a re-arm (below) without `rearmDelay`. **This paragraph is the single definition of a press; §6 rule 4b, §8 and §17 only refer to it.**

**Order of decisions on a press:** (1) if `alertPending == true`, refuse at once: no retry, no extra alert (merged, §8 rule 8), one `AUD-305`; (2) **permission preflight (§8 Permission denial, before any engine call):** if `RECORD_AUDIO` is not granted, request it when the platform can still ask (only here, never during a workout, §10); a denial is refused with `MIC-201` / `MIC-202` (or `MIC-201` with `state unknown`), the alert once, state `IDLE`, and **no engine call is made** (`dispose()`, `initialize()` and `startListening()` all wait); (3) if the engine is broken and retry-eligible (§8), run the retry **first** (a failed retry refuses with the §8 alert and code); (4) with a healthy engine, evaluate `canOpenMicrophone(button)` (§6 rule 4a): `mayOpen` → call `startListening()` in the same synchronous step; (5) after **every** asynchronous step (the permission request, the retry, the grace wait, any play requested meanwhile, which closes the gate) re-evaluate every condition fresh (§6 rule 4b) and call `startListening()` only on `mayOpen`.

**Gate closed only by app audio** (`audioGateSatisfied == false`, `alertPending == false`, engine healthy, `captureMayBeOpen == false`): enter the grace wait at once. The press never calls `stopAll()` (the app never manufactures proof of silence).

**Grace wait.** For at most `pressGraceWindow` (3000 ms, injectable) the press shows the transient label `Just a moment…` (a localized string from the language layer, §12; no alert, no fault record) and re-evaluates whenever `audioGateSatisfiedChanges` fires and at expiry. The gate opens within the window → proceed silently. At expiry the press is refused: alert once, with `AUD-305`. A second press during the wait is ignored. The press is never queued beyond the window.

**Every other refusal** (a broken engine that is not retry-eligible, a failed retry) stays in `IDLE`, plays the alert once, and records the code its own rule names (§8). **A pending alert is not in this list:** its refusal is decision (1) above (no alert, exactly one `AUD-305`).

**Refusal matrix (restates the rules above; if it ever differs, the rules above and §8 win):**

| Cause of refusal | Alert | Code | State |
|---|---|---|---|
| `alertPending == true` | none (merged, §8 rule 8) | `AUD-305` | `IDLE` |
| Microphone permission denied (preflight, before any engine call) | once | `MIC-201` / `MIC-202` (`MIC-201` + `state unknown` when undecidable, §8) | `IDLE` |
| Gate still closed at the end of `pressGraceWindow` | once | `AUD-305` | `IDLE` |
| Broken engine, not retry-eligible | once | the code its own §8 rule names | `IDLE` |
| Retry failed | once | the failed retry's code (§8) | `IDLE` | With no play requested yet in this process (first run) the gate is satisfied and the press proceeds at once. The press may start an engine retry but never calls `startListening()` before that retry has succeeded and every condition has been re-evaluated.

## Re-arm conditions (`Listen for Go` mode; deferred, not in 0.0.1)

Open the START listener only when: the engine is healthy; any pending `stopListening()` has completed (waiting at most `stopListeningHardTimeout`); `microphoneOpenGateSatisfied` (§6 rule 4a); and `rearmDelay` has elapsed (measured from `audioGateOpenedAt`). Otherwise return to `IDLE` with manual controls. Never start a second listener against uncertain microphone ownership or possibly active app audio. **A re-arm counts as a voice entry attempt:** `entryAlertPlayed` is reset when it begins, and a failure plays the alert once and shows the §8 message or label.

`rearmDelay` is an injectable named constant. Gate 0F measures it: if any activation occurs during or within 1.5 s after an app sound, raise it and record the final value in §20. The 1.5 s window starts at the **end of the audible sound** as seen on the external recording (the same method as the latency measurement); also log `audioGateOpenedAt` so the offset between it and the audible end is reported. The same own-audio check applies to a press of `START VOICE SESSION` shortly after a cue.

---

# 12. Language Architecture

Everything language-dependent comes from one `AppLanguage` (first release: `en`) through a `LanguagePack`:

```text
LanguagePack
├── UI strings        (Flutter localization resources)
├── audio             (none: all cues are language-neutral tones in `assets/audio/common/`, §6)
├── voice commands    (START/STOP words for this language and the engine/model that supports them)
└── notes             (text direction, number format)
```

- No user-visible string literal in widgets or Timer logic. Use Flutter's official localization mechanism; verify against the installed Flutter version. The Voice Test screen may keep English text isolated from production strings.
- Layouts are direction-agnostic (RTL-ready): `EdgeInsetsDirectional`, `AlignmentDirectional`; no hard-coded left/right. No RTL language ships in v1. Western digits.
- `AppLanguage` is fixed to `en` (nothing is stored); no selector while one language exists.
- Voice commands per language are separate engine evaluations; never assume the English model works for another language.
- The MVP must prove Timer logic works with the English pack and a second fake `LanguagePack`.
- Future languages (Arabic, French) are non-binding and out of v1.

## Adding a language later (data, not code)

A language is a self-contained pack. Adding one must not touch Timer logic, `SpeechController`, or engine wrappers:

1. UI strings: a new `app_<lang>.arb`.
2. Audio: nothing to add; all cues are language-neutral tones in `assets/audio/common/` (§6).
3. Voice commands: the `LanguagePack` declares, per language, the START/STOP phrases and the voice resource it needs (model id, keyword file asset, engine id). A language without a verified voice resource ships with manual controls, or with English voice commands if the human approves.
4. Its own voice evaluation: the Gate 0 procedure (§14) is rerun for that language and engine. The English results never transfer.
5. Text direction and number format in the pack notes.

The `SpeechController` asks the active `LanguagePack` for the command set and resource; it never contains a phrase or a model path.

## Extension points (kept deliberately open)

These abstractions exist so improvements do not require a refactor. Add features only through them, each as a separate human-approved change:

| Seam | Today | Possible later |
|---|---|---|
| `SpeechEngine` | Sherpa-ONNX | another engine (Appendix A), a per-language engine |
| `LanguagePack` | English | more languages, per-language engine/model |
| `AudioFeedbackEngine` | `audioplayers`, one alert tone | another player, a microphone-specific alert tone, audio-route handling (Bluetooth) |
| Command ID → phrase | fixed `Go` / `Stop` | custom phrases (needs approved spec revision + gates) |
| `SettingsController` | Halfway cue, mode, duration, After a set | user-facing options |
| Diagnostics interface | generic name/value parameters | any engine's verified tunables |
| `SessionController` / `ResultsStore` (§21) | none | rounds and goal in memory (Extension S); saved Count-up results on the device (Extension P) |

Haptics, Bluetooth, music robustness, and personalization remain out of v1 and are not designed in advance beyond these seams.

## Microphone / status labels

Short and localized. The audible alert is the primary notification; fixable microphone problems also show one sentence (§8 User messages).

- `Listening for Go` — only while the START listener is healthy and active.
- `Listening for Stop` — only while the Count-up STOP listener is healthy and active.
- `Voice unavailable` — voice failed, STOP unavailable, or permission missing.
- `Stopped listening` — the listening timeout expired (no tone).
- `Preparing voice…` — a voice request is waiting for a normal initialization to finish (§8).
- Otherwise show no microphone label. **Never claim the microphone is off.**

Permission problems show the §8 user message (`MIC-201` / `MIC-202`); `[ Use manual controls ]` is always available. Use a verified package only if needed to open settings or to tell denied from permanently denied; do not invent `openSettings()`.

---

# 13. UI, Settings, Wakelock, Lifecycle, Privacy

## Home
Mode selector `COUNTDOWN | COUNT-UP` with **no mode preselected at a cold start** (the athlete chooses each time; both start buttons stay disabled until a mode is chosen; the choice is not remembered across launches); after `COUNTDOWN` is chosen, durations `20 30 45 60` (30 preselected); `[ START VOICE SESSION ]`; `[ START MANUALLY ]`; hint `Voice command: Go` (Count-up: `Go / Stop`, when `voiceStopCapability` is ON); `[ Settings ]`. In `DIAG_DETAIL` builds also the `Voice:` status line (§8).

## Screens
- **Ready:** `GET READY`, duration, `Take your position`, `Say "Go" when ready`, label per §12, `[ START MANUALLY ]`, `[ CANCEL ]`.
- **Preparing** (exists only when `startDelay` > 0): `START DETECTED` and `[ CANCEL ]` (secondary, always available in `PREPARING`). With the default `startDelay` = 0 there is no Preparing screen: the Running screen shows the `START DETECTED` badge for `startDetectedDisplay`.
- **Running:** very large timer. Countdown shows remaining time with `[ CANCEL ]`; Count-up shows elapsed with `[ STOP ]` (always visible) and `[ CANCEL ]` (secondary).
- **Finish:** `REST` with `[ CANCEL ]` (the only action in `REST`; it returns to `IDLE` and keeps the completed result, §7/§9), then `IDLE` with the result visible and `[ START VOICE SESSION ]` / `[ START MANUALLY ]` (or the automatic re-arm in `Listen for Go` mode, §11).
- **Result (both modes):** `TIME 01:23` under the `STOPPED` badge (Count-up) or `Set done` (Countdown) stays visible until a newer session enters `RUNNING`, the athlete leaves the screen, or the app goes to the background (§9 Result lifetime); in `IDLE` (not in `REST`) the mode selector and both start buttons stay available so the athlete can change mode or duration while resting. In `REST` the mode selector and both start buttons are disabled; only `[ CANCEL ]` is available. **The `~` suffix (`TIME 01:23 ~`) is shown exactly when the result's `stopSource` is `voice` and the injectable flag `voiceStopResultApproximate` is true** (default true; set to false only by the build define `VOICE_STOP_APPROX=false`, used only after the keep-vs-correct decision is recorded in §20, §3). A manual STOP, the safety cap, an unknown source, and the Countdown `Set done` never show `~`.
- If voice STOP becomes unavailable during a run, keep `[ STOP ]` prominent and show the `Voice unavailable` label and, for fixable microphone faults, the §8 message card.
- Dark, high-contrast, large digits, no unnecessary animation, screen awake. Active workout screens are portrait-only; a test prevents rotation.

## Manual fallback
`[ Start manually ]` and `[ Cancel ]` (where the state permits; not in `REST` for Start) work when permission is denied, the model fails, or the engine is unavailable. Voice failure never crashes the timer.

## Settings
```text
Halfway cue (Countdown): ON/OFF (default ON)
Mode: countdown | count-up (none selected at launch; chosen each time)
Countdown duration: 20 | 30 | 45 | 60 (default 30)
Language: en (fixed; no selector yet)
After a set: Wait for me (fixed in 0.0.1; `Listen for Go` is deferred, §0 Release map)
```
Changes apply as follows: mode and duration only in `IDLE`; the Halfway cue is read when a set starts (a change during a set applies from the next set); `After a set` is read when `REST` ends. `Voice Stop` is not a user setting. **Settings live in memory only and return to their defaults at every cold start; nothing is stored (§9 Result lifetime).** An informational `About testing` screen is reachable from Settings.


## Wakelock (`wakelock_plus`)
The wakelock is a **computed state** (single definition), not a set of ad-hoc acquire/release calls: it is held if and only if at least one of these holds, and released the moment none does — (a) the state is `LISTENING_FOR_START`, `PREPARING`, `RUNNING`, or `REST` (`COMPLETED` keeps the current state; `ERROR` and `CANCELLED` hold nothing); (b) `Check voice` is running (§8; it holds only while it runs); (c) the **result hold** is active. The **result hold** starts each time the controller enters `IDLE` from `REST` (the normal end of a set, or Cancel in `REST`, which keeps the result) and lasts `resultVisibilityTimeout` (default 45 s, injectable) so the athlete at 1–2 m can read the result. Its timer is cancelled (the timer itself never releases anything then) on: entering `LISTENING_FOR_START` or `PREPARING`, starting `Check voice`, leaving the screen that shows the result, and `AppBackgrounded`; so a stale expiry can never release a wakelock that (a) or (b) now needs. Entering `IDLE` any other way (listening timeout, failed voice entry, error cleanup, Cancel before `REST`) starts no result hold. Acquire immediately before entering `LISTENING_FOR_START` (and while waiting for unfinished initialization after a voice request) and before `PREPARING` on the manual path. Prewarming does not acquire it. Wakelock does not guarantee background execution; the MVP assumes the screen stays on and the app visible (known limitation of 0.0.1, §2). Test each acquire/release edge.

## Lifecycle
- **Cancel the session** (cleanup §9) on: `AppBackgrounded` (answering a call sends the app to the background); a verified permanent audio-focus loss. A call that is only ringing, declined, or shown as a banner while the app stays in the foreground is **not** a cancel: the clock continues, voice is off (`MIC-206`), and manual STOP works. Microphone or route loss reported through `errorStream` in `LISTENING_FOR_START` is a `MIC-2xx` fault (§8): it ends voice entry and returns to `IDLE`; it is not a session cancel.
- **Precedence.** A verified permanent interruption signal (permanent audio-focus loss reported by the audio-focus listener) cancels the session (`InterruptionBegan(permanent)`). A transient loss, or a focus-loss error reported only by the speech engine without a permanent signal, is `MIC-206` (§8): voice off, timer continues. If both arrive in the same dispatch turn the permanent signal wins. Classify by the verified event type, never by guessing from an ambiguous error message.
- **`AppBackgrounded`** is raised only for Flutter `AppLifecycleState.paused` / `detached`. `inactive` and `hidden` never cancel (notification shade, dialogs, overlays). When accepted: synchronously `speechEpoch++` and mark shutdown in progress before async cleanup; shutdown-generated errors are stale. **Known limitation (0.0.1):** because the session is cancelled on `AppBackgrounded`, locking the phone, pressing Home, or switching apps during a session ends it; the time is lost and nothing is saved. Running with the screen off needs a foreground service with a persistent notification, is unreliable on some vendors' devices (for example Transsion) without a manual battery exemption, and is deferred to a later version. `AppBackgrounded` also clears the displayed result (§9 Result lifetime); `inactive` and `hidden` do not.
- **Do not cancel** on transient focus loss or ducking during `PREPARING`/`RUNNING`/`REST`; the monotonic `Clock` stays the truth. In Count-up with STOP listening, a transient interruption reported by the engine is `MIC-206` (an environmental fault, §8).
- A cue interrupted by focus loss is skipped and logged, never replayed.
- Verify the exact lifecycle sequence for screen lock, Home, app switcher, and notification shade on the installed Flutter version (API 26 emulator and reference device); record the result. If `paused` does not arrive on screen lock on a device, use the verified platform signal; do not invent one.
- If the installed audio API cannot report focus changes, record that and implement only background / screen-lock / call handling.
- On return, show a deterministic `IDLE`; voice entry is available only if the engine is healthy. No automatic recovery promise. No result is shown (§9). With Extension S active, `AppBackgrounded` still cancels the running round and clears `lastCompletedResult` exactly as stated here; only the session's completed rounds follow §21.2 (decision D-S1).

## Privacy
The app must not: require an account, use a backend, upload recordings, store raw recordings, use location/camera/cloud speech, or use analytics/crash SDKs.

Correct statements: *"The app does not send user audio to an application-owned server."* Say "the app has no internet permission" only after release-manifest verification (Appendix B). Never claim "audio never leaves the device" unless the exact speech stack is verified fully on-device.

Product principle: nobody should fear being listened to. The microphone is opened only while a listening label shows (§12); after the label disappears it normally closes within a fraction of a second, and if the stop fails or hangs it may stay open until a later stop succeeds or the app process ends (§4 Honest limit, §8); the app then shows `Voice unavailable`, never claims the microphone is off, and the owner's diagnostics record `ENG-106`. Audio is processed in memory and discarded. **The app stores nothing on the device: no settings, results, or recordings, and (from version 0.0.2) no log files.** The workout result and the settings live in memory only and are gone when the app closes. Nothing is used to learn dialect or personalize recognition. Version 0.0.1, the owner's own build (`DIAG_DETAIL`), keeps a local `diagnostics_log.txt` of the last 5 faults for debugging: it contains no audio and no workout result, is never sent anywhere, and can be deleted in Settings → Diagnostics. (The packaged speech model may be copied once to the app's private storage by the engine integration; it contains no user data.) No feedback form; the only feedback channel is an email shown as text (at most a plain `mailto:` with an empty body). Android's own microphone indicator is never suppressed. Voice personalization/dialect adaptation is deferred and needs its own decision and privacy review. With Extension P activated, the stored-data statement above is replaced as §21.5 states.

## Diagnostics (local only)
A bounded in-memory ring buffer (injectable size, default 200 records) of `AppFault` records (§8 Fault codes) and key events: timestamp, mode, state, event, engine id, model id, code, stage, detail, raw message, timeout type, build version, Android API, device model. No raw audio, no network, no analytics. Viewable and copyable (clipboard) in debug builds, in `DIAG_DETAIL` builds (Settings → Diagnostics and the Fault panel); **never shown in public builds**. In public builds it is never written to disk and is lost when the app closes. In `DIAG_DETAIL` builds the last 5 faults (each with its event snapshot, §8) are additionally persisted in `diagnostics_log.txt` (single definition: §8 Fault codes and detail); that file is local-only and is deleted by `[ Delete log ]`. "Log a diagnostic" means this record. **This is the owner's main tool for finding out why voice failed.**

---

# 14. Speech Engine, Selection, and Test Levels

## Interface

```dart
enum SpeechErrorCode {
  permissionDenied, permissionPermanentlyDenied,
  microphoneInUse, microphoneBlocked, routeLost, focusLost,
  silentInput, inputStalled,
  modelFileMissing, keywordFileInvalid, nativeLibraryFailed, commandUnsupported,
  invalidConfiguration, engineFailure, lifecycleTimeout, unknown,
}

final class SpeechError {
  const SpeechError(this.code, {this.stage, this.detail, this.rawMessage});
  final SpeechErrorCode code;
  final String? stage;       // 'initialize' | 'startListening' | 'listening' | 'stopListening' | 'dispose'
  final String? detail;      // path, sizes, ABIs, elapsed ... (§8 table)
  final String? rawMessage;  // exact platform/native text
}

abstract interface class SpeechEngine {
  Future<void> initialize();                    // must never open the microphone
  Future<void> startListening(Set<VoiceCommand> commands);
  Future<void> stopListening();                 // releases the microphone; not disposal
  Stream<VoiceCommand> get commandStream;
  Stream<SpeechError> get errorStream;
  Set<VoiceCommand> get supportedCommands;
  Future<void> dispose();
}
```

- Normalize errors at the engine boundary. Never infer a permission category from an ambiguous message; unclassified = `unknown` = voice failure (§8). The raw/native text is stored in the diagnostics buffer and shown only in `DIAG_DETAIL` builds (§8 Fault codes); public builds never show it. `SpeechController` maps `SpeechErrorCode` → `AppFault` code through one table (`fault_catalog.dart`).
- The interface exposes no Sherpa/ONNX types; `TimerController` never imports an engine package. `SpeechController` checks `supportedCommands` first and never requests an unsupported command (§10). If an engine nevertheless rejects a command as unsupported it raises `commandUnsupported` (→ `ENG-109`, a capability limit that sets `voiceStopAvailable = false` without breaking the engine); `invalidConfiguration` is for any other bad configuration. The Voice Test records an unsupported STOP as `unsupported`, not a failure.
- Engine tunables (threshold, boost, sensitivity) go only through a generic **diagnostics interface** (engine id, model id/hash, name/value parameters, optional setter by name), created in Milestone B. No parameter names are invented in advance; each engine maps its verified parameters when integrated.
- Phone microphone only. Verify per engine whether it captures the microphone itself or expects app-supplied PCM. Exactly one component owns the microphone at a time. Speech input format is the model's (commonly 16 kHz mono); audio assets have different formats.

Implementation: `SherpaKwsSpeechEngine` (the engine of record). A second engine is possible only through Appendix A; only one engine ships.

## Engine of record

**Sherpa-ONNX KeywordSpotter.** Official Android/Dart/Flutter examples; keywords are supplied through the model's supported keyword mechanism, generated with that model's tooling; a phrase must be representable by the model's tokens. Primary model candidate: `sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01` (verify it loads and that its tooling can produce `Go` and `Stop`). Do not hand-write Sherpa keyword token definitions; use the model's official tooling.

**Second Sherpa model, for evaluation only:** `sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20` (English words through the phone lexicon `en.phone`; the docs page lists `chunk-8` at 160 ms and `chunk-16` at 320 ms latency). B0a verifies that both models load; the second is not shipped unless its license record is `clear` (§15). A model change consumes a tuning attempt (below).

An alternate engine (onnx-wakeword and others) is **not** evaluated in v1 unless the owner activates Appendix A, typically after a Sherpa DROP or a Gate 0H failure within the tuning cap.

## Gate funnel (cheap to expensive)

Gates: 0A, 0B, 0C, 0E, 0F, 0H. The second-engine process is in Appendix A.

- **0A Availability (no production code):** English `Go`/`Stop` representable; exact model files; license and training-service terms (any audio uploaded?), and the license, training data, and terms of the exact model archive you will ship (§15 Model license record); Android runtime and Flutter path; `minSdk 26` and ABIs incl. emulator ABI; PCM requirements and capture ownership; size impact; project maturity; vendor claims recorded as *claims*. Outcome PASS/DROP in §20. An engine without STOP is not dropped here but cannot ship the full MVP.
- **0B Offline fixture (AI) = B0a:** WAV → KWS → result → reset. No microphone claims. Mark audio-word evidence **unverified** if no fixture exists.
- **0C Live smoke (human) = B0b:** reference device, quiet room, 1–2 m, Airplane Mode; microphone PCM reaches KWS, START detected, STOP detected if supported (diagnostic only), stream resets and continues. Failure → DROP the engine.
- **0E Full matrix (reference device):** §14 targets below, plus model load, RAM, CPU, battery, app size. RAM/CPU are report-only unless numeric thresholds were recorded *before* measuring.
- **0F Real workout behavior:** ≥30 aggregate minutes, ≥50 first-attempt START utterances per speaker, heavy breathing and self-talk, no music. **Own-audio check:** zero activations during or within 1.5 s after any app sound (incl. re-arm after the finish beep); otherwise raise `rearmDelay` and rerun. The window starts at the end of the audible sound (external recording), not at a callback. The same check applies to the Count-up STOP listener: the listener opens at `settleAnchor + stopListenSettleDelay` (§10; 500 ms provisional), so the part of the window after the listener opens is exactly what is tested; any activation there raises `stopListenSettleDelay`, and Gate 0H evidence sets the final value.
- **0H Voice Stop:** on the selected engine (see levels below).

**One engine ships.** Sherpa is the engine of record. If a second engine is ever activated, Appendix A defines admission, ranking, and removal of the loser.

**Fallback chain on a 0H failure:** next documented threshold/boost configuration on the same engine (within the tuning cap defined below) → **conditional alternative phrase branch (recorded human decision, STOP or START):** test a longer phrase for the failing command (e.g. "Stop timer"; a START example is chosen by the owner) with a full gate rerun (0E/0F for START, 0H for STOP); it counts as one configuration for the tuning cap → activate Appendix A (only by recorded human decision, and only for an engine verified to support `Stop`) → NO-GO for the full MVP (NO-GO also means: speech needs Internet, the model cannot load on API 26, START is not reliably recognized (after the alternative-phrase branch above), or false positives are irreducible — it triggers this chain, not an endless experiment). A manual-Stop-only build is an internal or explicitly labeled reduced product only by recorded human decision. The same chain (configuration → alternative phrase → Appendix A → NO-GO) applies to a START failure at 0E/0F.

**License-blockage chain (§15):** as soon as a candidate model's license record is `unclear` at Gate 0A/B0a, send the written clarification request to the maintainers immediately (record date, recipient, and reply in §20) and run it in parallel with all subsequent gates; **in parallel, begin evaluating alternative models with clear licensing terms (Apache-2.0, MIT, or explicit commercial-use grants) as a fallback — record candidates and their gate status in §20**; the release gate only checks the outcome. If the selected model's license record is still not `clear` at the release gate → replacement model with its own full gate rerun (0E/0F/0H; a model change consumes a tuning attempt per the cap below) → NO-GO for that release level. Level 2 (store) requires `clear`; Level 0/1 may use an `unclear` model only by recorded owner decision naming the risk; a `blocked` model is never shipped at any level.

**Tuning caps:** at most 3 complete parameter configurations per engine for START across 0E/0F combined, and 3 for STOP across 0H. A configuration is one tuple of all adjustable parameters, including the keyword phrases. A model change or a phrase change consumes an attempt; neither resets the cap.

## Test levels (match the audience)

**Level 0 — Personal (owner only).** B0b smoke test passes; ~20 START and ~20 STOP live attempts in a quiet room including exertion; 15 minutes of live false-activation observation; alert audibility checked on the reference device. Purpose: use it yourself and find faults. Nothing is claimed publicly.

**Level 1 — Friends (closed test).** Requires:
- START (Gate 0E/0F): ≥2 speakers, 50 attempts each, overall ≥ 90%, each speaker ≥ 80%.
- STOP (Gate 0H, Level 1 criteria): ≥2 speakers, ≥100 qualifying first-attempt utterances (50 each) with **zero misses**; zero confirmed false stops in an automated recorded run (≥30 min) and a live run (≥15 min); p95 ≤ 1.5 s. Results are labeled **indicative** (95% upper bound ≈ 3/N).
- Controlled false-positive run: 60 minutes (20 silence/resting speech, 20 exertion, 20 mixed with command-like near-words) with 0 false activations.

**Level 2 — Store (public release).** Requires:
- ≥3 speakers; STOP ≥300 qualifying utterances (≈100 per speaker) with zero misses; false-stop observation ≥3 aggregate hours (recorded + live) with **zero** false stops; latency p95 ≤ 1.5 s.
- Natural false-positive observation for START (`Go`) ≥3 aggregate hours with rate ≤ 1/hour (report "verified zero events" separately and only if exactly zero).
- All release checks in Appendix B.

**Definitions.** A *qualifying utterance* is a distinct, intentional first-attempt STOP at 1–2 m; replayed or augmented copies count once; exclusions only for speaker error, each logged. A STOP recognized only on a second try is a **miss**. A false stop ends a workout irreversibly: one confirmed false stop fails that configuration and the stage is repeated after any model or parameter change. Any STOP activation within 1.5 s after the **audible end** of `start_confirm` (external recording, as in 0F) counts as a false stop; the STOP listener under test opens at `settleAnchor + stopListenSettleDelay` (§10), the only anchor allowed (not the timer start, not the end of the last sound as reported by a callback, not engine readiness). Raise `stopListenSettleDelay` until none occurs and record the value. Do not relax the zero-miss criterion by exception. Report exact counts, exclusions, misses, p50/p95, recorded hours, and live minutes; never hide a miss.

**Why the STOP bar differs from START (≥ 90%).** A missed `Go` costs the athlete a button press while standing at the phone; a missed `Stop` costs breaking off mid-effort to walk 1–2 m to the phone — and voice Stop is the Count-up differentiator (§1). The miss criterion is strict where the fallback is expensive; the false-stop criterion is zero everywhere because a false stop ends a workout irreversibly.

## Targets (apply at Gate 0E unless a level above says otherwise)

- START (quiet): overall ≥ 90%, each speaker ≥ 80%. Pilot may use 2 speakers; the full 3-speaker rerun is required before any public release.
- **Latency:** end of the spoken keyword to the confirmation beep (START) or finish-beep onset (STOP): p95 ≤ 1.5 s. Measure externally (a phone/camera capturing speaker and device beep in one continuous recording; annotate; report p50/p95; do not mix methods). Use release or profile builds, never debug. Record build variant, Flutter/Dart versions, engine and model versions, device, and API.
- **Model startup:** ready ≤ 2 s after process initialization on modern Android; report API 26 separately.
- **Battery:** Countdown ≤ 3% per 10 minutes (listening + workout, microphone off during `RUNNING`). Count-up with voice Stop is a **separate** scenario (microphone on during `RUNNING`): report actual consumption and duration; its release threshold is set only after the first measurement is recorded. Also a full 5-set session with listening before each set.
- **App size:** ≤ 60 MB if feasible (measure AAB, per-ABI APK, native libs, models, keywords, audio); otherwise record `CONDITIONAL` with the real size.
- Do not silently change a target after seeing results; record a decision and mitigation. `CONDITIONAL` means a target is missed but a bounded mitigation and a re-test condition are written in §20.

## Device policy

- **Reference device:** the owner's primary physical Android phone; all accuracy, latency, false-positive, battery, and audio measurements are taken on it.
- **Emulator** (API 26 and one recent API) is valid only for build, install, launch, native-library/model loading, permission flows, timer/state/UI logic, lifecycle, and manifest checks. Never use it as evidence for accuracy, latency, battery, or thermals.
- Other physical devices are optional; record each separately. Any untested device class is documented as "compatibility verified on emulator only; voice accuracy unverified".
- ABI check: if the engine's native runtime lacks the emulator ABI, record an environment/ABI blocker for that candidate; it is not a generic failure.


---

# 15. Models, Dependencies, Permissions

## Model assets
Do not guess paths. Follow the selected engine's official Flutter/Dart example for: packaging model files as assets, copying them to a readable local path (`path_provider` if the example needs it), creating the config with real paths, loading the keyword file/buffer, and releasing resources. Keep model loading out of widgets; load once per process where practical. Never reuse one candidate's paths/formats for another. Record model file names and hashes per engine.

## Official Sherpa references (for B0a — reference, not permission to invent APIs)

- Android KWS sample: `https://github.com/k2-fsa/sherpa-onnx/blob/master/android/SherpaOnnxKws/app/src/main/java/com/k2fsa/sherpa/onnx/MainActivity.kt` (lifecycle and resource ownership)
- Dart API examples: `https://github.com/k2-fsa/sherpa-onnx/tree/master/dart-api-examples`
- Flutter example index: `https://github.com/k2-fsa/sherpa-onnx/blob/master/flutter/sherpa_onnx/example/example.md`
- Dart KeywordSpotter example: `dart-api-examples/keyword-spotter/bin/zipformer-transducer.dart` (`createStream()`, chunks, `isReady()`, `decode()`, `getResult()`, `reset()`, `free()`)
- C++ API (semantics only): `sherpa-onnx/csrc/keyword-spotter.h`

If a path moved, search the official `k2-fsa/sherpa-onnx` repository for the current equivalent.

**B0a procedure (AI-only, no live microphone claim):** verify the resolved `sherpa_onnx` version; the exact model and token files; the keyword-generation mechanism; reproduce the official lifecycle; load the model and create a stream; feed a WAV fixture where supported; verify result format and `reset()`; verify resource release; record every API/model mismatch; record whether results expose keyword start/end or token timestamps (a finding only — the Count-up displayed time stays the instant STOP was accepted unless the human records a correction decision in §20; see the bias note in §3). Also record whether the integrated stack lets the app observe microphone frames or a PCM peak (needed for `MIC-210` / `MIC-211`, §8). **STOP and report** if the lifecycle can't be established, files can't be obtained, keyword generation can't be verified, the Dart API differs from the assumed interface, the model can't accept the needed PCM format, or the combination can't be built for Android. B0a is not proof of live accuracy; without a fixture, mark recognition evidence **unverified**.

## Dependencies
Use only what the selected implementation needs. Versions below are starting hints; resolve with `flutter pub add`, record exact resolved versions in the README, and commit `pubspec.lock`, and record changes in §20. Never assume an API from the version number.

```yaml
dependencies:
  flutter: { sdk: flutter }
  flutter_riverpod: ^3.4.3
  sherpa_onnx: ^1.13.8
  audioplayers: ^6.8.1
  wakelock_plus: ^1.8.1
  flutter_localizations: { sdk: flutter }
```

Add `intl` only after verifying the SDK's localization setup. Add `record` and `permission_handler` only if the official integration needs them; add `path_provider` for `diagnostics_log.txt` (§8) in `DIAG_DETAIL` builds, and wherever the official integration needs it (the app never stores user data). Do **not** add `shared_preferences` or any other package whose purpose is storing data, `flutter_tts`, `firebase_analytics`, `firebase_crashlytics`. An alternate engine's runtime dependencies are added only if Appendix A is activated. Android application ID: use a clearly temporary `com.example.voiceworkouttimer` unless the owner provides one; record it.

## Model license record

For every model shipped **or** evaluated, keep in `third_party/<model-id>/`: the archive's `README.md` and any license or notice file, the archive SHA-256, the training data and its stated terms, and the source and license of any lexicon (e.g. `en.phone`). Record a verdict in §20: `clear`, `unclear`, or `blocked`.
- **Automatic integrity check (mandatory).** A script in the repository (for example `tool/verify_models`) compares the SHA-256 of each model archive, and of every model, token, keyword, and lexicon file that goes into the app, with a pinned list committed next to `third_party/<model-id>/`. It runs before the archive is unpacked into the project, at the start of every release build, and in the Appendix B release verification, and it **fails the build on any mismatch or on any model file that is not on the list**. Changing a pinned value is allowed only with a new `[owner]` line in §20 naming the new file, its source, and the review done.
- Verdicts: `clear` = the available evidence is enough to ship under the release criteria; `unclear` = rights or terms are not settled yet; `blocked` = a known condition or conflict forbids the intended use.
- An owner decision accepts a risk; it does not make a model `clear`, and giving a build to friends is distribution. Level 0 and Level 1 may use an `unclear` model only by a recorded owner decision naming the risk. Level 2 (store) requires `clear`: a written clarification from the maintainers or a model with explicit terms. A `blocked` model is never shipped.
- A license statement that exists only in a README front matter or a hosting page is recorded as stated, but is not treated as a clarification of the training-data terms.
- Known state, checked when this spec was written (re-verify before shipping): the English model archive (`sherpa-onnx-kws-zipformer-gigaspeech-3.3M-2024-01-01`, SHA-256 `f170013b4716e41b62b9bfd809687c207cef798ef9bc6534d524e17af9b6561a`) contains a `README.md` stating Apache-2.0 and no separate license file; it was trained on GigaSpeech XL, whose dataset terms are discussed in `k2-fsa/sherpa-onnx` issue #3802 as non-commercial research use and are unresolved for models. The `zh-en-3M-2025-12-20` archive reportedly has no README or license file and states no training data (reported in the same issue; not verified by us).
- Apache-2.0 distribution normally needs the license text and any notices shown with the app (Appendix B).

## Android permissions
Likely only `android.permission.RECORD_AUDIO`. Do not add network access merely because packages often include it. Set `android:allowBackup="false"` in the app manifest: the app stores no user data (the `DIAG_DETAIL` fault log holds none), and this keeps anything a package writes from being copied to the user's Google account. After building, inspect the merged manifest, record every permission, and verify `allowBackup` is false (plus any `dataExtractionRules` / `fullBackupContent` on recent API levels).

## Offline verification (before any release claim)
Airplane Mode → launch → Countdown: say `Go`, confirm detection, start beep, timer, halfway tone, finish beep. Count-up: say `Go`, run, say `Stop` (also verify manual STOP), confirm the result. **With `After a set = Listen for Go`**: start set 2 by voice without touching the phone (re-arm within `afterRestListeningTimeout`). With the default `Wait for me` the second set requires a manual press; hands-free applies within a set (voice Go/Stop), not automatically between sets unless `Listen for Go` is selected. If required voice control fails in Airplane Mode, the engine is unacceptable.

## Claims
Never claim an unverified result. Exact wording for support, environment, and testing claims is in Appendix B.

## First run
Open → (Level 1 and Level 2 builds only: the dismissible testing notice, shown at every cold start because the app remembers nothing; owner builds show it only on `About testing`) → choose a mode (none is preselected) → choose the duration if Countdown → `START VOICE SESSION` → grant permission when requested → Get Ready → say "Go". Keep the screen on and stay in the app during the session (known limitation of 0.0.1, §2). No mandatory calibration. If permission is denied, manual mode stays usable.

---

# 16. Architecture, Structure, Build Order

## Clock, Scheduler, Halfway

- `abstract interface class Clock { Duration now(); }` — a monotonic source (wrap a `Stopwatch`); `FakeClock` returns a manually controlled `Duration`. Never use `DateTime.now()` for elapsed workout time; never invent a Dart `Instant` type. Verify whether the monotonic source advances while the device sleeps; because an interrupted session isn't resumed, the shown value after an interruption is the last known elapsed and is never extrapolated.
- Elapsed is always `clock.now() - startTime`; never `remaining--`.
- **`startTime` definition.** `startTime = preparingStartedAt + startDelay` (§5; default 0), computed when `PREPARING` begins. A late `tick()` moves the moment `RUNNING` is *observed*, not `startTime`: elapsed, `halfwayMs`, and the Countdown finish time (`startTime + totalDurationMs`) are all measured from it, so scheduler lag never changes the workout length. Before handling any input (manual STOP, voice Stop, Cancel, Start) the controller first runs the same due-event evaluation as `tick()`, so an input cannot be processed while the state is still `PREPARING` after logical `startTime`. Stop is accepted only in `RUNNING`; `finalElapsed` is never negative.
- Do not use `Future.delayed` as the source of truth. Build an injectable scheduler around `Clock`; production may poll every ~50–100 ms (not part of correctness). Required test hook: `void tick();` — it only observes time and triggers due events; tests advance `FakeClock` then call `tick()`. Audio duration never defines timer correctness.
- UI may refresh once per second; no decimals.
- **Halfway:** threshold is exactly 50% of the duration in milliseconds: `final halfwayMs = totalDurationMs ~/ 2;` (20→10.0 s, 30→15.0 s, 45→22.5 s, 60→30.0 s). Emitted exactly once, on the first `tick()` at or after the threshold, behind an idempotent guard; dispatch may lag by up to one scheduler interval. The finish event follows the same rule.
- `countUpMaxDuration` is injectable (default 30 min; tests override it with a small value).
- If a late `tick()` finds both `halfway` and `finish` due, process them in time order but skip the halfway cue (log it); never play `halfway` after `finish`; `finish` fires once.

## Riverpod 3
Use modern Riverpod 3 APIs (`NotifierProvider`, `AsyncNotifierProvider`, `Provider`); no legacy `StateProvider`; verify the installed docs. UI contains no core timer logic. The injectable `Scheduler` is a core service injected alongside `TimerController`.

```text
TimerController    selected duration, phase, elapsed, halfway flag, sessionId
SpeechController   sole owner of SpeechEngine lifecycle (serialized), failure policy §8, permission state
SettingsController halfway cue, mode, duration (in memory; defaults at every cold start)
```

Responsibilities stay separate: **Speech** detects a command; **Audio** gives feedback; **Clock** defines elapsed time; **Scheduler** evaluates due events; **TimerController** owns workout behavior; **UI** displays state and fallback controls.

## Suggested structure

```text
l10n.yaml                      (follow the installed SDK's generated-localization tooling; don't invent fields)
lib/
├── main.dart
├── app/                       app.dart, theme.dart, lifecycle_observer.dart
├── l10n/app_en.arb
├── core/
│   ├── enums/                 timer_phase.dart, voice_command.dart
│   ├── models/                speech_error.dart, app_fault.dart
│   ├── language/              app_language.dart, language_pack.dart
│   ├── services/              speech_engine.dart, sherpa_kws_engine.dart,
│   │                          speech_engine_diagnostics.dart (Milestone B, generic),
│   │                          audio_feedback_engine.dart, audioplayers_feedback_engine.dart,
│   │                          diagnostics_buffer.dart, fault_catalog.dart,
│   │                          clock.dart, scheduler.dart
│   └── utils/                 duration_formatter.dart
├── features/
│   ├── home/                  home_page.dart
│   ├── timer/                 timer_controller.dart, speech_controller.dart, timer_state.dart,
│   │                          timer_page.dart, widgets/{timer_display, duration_selector, voice_status}.dart
│   ├── voice_test/            voice_test_controller.dart, voice_test_page.dart
│   ├── diagnostics/           voice_check_controller.dart, voice_check_page.dart, voice_status_line.dart (DIAG_DETAIL only)
│   └── settings/              settings_controller.dart, settings_page.dart
docs/ERROR_CODES.md           (every fault code: meaning, cause, fix; kept in sync by a test)
third_party/<model-id>/       (model README, license notices, archive SHA-256; §15)
assets/audio/common/           ( directories listed explicitly in pubspec.yaml)
test/
├── fakes/                     fake_clock, fake_speech_engine, fake_audio_feedback_engine, fake_language_pack
└── timer_controller_test.dart, timer_state_test.dart, duration_formatter_test.dart
```

Optional E2E projects live outside `lib/` and `test/` and never ship. The structure is guidance, not permission to create duplicate state models or parallel timer controllers.

## Voice Test screen (development / Phase 0; engine-neutral)

The Voice Test screen is the engine-evaluation harness (counters, tags, copy as text); `Check voice` (§8) is the quick health check for fixing faults. They are two screens that share the same `SpeechController`.


Shows: engine id; model id/hash; keywords START/STOP; STOP support (supported/unsupported); status; last result; score, threshold, boost (each may be `n/a` when the engine does not provide it); latency (a numeric field on each attempt row, entered by hand from the external measurement); Android version/API; device model; audio input format; expected vs actual command; counters (Hits / Misses / False positives); tags (distance, noise, music, speaker); buttons `Start Listening`, `Stop Listening`, `HIT`, `MISS`, `FALSE POSITIVE`. `Start Listening` opens only through `canOpenMicrophone(voicetest)` and is refused while `alertPending == true`. It can run the **Count-up STOP scenario** (START voice or manual, elapsed counter, microphone listening for STOP only, exertion tags). It runs against any `SpeechEngine`, so every candidate is measured with the same screen, procedure, and result format. Engine parameters are read/written only through the generic diagnostics interface.

**`[ Copy results ]` for every diagnostic run** (plain text to the clipboard, one row per attempt, tab-separated; no file is written and no dependency is added; the rows live in memory and are lost when the app closes, so copy them before leaving). Columns: `timestamp, engine_id, model_id_or_hash, device_model, android_api, keyword, expected_command, actual_command, result (HIT/MISS/FALSE_POSITIVE/unsupported), score (nullable), threshold (nullable), boost (nullable), speaker (optional tag), distance_m, noise, music, latency_ms (nullable; entered on the attempt row from the external method), mode (countdown|count-up), scenario (start|stop|manual_stop|cancel|safety_cap), notes`. An unsupported command is never recorded as MISS. Not shipped in public builds. Threshold winners are recorded in §20; no raw model-tuning controls in the consumer UI.

## Build order — MUST FOLLOW

**Milestone A — Timer Core.** No Sherpa, no live microphone, no polished UI.
**Milestone A-pre — Audio Semantics Spike (AI, required, before A).** Facts only; no Timer Core code. In a throwaway project outside `lib/` and `test/` (never shipped), with the installed `audioplayers`, verify Milestone A2 items 1–2 below (`stop()` semantics including whether it can cut a later sound, and request-to-audible-start latency) and record each as a **fact** in §20. Latency is measured on the physical reference device (an emulator figure is not usable for `audioTail` and is recorded only as a note). Result: a *provisional* `audioTail` for Milestone A and a yes/no on the §6 rule 6 isolation requirement. If a cue cannot be played at all, or stop isolation (rule 6) cannot be achieved, STOP and report before any Timer Core work. Milestone A then codes the gate against the `AudioFeedbackEngine` interface and its fakes. A2 later re-confirms items 1–2 on the production engine and adds items 3–4 and the final `audioTail`.
Flutter project → Riverpod → `FakeClock` → `FakeSpeechEngine` → `FakeAudioFeedbackEngine` → state machine (both modes) → `SpeechController` (§8 failure policy, fault catalog, `docs/ERROR_CODES.md`) → Halfway → Count-up elapsed + manual Stop + fake-voice Stop → finish/REST → `IDLE` with the result visible and the microphone closed (`Wait for me` only; `Listen for Go`, `rearmDelay` and any re-arm are deferred, §0, and are not built in 0.0.1) → second-set loop → `LanguagePack` abstraction (English pack) → unit tests. Includes all injectable constants with placeholder defaults. Do not block on Sherpa; no Bluetooth or extra languages.

**Milestone A2 — Audio Player Semantics (AI).** Build `AudioPlayersFeedbackEngine` (the real implementation behind `AudioFeedbackEngine`) as part of this milestone, then verify against the installed `audioplayers` version (official docs, installed source, emulator, reference device when available) and record each finding as a **fact** in §20 (items 1–2 were already recorded provisionally by A-pre; A2 re-confirms them on the production engine and adds items 3–4):
1. whether `stop()` stops a playing sound and how long it takes (p50/p95, ≥20 runs), whether a stop still pending can cut a sound requested after it (the isolation rule of §6 rule 6), and whether one reusable player or one player per cue is safer; whether two overlapping plays are both audible and neither is cut by the other (the §6 overlap contract; report yes/no);
2. p50/p95 latency from play request to audible start (≥20 runs, **on the physical reference device**, also over a Bluetooth output when one is available), measured with the external recording method of Gate 0F; and the actual audible length of each asset against its hard maximum;
3. audio focus and ducking behavior with and without other media;
4. whether the alert (media channel) is audible with the ringer in silent/vibrate and Do Not Disturb, with media volume zero, and with a connected Bluetooth output; record the result and whether a different audio usage is needed.

Record `audioTail`: the p95 latency from item 2 plus the output/scheduling latency plus a safety margin (value and derivation in §20). The gate is the quiet estimate of §6 rule 3: an estimate, never proof, and it must pass the Gate 0F own-audio check (§11). If a later measurement shows a start latency above the recorded `audioTail`, enlarge `audioTail` only; no other rule changes. Mandatory cues always play. The app has no audio-mode switch and no completion-based path: if item 1 shows a cue cannot be played or stopped safely, STOP and report; do not invent semantics.

**Milestone B — Engine-neutral Voice Test harness.** Depends only on A. Build: generic diagnostics interface → Voice Test screen → engine selector → counters/copy-as-text/device info → diagnostics display → `Voice:` status line and `Check voice` flow (§8), verified with `FakeSpeechEngine`. Verify with `FakeSpeechEngine` (emitting START/STOP). No Sherpa/ONNX dependency; no polished timer UI.

**B0a → B2 (Sherpa).** B0a (§15) does not depend on B and may run in parallel with it, but B2 needs both; if B0a fails, STOP and report — the owner decides whether to activate Appendix A. B2 (after B0a passes): implement `SherpaKwsSpeechEngine` behind `SpeechEngine` and plug it into the harness using the verified model/keyword/microphone/reset behavior. If the verified KWS behavior doesn't fit the abstraction, stop and propose an interface change; no hacks.

**B0b (human).** Live-microphone smoke test (Gate 0C) on the reference device using the Voice Test build.

**C — Device Gate (human).** After B0b passes for at least one engine; Run the §14 matrix and Gate 0F on the engine.

**C2 — Voice Stop Gate (Gate 0H).** AI runs the automated recorded-audio part, the human the live part, on the selected engine. Do not share a build with friends without a Level 1 STOP pass and an explicit human instruction; public release additionally requires Level 2.

**D — Production integration.** May start for owner-only builds once B0b has passed for the selected engine (Level 0; voice Stop may be enabled for the owner's own testing). Builds shared with friends need 0H Level 1 for that engine; store builds need Level 2. Integrate the engine through `SpeechEngine`, `TimerController`, production timer (both modes). Voice Stop requires the engine that passed 0H.

**E — Polish.** Production UI localized through the language architecture, settings, approved audio assets, wakelock, permission UX, manifest verification, release cleanup.

**E2 — Optional external E2E (explicit human approval).** Host-side `e2e` tests for UI/state flow only, separate project, never shipped, never evidence for the voice/device gates. Never blocks release.

Order: A-pre → A → A2 → B → B0a → B2 → B0b → C → C2 → D → E → E2. For owner-only use, D may begin right after B0b; C and C2 must complete before any build goes to friends. This is the default sequence. Real dependencies: A needs A-pre; A2 needs A and A-pre; B needs A; B0a needs only the development environment; B2 needs B and B0a; B0b needs B2; C needs B0b; C2 needs C; D needs B0b (owner builds) or C2 (friends' builds); E needs D; E2 needs E and explicit approval.

---

# 17. Required Tests (Milestone A and later)

**0.0.1 scope of the tests below:** every test whose subject is `Listen for Go`, the re-arm path, `rearmDelay`, or a non-zero `startDelay` is skipped in 0.0.1 (§0) and is not part of its acceptance; after `REST` the app is always in `IDLE` with the microphone closed.

Use `FakeSpeechEngine`, `FakeClock`, `FakeAudioFeedbackEngine`. State-machine tests never depend on real WAV playback. Each bullet is a test group; write at least one test per behavior listed.

## Timer and modes
- 20/30/45/60 s completion; halfway exactly once (45 s at 22.5 s); finish exactly once; both fire on the first `tick()` at or after their threshold.
- Manual Start uses the same preparation timeline as voice Start (also from `LISTENING_FOR_START`, where it stops the engine and cancels the listening timeout); ignored in `REST` and around `COMPLETED` (handled per the §7 table); a duplicate Start cannot start two timers.
- Cancel in listening / preparing / running / REST: no stale cue starts, all scheduled events cancelled, a late `REST`-end timer ignored.
- Inputs around `COMPLETED`: nothing is dispatched while the state is `COMPLETED` (single synchronous step with `REST` entry); Cancel/Back/Start/Stop arriving around the stop are handled in `REST` with the completed result kept (§7 table).
- `After a set` defaults to `Wait for me`: after finish feedback and REST the app is in `IDLE`, no listener is open, the result stays visible, and the wakelock follows the §13 result hold (`resultVisibilityTimeout`, default 45 s). In `Listen for Go` the app re-arms per §11; both timeouts are injectable and expire to `IDLE` with the label `Stopped listening`, never clearing a shown or protected result. The mode and duration can be changed in `IDLE` while the result is visible.
- Pressing `START VOICE SESSION` after a set follows the single §11 definition: no `stopAll()`, then the grace wait; outcomes are asserted in the dedicated bullets of the Audio group.
- `unknown` ignored; STOP ignored in Countdown, `PREPARING`, `REST`, `IDLE`, and when `voiceStopCapability` is OFF or `voiceStopAvailable` is false.
- Default `startDelay` = 0: `PREPARING` is entered and left in the same synchronous step; `start_confirm` is requested once in that step; `startTime == T+0`; elapsed counts from the Accepted Start. Tests of `PREPARING` behaviors (Cancel in `PREPARING`, alert pending through `PREPARING`, deferred clearing of the previous result) inject a non-zero `startDelay` (e.g. 4000 ms).
- Clock start under a late scheduler: with `startDelay` = 4000 ms injected for this test, advance past T+4000 ms by 700 ms before the first `tick()`: the state becomes `RUNNING`, elapsed reads 00:00 plus 0.7 s (i.e. measured from the logical `startTime`, not from the late tick), `halfwayMs` and the Countdown finish stay at `startTime + …`, and a Count-up Stop accepted right after that tick gets `finalElapsed` measured from the logical `startTime`.
- Count-up: shared preparation timeline; counts up from `startTime`; no halfway; START during `RUNNING` ignored; manual STOP works with a hung engine; simultaneous voice and manual STOP complete once; `countUpMaxDuration` auto-stops through the same path, also with a broken engine or pending stop.
- Count-up stop: the final elapsed freezes synchronously before any engine await; `finish_beep` is requested once **immediately**; no Undo; a new session restarts from 0.
- A late `tick()` with `halfway` and `finish` both due processes them in time order, skips the halfway cue, and fires `finish` once.
- Halfway cue ON plays once at 50%; OFF plays nothing there, other cues unchanged; the setting lives in memory and returns to its default at the next cold start.

## Identities and lifecycle
- Stale speech events are ignored by `speechEpoch`; stale scheduler/audio callbacks by `sessionId`. A speech failure increments only `speechEpoch`; the clock, safety cap, and milestones continue.
- `AppBackgrounded` synchronously invalidates `speechEpoch` before cleanup from every active state; a shutdown-generated late `SpeechError` is ignored and does not break the engine.
- `inactive`/`hidden` never cancel; `paused` cancels with cleanup; a notification-shade pull-down in `RUNNING` changes nothing. Permanent interruption cancels; transient is ignored in `PREPARING`/`RUNNING`/`REST`. A ringing or declined call while the app stays in the foreground is transient (clock continues, voice off, manual STOP works); answering a call (`paused`) cancels.
- `sessionId` is invalidated when `REST` ends; a late scheduler or audio callback from the finished session does nothing in `IDLE` or `LISTENING_FOR_START`.
- `stopAll()` is called on explicit cancel, lifecycle cancel, and unrecoverable Timer/core error; a voice failure follows §8 instead.
- Wakelock (§13 computed rule): acquired before listening/manual preparation; prewarm does not acquire it; held exactly while (a) a state of §13(a), (b) `Check voice`, or (c) the result hold applies. With `FakeClock`: the result hold starts on `REST` → `IDLE` (also after Cancel in `REST`), releases at `resultVisibilityTimeout`, and starts nowhere else; starting a new session (`START MANUALLY`, `START VOICE SESSION`, a re-arm) or `Check voice` inside the window cancels the timer, and when the old expiry time passes the wakelock is **still held** by (a)/(b) (never released mid-workout); `AppBackgrounded` and leaving the screen that shows the result cancel it; every edge is asserted.

## Audio
**Test structure:** the gate is tested with `FakeAudioFeedbackEngine` and `FakeClock` (injectable `audioTail`); there is one gate rule and no mode parameter (§6 rule 3).
- A mandatory cue is requested at its due time with no preceding `stopAll()` and no wait, even while an earlier sound may still be audible (brief overlap allowed); a `stopAll()` that hangs or fails changes nothing; the microphone stays blocked until `quietAt`.
- `REST` end: `REST` ends at the finish beep's `quietAt` as read right after `playFinishBeep()` returned; no completion callback exists or is needed; with the asset missing (no play request) `REST` ends at once; `rearmDelay` is applied from `audioGateOpenedAt`.
- A delayed `start_confirm` cannot open the Count-up STOP microphone before `quietAt`; the clock still starts at `startTime`; a gate that opens within `stopListenerOpenGrace` still lets the listener open; gate still closed at grace expiry → `voiceStopAvailable = false`, manual STOP stays, session alert plays once.
- Play failure: a missing asset makes no play request and leaves `quietAt` unchanged (`AUD-301`); a failing play call records `AUD-302`, still extends `quietAt` (conservative), and nothing waits for it.
- `quietAt` rule: `quietAt = max(quietAt, requestedAt + hardMax(asset) + audioTail)` is evaluated when the player API call is invoked; with two plays the later one never shortens `quietAt` (older play longer than newer); overlapping plays keep the gate closed until the latest estimate; a newer play closes the gate again.
- The gate opens exactly when `Clock.now() >= quietAt` and no play request is pending, for the STOP listener, the START re-arm, and `START VOICE SESSION` alike; no path reads any other audio state for the microphone decision.
- `stopAll()` isolation: it is called only by the cancel paths (§10); no gate, press, re-arm, `REST`, or cue path calls it, and it never changes `quietAt`; a `stopAll()` in flight never cuts a cue requested after it (the fake models this rule).
- `START VOICE SESSION` with the gate closed by app audio: zero `stopAll()` calls and the grace wait (§11); the re-arm uses the same gate verdict.
- First run: fresh process, no play requested, press `START VOICE SESSION` immediately: it listens at once (`audioGateOpenedAt == null`, no `rearmDelay` wait).
- Settle anchor: gate opens **before** `RUNNING` → STOP listener opens at `startTime + stopListenSettleDelay`; gate opens **after** `RUNNING` → at `audioGateOpenedAt + stopListenSettleDelay`; never earlier, never delayed twice.
- Stale open decision: with a `FakeSpeechEngine` whose `initialize()` hangs, the gate is satisfied when `START VOICE SESSION` starts an engine retry; a new play is requested while `initialize()` is pending; when `initialize()` completes, `startListening()` is **not** called (the fresh re-evaluation finds the gate closed) and the attempt re-evaluates from the current state. Repeat for a pending `dispose()`/`stopListening()`, for the Count-up STOP listener, and for the START re-arm.
- Pending alert before opening: with `alertPending == true`, no path opens the microphone; the alert is requested as an ordinary play (extending `quietAt`), and only then does the gate follow `quietAt`; the alert's turn arrives at the first instant the gate is satisfied with no mandatory cue due within its own length.
- Delayed audible start: with a fake player that starts output late (request-to-start latency up to the approved p95), the gate does not open before `requestedAt + latency + hardMax`; a latency above the approved `audioTail` is reported as a test failure and the value is not approved.
- Duration over nominal: an asset longer than its hard maximum (e.g. `finish_beep` beyond 1500 ms) fails the asset test; the gate computation uses the hard maximum, never a nominal or recommended duration.
- Estimate expired: after `quietAt` has passed the gate is open and no log, UI, or diagnostic text claims silence was proven or that a sound ended.
- Gate ownership: with a `FakeAudioFeedbackEngine` reporting `audioGateSatisfied == true`, `microphoneOpenGateSatisfied` is false while `alertPending == true`, while the speech engine is broken (except the retry-eligible button path), and while `captureMayBeOpen == true`; the fake audio engine is never told about alerts or engine health.
- Fresh re-evaluation: `START VOICE SESSION` makes no `stopAll()` call; the retry's own health change does not abort the press; after every asynchronous step the press re-evaluates from the current state and proceeds on an open gate, or is refused on a closed one after the §11 grace wait with `AUD-305`.
- Button while `alertPending == true`: no extra alert, exactly one `AUD-305`, state stays `IDLE`.
- Grace wait (§11): a press while only app audio closes the gate shows `Just a moment…` (no alert, no fault record); if the gate opens within `pressGraceWindow` the press proceeds silently and calls `startListening()` once; at expiry it is refused with alert and `AUD-305`; a second press during the wait is ignored; a play requested during the wait is re-evaluated fresh.
- Alert timing: an alert pending behind a playing `finish_beep` (or, with `startDelay` > 0, through `PREPARING`) plays at the first instant `audioGateSatisfied` is true and no mandatory cue is due within its own length; no timeout constant exists; `alertPendingSince` is not used for any decision.
- Alert cancellation: an accepted Start discards a pending `entry` alert (rule 4); Cancel, lifecycle cancel, and Timer/core error clear any pending alert (rule 5); nothing else does.
- Play request pending: while a play request is decided but its player call has not been invoked, the STOP listener, the re-arm, and `START VOICE SESSION` each try to open; `startListening()` is **not** called until the call has been invoked and `quietAt` updated.
- Hung invocation: with a `FakeAudioFeedbackEngine` whose invocation stage hangs (decided, never invoked), `playRequestPending` becomes false at `playInvokeTimeout`, `AUD-302` (`invoke timeout`) is recorded, `quietAt` is extended conservatively from the abandonment, the gate then opens at `quietAt`, a later `START VOICE SESSION` press listens normally (voice recovers), and when the hanging stage later completes it makes **no** player call (no sound, no `quietAt` change, the generation token is invalid); a Count-up STOP listener or `START VOICE SESSION` listener that opened after the abandonment never hears an app sound from that request; `alertPending` never waits longer than these bounds.
- Audio initialization (§6 rule 9): with a fake whose `initialize()` fails, then one that hangs past `audioInitTimeout`: exactly one `AUD-303`, no retry, every later play request skipped silently with no fault and no `quietAt` change, `audioGateSatisfied` true, `REST` ends at once, the Timer and manual controls never wait, no alert plays and the Fault panel shows the fault; a late success is ignored.
- Press grace worst case: with a finish beep whose invocation hangs until `playInvokeTimeout`, the gate opens at 3000 ms after the decision; a press made at the decision instant is not refused before then (`pressGraceWindow` 3000 ms), and a shorter injected window is refused with `AUD-305` as documented.
- `REST` end after an abandoned finish beep: `quietAt` as read right after `playFinishBeep()` returned already includes the abandonment extension, and nothing can change it later; the `REST` timer is not re-armed and never fires early.
- No gap: from the decision of a play request until its player call is invoked and `quietAt` updated, `audioGateSatisfied` stays false at every synchronous step.
- Alert hand-off: at no synchronous step is `alertPending == false` observable while no play request or `quietAt` update exists for the alert.
- Button path with a broken engine: `canOpenMicrophone(button)` returns `mayStartEngineRetry`; `startListening()` is not called before a successful retry; after the retry the controller re-evaluates every condition fresh from the current state (the retry's own health change does not abort the press) and requires `mayOpen` before opening; a failed retry opens nothing and plays the alert once.
- `audioGateOpenedAt`: equals `max(quietAt, moment playRequestPending last became false)`; `settleAnchor` and `rearmDelay` follow it; before the first play request it is `null`.
- Button order with both conditions: engine broken and gate closed: the recorded call order is retry, then the grace wait or an immediate open on a re-evaluated `mayOpen`, then `startListening()`; `startListening()` is never called before the final evaluation. A play requested during the retry closes the gate (no `startListening()`); the retry's own health change does not abort the press, while any other state change observed at re-evaluation does.
- Late timers and events: a delay timer that fires late, or an event delivered after a state change, re-checks the gate at the actual open and never executes an earlier decision (Fake clock plus Fake engine).
- No ordinary workout cue is requested while the STOP listener is active. `START DETECTED` appears even if audio fails.

## Voice failure (§8)
Every numbered rule in §8 *The alert*, every bullet in *Engine call discipline*, and every exception in §8 has at least one test. These are easy to miss and must be tested:
- Timeouts are named and exposed so tests can assert the values.
- After an accepted Start or cancel the capture may still be closing: the label disappears at once, speech events are ignored, and `capture closing` is logged until the stop completes.
- Any engine failure sets `engineBroken`, disables commands, and plays the alert once; a `stopListening()` slower than 300 ms but finished before 2000 ms with capture closed leaves the engine healthy.
- **Microphone closes after any engine failure:** a failure (e.g. `ENG-108`) in Count-up `RUNNING` with the listener open triggers exactly one bounded `stopListening()` even though the engine is broken; the same single call happens at cancel and at Count-up stop; a failing/hanging stop records `ENG-106`; no other call is made on a broken engine during a workout.
- No engine call starts behind a pending one; a queued `startListening()` is cancelled if the engine breaks; a `startListening()` in flight at background/cancel/Manual Start is followed, if it later succeeds, by one bounded `stopListening()`.
- Retry from `IDLE` (§8): only after a completed error, never after a pending call, never while `captureMayBeOpen` is true (e.g. after `ENG-106`), never while `alertPending == true`; a failing or hanging `dispose()` during a retry leaves the engine broken and `captureMayBeOpen` unchanged; a `startListening()` that completes late triggers the one closing stop; max 3 consecutive failures; success resets `engineRetryCount`; each failed retry records its code and — except retries run inside `Check voice`, which never plays an alert — plays the entry alert once. A late successful `initialize()` is accepted only in `IDLE`. A press during a normal initialization waits for the same Future and starts no second one.
- Permission: a pure denial leaves the engine healthy (`MIC-201` / `MIC-202`); `MIC-202` through `errorStream` mid-listening, and `MIC-203..206/210/211`, close the capture via a bounded stop and leave the engine healthy if it succeeds; conflicting evidence breaks the engine.
- Alert: at most one per entry attempt and one per session across both triggers (voice failure and STOP unavailable); at most one alert is pending at a time (§8 rule 9); an alert pending behind a cue plays after it; **normal completion of a set never discards a pending alert** (it plays after the finish beep, survives the end of `REST` and `sessionId` invalidation, and plays at the first instant the gate is satisfied; afterwards the app follows §11; a later Accepted Start discards it per §7); Cancel, lifecycle cancel, and core error clear it; a due mandatory cue is requested on time even while an alert is playing (overlap allowed); a missing/hung alert file never changes Timer state; it never starts while the STOP listener is listening; it has no `SpeechEngine` dependency.
- `supportedCommands` without `stop`: no STOP listener is requested, `voiceStopAvailable = false`, `ENG-109` is recorded, no alert, the engine stays healthy (Milestone A); `Check voice` lists the supported commands, does not resolve `ENG-109`, and the status line reads `Voice: OK (Stop unsupported)` (Milestone B; in Milestone A tests stub the line/check verdict or skip this half of the bullet).
- A failure in `PREPARING`/Countdown `RUNNING`/`REST` never cancels the clock; failure or permission loss in Count-up `RUNNING` sets `voiceStopAvailable = false`, keeps the timer and manual STOP, and attempts no recovery.
- Prewarm loads the model without opening the microphone, asking permission, taking the wakelock, or blocking manual controls; a prewarm failure sounds no alert (the next explicit request does) and records its fault (Milestone A); showing `Voice: <CODE>` for it is Milestone B (status line).
- Labels: `Listening for Go`/`Listening for Stop` only for a healthy active listener; `Voice unavailable` otherwise; never a "microphone off" claim; late completions cannot restore a broken engine, except a successful `initialize()` while idle.
- A Count-up session started with `START MANUALLY` and no microphone permission: `voiceStopAvailable = false`, `MIC-201`/`MIC-202` recorded, the message card shown, **no session alert** (alert only for failed voice entry), and no permission prompt during the workout.
- A `MIC-2xx` error in `LISTENING_FOR_START` returns to `IDLE` (not a session cancel) and leaves the engine healthy if the closing stop succeeds.

- Playback faults and the alert (§8 rule 10): each row of the matrix with a fake player; a failing `voice_unavailable` play records `AUD-302` once, plays no second tone, consumes the entry/session alert budget, and starts no retry; the clock, `REST`, and the microphone gate behave as the matrix says.
- `CORE-101`: a known exception injected into `Clock`, `Scheduler` and `TimerController` each returns to `IDLE` with cleanup and the generic message in every build; a copyable stack trace exists only with `DIAG_DETAIL`; no alert plays.
- Diagnostics hygiene: a very long, multi-line `rawMessage` and stack trace are capped and sanitised, the log file stays valid, and nothing outside the capped fields is written; several faults raised at once are written by the single writer in order, none lost to a race; the write interrupted before the rename leaves the previous valid file intact.
- Upgrade cleanup: a failing `delete()` never blocks startup, is retried at the next startup, and no text claims removal.

- Permission order: engine broken and permission denied, then `START VOICE SESSION`: `MIC-201` or `MIC-202` is recorded, the alert plays once, and no `dispose()`, `initialize()` or `startListening()` call is made; after the permission is granted a new press follows §8 and §11; a permission request that returns later triggers the fresh re-evaluation.
- Fault resolution scope: a failing `start_confirm` (`AUD-302`) followed by an opened listener leaves `AUD-302` listed; `fault resolved AUD-302` is recorded only after a later successful `start_confirm`; `CORE-101` stays listed until `[ Dismiss ]` or restart; a successful `Check voice` or opened listener resolves `ENG-*` (not `ENG-109`) and `MIC-*` only.
- Event snapshot: a fault written to the file carries at most 20 events, each capped at 200 characters; the file stays valid and bounded; no audio, result or earlier `rawMessage` is stored; `[ Delete log ]` removes it.

## Voice health and Check voice (§8; Milestone B — Milestone A implements the fault records only, with the line/check halves stubbed or skipped)
- A successful `initialize()` that completes after `initializeTimeout` while in `IDLE` (no other call started) restores the engine, clears `ENG-104`, resets `engineRetryCount`, and logs the real elapsed time; the same completion during a workout, or after another call started, is ignored. `START VOICE SESSION` while that `initialize()` is pending is refused with the alert; after it completes the press works.
- Status line: `Voice: not checked` → `Voice: OK` (or `Voice: OK (input level not observable)` when frames/level could not be observed) after a successful step → `Voice: <CODE>` on a fault → `OK` again when the fault is resolved (by `Check voice`, a successfully opened listener, or a late init success); it is text, not color alone; `fault resolved <CODE>` is logged.
- `Check voice`: runs only in `IDLE`; steps run in order and each shows ✅/❌ with code and time; each failing step yields the right code (`MIC-201/202`, `ENG-1xx`, `MIC-203..211`, `ENG-106`); "Say Go not heard" is a warning and does not break the engine; it plays no alert; the result list is copyable; it uses serialized, bounded engine calls; steps 1–3 and 5 passing clears the fault. A public build has neither the status line nor `Check voice`. Lifecycle: the indicator shows whenever the microphone is open; the start buttons and Settings are disabled while it runs; background or leaving the screen cancels it with one bounded stop and releases the wakelock; a `Go` heard is listed and never starts a workout; a successful check resolves all `ENG-*`/`MIC-*` faults; multiple unresolved faults list as `+N earlier`.

## Result and cancel
- Nothing is stored: fakes for every storage API (key-value store, file, documents directory) are wired in and assert zero calls (in `DIAG_DETAIL` builds the only allowed call is the fault-log append of §8) across a full Count-up and Countdown session, a Cancel, a background, and a fault; the manifest check verifies `allowBackup` is false.
- Result lifetime (§9): the Count-up result appears at stop and stays through `REST` and `IDLE`; it is cleared when a new session enters `RUNNING` (either mode), when the athlete leaves the screen for Settings/About testing/Voice Test, and on `AppBackgrounded`; `inactive`/`hidden` do not clear it; a fresh process shows no result.
- No interruption prompt: background, permanent interruption, or process death during a session leaves nothing to restore; the next launch shows `IDLE` with no result and no prompt.
- Cancel cases: Cancel in `REST` or `LISTENING_FOR_START` keeps the result; Cancel in `PREPARING` (`startDelay` > 0) keeps the previous result; Cancel in Count-up `RUNNING` leaves no result.
- Deferred clear: an Accepted Start clears nothing; the previous result is cleared only when the new session enters `RUNNING`; a Cancel in `PREPARING` leaves the previous result displayed.
- Mode and settings reset: at a cold start no mode is preselected and both start buttons are disabled until one is chosen; Countdown duration is 30 after choosing Countdown; Halfway cue ON and `After a set` = `Wait for me`; changes apply per §13 and are gone at the next cold start.
- Known limitation (0.0.1): screen lock, Home, an app switch, or answering a phone call during `PREPARING`/`RUNNING`/`REST` cancels the session with cleanup (back to `IDLE`, no result, microphone closed) and no counter keeps running; `inactive`/`hidden` change nothing; a ringing or declined call alone does not cancel (the clock continues, voice is off, manual STOP works).
- Back during an active session confirms `Cancel session` / `Continue` in both modes; there is no keep option.
- Back dialog auto-dismiss: if Countdown finishes or Count-up Stop/safety cap fires while the Back confirmation is visible, the dialog is dismissed and the completed result is shown; no choice is applied. Test with FakeClock advancing time while dialog is open.

## Fault codes (§8)
- Every `SpeechErrorCode` and playback fault maps to exactly one code in `fault_catalog.dart`; every code appears in `docs/ERROR_CODES.md` and the reverse (test fails on a mismatch); unmapped input becomes the group's unclassified code with the raw message, per the §8 error-mapping table.
- `DIAG_DETAIL=true` shows the Fault panel with code, stage, detail, raw message, `[ Copy details ]`; `false` shows only `Voice unavailable`, the §8 user message for `MIC-2xx` faults, and the alert, with no raw text anywhere in the widget tree.
- A fake capture of all-zero frames for `micSilenceTimeout` raises `MIC-210`; frames that stop mid-session raise `MIC-211`; a quiet but non-zero signal raises nothing. Each `MIC-2xx` environmental fault closes the capture via a bounded `stopListening()`, leaves the engine healthy if that succeeds (a later `START VOICE SESSION` can retry), and breaks it if the stop fails.
- `MIC-201` and `MIC-202` leave the engine healthy; every `ENG-1xx` fault that the §8 fault table classifies as an engine failure breaks it, **except `ENG-109`** (`commandUnsupported`), a capability limit: it leaves the engine healthy, plays no alert, and disables only the unsupported command (§8, §10).
- Faults never block manual controls and never cancel the clock.
- In `DIAG_DETAIL` builds: a fault is appended to `diagnostics_log.txt` (rolling window of 5; the 6th drops the oldest); several consecutive restarts with no new fault leave the file unchanged; after a forced process kill following `ENG-106` the next launch lists it under `Previous session` as history only — `Voice:` reads `not checked`, `engineBroken` is false, no alert plays, nothing is shown as an unresolved fault; a malformed file is treated as empty and the next append rewrites it from the valid records; a non-`DIAG_DETAIL` build started over an existing `diagnostics_log.txt` deletes it and touches no other file (upgrade test with a fake file store, including a `results.json` that must survive when Extension P is present); `[ Delete log ]` removes the file and is enabled only in `IDLE`; public builds write no file and show no diagnostics. Test with a fake file store.

## Language, UI, assets
- Timer logic runs with a second fake `LanguagePack` (e.g. `xx`); no audio asset depends on the language (all cues are in `common/`); no user-visible string literal outside the localization layer (static check as practical). A fake RTL pack drives `Directionality.rtl` in widget tests.
- Asset duration limits (§6); the four cues (`start_confirm`, `halfway`, `finish_beep`, `voice_unavailable`) differ pairwise in content hash and pattern; every cue exists and is declared in `pubspec.yaml`.
- Active workout screens do not rotate.
- Tilde rule (§13): `stopSource == voice` with `voiceStopResultApproximate` true shows `TIME mm:ss ~`; manual STOP, safety cap, an unknown source, and `Set done` show none; the flag false removes it. Widget tests.

## Fake behaviors required
`FakeSpeechEngine`: `emit(VoiceCommand)`, `emitError(SpeechError)`, configurable `supportedCommands`; success/failure/hang for `initialize()`, `startListening()`, `stopListening()`, `dispose()` incl. late completion of abandoned operations; `stopListening()` completing before 300 ms, between 300 ms and 2000 ms, failing, or hanging; records lifecycle order, outcomes, and calls in flight so overlap fails a test.
`FakeAudioFeedbackEngine`: uses the injected `Clock` and `audioTail`; records requests (with `requestedAt`), stops, and `quietAt`; optional output-start latency; configurable play failure, missing asset, and a hanging invocation stage (decided but never invoked); `stopAll()` configurable to hang or fail (to prove no decision depends on it); a strict serial mode (never overlaps cues, never starts a stale cue) and an overlap mode. No completion callbacks.

---

# 18. Definition of Done (by level)

Numeric criteria live in §14; this is the checklist.

## Level 0 — Personal (owner)
- [ ] `diagnostics_log.txt` exists in this build (`DIAG_DETAIL=true`), holds the last 5 faults with full detail, and `[ Delete log ]` removes it
- [ ] Known limitation (0.0.1) confirmed on the reference device: locking the phone, Home, switching apps, or answering a call during a session cancels it clearly; a ringing or declined call that leaves the app in the foreground does not cancel (the state stays, the clock continues, voice is off, manual STOP works), and the Home screen shows no preselected mode
- [ ] Milestone A tests pass; A2 facts recorded
- [ ] Countdown and Count-up work end to end by voice and manually
- [ ] Voice failure before Start: alert plays, manual controls remain
- [ ] Voice failure during a workout: alert plays, timer continues, manual Stop works
- [ ] Alert audibility checked on the reference device (silent mode, zero media volume, Bluetooth) and recorded
- [ ] After an engine fault in Count-up, the listening indicator disappears and Android's microphone indicator turns off (or the limitation is recorded)
- [ ] Own-audio check: no `Go`/`Stop` activation during or within 1.5 s after the audible end of `start_confirm`, `halfway`, `finish_beep`, or `voice_unavailable` (also right after pressing `START VOICE SESSION` following a set)
- [ ] Engine retry: after a retry-eligible error (a completed error, §8) with `captureMayBeOpen == false` and no engine call pending, pressing `START VOICE SESSION` in `IDLE` allows one re-initialization attempt; never after `ENG-106` or any state where `captureMayBeOpen == true`, never after a pending call, never for `ENG-109`; three consecutive failures stop retries until restart
- [ ] Version `0.0.1` built with `DIAG_DETAIL=true`; the Fault panel shows code, stage, detail, and raw message
- [ ] **Fault drill:** trigger each situation below on the reference device, record the code shown, and note whether the detail matched the real cause (mark "not reproducible" where impossible), then fix or document each: deny permission; deny permanently; revoke the permission while listening; another app recording; microphone privacy switch off (Android 12+); connect/disconnect a Bluetooth or wired headset; incoming call; app to background; delete a model file and a keywords file in a debug build; Hot Restart; first-launch slow initialization
- [ ] After each fix from the fault drill, `Check voice` passes and the status line shows `Voice: OK`
- [ ] `docs/ERROR_CODES.md` updated with every new cause found; README has every note of the Appendix B microphone-notes list
- [ ] The model integrity script of §15 exists, passes on the files in use, and fails on a deliberately altered copy and on an extra model file

## Level 1 — Friends
- [ ] No `diagnostics_log.txt` is written (`DIAG_DETAIL=false`; verified with the build flag and a file-system check)
- [ ] Everything in Level 0
- [ ] Gate 0F recorded (own-audio check and final `rearmDelay`)
- [ ] Gate 0H at Level 1 criteria; START targets; 60-minute false-positive run (§14)
- [ ] p95 latency ≤ 1.5 s; API 26 emulator compatibility; Airplane Mode; wakelock/screen test; 5-set session
- [ ] After integration: ≥30 first-attempt STOP trials across ≥2 speakers with zero false stops or duplicate completions
- [ ] Testing disclosure shown at every cold start (Level 1/2 builds) and on `About testing` (Appendix B), figures matching §20
- [ ] Built-artifact checks: APK content matches the written allowlist, no fake-engine or diagnostics entry point, merged manifest and `allowBackup=false` verified on the built file (Appendix B)
- [ ] Direct dependencies checked for known advisories and abandonment; result recorded in §20
- [ ] Upgrade test: a 0.0.1 install with an existing `diagnostics_log.txt` updated to the friends' build leaves no such file and no other file touched (§8)

## Level 2 — Store
- [ ] No `diagnostics_log.txt` is written (`DIAG_DETAIL=false`; verified with the build flag and a file-system check)
- [ ] Everything in Level 1 at Level 2 criteria (§14)
- [ ] Version is 0.9.0 or later and `versionCode` increased; the Appendix B dependency review is recorded in §20
- [ ] Battery measured and reported (Countdown and Count-up separately)
- [ ] Appendix B checklist complete

---

# 19. First Task for the Coding AI and Prompt

**First pass: Milestone A-pre only** (§16: audio-semantics spike, facts only; then STOP and report). **Milestone A starts only after the report and an explicit instruction** (never both in one pass). The steps below are Milestone A.

```text
1. Create the Flutter Android project.
2. Report the actual Flutter, Dart, Android Gradle Plugin, Gradle, compileSdk, targetSdk, and minSdk environment.
3. Add Riverpod.
4. Add a minimal test harness.
5. Create the Clock, SpeechEngine, and AudioFeedbackEngine abstractions the Timer Core needs, plus the injectable constants in §10 and §6 with placeholder defaults.
6. Create FakeClock, FakeSpeechEngine, and FakeAudioFeedbackEngine.
7. Build the Timer state machine (Countdown and Count-up) and the scheduler/tick abstraction, including the §8 failure policy and the §9 stop/cancel/result-lifetime rules.
8. Create the AppLanguage / LanguagePack abstraction with the English pack; no user-visible string literals in Timer logic.
9. Create the `docs/ERROR_CODES.md` skeleton (§16); record every verified version, measured value, owner decision, and fault-code addition as you go: facts and decisions in §20, fault codes in `docs/ERROR_CODES.md`.
10. Implement the required unit tests in §17 (Milestone A scope: Voice-health line/check halves stubbed or skipped per §17).
11. Run tests and build.
12. STOP and report the result.
```

**After every milestone**, stop and report: implemented, test results, build status, exact resolved dependency versions, exact model files used (when applicable), known limitations, unverified real-device requirements. Never proceed to the next milestone unless explicitly instructed.

Stop conditions per milestone: **A-pre** — facts only, no Timer Core code; stop and report (stop earlier if a cue cannot be played or stop isolation, §6 rule 6, cannot be achieved). **A** — no Sherpa or live microphone. **A2** — stop if a cue cannot be played or stopped safely (§16 A2 item 1). **B0a** — stop on any blocker defined in §15. **B** — no candidate engine; verify with `FakeSpeechEngine`. **B0b / C** — the human performs live tests; the AI may process recorded results but never invents them. **B2** — if the abstraction doesn't fit, report and propose an interface change. **D** — no Sherpa in the production Timer until the human gate has passed; voice Stop in Count-up is enabled for the owner's own builds after B0b (Level 0), for friends' builds only after 0H Level 1, and for public release only after Level 2. **E2** — not without explicit approval.

## Prompt for the coding AI

```text
Use this document as the implementation contract. Follow the Verification Rule in §0: inspect actual resolvable package versions and official examples before using any dependency; never invent APIs, model files, keyword formats, or config formats; STOP and report any blocker.
Follow the build order in §16 and the stop conditions in §19. Do one milestone per pass. Start with Milestone A-pre only (§16: audio-semantics spike, facts only); Milestone A starts only after explicit instruction: Timer Core (Countdown and Count-up), the failure policy (§8), the LanguagePack abstraction with the English pack, fakes, and tests. No Sherpa, no live microphone.
Humans perform live-device tests (B0b, C, C2 live part); you may process recorded results but never invent them.
Never claim a real-device test passed unless it was measured and recorded.
```


---

# Appendix A. Alternate Engine (activated only by the owner)

**Activation.** The owner records the decision in §20. Typical triggers: Sherpa is DROPped at B0a/B0b, or Gate 0H fails for every Sherpa configuration within the tuning cap. Nothing in this appendix is built or evaluated in v1 without that decision. Everything below plugs in behind `SpeechEngine`; `TimerController` is never modified.

## Candidates

| Engine | Role | Notes (claims until measured) |
|---|---|---|
| onnx-wakeword (`github.com/voicute/onnx-wakeword`) | First alternate | Claims: model < 130 KB, multi-keyword, Java + ONNX Runtime, MIT. Keyword sets are trained models; custom multi-keyword models come from the project's training service (check whether any audio is uploaded, §13 Privacy). Published English demos are wake-word style; the published command demo is Chinese. Young project; no known Flutter plugin (a platform-channel bridge is likely). Forks and mirrors exist: record the canonical repository, tag, and commit. |
| Vosk | Last resort | Only by a recorded human decision for a bounded evaluation. |
| GPTWake, mica-voice | Reference only | Never use Chinese-model files or values for the English model. |

## Gate 0A for any additional engine (no production code)

Record: whether an English START model exists or can be produced within an approved budget, with exact files and identifiers (STOP support is recorded; an engine without STOP cannot ship the full MVP); how keywords are defined and what changing a phrase costs; license and terms for engine code, models, and any training service; Android runtime and Flutter path (verified plugin or bridge) and build effort; `minSdk 26` and ABIs incl. the emulator ABI; PCM requirements and capture ownership; size impact; project maturity; languages supported (information only); vendor claims recorded as *claims*. Outcome PASS or DROP with the reason.

## Milestone B1 — onnx-wakeword spike (after Gate 0A PASS and explicit human approval)

**B1 (onnx-wakeword spike, conditional on Gate 0A pass and human approval):** identify the canonical repository/tag/commit; verify English START (and STOP if available) model files, license, and provenance; verify the Android path and a Flutter bridge; verify PCM requirements; run a WAV fixture if supported; implement `OnnxWakewordSpeechEngine` behind `SpeechEngine` without modifying `TimerController`; report mismatches. DROP and stop if no acceptable English START model exists, the runtime can't run at `minSdk 26`/emulator ABI, integration needs hacks around `SpeechEngine`, or the flow can't be verified without inventing details. A Sherpa failure never blocks B1.

A human then repeats the live smoke test (B0b) and the §14 gates with the same procedure used for Sherpa. A retry of the same engine needs a new recorded human decision.

## Admission, ranking, and removal (only when two engines are evaluated)

- Both run the identical procedure, speakers, scripts, and device. An engine is eligible only if it meets the §14 targets at Gate 0E and Gate 0F.
- Rank eligible engines in this order: false-positive rate; recognition under quiet conditions; latency p95; battery and size; integration and maintenance risk (licensing and maturity included). If still tied, choose the lower integration risk and record why. Sherpa holds no tie-break advantage.
- **STOP never affects admission or ranking.** After ranking, Gate 0H runs on the top eligible engine; on failure, the next documented configuration, then the next eligible engine only if verified to support `Stop`, then NO-GO.
- Exactly one engine ships. Remove the other's code, model files, and dependencies before release; record the decision with evidence for every evaluated candidate in §20.
- Never reuse one engine's paths, files, or configuration formats for another; record model file names and hashes per engine.

---

# Appendix B. Before Friends and Before the Store

Nothing here blocks Milestone A. The Level 1 items (testing disclosure, claim wording) apply when you share a build with friends; the rest applies before a public release (Level 2).

## Microphone notes for users (app description, README, `About testing`)

These seven notes go into the store/app description and the project README (and are shown on `About testing`). Localize them through the language pack:

1. **Permission.** Voice commands need microphone permission. If you denied it, enable it in Settings → Apps → Voice Workout Timer → Permissions. The manual buttons always work.
2. **Other apps.** Close apps that may be using the microphone (calls, voice recorders, voice assistants, video calls) before you start.
3. **Keep the app open.** Voice listening works only while the app is open and the screen is on. Leaving the app stops it.
4. **Microphone privacy switch (Android 12 and later).** Make sure microphone access is on in the Quick Settings panel and in Privacy settings. If it is off, the phone may deliver silence without any error.
5. **Headsets.** A Bluetooth or wired headset can switch the microphone input. For best results disconnect it; this version is tuned for the phone's own microphone.
6. **Other audio.** Music, videos, or calls playing on the phone can interfere with voice commands and with the app's own sounds. Pause them while you train.
7. **Calls.** Turn on Do Not Disturb (or Airplane Mode) before you train. A ringing or declined call does not stop the timer, but answering a call sends the app to the background and ends the session.

## No unverified claims (Level 1 and 2)
Never write "Speech is accurate", "Offline verified", "Works on Android 8", "No false positives" unless the corresponding test was actually run and recorded. Allowed: "Implemented but not yet verified on physical hardware." Supported wording: *"Supported from Android 8 (API 26). Compatibility verified on emulator; voice accuracy measured on <device model, Android API>."* Environment wording: *"Voice commands work in a quiet environment (no music, no other voices). Music support may come in a later version."* Voice Stop claims follow the Level wording above. Timing wording: a voice-Stop result includes the recognition latency (target p95 ≤ 1.5 s, §14); never present a voice-Stop time as more precise than that.

**Testing disclosure** (localized template; the figures come from §20, never invented or rounded up; until measured, state "not tested yet"):
> This is an early version. Voice commands were tested in a quiet room (no music, no other voices) on <N> phone model(s) (<model, Android version>), with <S> speakers, about <U> "Stop" attempts and <V> start attempts. Results with other phones, accents, music, or noise are not yet known. Please try it and tell us what happens: amine.triki@live.com (the app fills in and sends nothing for you; please do not send recordings).
> Privacy: the app starts listening only after you ask it to and shows "Listening" while it does, and voice commands are accepted only while that indicator is visible. When listening stops the indicator disappears at once and the microphone closes within a moment (normally well under a second); anything said during that closing moment is ignored. In the rare case it cannot be closed, the app shows "Voice unavailable" and the microphone may stay in use until you close the app completely, which releases it. The app does not send your audio to any server and needs no account. The app stores nothing on your phone: your settings and your timer result exist only while the app is open. This release does not learn your dialect, upload recordings, or collect analytics.
> While you use it: keep the screen on and stay in the app during a session. Locking the phone, pressing Home, switching apps, or answering a call ends the session and the time is lost.

**With Extension P activated,** the sentence that says the app stores nothing is replaced by the §21.5 statement, in the notice, on `About testing`, in the store listing, and in the Data Safety declaration.

Shown as a one-line dismissible notice at every cold start in Level 1 and Level 2 builds (the app remembers nothing, so it cannot know a first run), with a `More about testing` link; always on the `About testing` screen; and in the store listing and release notes, together with the 0.0.1 known limitation of §2. The contact address is a release configuration constant.

## Release verification (Level 2)
No `android.permission.INTERNET` unless a documented dependency requires it and the human approves. Verify: target/min SDK; merged manifest permissions (investigate unexpected `INTERNET`, `BLUETOOTH`, `LOCATION`) and the `allowBackup=false` application attribute (with `dataExtractionRules` / `fullBackupContent` where applicable); embedded model and audio files; ABIs/native libraries; only the selected engine's runtime and model present; no `FakeSpeechEngine`, E2E hook, E2-only entry point, or test-only dependency (dependency-graph inspection and APK/AAB content inspection; use mapping files where minification is enabled). **Allowlist method:** list the contents of the built APK/AAB and compare them with a written allowlist (the selected model's files, the four audio cues, the native libraries of the selected engine); any other file, any extra model, and any diagnostics, Voice Test, or fake-engine entry point is a failure. Run the permission and `allowBackup` checks on the **built** artifact, not on the source manifest. For Level 1 builds (friends) run the same allowlist check and the manifest checks as a shorter list. Repeat the voice-path smoke test (START, STOP, offline) on the final release artifact. Store requirements: privacy policy URL, Play Data Safety declaration (required even when nothing is collected), microphone disclosure in the first-run flow; verify current store policy text at release time.

## Level 2 extras

- `DIAG_DETAIL` is false in the release artifact (verify: no Fault panel, no raw error text); version is 0.9.0 or later.
- Every shipped model's license record is `clear` (§15); the license text and notices are shown in the app (Flutter shows pub-package licenses automatically; a model's license text must be added through the license registry — verify the mechanism at Milestone E).
- **Dependency review before release.** List every direct and transitive dependency from `pubspec.lock` and the Gradle graph; check each for published security advisories and for being discontinued or unmaintained, using what the installed SDK and the package registry actually offer (verify the tool; never assume a command); record the date, the findings, and any accepted risk `[owner]` in §20. Re-run it for any release after a dependency change. For Level 1 builds a quick check of the direct dependencies is enough.
- The store listing states: free, no ads, no purchases, no Internet use.
- All Level 2 criteria in §14 (≥3 speakers; ≥300 qualifying STOP utterances with zero misses; 3-hour false-stop observation with zero false stops; ≥3 aggregate hours of natural START false-positive observation at ≤ 1/hour).
- Battery: Countdown ≤ 3% per 10 minutes or documented CONDITIONAL; Count-up reported separately.
- Latency and battery measured on release/profile builds; voice path re-checked on the final artifact.
- Bluetooth and music recognition documented as deferred.

---

# 20. Measured Facts and Owner Decisions (the only record)

There is no separate decisions file or changelog. The coding AI fills this section as facts are measured, one plain line each: `date | fact | how measured | device`. Owner decisions are marked `[owner]`; only the owner adds them. The AI may propose a decision but never records one as made (§0).

- **Engine of record:** package and model versions, archive SHA-256, license record (§15).
- **Audio facts (A-pre, A2):** `stop()` semantics, request-to-audible-start p50/p95, `audioTail` (value and derivation), audio focus, alert audibility.
- **Gate results:** 0A, 0B, 0C, 0E, 0F, 0H, each with date, device, and counts.
- **Tuning configurations used** (cap in §14) and any phrase change `[owner]`.
- **Provisional constants approved** (`audioTail`, `playInvokeTimeout`, `audioInitTimeout`, `pressGraceWindow`, `rearmDelay`, others) `[owner]`.
- **Acceptable clock-versus-beep offset** (§5, from the A-pre/A2 latency facts) `[owner]`: not yet recorded.
- **Known faults (owner-visible list):** one line each `date | fault code or symptom | steps to reproduce | status`. A fault that can cause a false Stop, a lost result, a hung microphone or a crash blocks a friends' build until it is fixed or accepted `[owner]`.
- **Known limitations accepted** for the release (for example the 0.0.1 limitation of §2) `[owner]`.
- **Public/store version:** 2026-10-15 | the first public (store) release is **0.9.0**; the earlier 0.0.9 and 1.0.0 references are unified to it and `1.0.0` stays reserved for a later decision | owner instruction in chat `[owner]`.
- **Extension decisions (§21.8):** 2026-10-15 | **D-S1 = B**: when the app goes to the background (screen lock during a long rest, a notification, answering a call), the running round is cancelled and the session's completed rounds stay in memory; a process kill or app close still loses them | owner instruction in chat `[owner]`.
- **Extensions (§21), still not recorded:** activation of S and of P, and decisions D-S2, D-P1 to D-P5 (§21.8) `[owner]`.

---

# 21. Extensions After 0.0.1 (Owner-Activated)

> **Not required for 0.0.1 or for any release until the owner records the activation in §20 `[owner]`.** Nothing in this section is built early, and nothing here changes the behavior of the timer, the voice path, the audio path, or the failure policy of §8. Two extensions, in this order: **S** (sessions and rounds, in memory only) and **P** (saved Count-up results, on the device only). **S does not depend on P. P depends on S.** Neither may delay or replace the Go/Stop reliability work (Gates 0F/0H, §14): do not start either before Level 0 of §18 is met.

## 21.1 Principles

- **Free training is always available.** A round goal is an intention, never an obligation: the app reports progress and never blocks, stops, or ends anything because a goal was reached or missed.
- **The athlete decides everything between rounds.** There is no forced rest, no rest timer, and no plan. Every round is independent: the athlete may change mode (Countdown or Count-up), duration, or the exercise between any two rounds, or leave after one round. A round goal is not a "workout plan" in the sense of §0.
- **Completed work is the truth; the goal is not.** Ten chosen rounds and nine completed show `9/10`, not a failure.
- **The timer stays as specified.** Do not redesign the `TimerController` state machine (§7) or the Scheduler. The extensions are separate controllers that observe completed rounds and own their own data (§21.3). The integration points (complete, cancel, mode change, background) are tested (§21.7).
- No new voice command, no new microphone use, no new audio cue, no new asset. Everything shown is visual and localized through the language layer (§12).

## 21.2 Extension S — Sessions and rounds (memory only)

**Home.** A selector `Rounds`: `Single round` (default at every cold start, exactly the 0.0.1 behavior with no session object) or `Several rounds`. With `Several rounds`: an optional goal `roundGoal` (none, or 1–99; default none). The choice is not remembered across launches (§13). While a session exists, the selector is locked to `Several rounds`; to return to `Single round` the athlete ends the session first.

**Session.** Created at the first Accepted Start (§7) while `Several rounds` is selected. It holds: `roundGoal` (nullable), `startedAt` (wall clock, display only), and the list of `RoundRecord`. It exists in memory only and is never written to disk.

**What counts as a completed round (single rule):**

| Event | Completed round? |
|---|---|
| Countdown reaches 100% | yes |
| Count-up stopped by voice `Stop`, manual STOP, or the safety cap | yes, whatever its length |
| Cancel in any state, Back-confirmed cancel, lifecycle cancel, `ERROR` | **no**: nothing is recorded |

A partial round (a cancelled round recorded with its real elapsed time) is **deferred**. When it is added later it records the elapsed time actually spent, never the chosen duration and never the remaining time, and it needs its own owner decision.

**`RoundRecord` (memory):** `index` (1-based completion order), `mode`, `chosenDurationMs` (Countdown only), `finalElapsedMs` (Count-up only), `stopSource` (Count-up only), `completedAt` (wall clock, display only). No exercise name in S.

**Progress.** While a session exists the Home and Ready screens show the completed count as `done/goal` (`done` when there is no goal) and the Timer screen a small label `Round N` with N = completed + 1. Reaching the goal shows the visual label `Goal reached` and nothing else (no tone, no change of state, no stop). The athlete may continue: `11/10` is valid.

**Between rounds** the app is in `IDLE` exactly as in §11: microphone closed, mode and duration selectable, `START VOICE SESSION` / `START MANUALLY` available. The round just completed stays visible as in §9.

**Ending.** `[ END SESSION ]` is offered only in `IDLE` (never in an active state). It opens the **Session summary** (list of rounds: index, mode, result with the `~` rule of §13; the count `9/10`; total work time) and, when dismissed, clears the session and returns to `Single round`.

**Leaving or interrupting.** Back during an active round follows §9 and cancels only that round (the session and its completed rounds are not affected). `AppBackgrounded` (decision **D-S1 = B**, §20): the running round is cancelled with the cleanup of §9 and `lastCompletedResult` is cleared exactly as §9 says; the session's completed rounds stay in memory and the session continues when the athlete returns. A process kill or app close always loses the session (no restore, no prompt). The Home screen of a surviving session shows `Session in progress: N` with `[ CONTINUE ]` and `[ END SESSION ]`.

## 21.3 Integration contract (the seam between the timer and the extensions)

```text
TimerController ──RoundCompleted(sessionId, mode, chosenDurationMs?, finalElapsedMs?, stopSource?)──▶ SessionController
```

- `TimerController` emits `RoundCompleted` exactly once per completed round, in the same synchronous step that creates `lastCompletedResult` (§9) or the Countdown completion. It knows nothing about goals, rounds, sessions, names, or storage.
- `SessionController` (a Riverpod 3 Notifier, §16) appends one `RoundRecord` per event, ignoring a duplicate or stale event by the timer's `sessionId` (§7). It never touches `Clock`, `Scheduler`, `SpeechController`, or `AudioFeedbackEngine`, and never starts, stops, or delays anything.
- Cancel and error paths emit nothing.
- Files (only when S is activated): `features/session/{session_controller, session_state, round_record, session_summary_page}.dart`. `TimerController` must construct and pass every §17 test with no `SessionController` present.

## 21.4 Extension P — Saved Count-up results (on the device only)

**Scope.** Only Count-up results are saved. A Countdown completion carries no information beyond the duration the athlete chose; saving completed Countdown rounds (for example `8 rounds × 45 s`) is deferred (D-P3). Nothing is saved automatically: the athlete presses `[ SAVE RESULT ]`.

**Data model (pseudonymous, no account, no personal data):**

- `Exercise`: `id` (UUID v4, internal, **never shown**), `name` (shown), `createdAt`.
- `SavedResult`: `id` (UUID v4), `exerciseId`, `mode` (always Count-up now; kept for later), `finalElapsedMs`, `stopSource`, `approximate` (the `~` flag of §13 as it was when saved), `savedAt` (wall clock, display and ordering only; never used for timing).
- Results reference the exercise by `id`: **renaming changes the name only; results never detach.**
- Default name: a localized pattern `Exercise N` (language layer). Names are trimmed, 1–40 characters, unique ignoring case; a clash is refused with a plain message.

**Saving.** `[ SAVE RESULT ]` appears under a Count-up result in `IDLE` only (not in `REST`; never during a workout) with an exercise chip `Exercise: <name> ▾` that defaults to the exercise used last in this process. If no exercise exists, the first save creates one with the default name, editable before confirming. One tap saves to the chip's exercise. A result can be saved once (`Saved ✓`, disabled). The in-memory `lastCompletedResult` keeps following §9; saving does not change its lifetime.

**History** (reachable from Home): exercises with their result counts → one exercise: results newest first with date, time, `~` where applicable, and the neutral signed difference to the previous result of the same exercise (for example `+0:07`). **No automatic "best", "worse", or colour judgement:** a longer Count-up time is better for a plank and worse for a timed task (decision D-P2). Results of different exercises are never compared or merged.

**Manage:** rename an exercise; delete an exercise (the confirmation states how many results go with it); delete one result; move a result to another exercise (fixes a wrong assignment); Settings → `[ Delete all saved results ]` (IDLE only, with confirmation; it deletes `results.json` **and** any `results.corrupt.json`, so no copy of the athlete's data is left behind). Deletions are final.

**Storage rules.**
- One JSON file `results.json` in the app's documents directory, `schemaVersion: 1`, loaded on first need (opening History or pressing Save), never on the timer path.
- **Writes are atomic** (write a temporary file, then rename over the old one) and **serialized** (one write at a time, the last state wins).
- **No storage I/O in `LISTENING_FOR_START`, `PREPARING`, `RUNNING`, `REST`, `COMPLETED`, or during `Check voice`.** Save, History, and delete actions are available only in `IDLE` and never block the timer, the scheduler, or any state change; the timer cannot observe a storage call.
- An unreadable or malformed file is renamed once to `results.corrupt.json` (overwriting an older one), the app starts empty, and `STO-401` is recorded: unreadable data is never silently overwritten.
- Provisional caps (owner may change in §20): 100 exercises, 5000 results; at the cap a save is refused with `STO-403` and a plain message.
- `allowBackup=false` is unchanged (§15): uninstalling the app deletes every saved result. Export is deferred (D-P4).
- Storage faults use the same `AppFault` record (§8; add `storage` to the `stage` values) with the codes `STO-401` (read failed or file malformed), `STO-402` (write failed), `STO-403` (cap reached; a policy refusal, not a fault of the device), `STO-499` (unclassified). They play **no** alert tone (the tone is the voice-unavailable tone) and show one plain sentence (`Could not save the result.` / `Saved results could not be read.`; `DIAG_DETAIL` builds add code and detail). A failed save leaves the result on screen and the button available for a retry. Add every code to `docs/ERROR_CODES.md` (the §8 sync test applies).

Files (only when P is activated): `core/services/results_store.dart` (an interface plus a file implementation and a fake), `features/history/{history_controller, history_page, exercise_page}.dart`, `features/session/save_result_sheet.dart`.

## 21.5 Privacy and disclosure when P is active

This section replaces, only when P is activated: the "nothing is stored" paragraph of §2, the stored-data sentences of §13 Privacy, and the matching sentence of the Appendix B disclosure. The replacement statement (localized; wording approved by the owner):

> The app stores on your phone only the Count-up results you choose to save, with an exercise name you choose. They never leave your phone, need no account, and are deleted when you delete them or uninstall the app. Settings and the current session exist only while the app is open.

Unchanged: no audio is stored or sent, no analytics, no backend, no Internet use. The store Data Safety declaration states that no data is collected and that results are stored on the device only (verify the policy text at release time). The `DIAG_DETAIL` fault log stays a 0.0.1 owner-only exception and is never shipped in 0.0.2 or later.

## 21.6 UI notes

Home stays minimal: mode, duration, start buttons. Rounds, the goal, and History are secondary and collapsed by default so an athlete who wants one quick set never sees them. Active workout screens are unchanged except for the small `Round N` label. All new strings come from the language layer; layouts stay direction-agnostic (§12). Large high-contrast digits and no unnecessary animation still apply.

## 21.7 Required tests (when activated)

**S**
- `Single round` behaves as in 0.0.1: the §17 tests pass unchanged and no `SessionController` exists.
- Countdown completion, voice Stop, manual STOP, and the safety cap each add exactly one `RoundRecord`; a simultaneous voice and manual STOP add one; a duplicate or stale `RoundCompleted` (old `sessionId`) adds none.
- Cancel in `LISTENING_FOR_START`, `PREPARING`, `RUNNING`, `REST`, Back-confirmed cancel, lifecycle cancel, and `ERROR` add none and leave the session intact.
- Goal: `9/10`, `10/10` shows `Goal reached` with no audio request, no state change, and no stop; the athlete continues to `11/10`; no goal shows only `done`.
- Mode and duration change freely between rounds; `[ END SESSION ]` is absent in every active state; the summary matches the records; `Single round` is restored afterwards.
- `AppBackgrounded` (D-S1 = B): the running round is cancelled, `lastCompletedResult` is cleared, the completed rounds and the goal stay, and the session continues on return; a fresh process has no session.
- Zero storage calls in S builds (the §17 "nothing is stored" fakes still pass).

**P**
- Save creates an exercise with the default name on first use; a second press does nothing (`Saved ✓`); a Countdown result offers no save.
- Rename keeps every result attached (compare by `id`); name rules (trim, length, unique ignoring case); delete exercise removes its results; delete one result; move a result; delete all (also removes `results.corrupt.json`).
- Atomic write: a fake file store that fails between the temporary write and the rename leaves the previous file intact; writes never overlap.
- Malformed file → renamed to `results.corrupt.json`, empty state, `STO-401`; write failure → `STO-402`, the result stays on screen and a retry works; cap → `STO-403`.
- Zero storage calls in `LISTENING_FOR_START`, `PREPARING`, `RUNNING`, `REST`, `COMPLETED`, and `Check voice`; the §17 "nothing is stored" test is replaced by: zero calls outside Save, History, and delete actions.
- `TimerController` and `Scheduler` construct and pass every §17 test with no store; the `~` flag survives saving; the neutral difference is signed and never coloured.

## 21.8 Activation record and open decisions

The AI may propose, never record (§0). Record each as `[owner]` in §20.

| Id | Decision | Proposed default | Blocks |
|---|---|---|---|
| Activate S / Activate P | whether and when | — | the extension |
| **D-S1** | What survives `AppBackgrounded`? **Decided 2026-10-15: B** (recorded in §20): the session's completed rounds stay in memory and only the running round is cancelled; the timer's own §9 rules are unchanged; a process kill still loses the session. | decided | — |
| D-S2 | Goal range and default | 1–99, none | — |
| D-P1 | Confirm Count-up-only saving, the explicit `SAVE RESULT` press, and the §21.5 wording | as written | P |
| D-P2 | Whether an exercise may later carry a "longer is better / shorter is better" flag | none now | — |
| D-P3 | Save completed Countdown rounds as a count (for example `8 × 45 s`) | deferred | — |
| D-P4 | Export or backup of saved results | deferred | — |
| D-P5 | Caps (100 exercises, 5000 results) | as written | — |

**Definition of done (adds to §18 Level 0 when activated).** S: a 10-round session with a goal of 10 completed 9 times shows `9/10`; modes switched between rounds; the session summary correct; the D-S1 = B behavior confirmed on the reference device by locking the phone during a long rest and returning to the same session. P: results saved under two exercises, one renamed, one result moved, one exercise deleted; app force-stopped and relaunched with the data intact; a deliberately corrupted file handled per §21.4; the replacement privacy text shown in `About testing`.
