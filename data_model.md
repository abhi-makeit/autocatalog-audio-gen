# Data model: Generate audio from another model

Back to [README](README.md) · ER diagram in [architecture.md](architecture.md#4-er-diagram-new-and-changed-only) · See also [technical_plan.md](technical_plan.md)

Summary:
- **New:** `video_audio_clips`
- **Changed:** `voice_profiles`, `video_jobs`
- **Reused:** `video_segments.narration_text` holds the scene's voice line (the column exists and nothing writes it yet)
- **Not a table:** the audio model registry is an `AUDIO_MODELS` dict in code, like `VIDEO_MODELS`. Tables hold data only: voices and audio clips.

---

## 1. New table: `video_audio_clips`

One row per generated audio clip. It's a separate table, not more columns on `video_segments`, because a clip has its own lifecycle (status, regenerate, retry) and its own timeline edits. `track` leaves room for future SFX or music lanes without another migration.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK, default uuid4 | |
| `video_job_id` | UUID FK `video_jobs.id`, `ondelete=CASCADE`, NOT NULL, indexed | |
| `segment_id` | UUID FK `video_segments.id`, `ondelete=CASCADE`, NULL | The scene this clip is anchored to |
| `track` | String(16), default `'voice'` | Lane name; only `voice` is used in v1 |
| `text` | Text NOT NULL | What was actually synthesized (changes on regenerate) |
| `voice_profile_id` | UUID FK `voice_profiles.id`, `SET NULL` | |
| `model_id` | String(64) NOT NULL | `AUDIO_MODELS` key used |
| `language_code` | String(20) NOT NULL | |
| `style_prompt` | Text NULL | Tone or style instruction sent (Gemini-TTS) |
| `audio_path` | String(500) NULL | Local path or `gcs://` / `s3://` URI, same as `video_path` |
| `format` | String(16) NULL | **As returned by the model** (e.g. `wav`, `mp3`); converted to AAC only at final mix |
| `sample_rate` | Integer NULL | |
| `duration_ms` | Integer NULL | Measured with ffprobe/`wave`, not estimated |
| `start_offset_ms` | Integer, default 0 | **Relative to its scene's start** in the final cut, so reorders and video trims keep sync. Can be negative or run past the scene end |
| `trim_start_ms` / `trim_end_ms` | Integer, default 0 / NULL | Audio in and out points; NULL means the full length |
| `muted` | Boolean, default false | Excluded from the mix |
| `status` | String(32), default `'pending'` | pending / generating / completed / failed (String, as `video_segments.status` is) |
| `error_message`, `retry_count`, `generation_time_ms` | Text / Integer / Integer | Same fields as `video_segments` |
| `created_at`, `updated_at`, `completed_at` | DateTime (`utc_now`) | |

**Constraints and indexes**
- `uq_video_audio_clips_segment_track (segment_id, track)`
- `idx_video_audio_clips_job (video_job_id)`
- `CheckConstraint` on `status`

---

## 2. Changed table: `voice_profiles`

This table becomes the "sample voice list": it holds both prebuilt Gemini voices (shared library) and cloned voices.

**Add columns**

| Column | Type | Notes |
|---|---|---|
| `voice_type` | String(16), default `'cloned'` | `cloned` or `prebuilt` |
| `provider` | String(32), default `'google_chirp3'` | `google_chirp3` or `gemini_tts` |
| `provider_voice_id` | String(64) NULL | Prebuilt voice name, e.g. `Kore`, `Puck` |
| `gender` | String(16) NULL | `male` / `female` / `neutral` |
| `age_group` | String(16) NULL | `child` / `young` / `adult` / `old` |
| `preview_uri` | String(500) NULL | Short playable sample |

**Change**
- Make `voice_cloning_key` **nullable**, because prebuilt Gemini voices have no key.

**Add check constraint**
- `voice_type='cloned'` requires a `voice_cloning_key`.
- `voice_type='prebuilt'` requires a `provider_voice_id`.

---

## 3. Changed table: `video_jobs`

| Action | Column | Notes |
|---|---|---|
| **Add** | `audio_source` String(16), default `'none'` | CheckConstraint `IN ('none','native','external')` |
| **Add** | `audio_model` String(64) NULL | `AUDIO_MODELS` key |
| **Add** | `audio_language_code` String(20) NULL | e.g. `hi-IN` |
| **Add** | `voice_tone` JSON NULL | `{gender, age_group, style}` |
| **Drop** | `narration_script` | Phase-1 only, replaced by per-scene lines, not on main |
| **Keep** | `no_audio` | The flag the pipeline reads. Set server-side as `no_audio = audio_source != 'native'`, so #63's `keep_audio=job.no_audio is False` keeps working unchanged |
| Existing | `voice_profile_id` FK | The selected voice |

---

## 4. Reused column: `video_segments.narration_text`

At segment creation the orchestrator copies the prompted `scene.voice_line` into `VideoSegment.narration_text`. Clip rows are created from segments that have a non-empty `narration_text`.

---

## 5. Migration

- New file `migrations/versions/20261002_external_audio.py`, written by hand like `20260929_voice_profiles.py`.
- `revision="20261002_external_audio"`, `down_revision="20260929_voice_profiles"`.
- `upgrade`:
  - `create_table("video_audio_clips", ...)` with the constraints and indexes above
  - `add_column` for the new `voice_profiles` and `video_jobs` columns
  - alter `voice_profiles.voice_cloning_key` to nullable and add the voice-type check
  - drop `video_jobs.narration_script`
  - backfill: `audio_source='native' WHERE no_audio=false`
- `downgrade` reverses each step.
- **Check:** `alembic upgrade head`, then `downgrade -1`, then `upgrade head` all run cleanly on a dev DB.
