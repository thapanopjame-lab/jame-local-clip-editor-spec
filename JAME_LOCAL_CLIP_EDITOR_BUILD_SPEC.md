# JAME Local Clip Editor — Build Specification

**Document purpose:** A public, implementation-ready specification for building a new local-first AI video editor with a desktop coding agent.

**Audience:** Developers and AI coding agents.

**Normative terms:** **MUST** means required; **SHOULD** means recommended unless a documented reason prevents it; **MAY** means optional.

This specification describes product behavior and interfaces. It does not provide or require access to any existing project's source code, private data, credentials, datasets, or model weights.

## 1. Product Overview

Build a desktop **Local-first AI Video Editor**. The user imports one or more video files, describes the desired edit in ordinary language, reviews a local preview, requests changes in ordinary language, optionally teaches a reusable preference, and exports the result.

### User workflow

1. Import video clips.
2. Optionally select a style and target duration.
3. Type an editing request and choose **AI Edit**.
4. Review a locally rendered preview.
5. Pause, mark a range if useful, and type a correction.
6. Choose **Apply Fix** to revise only the current project.
7. Optionally choose **Teach Brain** to save an explicit learning correction.
8. Export the final video.

### Core responsibilities

| Component | Responsibility |
|---|---|
| AI provider | Reason about the supplied compact context and propose an edit plan |
| Local application | Analyze media, validate plans, manage timelines, render previews and exports |
| User | Direct the edit, review results, and explicitly approve reusable learning |

The AI **MUST NOT** need to receive a full-resolution source video by default. Heavy media processing and rendering **MUST** remain local.

## 2. Core Stack

| Area | Recommended choice | Requirement |
|---|---|---|
| Desktop runtime | Python | Keep the application usable as a native desktop program |
| UI | PySide6 / Qt Widgets | Provide a responsive Simple Mode and a separate Advanced Mode |
| Media tools | FFmpeg and ffprobe | Probe, trim, concatenate, transform, preview, and export |
| Playback | QMediaPlayer and QVideoWidget | Reuse one player for the Simple Mode preview |
| Speech recognition | faster-whisper Small, local execution | Make model installation and download status explicit |
| Local analysis | Python modules around FFmpeg and ASR | Keep transcripts, silence data, scenes, proxies, and frames local |
| AI integration | Provider abstraction | Support multiple providers without coupling the editor core to one |
| Configuration | Versioned JSON files | Keep styles, rules, presets, and learning data inspectable |
| Tests | pytest and Qt tests | Cover units, UI behavior, mocked workflows, and opt-in media integration |

The implementation MAY substitute compatible components, but MUST document why and preserve the privacy, validation, and non-destructive editing requirements.

## 3. Architecture

~~~text
Source Video Files
       │
       ▼
Local Analyzer ──► Context Pack Builder
                         │
                         ▼
                 AI Model Router
                         │
                         ▼
                    Editor Brain
                         │
                         ▼
                  Structured Plan
                         │
                         ▼
                  Local Validation
                         │
                         ▼
                   ProjectTimeline
                         │
                         ▼
                  Local Renderer
                         │
                         ▼
                   Preview / Export
                         │
                         ▼
                 User Feedback
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       Apply Fix only          Optional Teach Brain
       to this project          after confirmation
~~~

### Component boundaries

- **UI:** Collect commands and preferences, show progress and previews, and never expose raw model payloads in Simple Mode.
- **Core Video:** Probe sources, resolve FFmpeg tools, and provide safe media operations.
- **Project Timeline:** Represent ordered, non-destructive source ranges and editing metadata.
- **Analysis:** Produce local transcripts, silence ranges, scene boundaries, speech ranges, representative frames, and analysis status.
- **Context Pack:** Select compact, relevant evidence for each model request.
- **AI Provider:** Send and receive structured requests through a documented provider boundary.
- **Model Router:** Select a model using provider capabilities and the task type, not a hard-coded model name.
- **Editor Brain:** Load and validate style profiles, decision rules, presets, and approved learning corrections.
- **Validation:** Reject malformed or unsafe plans before they can change the timeline.
- **Rendering / Export:** Render from source files plus the current timeline; never rewrite an imported source.
- **Learning:** Save only after an explicit user action; keep raw corrections distinct from any carefully approved general rule.

Modules SHOULD communicate through typed data structures or documented contracts. The UI SHOULD orchestrate existing modules rather than duplicate analysis, validation, or rendering logic.

## 4. Suggested Project Folder Structure

~~~text
app/
  main.py
  ui/
    main_window.py
    director_panel.py
    advanced_tools.py
    widgets/
  core/
    ffmpeg.py
    ffmpeg_resolver.py
    transcription.py
    project_timeline.py
    project_analysis.py
    context_pack.py
    simple_workflow.py
    model_provider.py
    model_router.py
    project_render.py
    credentials.py
  providers/
    base.py
    chatgpt_plan.py
    openai_compatible.py
  editor_brain/
    loader.py
    validator.py
    narrative.py
    timeline_adapter.py
    schema/
    styles/
    rules/
    presets/
    learning/
    examples/
tests/
config/
cache/
temp/
exports/
logs/
README.md
~~~

This is a starting point, not a mandate to create empty folders or duplicate modules. Keep modules cohesive, add only folders needed by the implementation, and keep generated media and private user data out of version control.

## 5. Source Model

Represent every imported file as an immutable source record.

### SourceClip fields

| Field | Type | Purpose |
|---|---|---|
| id | string | Stable opaque ID assigned locally |
| filepath | local path | Used only by local media tools; never sent to a provider |
| filename | string | Local display label; omit or redact from provider context when unnecessary |
| duration_seconds | finite number | Media duration |
| width, height | positive integers or null | Video dimensions |
| fps | positive number or null | Frame rate |
| has_audio | boolean | Audio availability |
| analysis_status | enum | PENDING, ANALYZING, READY, or FAILED |
| transcript_segments | local collection | ASR output with source-relative start/end times |
| silence_ranges | local collection | Detected low-volume intervals |
| scene_boundaries | local collection | Candidate visual cut boundaries |
| speech_ranges | local collection | Candidate speech intervals |
| representative_frames | local collection | Local frame paths and timestamps |

Stable IDs SHOULD be derived from a canonical local identity and a file fingerprint or equivalent scheme. They MUST be collision-resistant, repeatable for the same source, and independent of display order. Keep the derivation and local paths inside the application. Do not use an ID as evidence about the content of a clip.

Missing media, changed fingerprints, and stale cached analysis MUST be detected. Source records are references to imported media; editing MUST create timeline decisions and rendered outputs without mutating the source file.

## 6. Project Timeline

The timeline represents an ordered edit as references to ranges in imported source files. It is non-destructive and supports repeated use of a source, such as A → C → B → A.

### TimelineItem fields

| Field | Type | Purpose |
|---|---|---|
| source_id | string | Must identify an imported source |
| source_start_seconds | finite number | Inclusive source-relative start |
| source_end_seconds | finite number | Exclusive source-relative end |
| order | non-negative integer | Output ordering |
| role | allowlisted string | For example HOOK, MAIN, or DEMONSTRATION |
| reason | short string | Plain explanation for the decision |
| tags | array of allowlisted strings | Semantic labels |
| actions | array | Validated visual or audio preset actions |

### Required behavior

- Preserve an explicit ordering and allow the same source_id more than once.
- Validate source existence, finite timestamps, positive range length, source duration bounds, unique order values, roles, tags, and actions.
- Apply a complete candidate timeline atomically: invalid input MUST leave the previous timeline unchanged.
- Support snapshots and undo for accepted timeline changes.
- Keep source time distinct from preview/output time. Convert between them through timeline mappings.

## 7. AI Edit Plan Contract

The model returns one structured plan. The application validates the plan locally before applying it.

### Illustrative plan

~~~json
{
  "schema_version": "1.0",
  "plan_id": "plan_demo_001",
  "style_id": "default_youtube",
  "preset_id": "default_youtube",
  "sources": [
    {"source_id": "source_demo_a", "duration_seconds": 92.0},
    {"source_id": "source_demo_b", "duration_seconds": 48.0}
  ],
  "timeline": [
    {
      "clip_id": "opening_hook",
      "source_id": "source_demo_a",
      "start_seconds": 12.5,
      "end_seconds": 24.8,
      "role": "HOOK",
      "reason": "Opens with the clearest demonstration.",
      "tags": ["HOOK", "DEMONSTRATION"],
      "actions": []
    },
    {
      "clip_id": "supporting_detail",
      "source_id": "source_demo_b",
      "start_seconds": 4.0,
      "end_seconds": 13.5,
      "role": "MAIN",
      "reason": "Adds context for the opening.",
      "tags": ["CONTEXT"],
      "actions": []
    }
  ]
}
~~~

The IDs and media details above are synthetic examples.

### Validation requirements

Reject a plan if it has an unsupported schema version, unknown source or preset, duplicate clip IDs, invalid role/action, non-finite or negative timestamps, end <= start, ranges beyond source duration, invalid ordering, unsupported fields, or a reference to a source not in the current project. Enforce configured size and clip-count limits.

Do not apply partial plans. If a provider returns malformed content, the application MAY request one bounded repair attempt with the validation issue; it MUST still validate the repaired response. If validation fails again, keep the current timeline intact and show a short user-facing error.

### Revision patch contract

Revision requests SHOULD identify the current plan or revision as their base and express a bounded replacement or patch. Reject stale-base patches. Validate the entire proposed result and apply atomically. A revision may change the current project's timeline only.

## 8. Editor Brain

Editor Brain is a local, versioned configuration system. It is **not** a model-weight store.

| Part | Contents |
|---|---|
| Style Profile | Canvas, typography, caption policy, graphic preferences, transition limits |
| Decision Rules | Conditions over allowlisted tags and safe editing decisions |
| Presets | Target format, target duration, output shape, supported action names |
| Learning Corrections | User-approved guidance with provenance and context |
| Narrative Rules | Meaning, continuity, compression, and transcript-boundary checks |
| Schemas / Examples | Versioned contracts and synthetic examples |

Support several generic styles, for example default_youtube, short_form, tutorial, and product_review. Style data MUST be validated before use. Learning corrections MUST be stored separately from built-in rules so users can inspect, remove, or export them.

## 9. Narrative Rules

The narrative validator SHOULD check:

- COMPLETE_SENTENCE: avoid cutting off a spoken sentence when a safe alternative exists.
- PRESERVE_CONTEXT: retain enough setup to make a statement understandable.
- CAUSE_ACTION_RESULT: keep necessary cause, action, and result together.
- NUMERIC_CONTEXT: retain surrounding explanation for important numbers or comparisons.
- NO_EXTREME_COMPRESSION: warn when the requested duration removes too much meaningful content.
- SEMANTIC_ADJACENCY: flag edits that join unrelated ideas without a transition or explanation.

When goals conflict, use this priority:

1. Meaning
2. Story continuity
3. Natural speech
4. Remove clear repetition
5. Reduce duration

Warnings SHOULD include a stable code, severity, and concise explanation. The UI MUST NOT expose raw validator payloads in Simple Mode.

## 10. Compression Safety

Compute compression_ratio = edited_duration / source_duration for the relevant set of kept sources or for another clearly documented denominator. Use the same denominator consistently in the UI and tests.

| Ratio | Severity | Expected behavior |
|---:|---|---|
| Below 50% | WARNING | Explain that the edit is heavily compressed; allow review or request a safer plan |
| Below 25% | HIGH_RISK | Require an explicit review step or regenerate before applying |

Thresholds MUST be configurable. A warning does not authorize unsafe ranges or bypass structural validation.

## 11. Transcript Boundary Snap

Provide helpers equivalent to snap_to_segment_start() and snap_to_segment_end().

- Snap only when a proposed edge is within a configurable tolerance of a valid transcript boundary.
- Do not move an edge across a different source.
- Preserve the original plan and return a new candidate.
- Revalidate the candidate after snapping.
- Test that distant edges remain unchanged and edits never exceed source bounds.

## 12. Local Analysis

The Local Analyzer SHOULD support:

- faster-whisper transcription with source-relative timestamps.
- FFmpeg silence detection.
- Scene-boundary detection.
- Speech-range derivation.
- Representative frame extraction.
- Optional local proxy generation.

All raw analysis artifacts, caches, frame files, and proxies MUST remain local by default. Cache entries MUST be keyed by a source fingerprint, analysis version, and model/configuration identity so stale results can be invalidated.

Silence is evidence, not an editing command. The system MUST NOT remove a range solely because it contains no speech; silent demonstrations, reactions, atmosphere, and B-roll may carry meaning.

Analysis SHOULD be cancellable and report per-source status. If ASR or scene analysis is unavailable, continue with the remaining available evidence and clearly mark the missing capability.

## 13. Context Pack Builder

Build compact provider context on demand. Do not resend a full transcript for every revision.

| Level | Contents | When to include |
|---|---|---|
| Level 1 | Overview, chapter candidates, important segments, hook candidates, source metadata | Initial plan and ordinary revisions |
| Level 2 | Transcript segments near the current playhead or selected range | Focused revision |
| Level 3 | Selected representative frames | Only when useful and supported by the provider |

Use stable synthetic segment references such as CLIP01.SEG12 within a project context. They MUST be mapped locally to source-relative times. They are references, not personal or globally meaningful IDs.

The context pack MUST include only what the task requires. It SHOULD include the current style, target duration, compact timeline, relevant Brain rules, current preview time, and selected source ranges for revision requests. Keep file paths, unrelated transcript, private notes, and raw media out of the pack.

## 14. Model Router

Define task roles by capability rather than vendor model name.

| Role | Suitable work |
|---|---|
| Lightweight | Preprocessing, tagging, summarizing, routine revisions, context compression |
| Reasoning / editor | Story outline, hook selection, narrative planning, clip ordering, difficult revisions |

Choose from the active provider's discovered model catalog and capability metadata. Do not hard-code model names into the editor core or Simple Mode UI. If a role is unavailable, use an explicitly configured alternative or report the limitation; do not silently switch to a paid provider.

## 15. Simple Mode UX

Simple Mode is a Director interface: **the preview is on the left; the conversation is on the right**. Use a resizable horizontal splitter, with sensible minimum sizes.

### Left: Video Preview

- Video preview using the existing QMediaPlayer/QVideoWidget implementation.
- Seek bar that follows playback and seeks in milliseconds.
- Play/Pause, Stop, −5 sec, +5 sec.
- Current time / total time displayed as HH:MM:SS.mmm.
- Mark In, Mark Out, Clear Selection.
- Previous Cut and Next Cut when boundaries are available.

### Right: AI Editor

- Conversation history for the current project/session.
- A multiline natural-language input.
- Enter sends; Shift+Enter inserts a newline.
- One primary action labeled **AI Edit** before the first preview and **Apply Fix** after a preview exists.
- Visible **Teach Brain** button, disabled until an appropriate correction is available.
- Quick feedback buttons and compact progress messages.

Simple Mode MUST hide JSON, raw prompts, raw model output, Analysis Packs, internal source IDs, manual validation controls, and debug logs. Keep these in Advanced / Development Mode. Resizing the window MUST NOT make the command input, Apply Fix, or Teach Brain unreachable.

## 16. Natural-Language Editing

Example requests:

- “Cut this to about 30 seconds.”
- “Keep only the most important parts.”
- “Remove this section.”
- “Keep this section.”
- “This cut is too fast.”
- “Change the hook.”
- “Add a title here.”

The first message is an editing instruction. After a preview exists, each new message is a revision request. The application MUST attach current preview time and, when set, selected range automatically.

Revision context MUST include, when available:

- User feedback / requested change.
- Current preview time in milliseconds or exact seconds.
- Selected range and its mapping to source-relative ranges.
- Active style and target duration.
- Current timeline and current edit plan.
- Relevant transcript segments and stable segment references.
- Relevant Brain rules and approved learning guidance.

The provider response MUST be treated as untrusted data and validated before any timeline update.

## 17. Mark In / Mark Out

Mark In and Mark Out use preview/output time with millisecond precision. For example:

~~~text
IN  00:00:37.420
OUT 00:00:42.850
~~~

If the end is not after the start, reject the selection and show a plain-language message. Use Selection MAY insert a human-readable range in the command field, but users MUST NOT have to type timestamps themselves.

Map the selected preview range through ProjectTimeline to one or more source-relative ranges before sending it to a provider. Preserve both the preview range and source mapping so the model cannot confuse output time with source time.

## 18. Style and Duration

Provide a style selector and these duration choices:

- Auto
- 15 sec
- 30 sec
- 60 sec
- 3 min
- Custom

Custom duration MUST have a bounded positive numeric value. Selected style and duration MUST be added to the initial planning context automatically. Do not require users to repeat a selected duration in their natural-language instruction.

## 19. Quick Feedback

Provide buttons such as:

- Good
- Too Short
- Too Long
- Hard to Follow
- Cut Too Fast
- Keep This
- Remove This
- Too Many Effects

Clicking a button MUST fill or append text in the input. It MUST NOT send a request or save a learning correction automatically. The user must be able to edit the text before sending it.

## 20. Teach Brain

Keep **Apply Fix** and **Teach Brain** separate:

- **Apply Fix** changes the current project timeline and preview only.
- **Teach Brain** saves a correction for possible use in future projects.

Never learn automatically from every revision. Save at least:

| Field | Meaning |
|---|---|
| active_style | Style in effect |
| feedback | User's stated preference |
| correction_type | General category such as pacing or range preservation |
| current_timestamp | Preview/source timestamp context, if available |
| selected_range | Preview and mapped source range, if available |
| project_context | Minimal non-sensitive clip/project context |
| previous_decision | Relevant prior AI decision |
| user_correction | User-approved correction text |
| context | Minimal supporting context and provenance |
| created_at | UTC timestamp |

Store raw correction separately from any generalized lesson. Generalize only when a rule is directly supported by the user's wording and context; otherwise keep the raw correction. Do not infer a broad rule from a clip-specific instruction.

## 21. Feedback Learning Loop

~~~text
AI Edit
  → Preview
  → User correction
  → Apply Fix
  → User decides whether the correction is reusable
  → Teach Brain (optional)
  → Store correction with provenance
  → Retrieve relevant guidance in a future project
~~~

Use only relevant, validated learning entries in later context packs. Provide a way to inspect and remove learned entries. A user declining Teach Brain MUST NOT lose the current-project correction.

## 22. Subtitle System

Implement subtitles as a staged local pipeline:

~~~text
Audio
  → ASR
  → Spelling correction
  → Glossary correction
  → Context correction
  → Subtitle generation
  → Local render
~~~

Support a roadmap for:

- FULL_SUBTITLE
- KEYWORD_SUBTITLE
- HOOK_SUBTITLE

Keep subtitle timing tied to the source/timeline mapping. Let users review corrected text before export. Subtitle styling SHOULD use local presets.

## 23. Spelling Correction

Support a local glossary, user-approved learned corrections, and optional provider-assisted context correction.

- Preserve the original transcript and the corrected transcript separately.
- Mark uncertain corrections as UNCERTAIN for review.
- Do not hallucinate names, brands, numbers, or technical terms.
- Apply glossary entries only when the match and context are unambiguous.
- Record user-approved glossary changes separately from automatic proposals.

## 24. Text and Graphics Presets

Graphics are selected from a local allowlist; the model does not generate executable drawing code.

Example preset identifiers:

- TEXT_BIG_CENTER
- TEXT_KEYWORD_BOTTOM
- ARROW_POINT
- CIRCLE_HIGHLIGHT
- LABEL_RIGHT
- COMPARE_SPLIT

Validate preset ID, supported parameters, text length, timing, canvas bounds, and source range. The renderer owns visual implementation and versioning.

## 25. Effect Presets

Potential allowlisted effects include:

- PUNCH_IN_MEDIUM
- ZOOM_DETAIL_SLOW
- FREEZE_EMPHASIS
- BROLL_FULL

The model may choose a supported preset and provide bounded parameters. It MUST NOT emit arbitrary FFmpeg filter expressions or executable effect code. Unsupported effects should be rejected or omitted with a clear warning.

## 26. Local Rendering

The renderer SHOULD support:

- Source-range trimming.
- Ordered multi-clip concatenation.
- Consistent resolution, pixel format, and frame-rate normalization.
- Audio handling and optional normalization.
- Subtitle burn-in.
- Allowlisted text/graphics presets.
- Allowlisted effect presets.
- Preview-quality rendering and final export profiles.

Render from immutable sources plus a validated timeline snapshot. Write to a new output path. Check FFmpeg exit status, output existence, media metadata, duration, audio/video streams, disk space, and cancellation. A render failure MUST preserve source files and the previous valid project state.

## 27. Proxy Workflow

For large files, the application MAY generate local 480p/720p proxies:

~~~text
Original source
  → Local proxy
  → Local analysis and preview editing
  → Validated AI plan
  → Final render from original source ranges
~~~

Track proxy-to-original time mapping and verify duration/frame-rate assumptions before final render. Do not send an original or proxy to a provider unless an explicitly supported feature requires it and the user has been informed.

## 28. Privacy

Default policy:

- Raw video and audio stay local.
- Transcripts, silence data, extracted frames, proxies, and caches stay local.
- A provider receives only the compact context required for the current task.
- Omit absolute local paths, unrelated segments, and unnecessary filenames.
- Send images only to providers that explicitly support them and only when useful.
- Never upload original media by default.

Document exactly what leaves the machine for each provider mode. Give the user a way to disable provider requests while retaining local analysis and manual editing.

## 29. ChatGPT and AI Provider Authentication

Define a provider interface for connection status, model discovery, plan/API capability, and structured completion. Keep provider-specific authentication outside the timeline, renderer, and Editor Brain.

Support:

- Sign in with ChatGPT only through an officially supported SDK or connector.
- API providers through explicit user configuration.
- Additional providers through the same abstraction in the future.

Authentication requirements:

- Never scrape browser cookies or reuse an unrelated browser session.
- Never store passwords or hard-code access tokens.
- Store credentials only through OS secure storage or the provider SDK's secure mechanism.
- On startup, check bridge/provider status and restore an existing valid credential before offering a new sign-in.
- If already connected, do not reopen browser sign-in; refresh status and model discovery.
- Use explicit states such as DISCONNECTED, CONNECTING, CONNECTED, REFRESHING, and ERROR.
- Ensure every async path has a timeout or terminal result.
- Ignore stale login results if a valid session has already been restored.
- Report plan authorization separately from basic sign-in.

ChatGPT plan access is available only where the supported connector and account authorization allow it. Do not imply that a subscription grants an undocumented local API, and do not bypass provider terms or entitlement checks.

## 30. Paid API Safety

- Never fall back to a paid API automatically.
- Require the user to explicitly select and configure a paid API provider.
- Show the provider, billing mode, and relevant cost/usage warning before a paid request.
- Do not store an API key in project files, source control, logs, or plaintext settings.
- If the selected provider is unavailable or out of quota, stop or offer a non-paid/manual option.

## 31. Advanced Mode

Advanced / Development Mode MAY show:

- Structured plans and patches.
- Analysis and context packs.
- Source IDs and source-relative timing.
- Prompts and provider responses.
- Validation diagnostics.
- Debug logs and renderer details.

Simple Mode MUST remain a plain-language Director interface. Advanced diagnostics MUST still redact credentials, tokens, and private paths where practical.

## 32. Multi-Clip Editing

Support importing several sources into one project. The AI may identify hooks, choose shots or takes, order story sections, remove clear repetition, preserve demonstrations/B-roll, and propose a conclusion.

The timeline MUST support source reuse and ordering across multiple inputs. The context pack MUST label sources and transcript segments consistently. The validator MUST ensure every clip range belongs to the stated source, regardless of output order.

## 33. End-to-End Workflow

### User actions

~~~text
Import clips
  → Select style and duration (optional)
  → Type request
  → AI Edit
  → Watch local preview
  → Mark a range (optional)
  → Type correction
  → Apply Fix
  → Teach Brain (optional)
  → Export
~~~

### Application actions

~~~text
Local analysis
  → Compact context pack
  → Provider/model routing
  → Structured plan
  → Brain and narrative validation
  → Atomic timeline update
  → Local preview render
~~~

Every model-generated change must pass validation before it changes the timeline. Every preview/export must render from a known timeline snapshot.

## 34. Error Handling

Handle at least:

| Condition | Expected behavior |
|---|---|
| FFmpeg/ffprobe missing | Explain the missing tool and keep the application open |
| ASR unavailable | Continue with available analysis and mark transcript unavailable |
| Provider disconnected | Preserve the command and project; offer reconnect or manual mode |
| Credential refresh fails | Move to ERROR or DISCONNECTED; never loop forever |
| Invalid AI plan | Reject it; keep the current timeline unchanged |
| Stale revision patch | Reject it and request a fresh revision |
| Render failure | Keep source media and prior valid preview/project state |
| Quota exhausted | Explain the provider limit; never silently switch to paid usage |
| Disk space low | Stop before writing a partial final export when feasible |
| User cancels | Stop work safely and leave source files unchanged |

User-facing errors SHOULD be short and actionable. Detailed diagnostics belong in logs or Advanced Mode, with secrets and sensitive paths redacted.

## 35. Testing

### Unit tests

- Brain configuration and schema validation.
- AI plan and timeline patch validation.
- Narrative warnings and compression thresholds.
- Transcript-boundary snapping.
- ProjectTimeline order, repeated sources, atomic apply, and undo.
- Multi-source context mapping and stable segment references.
- Context Pack size, selection, and omission of local paths.
- Apply Fix changes only the current timeline.
- Teach Brain writes only after explicit action and includes required context.
- Mark In/Out validation and preview-to-source mapping.
- Playback controls, seek, duration formatting, and cut navigation.
- Duration presets and style context.
- Provider routing, model discovery, connection state, and timeout behavior.
- No automatic paid-provider fallback.
- Source file hash remains unchanged across edit operations.

### Integration tests

Use a mock model provider for deterministic tests:

~~~text
Import
  → Analyze
  → Mock AI plan
  → Validate and apply
  → Preview
  → Mock revision
  → Apply Fix
  → Export
~~~

Real FFmpeg render tests SHOULD use small synthetic fixtures, be opt-in when environment-dependent, and verify source hashes before and after. Authentication tests MUST use mocks or dedicated test credentials; never require a developer's personal account.

## 36. Fast Build Order for AI Coding Agents

Build phase by phase. Do not begin a later phase until its acceptance criteria pass. Keep providers, renderer, and timeline independently testable.

| Phase | Deliverable | Acceptance criteria |
|---|---|---|
| 1. Desktop shell + playback | PySide6 window, import/probe one video, preview playback | App opens; user can load, play, pause, stop, seek, and read millisecond time; window resizes |
| 2. FFmpeg + ProjectTimeline | Non-destructive timeline model and renderer boundary | Unit tests cover range validation, ordering, repeated sources, atomic apply, undo; source hash unchanged |
| 3. Local transcription + silence + scenes | Local analyzer and cache | Transcript, silence, scene, and speech results are local; failures are reported per source; silence alone does not auto-cut |
| 4. Analysis Pack + Context Pack | Versioned compact evidence builder | Stable segment references map to local timestamps; current/range context is focused; paths and full transcript are omitted unless needed |
| 5. Editor Brain + AI Edit Plan | Config loader, plan contract, validator, timeline adapter | Synthetic valid plan adapts to timeline; malformed IDs/times/actions are rejected without mutation |
| 6. Narrative Validator | Continuity rules, compression checks, transcript snapping | Warning severities and thresholds are configurable; boundary tests and compression tests pass |
| 7. AI Provider + Model Router | Provider interface, model discovery, task routing, secure credential boundary | Mock providers pass; connection states terminate; no cookie scraping, secrets in logs, or paid fallback |
| 8. Simple Director UI | Split preview/conversation layout and one-command flow | User can import, request AI Edit, view progress/history, and reach all controls at minimum supported window size |
| 9. Apply Fix | Natural-language revision context and validated timeline patch | Current time/range/style/timeline context is attached; stale/invalid patches do not mutate; successful fix renders a new preview |
| 10. Teach Brain | Explicit, contextual learning correction | Apply Fix does not save learning; Teach Brain does; clip-specific lessons require confirmation; entries can be inspected/removed |
| 11. Subtitle Correction | Local ASR review, glossary, uncertainty state, subtitle export/render | Original and corrected text stay separate; uncertain names are flagged; subtitle timing follows timeline mapping |
| 12. Text / Graphics | Allowlisted graphics presets and renderer support | Unknown presets/parameters are rejected; visual bounds/timing are validated; presets render in a fixture test |
| 13. Effects | Allowlisted effects such as zoom or freeze | No arbitrary model-generated filter code; unsupported actions fail safely; effect output has a render test |
| 14. Multi-clip production testing | Representative multi-source projects and regression suite | Repeated sources, mixed resolutions, audio, revisions, and source immutability pass end-to-end tests |
| 15. Packaging | Installer or portable distribution, setup checks, documentation | Clean machine setup is documented; required tools/models/licenses are disclosed; app starts without developer-only paths |

## 37. Acceptance Criteria

The MVP is accepted when all applicable items pass:

- [ ] Import multiple video clips.
- [ ] Run local analysis and cache results safely.
- [ ] Build compact model context.
- [ ] Obtain a structured AI plan through a provider abstraction or a deterministic mock.
- [ ] Validate plans before application.
- [ ] Create and render a multi-source ProjectTimeline.
- [ ] Review a local preview.
- [ ] Request a natural-language revision and apply it safely.
- [ ] Mark In/Out and map preview ranges back to sources.
- [ ] Keep Apply Fix separate from Teach Brain.
- [ ] Export an MP4.
- [ ] Verify imported source files remain unchanged.
- [ ] Keep paid-provider use opt-in and explicit.
- [ ] Keep editing usable in manual mode when AI is disconnected.
- [ ] Keep raw plans and diagnostics out of Simple Mode.

## 38. Security Checklist

- [ ] No plaintext passwords, tokens, or API keys.
- [ ] No browser-cookie scraping or unrelated session reuse.
- [ ] No hard-coded secrets in source, configuration, tests, or examples.
- [ ] No automatic paid-provider fallback.
- [ ] Source files are treated as immutable inputs.
- [ ] Logs redact credentials and minimize sensitive paths.
- [ ] Model output is treated as untrusted and validated.
- [ ] File paths are constrained to intended local inputs/outputs.
- [ ] Timestamps, durations, IDs, and preset parameters are validated.
- [ ] Temporary artifacts have bounded storage and cleanup behavior.
- [ ] Project exports do not accidentally include credentials or private cache data.
- [ ] Tests use synthetic media and mock credentials by default.

## 39. Licensing

Before commercial distribution, review the license and redistribution requirements for every dependency and SDK, especially authentication SDKs, media libraries, fonts, ASR/model packages, model licenses, and any bundled binary. Record notices and attribution requirements in the distributed application.

## 40. What This Specification Intentionally Does Not Include

This document does not include:

- Source code from any existing application.
- Credentials, passwords, API keys, or tokens.
- User data or private transcripts.
- Private datasets or model weights.
- Proprietary or personal Editor Brain rules.
- Personal learning corrections.
- Local machine paths.
- Private deployment or account configuration.

All identifiers and media values shown here are synthetic examples.

## 41. Build Spec Usage

Give this file to a coding agent and use an instruction such as:

> Build the application described in this specification. Work phase by phase. Do not skip acceptance tests. Do not invent undocumented behavior. Keep all video processing local unless the specification explicitly allows otherwise.

The agent SHOULD first inspect its target workspace, preserve existing user changes, identify available dependencies and test constraints, then implement only the current phase. It MUST report assumptions, tests, and environmental blockers.

## 42. Output

The deliverable is a Markdown document named JAME_LOCAL_CLIP_EDITOR_BUILD_SPEC.md. Keep it readable in a text editor and portable across operating systems. Do not add local machine paths, credentials, private source data, or copied application code when adapting or redistributing this specification.
