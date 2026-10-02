# Architecture: Generate audio from another model

## 1. System architecture

The frontend adds a second, mutually exclusive audio option and a phase-2 Audio section. The API gains endpoints for audio models, voice filters, clip regeneration and audio edits at assembly. The orchestrator generates silent video clips and voice clips side by side, and the assembler mixes the placed voice clips into the final MP4.

```mermaid
flowchart LR
  subgraph FE["Frontend (video.html / video.js)"]
    F1["Generate audio ☐ (native)"]
    F2["Generate audio from another model ☐"]
    F3["Phase-2 Audio section<br/>model · voice sample/upload · language · tone · format"]
    F4["Scene editor<br/>+ Voice line per scene"]
    F5["Clip review + Audio lane<br/>shift · trim · mute · regenerate"]
    F1 -. mutually exclusive .- F2
    F2 --> F3 --> F4
  end

  subgraph API["FastAPI /api/v1/video"]
    A1["GET /audio-models"]
    A2["GET /voices (filters)<br/>POST /voices/clone"]
    A3["POST /projects/{id}/create<br/>audio_source + audio_config"]
    A4["POST /{id}/audio-clips/{clip}/regenerate"]
    A5["POST /{id}/assemble<br/>+ audio_edits[]"]
    A6["GET /{id}/progress (SSE)"]
  end

  subgraph SVC["Services"]
    R["audio_models.py<br/>AUDIO_MODELS registry"]
    O["FashionVideoOrchestrator"]
    VG["VideoGenerator<br/>(silent clips)"]
    AG["audio_generator.py<br/>synthesize_voice_clip()"]
    AS["VideoAssembler<br/>merge_segments + mix_audio_layers"]
  end

  subgraph EXT["Google"]
    G1["Gemini-TTS<br/>(prebuilt voices + style prompt)"]
    G2["Chirp 3 Instant Custom Voice<br/>(cloned voices)"]
  end

  subgraph DATA["Postgres + Object storage"]
    T1[(voice_profiles)]
    T2[(video_jobs)]
    T3[(video_segments)]
    T4[(video_audio_clips · NEW)]
    S[("GCS/S3<br/>jobs/{id}/audio/*.wav")]
  end

  F3 --> A1 & A2
  F4 --> A3
  F5 --> A4 & A5
  A1 --> R
  A3 --> O
  A5 --> O
  A4 --> AG
  O --> VG
  O --> AG
  O --> AS
  AG --> G1 & G2
  AG --> T4
  AG --> S
  O --> T2 & T3
  A2 --> T1
  AS --> S
```

---

## 2. End-to-end sequence

From ticking the checkbox to the final MP4. Voice clips need only text, so they are generated **in parallel** with the silent video clips (Phase 4 and Phase 4b) and are ready when the job pauses at `clips_ready` for review.

```mermaid
sequenceDiagram
  actor U as User
  participant FE as video.js
  participant API as routes/video.py
  participant OR as Orchestrator
  participant VG as VideoGenerator
  participant AG as audio_generator
  participant G as Google TTS
  participant AS as Assembler
  participant DB as Postgres

  U->>FE: tick "Generate audio from another model"
  FE->>API: GET /audio-models, GET /voices?language=&gender=&age_group=
  U->>FE: pick model, voice (library / upload), language, tone, write voice line per scene
  FE->>API: POST /projects/{id}/create {audio_source:"external", audio_config, scenes[].voice_line}
  API->>API: validate (model exists, voice accessible, not native, lines fit)
  API->>DB: video_jobs (no_audio=true, audio_*), structured_input.scenes
  FE->>API: POST /{id}/generate
  API->>OR: BackgroundTasks.run(job_id)
  OR->>DB: segments (narration_text = voice_line)
  par Phase 4 — video clips (silent)
    OR->>VG: generate each clip (no_audio=true)
  and Phase 4b — voice clips (text-only, no video dependency)
    OR->>DB: insert video_audio_clips (pending)
    OR->>AG: synthesize_voice_clip() per line
    AG->>G: Gemini-TTS / Chirp 3 clone
    G-->>AG: audio bytes (model-native format)
    AG->>DB: clip.audio_path, format, duration_ms, status=completed
  end
  OR-->>FE: SSE clips_ready (job pauses, auto_approve=false)
  FE->>API: GET /{id} (segments + audio_clips)
  U->>FE: reorder/trim video, shift/trim/mute audio, regenerate a line
  FE->>API: POST /{id}/audio-clips/{clip}/regenerate {text}
  API->>AG: background re-synthesize, SSE update
  U->>FE: Approve & Assemble
  FE->>API: POST /{id}/assemble {segment_order, trims, audio_edits[]}
  API->>DB: persist audio_edits onto clip rows
  API->>OR: assemble_with_order
  OR->>AS: merge_segments (silent) → mix_audio_layers(placed voice clips)
  AS-->>OR: output.mp4 (single video, one audio track)
  OR-->>FE: SSE completed
```

---

## 3. Audio clip lifecycle

States of one `video_audio_clips` row. A failure is non-fatal for the job and can be retried. Shift, trim and mute only change edit fields and never re-synthesize; regenerate does.

```mermaid
stateDiagram-v2
  [*] --> pending: job generate (voice line present)
  pending --> generating
  generating --> completed
  generating --> failed: provider error (non-fatal for job)
  failed --> generating: Regenerate
  completed --> generating: Regenerate (edited text)
  completed --> completed: shift / trim / mute (edit fields only)
  completed --> [*]: mixed at assembly (unless muted)
```

---

## 4. ER diagram (new and changed only)

`video_audio_clips` is new. `voice_profiles` and `video_jobs` gain columns. `video_segments.narration_text` is reused to hold the scene's voice line. Column details are in [data_model.md](data_model.md).

```mermaid
erDiagram
  users ||--o{ voice_profiles : "owns (NULL = library)"
  video_jobs ||--o{ video_segments : has
  video_jobs ||--o{ video_audio_clips : has
  video_segments ||--o| video_audio_clips : "anchors (track=voice)"
  voice_profiles ||--o{ video_jobs : "selected voice"
  voice_profiles ||--o{ video_audio_clips : "spoken by"

  voice_profiles {
    uuid id PK
    uuid user_id FK "NULL = shared library"
    string name
    string language_code
    string sample_uri "clone source / preview"
    text voice_cloning_key "NOW NULLABLE (prebuilt has none)"
    string voice_type "NEW: cloned | prebuilt"
    string provider "NEW: google_chirp3 | gemini_tts"
    string provider_voice_id "NEW: e.g. Kore, Puck"
    string gender "NEW: male | female | neutral"
    string age_group "NEW: child | young | adult | old"
    string preview_uri "NEW: short playable sample"
    datetime created_at
  }

  video_jobs {
    uuid id PK
    bool no_audio "derived: audio_source != native"
    string audio_source "NEW: none | native | external"
    string audio_model "NEW: AUDIO_MODELS key"
    string audio_language_code "NEW: e.g. hi-IN"
    json voice_tone "NEW: {gender, age_group, style}"
    uuid voice_profile_id FK "existing"
  }

  video_segments {
    uuid id PK
    text narration_text "REUSED: the scene's voice line"
  }

  video_audio_clips {
    uuid id PK
    uuid video_job_id FK
    uuid segment_id FK
    uuid voice_profile_id FK
    string track
    text text
    string model_id
    string language_code
    text style_prompt
    string audio_path
    string format
    int sample_rate
    int duration_ms
    int start_offset_ms
    int trim_start_ms
    int trim_end_ms
    bool muted
    string status
    text error_message
    int retry_count
    int generation_time_ms
    datetime created_at
    datetime updated_at
    datetime completed_at
  }
```
