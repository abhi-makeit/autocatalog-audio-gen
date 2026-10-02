# Technical plan: Generate audio from another model

## Background

**What we build on**
- **`origin/main` (#63, `993df6a`):** the "Generate audio" checkbox (`#promptedAudio`). With `no_audio=false`, `merge_segments(keep_audio=True)` keeps each clip's native audio.
- **This branch (`ea10545`, phase 1):**
  - `voice_profiles` table and Chirp 3 Instant Custom Voice cloning (`create_cloned_voice`, `synthesize_with_clone` in `src/services/video/audio_generator.py`).
  - `GET /video/voices`, `POST /video/voices/clone` and `scripts/seed_voice_library.py`.
  - One global `narration_script`, laid over the finished video by `_overlay_narration`.

**Decisions made with the user**
- Providers: Google only (Gemini TTS voices plus Google voice cloning).
- **Audio prompt comes from the video form.** There is no separate voice-line input. Each scene's existing `visual_prompt`, `mood`, `setting` and `duration` drive both its video clip and its audio clip, so the two share the same context and the same length.
- One **audio clip per scene**, anchored to its segment, with a target length equal to the scene `duration`.
- **Two text modes, decided by the scene description:**
  - **`exact`:** text in quotes or after `VO:` / `Voiceover:` in the description is spoken word for word and is never rewritten.
  - **`generated`:** with no such text, Gemini writes a spoken line from the description, sized to the scene.
  - A per-scene `audio_mode` override (`auto` default, `exact`, `generate`) covers the cases where detection is wrong. Scenes in one job can mix modes.
- Review shows a **video track** (reorder, trim, exclude, as today) and an **audio track**. Audio edits: **shift start offset**, **trim head/tail**, **regenerate** (edited text spoken exactly, or rewrite from the prompt), **mute/remove**. Assembly mixes both into one MP4.
- **Mutually exclusive:** if native "Generate audio" is ticked, "Generate audio from another model" is disabled, and the other way round.

**Assumptions (correct me at review)**
0. **The spoken part is stripped from the prompt sent to the video model.** Quoted or `VO:` text is audio-only, so the video model gets just the visual description. This stops models from drawing the words as on-screen captions or trying to lip-sync them.
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

**How the audio prompt is built from the video form**

New file `src/services/video/audio_script.py`:

```python
def split_spoken_text(prompt: str) -> tuple[str, str | None]
    # -> (visual_prompt_without_speech, exact_text or None)

def resolve_text_mode(scene, exact_text) -> Literal["exact", "generated"]

async def write_voice_line(scene: FashionSubScene, language_code: str, tone: VoiceTone,
                           max_words: int | None = None) -> str
```

**`split_spoken_text`** pulls out the text to speak, using these rules:
- Text inside `"…"`, `“…”` or `「…」` quotes.
- The rest of a line that starts with `VO:`, `Voiceover:` or `Narration:` (any case). The quotes around it are optional.
- When there are several pieces, they are joined in order with a space.
- Escaped quotes (`\"`) are kept as a literal quote, so a prompt can still mention a quoted product name without it being spoken.
- The prompt it returns has the spoken parts removed. That is what the video model receives.

**`resolve_text_mode`**

| `scene.audio_mode` | Exact text found? | Mode used |
|---|---|---|
| `auto` (default) | Yes | `exact` |
| `auto` (default) | No | `generated` |
| `exact` | Yes | `exact` |
| `exact` | No | `exact`, speaking the whole description as written |
| `generate` | Either | `generated`. Any found text is passed to Gemini as a hint, not spoken verbatim |

**`write_voice_line`** (only for `generated`)
- Input: the scene's visual prompt (with speech removed), `mood`, `setting`, `scene_purpose` and `duration`, plus the job's language and tone.
- Word budget: `max_words = floor(duration × WORDS_PER_SEC[language] × 0.9)`. `WORDS_PER_SEC` defaults to 2.5 and lives next to `AUDIO_MODELS`; the 0.9 leaves a little breathing room at the end of the scene.
- Uses the existing `GeminiClient.generate_structured` with a short system prompt: "Write one spoken line for this scene, in `<language>`, `<tone>`, at most N words. Describe the product and feeling; do not mention camera moves." Returns `{"line": str}`.
- Exact text is spoken as written, in whatever language it is written in. The job's `language_code` is used only as the TTS locale.

**How each audio clip is fitted to its scene's length**

New `fit_to_duration(path, target_ms) -> (path, fit_mode)` in `audio_generator.py`:

`fit_to_duration(path, target_ms, text_mode)` behaves differently depending on whether the words may change:

| Measured length vs. target | `generated` | `exact` |
|---|---|---|
| Within ±50 ms | Nothing → `none` | Nothing → `none` |
| Shorter | `apad` with silence to exactly `target_ms` → `padded` | Same → `padded` |
| Up to 15% longer | `atempo` (≤ 1.15) to exactly `target_ms` → `sped_up` | Same → `sped_up` |
| More than 15% longer | Rewrite once with `max_words × target/actual`, re-synthesize, re-fit. If still too long, `atrim` the tail → `rewritten` / `trimmed` | `atempo` 1.15, then **keep the whole clip** (never cut the user's words) → `overflow` |

After fitting, `duration_ms == target_ms` in every case except `overflow`, so by default each audio clip starts and ends exactly with its video clip. An `overflow` clip runs into the next scene until the user fixes it at review, for example by lengthening the scene, shifting or trimming the clip, or shortening the text.

---

## 2. Backend changes

| File | Change |
|---|---|
| `src/services/video/audio_models.py` (new) | Registry above, `WORDS_PER_SEC`, plus a `get_audio_model(id)` helper |
| `src/services/video/audio_script.py` (new) | `split_spoken_text()`, `resolve_text_mode()` and `write_voice_line()` above, the last one built on `GeminiClient.generate_structured` |
| `src/services/video/audio_generator.py` | Add `synthesize_gemini_tts(text, voice_name, language_code, style_prompt, output_path)`, which reuses `_tts_beta_post` and `_access_token` (same Google auth). Add a dispatcher, `synthesize_voice_clip(model_id, voice: VoiceProfile, text, language_code, style_prompt, output_path) -> (path, format, sample_rate, duration_ms)`, that routes by `provider` to it or to the existing `synthesize_with_clone`. Write the model's native bytes and measure the real duration. Add `fit_to_duration()` above |
| `src/models/database.py` | `VideoAudioClip` model, new `VoiceProfile` and `VideoJob` columns, `VideoJob.audio_clips` relationship |
| `src/models/video.py` | `VoiceTone`, `AudioConfig {model, voice_profile_id, language_code, tone}`. On `FashionVideoCreateRequest`: `audio_source`, `audio_config`; remove `narration_script`. On `FashionSubScene` and the prompted scene input: optional `audio_mode: Literal["auto","exact","generate"] = "auto"`. Add `AudioClipResponse`, and `audio_clips: list[AudioClipResponse]` plus the audio fields on `VideoJobResponse`. Extend `VoiceProfileResponse` with type, gender, age_group and `preview_url` |
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
  - The existing "at least one scene with a prompt" check already covers audio. `audio_mode` is optional per scene.
  - For each scene that resolves to `exact`, if `words ÷ WORDS_PER_SEC > duration × 1.15`, the response includes a warning (not a rejection), the same one the scene tag shows.
  - Set `no_audio = audio_source != "native"`.
- `AssembleRequest`: add `audio_edits: list[AudioClipEdit {clip_id, start_offset_ms, trim_start_ms, trim_end_ms, muted}]`. The handler persists them onto the clip rows before calling `assemble_with_order`, mirroring how `trims` are handled.
- `POST /{id}/audio-clips/{clip_id}/regenerate {text?, style?, rewrite?}`: new. With `text`, that text is voiced as-is and the clip becomes `text_mode='exact'`. With `rewrite=true`, a new line is written from the scene prompt and the clip becomes `generated`. Either way the result is fitted to the clip's `target_duration_ms` using that mode's rules. Allowed only while the job is at `clips_ready`. Sets the clip to `generating`, runs `BackgroundTasks`, and progress goes over the existing SSE. Same pattern as `regenerate_segment`.
- `regenerate_segment` (video): when a video clip is regenerated with an edited prompt on an external-audio job, its audio clip is **not** regenerated automatically. The lane marks it "prompt changed", and the user can click rewrite.
- `get_video_job`: load and return `audio_clips`, with signed or `/storage` URLs, the way segment `video_url`s are built.

### Orchestrator (`src/services/video/orchestrator.py`)

- **Where segments are created (around line 541):** for external-audio jobs, run `split_spoken_text(scene.visual_prompt)`. The video segment's prompt becomes the visual part only, and the exact text (if any) goes into `VideoSegment.narration_text` (this column already exists and nothing writes it yet).
- **New `_phase_generating_voice_clips(job_id)`:**
  - Skipped unless `audio_source=='external'`.
  - Inserts one `pending` clip row per segment, with `source_prompt` = the scene's original `visual_prompt`, `target_duration_ms` = scene `duration × 1000`, and `text_mode` from `resolve_text_mode`.
  - For each clip, with bounded concurrency (semaphore of 3):
    1. Get the text:
       - `exact`: use the extracted text as it is.
       - `generated`: call `write_voice_line(scene, ...)`.
       
       Store the result in `clip.text` and `VideoSegment.narration_text`.
    2. `synthesize_voice_clip(...)`.
    3. `fit_to_duration(path, target_duration_ms, text_mode)` → store `fit_mode` and the final `duration_ms`.
  - A clip failure is **non-fatal**: that clip is marked `failed` and the user can retry it.
  - It runs **concurrently with Phase 4** (`asyncio.gather`) because the audio needs only the scene prompt and duration, which are known before any video exists. This is required: with `auto_approve=False` the pipeline stops at `clips_ready` *before* Phase 5 (around line 255), and audio has to exist at review time.
- **Phase 5 (`_phase_generating_audio`):** remove the clone or narration branch; it becomes a no-op for external jobs.
- **`assemble_with_order` (around line 1620) and `_phase_assembling`:**
  1. Merge silent clips as today.
  2. Compute each included segment's start in the final cut by summing the trimmed durations of the clips before it in `segment_order`.
  3. Build `SpeechSegment(start_ms = scene_start + start_offset_ms, trim_*)` for each completed, unmuted clip whose segment is not excluded. Clamp the start to at least 0.
     - **Video trims carry over to audio by default.** If the user trimmed `n` ms off the head of a video clip and has not set an audio `trim_start_ms`, the audio gets the same head trim; tail trims likewise. That way a trimmed scene still lines up with its audio. An explicit audio trim always wins.
  4. Call `mix_audio_layers`, with music when a background track is set. Output is one MP4 with one AAC track.
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
3. **Scene editor (`renderScenes`):** no new text input. When external audio is on, each scene card shows a 🔊 tag that updates as the user types. It uses a JS port of `split_spoken_text`, kept in step with the Python version by shared test fixtures.
   - Exact text found: **"Speaks your text · ~3s of 5s"**, turning amber with "too long for scene" past 1.15×.
   - None found: **"Model writes the line · 5s"**.
   - Clicking the tag cycles `audio_mode`: Auto → Exact text → Model writes. The value is sent as `structured_input.scenes[].audio_mode`.
   - A one-line hint under the first scene: *Put words in "quotes" or after VO: to have them spoken exactly.*
4. **Payload:** `_voiceOverPayload()` becomes `_audioPayload()`. It returns `{audio_source:'native', no_audio:false}`, `{audio_source:'external', no_audio:true, audio_config:{...}}`, or `{audio_source:'none', no_audio:true}`.
5. **Video track + audio track** in the clip-review panel, under `#clipReviewGrid`:
   - `renderAudioLane()` draws a horizontal timeline of the current clip order: a video row with widths proportional to trimmed durations, and an audio row under it. Audio clip blocks are positioned at `scene_start + start_offset_ms` and, by default, are the same width as their video block.
   - Each audio block shows its spoken `text`, an **Exact** or **Written by model** label, and a `fit_mode` badge (padded, sped up, trimmed). An `overflow` clip is drawn past its scene in amber and labelled "runs over".
   - **Video edits:** the existing reorder, trim and exclude controls. Trimming a video block also trims its audio block, unless the audio has its own trim.
   - **Audio edits:** drag a block to shift it, drag its edge handles to trim, use the mute toggle, or click ✎ to regenerate, either with edited text (spoken exactly) or with "rewrite from prompt" (model writes).
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
2. **Registry, `audio_script.py`, `synthesize_gemini_tts`, the dispatcher and `fit_to_duration`.**
   - Check: unit tests with `_tts_beta_post` and `GeminiClient` mocked, in the same style as `tests/services/video/test_voice_clone.py`.
   - `split_spoken_text` table tests cover straight, curly and CJK quotes, `VO:` lines, several pieces, escaped quotes and no speech at all. The same fixture file drives the JS tests.
   - `resolve_text_mode` covers every row of its table.
   - `fit_to_duration` tests on generated sine WAVs cover short, slightly long and much too long input in both modes. `ffprobe` shows exactly `target_ms`, except `exact` overflow, which keeps the full length.
3. **API**: endpoints, validation, `audio_edits`, regenerate.
   - Check: route tests for the exclusivity 400, an unsupported language, and a foreign voice_profile.
4. **Orchestrator** (concurrent voice phase, placement maths) **and assembler** (mixing without music, trims, `normalize=0`).
   - Check: tests that clip starts follow reorder and trims, that video trims carry over to audio, that muted clips are excluded, and that `ffprobe` on the output shows one audio stream with full video length.
5. **Frontend:** checkboxes, phase-2 section, scene audio tag and mode switch, video and audio tracks at review.
6. Update `tests/services/video/test_voice_clone.py` for the removed global narration.

---

## 5. Verification (end to end)

- **Run** the stack with `docker compose up`, then `scripts/migrate.sh`. Seed voices with `python scripts/seed_voice_library.py --prebuilt`.
- **Prompted job:** choose a model without native audio (e.g. Veo Fast) and confirm "Generate audio" is disabled. Tick "Generate audio from another model", pick Gemini TTS with a library voice, `hi-IN`, female/adult. Write 3 scene prompts:
  - **5s, model writes:** a plain visual description. The tag reads "Model writes the line".
  - **8s, exact:** a description plus `VO: "<a Hindi sentence>"`. The tag reads "Speaks your text".
  - **4s, exact but too long:** a description plus a quoted line of about 20 words. The tag turns amber.

  Generate.
- **At `clips_ready`:**
  - The audio track shows 3 completed clips under their video clips.
  - Scene 1 is 5s of model-written Hindi that matches the scene.
  - Scene 2 speaks the quoted sentence word for word, fitted to 8s.
  - Scene 3 is labelled "runs over".
  - The video for scene 2 shows no on-screen caption of the quoted text.

  Then:
  - Reorder the scenes and check the clips move with them.
  - Trim a video clip and check its audio is trimmed too.
  - Shift one audio clip by +500 ms and trim another.
  - Regenerate one clip with edited text and one with "rewrite from prompt", and check the labels switch between Exact and Written by model and the SSE updates arrive.
  - Approve & Assemble. Reorder scenes and check the clips move with them. Trim a video clip and check its audio is trimmed too. Shift one audio clip by +500 ms, trim another, regenerate one with edited text and one with "rewrite from prompt", and watch the SSE update. Then Approve & Assemble.
- **Output:** `ffprobe output.mp4` shows one AAC stream with the same duration as the video. Listening, each line starts at its scene plus offset and ends with its scene, except an overflow clip the user left as it is.
- **Exclusivity:** ticking both through the API (`audio_source` native with an `audio_config`) returns 400. Native-only jobs behave exactly as on main (#63).
- **Cloned voice:** upload a sample and consent recording, clone it, and run the same flow with `chirp3-clone`.
- **Storage:** with GCS configured, audio clips appear under `videos/<job_id>/` and the job re-assembles after a local cache wipe.
