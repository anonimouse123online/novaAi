# NOVA — Proactive AI Companion

A small Kotlin / Jetpack Compose Android app. One foreground service coordinates
on-device speech recognition, offline Android TTS, llama.cpp inference, SQLite
conversation memory, proactive decisions, and optional MediaProjection snapshots.
There is no backend or cloud database. Conversation stays local, app control uses Android
intents, and ordinary searches open browser results. Gemini is used only when explicitly
requested. The app requests Internet access, but no camera permission.

## Hybrid routing

`SpeechRecognizer -> deterministic parser -> local classifier if needed -> AiRouter -> Android action/chat -> existing TTS` preserves the voice
state sequence and the already-loaded local model. STOP cancels both native inference
and an in-flight HTTP request. Proactive decisions and screen processing stay local.

| Request | Execution | Gemini calls |
| --- | --- | --- |
| Chat, jokes, memory, stories, proactive comments | Existing GGUF / llama.cpp | 0 |
| Recognized app, media search, settings, dial/draft, calendar, alarm/timer commands | Kotlin intents and short action status; zero local inference | 0 |
| Latest/current news, weather, versions, ordinary web searches | URL-encoded search in preferred browser; SEARCH_AND_SHOW | 0 |
| Ambiguous action hints, e.g. `Bring up YouTube so I can watch videos` | Loaded local GGUF returns a validated compact action; Android executes it | 0 |
| Explicit `Search Gemini for ...` / `Check Gemini ...` | Existing grounded Gemini lookup, then local NOVA phrasing | At most 1 |
| Missing key, offline, timeout, quota or invalid Gemini result | Honest local explanation, no retries | 0 or 1 |

Ordinary reasoning and casual questions remain local. Merely mentioning an app or
asking how NOVA is doing today does not trigger Gemini. Commands are checked first.
An unspecified/invalid/ambiguous alarm time opens Clock's alarm UI rather than guessing.
Tomorrow's alarm opens Clock for day/time review; Android's weekday extra means repeating
alarms and is deliberately not used for a one-off tomorrow request. Spotify commands open
a search; they do not promise automatic playback. Missing apps
produce an honest status. Web search links can open in a browser if the target app is absent.
Android background activity restrictions require a notification tap when NOVA's activity
is not visible. Notification permission must be granted for that path.

The existing controls are retained, with four small preferred-app dropdowns and an optional
Contacts permission button. App preferences use a separate `nova_apps` file and are written
only when the user changes a selection. Existing model, memory and preferences are retained.
Private communication/credential commands are redacted in speech logs and skipped by new
memory writes. Calls use the dialer; SMS/email use drafts; calendar events require saving
in the calendar editor. No accessibility service, simulated typing or automatic sending.

See [NOVA_APP_ACTIONS.md](NOVA_APP_ACTIONS.md) for supported apps/actions, limitations,
changed files, device checks and the expected action logs. Browser SEARCH_AND_ANSWER via
page extraction remains a future feature; explicit Gemini requests retain the existing path.

Add your development key to the ignored root `local.properties` (keep its existing SDK entry):

```properties
GEMINI_API_KEY=your_existing_key
```

Gradle exposes it as `BuildConfig.GEMINI_API_KEY`. A missing key becomes an empty string
and online requests fall back locally. The configured key is sent only in Google's
`x-goog-api-key` HTTPS header. It never enters prompts, URLs, activity entries, or logs.
BuildConfig embeds the development key in the APK; local.properties is excluded from Git.

`GeminiProvider` uses Android's HTTP/JSON APIs without a runtime networking dependency:
`gemini-3.5-flash-lite`, minimal thinking, a 5-second connection timeout, 8-second read
timeout and 10-second overall deadline. Requests run on `Dispatchers.IO`, are stateless,
and contain only one query (up to 600 characters) and brief factual instructions.
There is no conversation history, memory database, personality prompt, screen context,
or previous Gemini response in the API request. Queries longer than the small limit
fall back locally instead of silently truncating the user's question.

Typical short English queries target **30–200 input tokens**. Output is capped at
**80 tokens**, with a maximum 50-word factual answer; typical answers use 10–80 tokens.
Search grounding can add provider-side context, minimal thinking and search charges,
so that target is not a guarantee of the total billed usage. Token counts are logged
when a request succeeds. Google may execute multiple search queries within our one
HTTP request. Incomplete, blocked, empty or ungrounded search answers are treated as
failures rather than presented as verified current facts.

Only the returned fact enters a fresh local prompt for NOVA's natural 1–2 sentence
reply. That prompt asks NOVA to preserve the facts and add no unsupported claims.
The latest original grounded result, source links and Google's search suggestions
appear beneath the existing controls; they are transient and are not written to disk.
The existing conversation memory still stores the user's turn and NOVA's spoken reply.

Expected logs, all under the `NOVA` tag:

```text
NOVA_ROUTER: LOCAL_CHAT
NOVA_ROUTER: APP_ACTION spotify_search
NOVA_ACTION: SPOTIFY_SEARCH queryChars=10
NOVA_ROUTER: WEB_SEARCH web_search
NOVA_ACTION: WEB_SEARCH queryChars=13
NOVA_ROUTER: WEB_SEARCH Gemini
NOVA_API: Gemini request
NOVA_API: Gemini completed in <ms> ms
NOVA_API: Tokens input=<N>, output=<N>, thinking=<N>
```

An offline/quota failure instead logs `NOVA_API: Gemini unavailable: <short reason>;
local fallback`, followed by the existing native inference and TTS logs. API errors
do not print server bodies, headers, keys or raw networking exception messages.

API reference: [Google Search grounding](https://ai.google.dev/gemini-api/docs/generate-content/google-search),
[Gemini thinking](https://ai.google.dev/gemini-api/docs/generate-content/thinking), and
[model availability](https://ai.google.dev/gemini-api/docs/deprecations).

## Build and run

Open this existing project in Android Studio with JDK 21, SDK 36.1, NDK
29.0.13113456, and CMake 3.31.6. Build with `./gradlew assembleDebug` (Windows:
`./gradlew.bat assembleDebug`). The APK is `app/build/outputs/apk/debug/app-debug.apk`.

The first native build fetches **official llama.cpp v0.6.0**, pinned to commit
`d81235049384534c167caea52b85a694f6103d14` and an archive SHA-256. Subsequent builds
reuse its sources in `.cxx`; no vendored source tree is added to the app.
Only its CPU library is compiled. Its servers, tools, examples and HTTP support are disabled.
All application logic is Kotlin; `nova_jni.cpp` is the small native bridge required by llama.cpp.
The initial APK targets **arm64-v8a phones**. UI supports Android 7+; hands-free mode
requires **Android 12+ with an on-device recognizer and an installed offline TTS voice**.
NOVA deliberately does not fall back to Android's network speech recognizer.

1. Tap **Install AI Model**, then choose a single-file, text-only, instruction-tuned
   1B–3B quantized GGUF with a llama.cpp-supported chat template. Start with a
   non-reasoning model; response token limits suit short spoken replies.
2. Import copies the file to `noBackupFilesDir/models/model.gguf`. No broad storage
   permission is needed. Enough free storage is required for the new copy.
   Import validates the GGUF header; full compatibility is checked during model loading.
3. Allow microphone permission. Notification permission is requested separately.
4. Tap **START NOVA** yourself. Recognition starts after model and offline TTS initialization.
5. Optionally turn **Screen Awareness** on and accept Android's screen-sharing dialog.
   Each sharing session needs fresh consent. Consent is never saved or reused.
6. **STOP NOVA**, in the app or notification, immediately closes speech and screen sharing,
   cancels timers and signals native inference cancellation. Native resources are released
   by the worker after the in-flight call aborts. Process death does not restart Companion Mode.

## Behavior and limits

- Personality and battery/context limits are in `NovaConfig.kt`.
- A first proactive decision may occur after 20 seconds of quiet. Later decisions have
  a minimum five-minute cooldown, increasing as talkativeness decreases. Zero disables
  proactive calls. The LLM decides SILENT, SPEAK, QUESTION or REACT; malformed decisions
  remain silent. No stock greetings or random canned responses are spoken.
- Memory ON supplies up to six recent messages. SQLite keeps at most 200 messages.
  Older supplied messages are capped at 400 characters; the current request keeps its
  existing 1,500-character limit. Stored originals are not shortened by prompt building.
  Memory OFF skips memory reads and writes for new turns; previously saved memory is
  retained locally for re-enabling. Database backup and transfer are excluded.
- The model stays loaded during a session. A 1,024-token context, at most four CPU
  threads, 64-token replies and 32-token proactive decisions bound inference work.
  Prompt construction aims for at most 300 tokens by dropping older turns. Longer
  current requests may use the remaining context rather than being silently truncated.
  Normal turns receive no proactive instruction or screen context. Proactive instructions
  are compact and include at most 240 characters of available screen description.
  Matching token prefixes reuse the in-memory KV cache; unmatched suffixes are removed.
  The final prompt token is always reevaluated to refresh logits. Unsafe/unsupported
  cache rollback and failed decoding reset the cache. Nothing is persisted to disk.
  The native batch, sampler and chat template are also reused; sampling resets each turn.
- The recognition session is paused while NOVA thinks or speaks and automatically resumes afterward.
  Its on-device recognizer is reused until Stop; completed results are not cancelled or destroyed.
  Recognition retries have backoff. Native inference has a timeout and abort callback.
  Native chat templates trim old context when it exceeds the token budget.
- Screen capture attaches its virtual display surface briefly about every 30 seconds.
  Frames have a maximum edge of 640 pixels; only meaningful visual changes produce a JPEG.
  Screenshots stay in RAM and are never saved or sent to an online service.
- **Optional local vision is not implemented.** The UI and notification explicitly say
  vision is unavailable. `AiEngine.visionAvailable` / `describeScreen` are the TODO integration
  point for a local multimodal model and matching projector (llama.cpp mtmd/JNI).
  Only a real vision description may enter model context; text-only models never receive
  guessed screen descriptions. Screen change events remain subject to proactive cooldowns.
- Android SpeechRecognizer is a session-based first-version solution. OEM support,
  speech model availability, device memory and battery policies affect sustained operation.

## Verification

Each implementation phase is compiled before continuing. Run:

```text
./gradlew.bat :app:assembleDebug :app:testDebugUnitTest :app:lintDebug
./gradlew.bat :app:assembleDebugAndroidTest
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb install -r -t app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk
adb shell am instrument -w -e class com.example.vyra.NovaInferenceDeviceTest com.example.vyra.test/androidx.test.runner.AndroidJUnitRunner
adb shell am instrument -w -e class com.example.vyra.NovaHybridDeviceTest com.example.vyra.test/androidx.test.runner.AndroidJUnitRunner
```

Stop NOVA before the native test so a second model context cannot compete with the service.
Use replacement installs and direct instrumentation to retain the imported model and memory;
the Gradle connected-test deployment/reset path removed app data during validation.

Earlier hybrid validation: debug APK and Android test APK assembly, all 27 JVM tests, and
Android lint passed (no errors; existing SDK/dependency/style warnings remain).
Tests cover routing, zero-call local/actions, one-call online phrasing, offline/quota/
invalid-result fallback, bounded JSON, redacted logs, background networking and STOP.
All four headless `NovaHybridDeviceTest` checks passed on the physical Xiaomi phone:
permissions/system intent resolution, special-character deep links, Android JSON query
construction, and zero-request offline fallback. Replacement installs preserved the
GGUF SHA-256 and every pre-existing database/preferences file byte-for-byte. NOVA was
left stopped. The data backup and preservation report are under `app/build/reports/`
as `nova-hybrid-device-data-before.tar` and `nova-hybrid-device-check.json`.
The one live Gemini check returned HTTP 429 / RESOURCE_EXHAUSTED; successful online
fact retrieval cannot be confirmed until the project's quota is available. Its sanitized
status/timing report is `app/build/reports/nova-gemini-live-check.json`.

Files changed for hybrid routing:

- Added `app/src/main/java/com/example/vyra/AiRouter.kt`, `AndroidActionManager.kt`, and `GeminiProvider.kt`.
- Updated `app/src/main/java/com/example/vyra/CompanionService.kt`, `MainActivity.kt`, and `NovaConfig.kt`.
- Updated `app/src/main/AndroidManifest.xml`, `app/build.gradle.kts`, and ignored root `local.properties`.
- Added `app/src/test/java/com/example/vyra/AiRouterTest.kt` and `GeminiProviderTest.kt`.
- Added `app/src/androidTest/java/com/example/vyra/NovaHybridDeviceTest.kt`; updated `NovaDeviceTest.kt`'s network-permission expectation.
- Updated `README.md`.

Earlier local-inference validation: the direct native device test passed for repeated, longer, shorter and
changed prompts, old-history trimming, and in-flight STOP cancellation using the imported
Qwen model. Cancellation aborted the native call about 68 ms after its signal. The broader Compose test run stalled in its UI
harness and was stopped; its results are not a successful regression run.
The GGUF was restored/verified against its original SHA-256 after test deployment removed
it. Current preferences and all 14 existing memory entries were preserved; two missing
logged conversation entries were merged back in chronological order (16 final entries).
The original database/preferences backup is `app/build/reports/nova-device-data-before-final-test.tar`.
The earlier voice-pipeline physical-phone trace confirmed a loaded Qwen2.5 1.5B Instruct model, initialized
offline TTS, final transcription -> background inference -> generated answer ->
`TextToSpeech.SUCCESS` -> TTS `onDone` -> LISTENING. The provider also reported
`ERROR_NO_MATCH` (7), now explicitly logged and retried. The recorded transcription was
"Hello and answering"; the generated answer was "Hello! How can I assist you today?".
The diagnostic trace is `app/build/reports/nova-phone-log.txt`.
The user confirmed the generated response was audible on the phone.

Unit tests cover prompt/history bounds, current-request preservation, proactive instruction
separation, thread limits, final speech extraction, empty inference rejection, reasoning
filtering, decision parsing, corrupt GGUF headers, frame size limits, and screen change
thresholds. Device tests cover bundled JNI loading, invalid-model cleanup, safe
model import replacement, bounded/persistent SQLite memory, requested permissions and
the visible Stop action. Tests do not grant microphone or screen-capture permissions,
download model weights, or listen to the user. Real conversation and MediaProjection
acceptance need a compatible model and the user's explicit permission on a phone.

## Voice pipeline diagnostics

Filter Android Studio Logcat with `tag:NOVA`, or run `adb logcat -s NOVA`.
The diagnostic build logs ordinary recognized text and generated answers, as well as native
load/decode failures and full Kotlin exception stacks. Private command text and responses
with private input/history are redacted. Action logs omit recipients and draft content.

Start waits for the imported GGUF, a loaded llama.cpp model/context, and offline TTS
initialization before opening the microphone. The Qwen2.5 Instruct GGUF uses ChatML:
`<|im_start|>role\ncontent<|im_end|>\n`, ending in `<|im_start|>assistant\n` for generation.
Inference runs on the service's serial background worker. Recognition completion does
not cancel it; Stop/service shutdown does. A normal utterance follows
LISTENING -> THINKING -> SPEAKING -> LISTENING without an intermediate Idle state.

Expected successful flow (response text and lengths depend on the model):

```text
NOVA: Model installed = true
NOVA: Model loaded = true
NOVA: TTS ready = true, offline voice=...
NOVA: Restarting listener, language=...
NOVA: Speech final = "Hello NOVA"
NOVA: CompanionService received text = "Hello NOVA", active=true
NOVA: State LISTENING -> THINKING
NOVA: Starting inference, thread=...
NOVA: Chat template = Qwen ChatML, add_generation_prompt=true
NOVA: Prompt tokens=..., response limit=64
NOVA: Prompt preparation finished in ... ms
NOVA: Prompt evaluation started
NOVA: Prompt cache reused=..., evaluated=...
NOVA: Prompt evaluation finished in ... ms
NOVA: Generation started
NOVA: First token after ... ms (from inference start)
NOVA: Generation finished in ... ms
NOVA: Generated ... tokens
NOVA: Speed = ... tokens/sec
NOVA: Inference completed, chars=..., total=... ms
NOVA: AI response = "<generated model response>"
NOVA: Inference result delivered to CompanionService, active=true
NOVA: TTS ready = true
NOVA: State THINKING -> SPEAKING
NOVA: Speaking response
NOVA: TextToSpeech.speak = SUCCESS (0)
NOVA: TTS started
NOVA: TTS finished
NOVA: Restarting listener, language=...
NOVA: State SPEAKING -> LISTENING
```

An empty final transcription logs a speech error and retries. Every SpeechRecognizer
error logs its numeric code, including ignored stale callbacks. Empty AI output is an
error, not a silent successful turn. Load/generation exceptions show `AI error: <reason>`
in the activity log and stop the session. TTS initialization/playback failures are explicit;
successful completion reopens recognition after a short speaker-settling delay.

## Inference latency

The earlier 500-token greeting took 21,045 ms. With the same imported Qwen2.5 1.5B
Instruct Q4_K_M model on the connected phone, the isolated native benchmark measured:

| Request | Prompt tokens | Reused tokens | Prompt evaluation | Generation | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| First greeting | 92 | 0 | 4,123 ms | 892 ms (12 tokens) | 5,015 ms |
| Repeated greeting | 92 | 91 | 69 ms | 871 ms (11 tokens) | 941 ms |
| Follow-up with conversation | 124 | 95 | 1,406 ms | 1,055 ms (12 tokens) | 2,462 ms |
| Shorter greeting | 92 | 91 | 76 ms | 666 ms (8 tokens) | 743 ms |
| Changed question | 100 | 84 | 669 ms | 404 ms (6 tokens) | 1,073 ms |

The trace is `app/build/reports/nova-inference-benchmark.txt`. A second, final direct
instrumentation run measured 4,400 ms for the first greeting (92 prompt tokens, 3,162 ms
prefill, 1,237 ms generation / 20 tokens, 16.16 tokens/sec), 552 ms for a repeated greeting,
and 2,189 ms for a follow-up. Long-history trimming produced a 185-token prompt. Its
complete trace is `app/build/reports/nova-inference-final-test.txt`.
These are inference-only
measurements, excluding model loading, recognition and playback; prompt history and
generated answers differ from the original trace. A first short prompt still needs full
prefill, while later matching prefixes can substantially reduce evaluation time. New
history, longer answers, thermal throttling and CPU contention change actual latency.
The 64-token cap bounds long answers; it does not accelerate an answer that already ends
after a few tokens. First-token timing includes prompt preparation and evaluation.
Speed counts non-EOG output tokens divided by generation time; prompt work is excluded.

Upstream references: [llama.cpp Android](https://github.com/ggml-org/llama.cpp/blob/master/docs/android.md),
[Android on-device speech](https://developer.android.com/reference/android/speech/SpeechRecognizer),
[MediaProjection consent](https://developer.android.com/media/grow/media-projection).
