# Technical plan: Generate audio from another model

Back to [README](README.md) · Diagrams in [architecture.md](architecture.md) · Tables in [data_model.md](data_model.md)

For engineering review. Implementation starts only after the manager signs off on the design.

---

## Background

**What we build on**
- **`origin/main` (#63, `993df6a`):** the "Generate audio" checkbox (`#promptedAudio`). With `no_audio=false`, `merge_segments(keep_audio=True)` keeps each clip's native audio.
- **This branch (`ea10545`, phase 1):**
  - `voice_profiles` table and Chirp 3 Instant Custom Voice cloning (`create_cloned_voice`, `synthesize_with_clone` in `src/services/video/audio_generator.py`).
  - `GET /video/voices`, `POST /video/voices/clone` and `scripts/seed_voice_library.py`.
  - One global `narration_script`, laid over the finished video by `_overlay_narration`.

**Decisions made with the user**
- Providers: Google only (Gemini TTS voices plus Google voice cloning).
- Text shape: one **voice line per scene**. Each line becomes its own audio clip, anchored to its scene.
- Assembly edits: **shift start offset**, **trim head/tail**, **regenerate** with edited text, **mute/remove**.
- **Mutually exclusive:** if native "Generate audio" is ticked, "Generate audio from another model" is disabled, and the other way round.

**Assumptions (correct me at review)**
1. The per-scene design **replaces** phase 1's single global narration, which is not on main yet. `narration_script`, `_ensure_narration_audio` and `_overlay_narration`, plus the narration textarea, are removed and folded into the new section. The voice library and clone code are kept and extended.
2. **Batch video is out of scope.** It stays `voice_engine="none"`.
3. **The audio model registry lives in code, not a table.** It is an `AUDIO_MODELS` dict, the same as `VIDEO_MODELS` in `video_generator.py:50`, served by a new endpoint.
4. **Open item to verify at implementation time:** whether Gemini-TTS itself accepts a voice-cloning key. If it does, there is one model. If not, cloned voices synthesize through the existing Chirp 3 `voiceCloningKey` path, and prebuilt voices plus tone/style go through Gemini-TTS. The UI is driven by capabilities, so either result fits the same design.

---

## 1. Audio model registry (code)

New file `src/services/video/audio_models.py`, shaped like `VIDEO_MODELS`:

```python
AUDIO_MODELS = {
    "gemini-tts": {            # confirm the current Gemini-TTS model id at implementation
        "label": "Gemini TTS", "provider": "gemini_tts",
        "voice_types": ["prebuilt"],            # + "cloned" if Gemini accepts cloning keys
        "supports_style_prompt": True,           # tone/emotion through natural language
        "languages": [...],                     # BCP-47 codes
        "output_format": "wav", "sample_rate": 24000,
    },
    "chirp3-clone": {
        "label": "Google Voice Clone (Chirp 3)", "provider": "google_chirp3",
        "voice_types": ["cloned"], "supports_style_prompt": False,
        "languages": [...], "output_format": "wav", "sample_rate": 24000,
    },
}
TONE_OPTIONS = {"gender": ["male","female","neutral"], "age_group": ["child","young","adult","old"]}
```

`GET /video/audio-models` returns this registry with `TONE_OPTIONS`. The UI shows only the fields a model supports, and the format is a read-only label taken from `output_format`.

**How tone is applied**
- **Prebuilt voices:** `gender` and `age_group` filter the voice list, since each library voice is tagged. `style`, plus gender and age when the model supports a style prompt, is turned into `style_prompt`, for example "Speak as a warm, elderly male narrator".
- **Cloned voices:** the sample fixes the timbre, so gender and age are display-only. `style` is applied only when `supports_style_prompt` is true.

---

## 2. Backend changes

| File | Change |
|---|---|
| `src/services/video/audio_models.py` (new) | Registry above, plus a `get_audio_model(id)` helper |
| `src/services/video/audio_generator.py` | Add `synthesize_gemini_tts(text, voice_name, language_code, style_prompt, output_path)`, which reuses `_tts_beta_post` and `_access_token` (same Google auth). Add a dispatcher, `synthesize_voice_clip(model_id, voice: VoiceProfile, text, language_code, style_prompt, output_path) -> (path, format, sample_rate, duration_ms)`, that routes by `provider` to it or to the existing `synthesize_with_clone`. Write the model's native bytes and measure the real duration |
| `src/models/database.py` | `VideoAudioClip` model, new `VoiceProfile` and `VideoJob` columns, `VideoJob.audio_clips` relationship |
| `src/models/video.py` | `VoiceTone`, `AudioConfig {model, voice_profile_id, language_code, tone}`. On `FashionVideoCreateRequest`: `audio_source`, `audio_config`; remove `narration_script`. Add `AudioClipResponse`, and `audio_clips: list[AudioClipResponse]` plus the audio fields on `VideoJobResponse`. Extend `VoiceProfileResponse` with type, gender, age_group and `preview_url` |
| `src/api/routes/video.py` | See [API endpoints](#api-endpoints-srcapiroutesvideopy) |
| `src/services/video/orchestrator.py` | See [Orchestrator](#orchestrator-srcservicesvideoorchestratorpy) |
| `src/services/video/assembler.py` | Add `trim_start_ms` and `trim_end_ms` to `SpeechSegment`, applied with `atrim` before `adelay`. Let `mix_audio_layers` work **without music**: when the merged video has no audio stream, use an `anullsrc` base the length of the video. Use `amix ... normalize=0` so voice isn't quietened. Keep `apad` and `duration=first` so the video is never cut |
| `src/services/video/storage_manager.py` | `publish_job_artifacts` also mirrors `video_audio_clips.audio_path` and rewrites them to URIs, the same way it handles segment `audio_path` today |
| `scripts/seed_voice_library.py` | Add a `--prebuilt` mode that seeds Gemini voices (name, `provider_voice_id`, gender, age_group) and synthesizes a short `preview_uri` sample for each |

### API endpoints (`src/api/routes/video.py`)

- `GET /audio-models`: new.
- `GET /voices`: add `?audio_model=&language=&gender=&age_group=` filters and return `preview_url`.
- `POST /voices/clone`: add optional `gender` and `age_group`.
- `create_video_job`: replace the `voice_engine=="clone"` and `narration_script` validation with `audio_source` validation:
  - `external` with native audio requested → 400.
  - The model must exist in `AUDIO_MODELS`.
  - The voice must be the user's own or a library voice, and its `voice_type` must be one the model supports.
  - The language must be in the model's `languages`.
  - There must be at least one non-empty `scenes[].voice_line` (≤ 500 chars).
  - Lines that likely run past their scene (words ÷ 2.5 > scene duration + 2s) get a warning, not a rejection.
  - Set `no_audio = audio_source != "native"`.
- `AssembleRequest`: add `audio_edits: list[AudioClipEdit {clip_id, start_offset_ms, trim_start_ms, trim_end_ms, muted}]`. The handler persists them onto the clip rows before calling `assemble_with_order`, mirroring how `trims` are handled.
- `POST /{id}/audio-clips/{clip_id}/regenerate {text?, style?}`: new. Allowed only while the job is at `clips_ready`. Sets the clip to `generating`, runs `BackgroundTasks`, and progress goes over the existing SSE. Same pattern as `regenerate_segment`.
- `get_video_job`: load and return `audio_clips`, with signed or `/storage` URLs, the way segment `video_url`s are built.

### Orchestrator (`src/services/video/orchestrator.py`)

- **Where segments are created (around line 541):** copy the prompted `scene.voice_line` into `VideoSegment.narration_text`. This column already exists and nothing writes it yet.
- **New `_phase_generating_voice_clips(job_id)`:**
  - Skipped unless `audio_source=='external'`.
  - Inserts `pending` clip rows for segments that have `narration_text`, then synthesizes them with bounded concurrency (semaphore of 3).
  - A clip failure is **non-fatal**: that clip is marked `failed` and the user can retry it.
  - It runs **concurrently with Phase 4** (`asyncio.gather`) because the audio needs only text. This is required: with `auto_approve=False` the pipeline stops at `clips_ready` *before* Phase 5 (around line 255), and audio has to exist at review time.
- **Phase 5 (`_phase_generating_audio`):** remove the clone or narration branch; it becomes a no-op for external jobs.
- **`assemble_with_order` (around line 1620) and `_phase_assembling`:**
  1. Merge silent clips as today.
  2. Compute each included segment's start in the final cut by summing the trimmed durations of the clips before it in `segment_order`.
  3. Build `SpeechSegment(start_ms = scene_start + start_offset_ms, trim_*)` for each completed, unmuted clip whose segment is not excluded. Clamp the start to at least 0.
  4. Call `mix_audio_layers`, with music when a background track is set.
  5. A mixing failure is non-fatal: deliver the silent video and log, matching the current `_overlay_narration` behavior.
- **Remove** `_ensure_narration_audio` and `_overlay_narration`.

---

## 3. Frontend changes (`frontend/pages/video.html`, `frontend/js/video.js`)

1. **Checkboxes** in the prompted section, starting from main's `#promptedAudio`:
   - Add `#externalAudio` "Generate audio from another model" directly below it.
   - `Video.toggleAudioSource()` enforces exclusivity: ticking one unchecks and disables the other.
   - `#promptedAudio` is disabled, with the hint "model has no native audio", when the selected model doesn't support it. The hint comes from a small `_supportsNativeAudio(model)` helper next to `_getMaxImagesPerScene` (around line 547).
2. **Phase-2 Audio section** `#externalAudioSection`. It reuses and reshapes this branch's `#voiceOverSection` markup and is shown when `#externalAudio` is checked. It contains:
   - **Audio model:** `#audioModelSelect`, filled from `GET /video/audio-models`.
   - **Language:** `#audioLanguageSelect`, from the model's `languages`.
   - **Tone:** `#toneGender` and `#toneAge` selects, plus `#toneStyle` text, shown only when `supports_style_prompt`. Gender and age re-filter the voice list.
   - **Voice:** `#voiceSelect` lists library and my voices, with a ▶ button that plays `preview_url`. The existing "+ Upload your voice" clone panel (sample + consent) is unchanged apart from extra gender and age fields.
   - **Output format:** a read-only label from `output_format`.
3. **Scene editor (`renderScenes`):** each scene card gets a "Voice line" textarea when external audio is on, with a live "~Ns of speech / Ms scene" hint. The value goes into `structured_input.scenes[].voice_line`.
4. **Payload:** `_voiceOverPayload()` becomes `_audioPayload()`. It returns `{audio_source:'native', no_audio:false}`, `{audio_source:'external', no_audio:true, audio_config:{...}}`, or `{audio_source:'none', no_audio:true}`.
5. **Audio lane** in the clip-review panel, under `#clipReviewGrid`:
   - `renderAudioLane()` draws a horizontal timeline of the current clip order, with widths proportional to trimmed durations. Audio clip blocks are positioned at `scene_start + start_offset_ms`.
   - **Edits:** drag a block to shift it, drag its edge handles to trim, use the mute toggle, or click ✎ to regenerate with edited text.
   - Pointer handling reuses the `_onTrimPointer*` pattern (around line 1797).
   - The lane re-renders when clips are reordered, excluded or trimmed, since offsets are relative to each scene.
   - **State:** `_audioEdits` is saved with the existing `_saveReviewState` (localStorage) and sent as `audio_edits` in `approveAndAssemble()` (around line 2352).
   - **Status:** `handleSSEMessage` updates clip status after a regenerate.
   - **Preview:** a per-clip ▶ button. A fully synced preview is out of scope for v1; the assembled output is the real preview.

---

## 4. Implementation order

0. **Rebase** `feat/audio-intigration-in-video` onto `origin/main` to pick up #63. Resolve the `create_video_job` `no_audio` conflict in favor of the new `audio_source` rule.
   - Check: `pytest tests/services/video` passes.
1. **Migration and models** (see [data_model.md](data_model.md) and the `database.py` / `video.py` rows above).
   - Check: `alembic upgrade head`, then `downgrade -1`, then `upgrade head` all run cleanly on a dev DB.
2. **Registry, `synthesize_gemini_tts` and the dispatcher.**
   - Check: unit tests with `_tts_beta_post` mocked, in the same style as `tests/services/video/test_voice_clone.py`.
3. **API**: endpoints, validation, `audio_edits`, regenerate.
   - Check: route tests for the exclusivity 400, an unsupported language, and a foreign voice_profile.
4. **Orchestrator** (concurrent voice phase, placement maths) **and assembler** (mixing without music, trims, `normalize=0`).
   - Check: tests that clip starts follow reorder and trims, that muted clips are excluded, and that `ffprobe` on the output shows one audio stream with full video length.
5. **Frontend:** checkboxes, phase-2 section, voice lines, audio lane.
6. Update `tests/services/video/test_voice_clone.py` for the removed global narration.

---

## 5. Verification (end to end)

- **Run** the stack with `docker compose up`, then `scripts/migrate.sh`. Seed voices with `python scripts/seed_voice_library.py --prebuilt`.
- **Prompted job:** choose a model without native audio (e.g. Veo Fast) and confirm "Generate audio" is disabled. Tick "Generate audio from another model", pick Gemini TTS with a library voice, `hi-IN`, female/adult, and add voice lines to 2 of 3 scenes. Generate.
- **At `clips_ready`:** the audio lane shows 2 completed clips. Reorder scenes and check the clips move with them. Shift one by +500 ms, trim one, mute nothing, regenerate one with edited text and watch the SSE update. Then Approve & Assemble.
- **Output:** `ffprobe output.mp4` shows one AAC stream with the same duration as the video. Listening, each line starts at its scene plus offset.
- **Exclusivity:** ticking both through the API (`audio_source` native with an `audio_config`) returns 400. Native-only jobs behave exactly as on main (#63).
- **Cloned voice:** upload a sample and consent recording, clone it, and run the same flow with `chirp3-clone`.
- **Storage:** with GCS configured, audio clips appear under `videos/<job_id>/` and the job re-assembles after a local cache wipe.
