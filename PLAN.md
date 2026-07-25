# VideoAnalyser — Implementation Plan

**Status:** final, post-grill. Ready to execute.
**Companion documents:** `docs/REQUIREMENTS.md` (the spec, with R-numbers referenced
throughout this plan) and `docs/GRILL_LOG.md` (the adversarial review that produced
this version).

---

## 0. How to execute this plan

You are building a local command-line tool. Read this section first.

1. This plan is **self-contained**. Everything you need is here or in
   `docs/REQUIREMENTS.md`. Do not assume knowledge of any prior conversation.
2. Every requirement is tagged `R<n>`. `docs/REQUIREMENTS.md` defines them.
   §17 is a traceability matrix — when you finish, every row must be satisfiable.
3. Build in the milestone order given in §16. Each milestone ends in something
   runnable. Do not build the persona system before the timeline is proven correct —
   §5 is the foundation everything else stands on, and it is the requirement the
   user cared about most (R2.4, "perfect synchronization, without losing anything").
4. **Where a value is given as a number in this plan, use that number.** They are
   defaults chosen deliberately and belong in the config file (§14.3), not inlined.
5. Where this plan says MUST, a deviation is a bug. Where it says SHOULD, deviating
   is allowed if you record why in a code comment.

### 0.1 The one-paragraph summary of what you are building

A Python CLI that takes any video file, decodes it against a single authoritative
timeline, transcribes the audio with word-level timestamps, samples the visual track
adaptively at points where the picture actually changes, binds every spoken phrase to
the exact visual state on screen when it was said, runs a roster of specialised
analysis personas over that aligned record, and emits one Markdown file plus a folder
of named screenshots — detailed enough that a reader, or another AI, can reconstruct
what happened in the video without watching it.

---

## 1. Scope and deliverable

### 1.1 Inputs

Any video file, any container, any codec ffmpeg can decode, any resolution, any
orientation, any frame rate (R1.1, R1.2). The five source archetypes that must all
work (R1.3):

| Archetype | Distinguishing traits | What matters most |
|---|---|---|
| Social / Instagram | Vertical 9:16, short, music-led, heavy grade | Mood, style, filters, music (R4.11) |
| Video call with screen share | 16:9, long, talking + shared screen, 2+ speakers | Speaker attribution, screen↔speech binding (R1.3.b) |
| Screen recording (macOS/iOS) | **Variable frame rate**, high resolution, UI-dense | Step reconstruction, OCR, exact buttons (R4.9) |
| AI-generated inspiration video | Stylised, no speech, music-led | Look, grade, style vocabulary, music (R4.11) |
| Workshop / woodworking | Handheld, tool noise, narrated or silent | Tools, materials, technique, real-world durations (R4.10, R5) |

### 1.2 Outputs

Per processed video, three artifacts at minimum (delivery modes in §10.8 may add
derived files, and a failed QA run quarantines instead — §12.3):

1. **One Markdown file** in `outputs/` (R6.2) containing everything in R4.
2. **One screenshot folder** in `screenshots/` (R6.3), named per R7.
3. **One JSON sidecar** alongside the Markdown — the machine-readable form of the
   same content, so downstream tooling never has to parse prose.

### 1.3 Explicit non-goals

Stated so the implementer does not gold-plate:

- No GUI, no web service, no daemon. A terminal command (R0.1).
- No video editing, re-encoding, or export of derived video.
- No cloud storage, no upload of the source video anywhere. Only sampled frames and
  transcript text leave the machine, and only to the Anthropic API.
- No real-time or streaming analysis. This is a batch tool over files at rest.

---

## 2. Target environment and dependencies

### 2.1 The machine

Apple MacBook, **M1 (base)**, **16 GB unified memory**, macOS (R0.3). This is a
binding constraint, not a hint:

- The 16 GB is **unified** — shared between CPU, GPU, and the OS. With a browser and
  an editor open, budget **7–8 GB** for this process, not 16.
- The M1 base chip has 8 GPU cores and ~68 GB/s memory bandwidth. Model inference is
  viable but not free.
- Architecture is `arm64`. Every wheel must be native; anything falling back to
  Rosetta or to an x86_64 build is a defect, not a slowdown.
- **macOS 14.0 or later is a hard precondition.** `av` 18.0.0 publishes its Apple
  Silicon wheel as `macosx_14_0_arm64`, and `mlx` likewise. On macOS 11–13 both fall
  back to building from source — PyAV needs a full FFmpeg dev toolchain and `mlx` has
  no usable sdist path — so the first run dies in an opaque compiler error. An M1
  MacBook shipped with macOS 11, so this is a real upgrade requirement, not a formality.
  `bootstrap.sh` MUST check it (§2.6).

Everything in §13 (memory and cost control) exists because of this section.

### 2.2 Python

Target **Python 3.12**. Accept **3.11–3.12 only**.

The ceiling is set by the tightest constraint actually in the tree, not by the loosest:
`whisperx` declares `>=3.10,<3.14`, but `rapidocr-onnxruntime` declares `<3.13`. On
3.13 the interpreter check would pass and `pip install` would then hard-fail — exactly
the failure mode §2.4 exists to prevent. If the OCR fallback is dropped, the ceiling
may rise to 3.13.

**Known resolver conflict:** `whisperx` 3.8.6 pulls `numpy>=2.1`. Do not pin a
`numpy<2` floor elsewhere; resolve the environment with `numpy>=2.1` and verify
`librosa` and `opencv`-adjacent wheels agree at lock time.

### 2.3 System dependencies (Homebrew)

| Package | Why |
|---|---|
| `ffmpeg` (≥7.0) | Decoding, frame extraction, audio extraction. Ships `ffprobe`, which §5 depends on for stream metadata. |
| `python@3.12` | Interpreter, if not already present. |

Install: `brew install ffmpeg python@3.12`

`ffmpeg` MUST resolve to `/opt/homebrew/bin/ffmpeg` (Apple Silicon Homebrew prefix).
If it resolves under `/usr/local`, the user has an Intel/Rosetta Homebrew and the
bootstrap MUST fail loudly with that diagnosis rather than proceeding.

### 2.4 Python dependencies

Every version below was resolved against the live PyPI index, not recalled. Pin
these as lower bounds in `requirements.txt`.

**Media and timing**

| Package | Min version | Why it is here |
|---|---|---|
| `av` | 18.0.0 | PyAV. Reads real per-frame presentation timestamps from the container. **This is the package that makes R2.4 achievable** — see §5.2. |
| `ffmpeg-python` | 0.2.0 | Ergonomic wrapper for ffmpeg/ffprobe invocations. |
| `scenedetect` | 0.7.1 | Shot/scene boundary detection (`ContentDetector`, `AdaptiveDetector`). Handles fades and dissolves that a naive histogram diff misreads. |
| `ImageHash` | 4.3.2 | Perceptual hashing for near-duplicate frame elimination (§5.5). |
| `Pillow` | 10.4.0 | Image loading, cropping, downscaling, annotation, PNG/WebP encoding. |
| `numpy` | 2.1.0 | Array backbone. Floor set by `whisperx`, not chosen independently. |
| `pyarrow` | 17.0.0 | Parquet I/O for `frames.parquet` (§4.2). Without it that artifact cannot be written. |

**Audio and speech**

| Package | Min version | Why it is here |
|---|---|---|
| `mlx-whisper` | 0.4.3 | **Default transcriber.** Apple's MLX runtime — runs Whisper on the M1 GPU. The fastest option on this specific hardware. |
| `faster-whisper` | 1.2.1 | Fallback transcriber (CTranslate2, int8 CPU). Note: CTranslate2 has **no Metal backend** — it is CPU-only on Apple Silicon. Chosen for robustness, not speed. |
| `whisperx` | 3.8.6 | Forced alignment to true word-level timestamps, and the diarization bridge. Whisper's native segment timestamps are **not** accurate enough for R2.3 — see §6.4. |
| `pyannote.audio` | 4.0.7 | Speaker diarization ("who spoke when") — required by R1.3.b and R4.6. |
| `librosa` | 0.11.0 | Tempo/BPM, key, spectral features, energy envelope — the music analysis behind R4.11. |
| `soundfile` | 0.12.1 | WAV I/O. |
| `silero-vad` | 5.1 | Independent VAD for the hallucination guard (§6.4). Must be a *different* VAD from any used inside the ASR, or the check is circular. |

**Text on screen**

| Package | Min version | Why it is here |
|---|---|---|
| `ocrmac` | 1.0.1 | **Default OCR.** Wraps Apple's native Vision framework: fast, accurate, no model download, no network, excellent on UI text. It is macOS-only, which is fine — R0.3 fixes the platform. |
| `rapidocr-onnxruntime` | 1.4.4 | Fallback OCR for non-macOS or if Vision is unavailable. |

> Do **not** use `imagetext-recognition`. It does not exist on PyPI. It appeared in an
> earlier draft of this plan and would have broken `pip install` on the first run.
> (`docs/GRILL_LOG.md`, finding 0.1.)

**LLM analysis**

| Package | Min version | Why it is here |
|---|---|---|
| `anthropic` | 0.120.0 | Official Anthropic SDK. All vision and synthesis calls. See §9 and §13.4. |

**Application plumbing**

| Package | Min version | Why it is here |
|---|---|---|
| `pydantic` | 2.9.0 | Every inter-stage artifact is a Pydantic model. This is what makes §4.3's contracts enforceable rather than aspirational. |
| `PyYAML` | 6.0.2 | Config file parsing. |
| `python-dotenv` | 1.0.1 | Loads `ANTHROPIC_API_KEY` from `.env`. |
| `rich` | 13.9.0 | Progress display, tables, and readable error output. |
| `tenacity` | 9.0.0 | Retry with exponential backoff around API and model calls. |

Not dependencies — `hashlib`, `json`, `pathlib`, `dataclasses`, `subprocess` are
standard library. They MUST NOT appear in `requirements.txt`.

### 2.5 One-time model and credential setup

The bootstrap MUST check all of these and report every failure at once, rather than
failing on the first and making the user re-run repeatedly:

1. `ANTHROPIC_API_KEY` present in `.env` or the environment.
2. Whisper weights present (downloaded on first use; **~3.1 GB** for `large-v3` — it is 1,550 M parameters at fp16, not 1.5 GB; §13.3 uses the same figure). The
   bootstrap SHOULD pre-fetch so the first real run is not a surprise download.
3. **`pyannote.audio` diarization models are gated.** With the pinned 4.0.7 the
   default pipeline is **`pyannote/speaker-diarization-community-1`** — 3.1 is the
   legacy pipeline, and community-1 is specifically better at speaker counting and
   assignment, which is the failure mode §6.5 guards against. The user must have a
   Hugging Face account, accept that one model's conditions, and provide `HF_TOKEN`.
   (The 4.x pin is forced: `whisperx` 3.8.6 requires `pyannote-audio>=4.0.0`.) This is a
   manual, interactive, browser step that cannot be automated. The bootstrap MUST
   detect a missing/unaccepted token and print the exact URLs to visit. If the user
   declines, diarization degrades to a single unnamed speaker (§6.5) — the run
   still completes.

### 2.6 Bootstrap script

`bootstrap.sh` at the repo root MUST, in order: verify `arm64`; verify the Homebrew
prefix; verify Python in range; create `.venv/`; install `requirements.txt`; verify
`ffmpeg`/`ffprobe` are callable; create the workspace of §3; write `.env.example`;
then print a checklist of anything still outstanding (§2.5). It MUST be idempotent —
running it twice changes nothing and reports the same state.

---

## 3. Workspace layout and naming — **HARD SPEC**

Everything in this section is a literal requirement (R6, R7). Do not improve on it,
reorganise it, or add folders that were not asked for.

### 3.1 The workspace

```
~/Desktop/Video Analysis/          <- root folder, named exactly "Video Analysis" (R6)
├── raw/                           <- user drops input videos here (R6.1)
├── outputs/                       <- all resulting .md and .json land here (R6.2)
└── screenshots/                   <- one subfolder per processed video (R6.3)
```

- The root lives on the **Desktop** by default, because that is where the user said
  they would use it. The path MUST be overridable via `workspace_root` in the config
  file and `--workspace` on the CLI.
- **The workspace is not the code checkout.** The Python package lives wherever the
  user cloned it; the workspace is a separate data directory. An earlier draft
  conflated them (`GRILL_LOG.md`, finding 0.10).
- On first run, all four directories are created if absent. Creation is silent if
  they already exist.
- Exactly three subfolders. No `logs/`, no `cache/`, no `tmp/` inside the workspace.
  Intermediate artifacts live outside it — see §3.4.

### 3.2 Screenshot subfolder naming — the literal format

One subfolder per processed video (R7), named:

```
<title> | <DD.MM.YYYY> | original: <source>
```

The user's own example, which is the acceptance test for this rule:

```
Claude skills and best practices | 28.07.2026 | original: instagram recording
```

| Part | Content | Derived from |
|---|---|---|
| 1 — title | What the video is about | The categoriser's title (§9), falling back to the input filename stem |
| 2 — date | Date the video was **processed**, `DD.MM.YYYY` | System clock at run start (R7.2) |
| 3 — origin | `original: <source>` | The provenance analyst (§9), e.g. `instagram recording`, `youtube video`, `iphone screen recording`, `zoom call recording`, `unknown source` |

Separator is exactly `space pipe space` (` | `).

### 3.3 Filename sanitisation — and the colon problem

The requested format contains a colon, and that has a real consequence the
implementer must handle deliberately rather than discover:

- **`/` (0x2F)** is the POSIX path separator and cannot appear in a filename. If a
  title contains `/`, replace with `-`.
- **`:` (0x3A)** is legal in an APFS filename at the POSIX layer, but the macOS
  Finder and Carbon-era APIs render it as `/`. A folder literally named
  `... | original: instagram recording` therefore **displays in Finder as**
  `... | original/ instagram recording`.
  **Resolution:** keep the colon. The user specified this format explicitly and will
  most often see these paths as Markdown links and in the terminal, where the colon
  renders correctly. The tool MUST print a one-time note on first run explaining the
  Finder rendering, and the config MUST expose `sanitize_colon: false|true` so the
  user can switch to ` - ` if the Finder display bothers them. Do not silently
  substitute — that would violate the literal spec of R7.
- **Control characters and newlines** in a derived title: strip.
- **Leading/trailing whitespace and dots**: strip (a leading dot hides the folder).
- **Length:** the practical macOS limit is 255 **UTF-8 characters** per path component
  (the on-disk field is larger). The date and origin parts are preserved intact; the
  **title** is truncated on a grapheme boundary until the whole component fits.
  Truncating at 255 *bytes* would also be safe but cuts non-ASCII titles up to four
  times earlier than necessary, which defeats the point of preserving the title.
- **Unicode normalisation:** normalise to NFC before writing, and on both sides of any
  path comparison. Note *why*: this is hygiene for network shares and external
  (non-APFS) volumes, not for APFS itself. APFS **preserves** whatever normalisation you
  write and is normalisation-*insensitive* — an NFC and an NFD spelling of the same
  accented title collide as the same directory, so the duplicate-folder failure cannot
  occur locally. It was HFS+ that forced NFD.
- **Collisions:** each screenshot folder contains a dotfile `.videoanalyser-source`
  holding the content hash of the video it came from — that is the ownership record.
  Dotfiles are exempt from R6's "exactly three folders" (they are files, not folders),
  and this avoids needing a registry the workspace is not allowed to hold. If the target
  folder exists and its recorded hash differs, append ` (2)`, ` (3)` … to the end of the
  whole name. Gate `G12`'s R7 regex therefore admits an optional numeric suffix:

  ```
  ^(?P<title>.+) \| (?P<date>\d{2}\.\d{2}\.\d{4}) \| original: (?P<source>.+?)( \((?P<n>\d+)\))?$
  ```

  With `sanitize_colon: true` the separator becomes ` - `, so `G12` must select its
  pattern from the active setting — `original[:] ` when false, `original - ` when true.
  A gate that hard-codes the colon would quarantine every run of a documented,
  supported configuration.

### 3.3a Who computes the folder name, and from what

The naming rule above is useless without a named owner. Stage `S9b_NAMING` (§4.2)
constructs the folder name after provenance and categorisation are known and before
any screenshot is written. Nothing else may construct it.

**Part 1 — title**, in strict precedence order:

1. The container `title` metadata tag, if present and not obviously junk
   (not empty, not the filename, not a camera default like `IMG_1234`).
2. The categoriser's inferred title (§9, persona `A2`).
3. The input filename stem, with underscores/hyphens → spaces and edge noise
   (`final`, `v2`, `copy`, trailing dates) stripped.

**Part 2 — date:** the run's start date, `strftime("%d.%m.%Y")`. Not the video's
creation date — R7.2 says "when the video was processed".

**Part 3 — origin:** the literal string `original: ` followed by a lowercase phrase.
The provenance analyst emits a machine enum; this table is the **only** sanctioned
mapping to the human phrase, and it MUST be reproduced verbatim in code:

| Provenance enum (`A1.source_category`) | Part 3 rendered as |
|---|---|
| `instagram_recording` | `original: instagram recording` |
| `tiktok_recording` | `original: tiktok recording` |
| `youtube_video` | `original: youtube video` |
| `iphone_screen_recording` | `original: iphone screen recording` |
| `macos_screen_recording` | `original: macos screen recording` |
| `zoom_call_recording` | `original: zoom call recording` |
| `google_meet_recording` | `original: google meet recording` |
| `teams_call_recording` | `original: teams call recording` |
| `camera_footage` | `original: camera footage` |
| `ai_generated_video` | `original: ai generated video` |
| `web_source_other` | `original: web source` |
| `unknown` | `original: unknown source` |

`--source "<phrase>"` on the CLI and `default_source` in config override the enum,
and the override is used verbatim after the `original: ` prefix.

**Sanitisation applies to Part 1 only.** The ` | ` separators and the literal
`original: ` prefix are structural and are never sanitised by the Part-1 rules —
sanitising them is what would destroy the format R7 requires. Sanitise the free-text
tail of Part 3 for path separators only.

**One exemption:** the `original:` colon is rewritten to ` - ` if and only if
`sanitize_colon` is set (§3.3). That flag is the single switch; the namer and gate
`G12` MUST read the same config key, or a supported configuration writes one form while
the gate demands the other and every run quarantines.

**Re-running on the same video on the same day** resolves to the same folder name.
Policy: with `--reprocess`, the folder is emptied before writing, so no screenshot
from a previous run survives as an orphan (R8.4). Without `--reprocess`, the run
stops and tells the user the output already exists.

### 3.3b Referencing these folders from Markdown — **the link-encoding rule**

This is the single highest-risk detail in the entire output path. The folder name
contains **spaces**, **pipe characters**, and (per R7) a **colon**. A naive
Markdown image reference to it is broken in every renderer:

```markdown
<!-- BROKEN — do not emit this. Space ends the target; pipes break enclosing -->
<!-- tables; a bare token followed by ':' can parse as a URI scheme.          -->
![Step 3](../screenshots/My Video | 28.07.2026 | original: instagram recording/03_click-upload.png)
```

Every image reference MUST use an **angle-bracket destination** with a
**percent-encoded path**:

```markdown
<!-- CORRECT -->
![Step 3 — the Upload button in the top right of the dashboard](<../screenshots/My%20Video%20%7C%2028.07.2026%20%7C%20original%3A%20instagram%20recording/03_click-upload.png>)
```

Rules, non-negotiable:

- Build the relative path with `pathlib`, then encode each **path segment** with
  `urllib.parse.quote(segment, safe="")` and rejoin with `/`. Encoding the whole
  path in one call would destroy the separators.
- Wrap the result in `<` `>`.
- Never emit an image reference inside a Markdown table cell — an unescaped `|`
  there breaks the table even when encoded. Screenshots go in their own block.
- §12 gate `G2` re-resolves every emitted link against the filesystem before the run
  is allowed to succeed. A link that does not resolve is a failed run, not a warning.

### 3.3c Screenshot file naming

One scheme, used everywhere (R7.4):

```
NN_lowercase-hyphen-description.png
```

`NN` is a zero-padded ordinal in timeline order, at a **fixed width chosen from the
final count** for the whole folder (two digits up to 99 screenshots, three from 100).
Mixing widths within a folder breaks lexicographic sort — `ls` and Finder would order
`09, 10, 100, 101, 11, …`, and a long tutorial exceeds 99 routinely under §10.6's
one-screenshot-per-step floor. The description is derived from what the screenshot shows, lowercased,
non-alphanumerics collapsed to single hyphens, truncated to 60 characters.
**No colons in filenames** — the colon is permitted only in the folder name, where
R7's format demands it. Example: `03_click-upload-button.png`.

### 3.4 Intermediate artifacts

Frames, extracted audio, model outputs, and per-stage caches are **not** part of the
deliverable and MUST NOT pollute the workspace (R6 permits exactly three folders).
They live in:

```
~/Library/Caches/VideoAnalyser/<video_content_hash>/
```

keyed by content hash so re-running on the same file resumes rather than redoes
(§4.4). A `--clean` flag deletes the cache for one video; `--clean-all` empties it.
§13.6 governs how large this is allowed to get.

`~/Library/Caches` is chosen deliberately: macOS offers an opt-in "Desktop & Documents
Folders" iCloud Drive feature, prominently promoted during setup, and **if the user has
it enabled** every intermediate frame written under the Desktop is queued for upload.
It is opt-in rather than a default, but it is common enough that the cache must not
depend on it being off. `~/Library` is outside that sync scope regardless. The
workspace itself (Markdown, JSON, a few dozen screenshots) is small and lives on the
Desktop as the user asked; the bulk cache never does.

**No component may write to `/tmp`.** Every intermediate path is derived from the
per-video cache directory. `/tmp` collides between concurrent videos and macOS purges
it mid-run.

---

## 4. Pipeline architecture

### 4.1 Principles

1. **One decode pass wherever physically possible.** Decoding is the dominant cost on
   an M1 (§13.2). The draft this plan replaces decoded the file three separate times.
2. **Every artifact is a Pydantic model, serialised to JSON.** Stages communicate only
   through these. No stage reaches into another stage's internals.
3. **Every timestamp entering the system is converted to master time exactly once,
   at the boundary where it is produced** (§5.3). This is the single most important
   architectural rule in the document.
4. **Every stage is resumable at its own granularity** (§4.4).

### 4.2 Stages

| ID | Stage | Consumes | Produces | Parallel with |
|---|---|---|---|---|
| `S1_PROBE` | Container/stream interrogation, **plus 5 evenly-spaced framing frames** (rotation: **none** — the ffmpeg CLI autorotates, §5.2; ≤1024 px, written to `framing_frames/`) | video file | `probe.json`, `framing_frames/*.jpg` | — |
| `S9a_FRAMING` | Personas A1, A2 — provenance and category. **Runs early**: `S6` needs `A2.multi_speaker` to decide whether to diarize (§6.5) | `probe.json`, `framing_frames/*.jpg` | `personas/A*.json` | S2, S3 |
| `S2_AUDIO_EXTRACT` | Extract ASR WAV (16 kHz mono) + analysis WAV (48 kHz stereo) | `probe.json` | `asr.wav`, `analysis.wav` | S3 |
| `S3_VISUAL_SCAN` | **Single dense decode**, segmentation included: per-frame PTS, per-tile features, activity envelope, **and** the state partition | `probe.json` | `frames.parquet`, `activity.npy`, `visual_states.json` | S2 |
| `S5_TRANSCRIBE` | ASR + forced alignment → word-level timestamps | `asr.wav` | `transcript.json` | — |
| `S6_DIARIZE` | Speaker turns (optional, §6.5) | `asr.wav`, `A2.multi_speaker` | `speakers.json` | — |
| `S7_AUDIO_FEATURES` | Silence, music, tempo/key, sound events | `analysis.wav` | `audio_features.json` | — |
| `S8_REPRESENTATIVES` | Choose representative frames; extract JPEGs; OCR them | `visual_states.json` | `representatives.json`, frame JPEGs | — |
| `S9b_NAMING` | Folder name (§3.3a) | `probe.json`, A1, A2 | `naming.json` | — |
| `S10_ALIGN` | Bind words → states; clauses → states; steps → screenshots | S3, S5, S6, **S7**, S8 | `alignment.json` | — |
| `S11_VERIFY_SYNC` | The sync proofs of §5.6 — **gate** | S10, **S7**, **S3**'s `activity.npy` | `sync_report.json` | — |
| `S12_PERSONAS` | Passes B, C, D of the roster (§9.2) | everything above | `personas/*.json` | internally parallel |
| `S13_ASSEMBLE` | Render Markdown + JSON sidecar + export screenshots | all | `.md`, `.json`, `screenshots/` | — |
| `S13b_VERIFY_OUT` | Persona `D5` — verifies exported files (§9.4) | `S13` output | `personas/D5.json` | — |
| `S14_QA` | Output gates of §12 — **gate** | S13b | `qa_report.json` | — |

The framing personas are split out as `S9a` deliberately: `S9b_NAMING` needs `A1`/`A2`,
and the rest of the roster needs the naming. Keeping all personas in one stage would
make `S9 → S12 → S9` a cycle. Likewise `D5` runs as `S13b`, after export, because its
job is to verify files that do not exist until then.

`S11` and `S14` are **gates**: on failure the run does not silently continue (§12.3).

**Why `S7` is an input to both `S10` and `S11`, not just a sibling:** `S10`'s clause
boundaries use silences ≥ 400 ms (§8.3) and `SC2` unions silence, music and non-speech
intervals (§5.6) — all of which live in `audio_features.json`. Omitting that edge is not
merely untidy: §4.4 skips a stage when its recorded `input_hash` still matches, so
changing `audio.min_silence_seconds` would re-run `S7` and leave `S10` cached, shipping
clause and step boundaries computed under the old threshold with no diagnostic.

**Why `S3` and segmentation are one stage.** Segmentation needs the full per-frame
descriptor series, but keeping 4 KB per frame for a 90-minute video is ~650 MB, which
this plan declines to store (§5.4). Splitting them would force either that 650 MB onto
disk or a second decode. Fused, the descriptors stay in a rolling buffer and only two
things are persisted:

- `frames.parquet` — one compact row per frame: PTS, 16 per-tile MADs, 16 per-tile mean
  luminances (~140 bytes/frame; ~23 MB for a 90-minute video). Enough to re-derive
  boundaries, fades and dissolves on resume without decoding again.
- `visual_states.json` — the partition, each state carrying the **full 4 KB descriptor
  of its representative frame** (§4.3). Only states keep descriptors — a few hundred per
  video, ~2 MB — and that is the field `S8`'s merge keys on (§13.1).

So `VisualState.descriptor` has exactly one writer, `S3`, and the full per-frame series
is genuinely never stored.

**Dependency note:** `S3` must not depend on `S5`. An earlier draft made frame
extraction depend on the transcript while also declaring the two parallel — an
unresolvable cycle. Resolved by splitting visual work in two: `S3` is a dense,
transcript-independent scan; `S8` does targeted extraction *after* the transcript
exists and may use it to request extra frames.

### 4.3 Artifact contracts

Define these as Pydantic models in `videoanalyser/models.py`. Selected critical fields:

```python
class TimeSpan(BaseModel):          # half-open [start, end)
    start: float                    # seconds on the master timeline
    end: float

class Word(BaseModel):
    text: str                       # VERBATIM, exactly as ASR produced it
    norm: str                       # lowercased, punctuation-stripped — for matching only
    start: float                    # master time
    end: float
    probability: float              # note: faster-whisper's field is `probability`
    speaker: str | None             # "SPEAKER_00" | "UNKNOWN" | None
    overlapped: bool = False
    t_asr: float                    # pre-alignment timestamp, kept for the residual check
    t_forced: float | None          # post-forced-alignment timestamp

class VisualState(BaseModel):
    index: int
    span: TimeSpan                  # states PARTITION the timeline — no gaps, no overlaps
    representative_pts: float       # ACTUAL decoded PTS, never a requested time
    descriptor: bytes               # the 64x64 gray descriptor, 4096 bytes. Persisted
                                    # because S8's merge (§13.1) needs it and S8 is
                                    # independently resumable — it may start in a fresh
                                    # process with nothing in memory. ~2 MB for a
                                    # 90-minute file; cheap insurance against two
                                    # engineers inventing two different merge keys.
    frame_path: Path | None
    ocr_text: str | None
    ocr_boxes: list[OCRBox] = []

# Why `visual_backup` is not a persona output: personas run in S12, but screenshots are
# selected, named and exported in S13, and §3.3c fixes the NN filename width from the
# final count — which itself depends on which claims are needs_visual. The id therefore
# does not merely happen to be unknown at emission time, it is not yet determined.
# Requiring it on the emission-time model would fail §9.6 validation, exhaust the two
# retries, and DROP every style claim on a talking-head or music-inspiration video —
# silently deleting R4.7's only mechanism while G14 passed over an empty set.
# S13 is the single owner: it freezes the export set, chooses the width, then backfills
# visual_backup. G14 validates after that, never before.

class Step(BaseModel):                # the assembler's contract for §10.4
    step_id: str                      # assigned by S10 only
    span: TimeSpan
    action_verb: str                  # MUST be from the controlled list, Appendix A
    target: str                       # what is acted on
    result: str
    why: str
    watch_out: str | None
    application: str | None           # software route (R4.9)
    url: str | None
    tool: str | None                  # physical route (R4.10)
    consumable: str | None
    parameters: dict[str, str] = {}   # grit, rpm, angle, passes, depth
    backing_states: list[int]         # >= 1 state indices, assigned by S10.
                                      # This is what S10 can actually know.
    screenshot_ids: list[str]         # filled in by S13 after the export set is
                                      # frozen; ALWAYS EMPTY at S10 emission, for
                                      # the same reason visual_backup is — see the
                                      # note above. G8 and D4 validate the >= 1
                                      # requirement (R4.8) after S13's backfill,
                                      # never before.
    evidence: list[str]
    merged_from: list[int] = []       # visual states merged under §13.1

class Claim(BaseModel):
    text: str
    kind: Literal["fact", "inference", "domain_estimate"]   # see §9.5
    timestamp: str                  # "HH:MM:SS.mmm" — see §9.6
    evidence: list[str]             # e.g. ["word:1043-1051", "state:17", "ocr:state17#3"]
    confidence: float
    needs_visual: bool = False      # set by the persona for any claim asserting
                                    # on-screen appearance
    backing_state: int | None       # the visual state the persona points at.
                                    # Personas emit THIS, never a screenshot id.
    visual_backup: str | None       # screenshot id, filled in by S13 after the export
                                    # set is frozen. Always None at persona-emission
                                    # time — see the note below.
```

### 4.4 Resume, caching, and the stage ledger

- The cache key is `sha256` of the **first and last 8 MiB plus the file size** of the
  input — full-file hashing a 4K video is minutes of I/O for no benefit.
- `<cache>/stages.json` is a ledger: `{stage_id: {status, completed_at, input_hash}}`.
  A stage is skipped only if `status == "complete"` **and** its recorded `input_hash`
  matches the hash of its current inputs. Changing the config invalidates downstream
  stages automatically.
- **Long-running stages checkpoint internally, not just at completion.** Per-frame
  vision results append to `vision.jsonl` (one JSON object per line, `fsync` every 10
  records). A crash at record 400 of 600 keeps 400. Writing one aggregate JSON at
  stage end — as the draft did — loses everything.
- Batch API job IDs are persisted the moment they are created (§13.4); results remain
  retrievable server-side for 29 days, so a crash after submission costs nothing.
- **Two distinct flags, not one:** `--resume` resumes an interrupted video from its
  last complete stage. `--skip-existing` skips videos that already have an output
  file. The draft conflated them under one name with two contradictory definitions.

---

## 5. The master timeline and synchronization — **the core of the project**

R2.4 asks for "perfect synchronization, without losing anything." Everything in this
section exists to make that a property of the system rather than a hope.

### 5.1 Definition

The **master timeline** is a single axis in seconds, `t = 0` at the presentation start
of the earliest stream. Every artifact in the system — word, frame, state, screenshot,
scene, claim — carries a master-time value. There is exactly one clock.

### 5.2 Probing (`S1_PROBE`)

Use `ffprobe -v error -print_format json -show_format -show_streams`, plus
`-show_packets -read_intervals "%+#200"` for the leading packets.

**Never** run `ffprobe -show_frames` over a whole file and pipe it to `jq`. That
decodes the entire video and buffers gigabytes of JSON to print a few numbers. The
full per-frame timing table comes from `S3`'s single decode pass instead.

Robustness rules, each of which fixes a crash the draft would have hit:

- **Numeric parsing:** with the JSON writer this section mandates, ffprobe **omits**
  `duration` and `nb_frames` entirely on MKV/WebM rather than emitting `"N/A"` — the
  literal `N/A` appears only in the default flat writer. So the failure to absorb is a
  **`KeyError` on an absent key**, not a `ValueError` from parsing `"N/A"`.
  (`start_time` is present and numeric on both, so it is not one of the affected
  fields.) Route every numeric read through one helper that treats an absent key,
  `None`, `""` and `"N/A"` alike as missing — the last for safety if anyone switches
  writers.
- **Missing audio is normal, not exceptional.** `next(s for s in streams if ...)`
  raises `StopIteration` on a silent screen recording or an AI-generated clip
  (R1.3.d). Use a default and carry an explicit `has_audio: bool`. With no audio,
  `S5`/`S6`/`S7` are skipped and the output states plainly that the video is silent.
- **Video stream selection:** skip streams with `disposition.attached_pic == 1`
  (embedded cover art), then choose by largest `width × height`.
- **Multiple audio tracks are the norm for the user's video-call archetype** (mic +
  system audio, or per-participant). Enumerate them all, record them in the manifest,
  transcribe the selected one, and pass an explicit `-map 0:a:<idx>` so the track
  transcribed is provably the track probed. Bare `-vn` lets ffmpeg pick a different
  "best" stream than the one described.
- **Rotation: never hand-roll it — but the two decoders disagree, and that matters.**
  The ffmpeg **CLI** applies the display matrix automatically (`-autorotate`, on by
  default). **PyAV does not** — it exposes the display matrix as side data and returns
  un-rotated frames. Since `S3` decodes with PyAV (§5.4) and `S8` exports with the
  ffmpeg CLI (§7.1), a rotated iPhone capture would otherwise produce a **descriptor
  series transposed relative to the exported screenshots**, breaking `SC6` on two of
  the five mandatory archetypes with no diagnosis.
  **Rule:** read `side_data_list[].rotation` at probe time into `probe.rotation`. In
  `S3`, which decodes with PyAV, apply that rotation explicitly to each decoded frame
  before computing the descriptor. In `S1` (framing frames) and `S8` (representative
  export), both of which go through the ffmpeg CLI, **apply nothing** — the CLI has
  already done it, and applying it again rotates twice.
  In `S11`, `SC6`'s re-extraction MUST reuse `S8`'s ffmpeg-CLI path and likewise apply
  nothing. That site is easy to miss and expensive to get wrong: the natural
  implementation reuses `S3`'s PyAV descriptor helper, where "apply nothing" is the
  wrong rule — `SC6` would then hard-fail on every rotated capture, and `M4` is a gate
  the build is told not to proceed past.
  `S1`, `S3`, `S8`, `S11`/`SC6` are every decode site in the pipeline; adding a new one
  without adding it here is a defect. The framing frames matter disproportionately because they
  are the *only* frames `A1` and `A2` ever see, and `SC6` samples representatives rather
  than framing frames, so a double rotation there would silently degrade provenance,
  category and the R7 folder name with nothing to catch it. Assert that each framing
  frame's aspect ratio matches the probed display aspect. Assert that the
  exported frame's dimensions match the descriptor's orientation.
  The draft's `-vf "rotate=90"` was wrong three times over regardless: that filter takes
  **radians** (90 rad ≈ 5157°), it does not resize the canvas so content is cropped
  away, and it double-rotates what the CLI already handled.
- **HDR / 10-bit:** if `color_transfer` is `smpte2084` or `arib-std-b67`, insert a
  tonemap filter before RGB conversion, or 4K HDR screen recordings produce washed-out
  screenshots.
- **Content class.** `S1` writes `content_class: screen | camera | mixed` into
  `probe.json`, computed from the five framing frames it already extracts. `S3` reads it
  to choose its descriptor (§5.5), and `S3` runs parallel with `S9a`, so it **cannot**
  use `A2.has_screen_content` — the only other screen-vs-camera signal in the plan.
  Without this field the choice is a judgement call at the one place the plan can least
  afford one.
  Rule, thresholds in §14.3: compute mean edge density (Sobel magnitude over a threshold)
  and palette flatness (fraction of pixels in the top 16 quantised colours) per framing
  frame. `screen` when edge density ≥ `screen_edge_density_floor` **and** flatness ≥
  `screen_flatness_floor`; `camera` when both are below; `mixed` otherwise.
  **`mixed` takes the per-tile-MAD branch.** The asymmetry is deliberate: choosing pHash
  on screen content silently deletes UI changes and defeats R4.9 unrecoverably, whereas
  choosing tile-MAD on camera footage merely over-segments, which the anchor rule and
  `max_state_seconds` already bound. R1.3.b — a video call with camera tiles *and* a
  shared screen — is exactly the `mixed` case.
- **VFR detection:** do not compare `r_frame_rate` to `avg_frame_rate` as strings —
  `"30000/1001"` vs `"29.97"` flags every ordinary 29.97 fps CFR file as variable.
  Parse both as `Fraction` and compare with tolerance; then confirm from `S3`'s real
  PTS series using the coefficient of variation of consecutive deltas.

### 5.3 The offset rule — one conversion, one place

The draft computed the audio/video `start_time` difference, stored it in a manifest,
and then never applied it anywhere — while asserting downstream that ASR timestamps
were already on the master timeline. They were not. Every word, silence, speaker turn
and audio feature in the entire output was uniformly shifted.

**The rule:**

```python
master_epoch = min(video_start_pts, audio_start_pts)   # from leading packets
audio_offset = audio_start_pts - master_epoch
video_offset = video_start_pts - master_epoch

def to_master(t_local: float, stream: Literal["audio", "video"]) -> float: ...
```

- `ffmpeg ... -vn -ar 16000 -ac 1 out.wav` normalises the output start to zero. Every
  timestamp produced from that WAV is therefore in **WAV-local** time and MUST pass
  through `to_master(t, "audio")` at the boundary of the stage that produced it.
- No raw extractor timestamp may reach `S10_ALIGN`. Enforce with an assertion.
- **Do not reason about edit lists or AAC encoder delay analytically.** The draft's
  detection commands (`ffprobe -show_format | grep editlist`) can never match anything,
  and its conclusion ("ffmpeg follows the edit list, do not double-correct")
  contradicted its own correction rule elsewhere with no tiebreak. Measure the offset
  empirically from leading packet PTS through the same decode path extraction uses,
  and apply it in exactly one function.
- **Test:** remux a known file with `-itsoffset 0.4` on the audio and assert the
  pipeline recovers it. **Generate the fixture with PCM/WAV audio**, where the offset is
  sample-exact. With AAC the offset snaps to the 1024-sample packet grid (~23.2 ms) and
  lands at 0.376 s, so an equality assertion against 0.400 fails by 24 ms through no
  fault of the pipeline. If AAC must be used, assert against the container's actual
  first-packet PTS with a tolerance of one audio frame. This test is mandatory (§15).

### 5.4 The single dense visual pass (`S3_VISUAL_SCAN`, part 1: decode)

This replaces three separate decode passes and removes a whole class of timing bugs.

**Decode in-process with PyAV. Do not pipe `rawvideo` from an ffmpeg subprocess.**
`-f rawvideo` emits headerless pixel bytes — no container, no `time_base`, no PTS — so
the only way to time those frames would be to *count* them, which is exactly the
positional join this section exists to eliminate.

```python
container = av.open(path)
stream = container.streams.video[idx]
stream.thread_type = "AUTO"
for frame in container.decode(stream):
    t = float(frame.pts * stream.time_base)          # master time, from the container
    d = frame.reformat(width=64, height=64, format="gray").to_ndarray()
```

If a subprocess is ever used instead (e.g. to force hardware decode), the pipe format
MUST be a timestamp-carrying container — `-f nut -` or `-f matroska -` — never
`rawvideo`, and `-fps_mode passthrough` replaces the deprecated `-vsync 0` on ffmpeg 7.

For every frame record:
PTS (master time), a 64×64 grayscale descriptor, and the mean absolute difference from
the previous frame. 64×64 gray is 4 KB per frame — a 90-minute 30 fps video is ~650 MB streamed through a
rolling buffer and never written. What persists is the compact per-frame feature table
plus one descriptor per *state* (§4.2), not per frame.

Why this matters:

- **Timestamps come from the decoder, keyed to the frame.** The draft ran `ffmpeg` to
  write `frame_%06d.png` and a *separate* `ffprobe` to list I-frame times, then joined
  them **by position**. One dropped or duplicated frame — a decode error, a corrupt
  GOP, an ffmpeg version difference — shifts every subsequent timestamp by a full
  keyframe interval, silently, for the rest of the file. There is no positional join
  here because there is one pass.
- **Use `best_effort_timestamp` / PTS, never DTS.** The draft used `pkt_dts_time`.
  With B-frames, decode order ≠ presentation order, so every frame was biased early by
  one to three frame intervals, and the earliest frames carried negative timestamps.
- `-hwaccel videotoolbox` uses the M1's hardware decoder. Without it, decode alone on a
  2-hour 4K file is hours (§13.2).

### 5.5 Visual states as a partition (`S3_VISUAL_SCAN`, part 2: segmentation)

Visual states are **half-open intervals `[t_i, t_{i+1})` that partition the timeline**.
Not a list of sampled instants. This one modelling choice removes an entire family of
alignment bugs — see §8.2.

Segmentation from the dense descriptor series:

- **Descriptor choice depends on `probe.content_class`** (§5.2), which `S1` computes;
  `mixed` uses the screen branch:
  - *Screen content* — tile the 64×64 descriptor into 16 blocks and declare a boundary
    when any single block's mean-absolute-difference exceeds `block_delta_floor`.
    Perceptual hashing is the **wrong** tool here: pHash keeps low-frequency DCT
    coefficients precisely to be invariant to small high-frequency change, but on a UI
    the small high-frequency changes *are* the content — a field being typed into, a
    dropdown highlighting, a checkbox toggling all yield a pHash distance of 0–1 and
    get deduplicated away. That directly defeats R4.9.
  - *Camera footage* — pHash with a threshold calibrated from the observed distance
    distribution for this file, not a hardcoded constant. Codec noise alone moves 2–4
    bits, so a fixed threshold that works for screen content over-segments here.
- **Compare against the group anchor, not the previous frame.** Chaining
  frame-to-frame lets a slow fade, a scroll, or a progress bar drift arbitrarily far
  from where the state began while every individual step stays under threshold — a
  30-second scroll collapses to one screenshot that misrepresents the whole span.
  Split when the step delta exceeds the floor **or** the distance from the anchor
  exceeds `anchor_delta_floor`.
- **Cap state duration** at `max_state_seconds` (default 10) so the worst case is
  bounded regardless.
- **Fade/dissolve detection runs in-process, not via `scenedetect`.** `ContentDetector`
  (which `AdaptiveDetector` inherits) calls `cv2.cvtColor(frame, COLOR_BGR2HSV)` and
  therefore requires a 3-channel BGR image — it cannot consume the 64×64 grayscale
  descriptor, and feeding it one raises inside `cvtColor`. Rather than pay a second
  decode, detect gradual transitions directly from the descriptor series: a fade is a
  sustained monotonic ramp in mean luminance with low block-variance, a dissolve a
  sustained elevated delta without a single-frame spike. `scenedetect` remains a
  dependency only for offline calibration of the thresholds, not for the hot path.
- The state list MUST satisfy: `states[0].span.start == 0`,
  `states[i].span.end == states[i+1].span.start`, and
  `states[-1].span.end == timeline_end`. Assert it.

**Timeline end:** `timeline_end = max(video_end, audio_end)`. Audio routinely outruns
video and vice versa. Any span past the last video frame is a state with
`visual: NONE`; any span past the last audio sample is `audio: NONE`. A length
difference > 0.5 s is a reported property of the input, **not** a drift failure — the
draft had two sections disagreeing about this on the same file.

### 5.6 Proving synchronization (`S11_VERIFY_SYNC`)

The draft had six checks and concluded "R2.4 satisfied". Inject a uniform +400 ms
shift into every transcript timestamp and all six still pass — they are
self-consistency checks, not sync checks. These are the replacements.

**SC1 — Cross-modal offset (the one that actually matters).** Compute a speech-energy
envelope from the audio at 100 Hz and the visual-activity envelope already produced by
`S3`. Cross-correlate over ±2 s in 10 ms steps. Assert `|peak lag| ≤ 50 ms`. On
screen-recording content, additionally measure the offset between spoken phrases and
the appearance of matching OCR text. **This is the only check that can detect a
constant A/V offset, which is the most likely real-world sync failure.**

*Applicability.* SC1 requires `has_speech` **and** a correlation peak whose prominence
clears `sc1_peak_prominence_floor`. It therefore does not apply to music-only,
silent, or AI-generated clips (R1.3.a, R1.3.d), and it is inconclusive — not failing —
on continuous narration over a static screen, where speech energy and visual activity
are genuinely uncorrelated. In both cases SC1 reports `INCONCLUSIVE`, which is recorded
in the Analysis Integrity block and does **not** block the run. Only a confident peak
outside ±50 ms is a failure. A genuine encoder offset in the source is an input
property the tool cannot repair: with `--allow-av-offset` the measured value is recorded
and compensated rather than aborting.

**SC2 — Interval coverage.** Coverage is the **union of analysed intervals** over the
timeline, not `max(last_word, last_frame) / duration`. Under the draft's formula, a
video with one word at the end scores 99.8% while forty minutes in the middle are
missing.

Coverage is computed **per modality** and both must hold:

- *Visual*: the visual-state intervals must partition `[0, timeline_end)` exactly — 100%
  by construction (SC3), so any shortfall is a bug.
- *Audio* (only when `has_audio`): `union(word intervals ∪ detected-silence ∪
  detected-music ∪ detected-non-speech)` must cover ≥ 99% of the audio span, with **no
  unexplained gap > 2 s**, where "explained" means overlapping a positively detected
  silence, music, or non-speech interval.

With no audio track the audio clause is skipped and the Analysis Integrity block records
the video as silent. It is never treated as an unexplained gap.

**SC3 — Visual partition integrity.** Assert the state list partitions the timeline
exactly (§5.5), all PTS strictly increasing, none negative after epoch subtraction.

**SC4 — Forced-alignment residual.** Median `|t_asr − t_forced|` ≤ 150 ms (§6.4). A
larger residual means the ASR timestamps drifted and the run must not proceed silently.

**SC5 — Offset round-trip.** The `-itsoffset` test of §5.3, run against a generated
fixture on every CI run.

**SC6 — Sampled re-extraction.** Re-extract 5% of representative frames at their
recorded timestamps and assert the descriptor matches the stored frame. This catches
any residual timestamp/file mismatch.

Explicitly **not** a sync check, and must not be reported as one: "≥90% of words are
within tolerance of a frame." That metric is circular — it is computed *with* the
tolerance it is checked against, and it mechanically improves as you sample more
frames while telling you nothing about synchronization. Report the binding-error
distribution instead, as a distribution.

---

## 6. Audio pipeline

### 6.1 Extraction (`S2_AUDIO_EXTRACT`)

Two WAVs, both with `t=0` recorded relative to the master timeline (§5.3):

- `asr.wav` — 16 kHz mono. For transcription only.
- `analysis.wav` — 48 kHz stereo. For music/tempo/key/sound events. The draft ran
  music analysis on the 16 kHz ASR file, discarding everything above 8 kHz and then
  claiming to characterise genre, key and timbre (R4.11) from band-limited mono.

### 6.2 Loudness

Normalise for ASR only, as a **gain-only** `volume=` filter, and record the applied
gain. A re-encoding normaliser can change sample count or padding, which silently
shifts every timestamp on top of §5.3.

**The silence threshold must be relative, not absolute.** The draft normalised to
−23 LUFS and then applied a fixed −40 dB silence threshold: a quiet phone capture gets
+18 dB of gain, lifting room tone to −35 dB, so `silencedetect` reports **zero**
silence for the entire file and every downstream "explained gap" check breaks. Measure
the noise floor first (`ebur128`/`volumedetect`) and set the threshold at
`noise_floor + 10 dB`.

### 6.3 Transcription (`S5_TRANSCRIBE`)

Default `mlx-whisper` with `large-v3`. Fallback `faster-whisper`.

**The two backends have different APIs. Write an adapter; do not write against one and
assume the other.** `mlx_whisper.transcribe()` returns plain dicts and does not accept
`vad_filter`; `faster-whisper` returns objects and does. The mandatory behaviours below
are stated once, with the per-backend realisation:

| Behaviour | `mlx-whisper` (default) | `faster-whisper` (fallback) |
|---|---|---|
| Word timestamps | `word_timestamps=True` | `word_timestamps=True` |
| Confidence field | `word["probability"]` (dict) | `word.probability` (attribute) |
| VAD filtering | not applicable — does not VAD-filter | `vad_filter=False` (see below) |
| Prior conditioning | `condition_on_previous_text=False` | `condition_on_previous_text=False` |

Mandatory call parameters — each fixes a specific defect:

- **`word_timestamps=True`.** Without it `segment.words` is `None` and the word loop
  raises `TypeError` on the first segment. R2.3 and R2.4 rest entirely on word-level
  timestamps; this is the foundation stage.
- **The confidence field is `word.probability`, not `word.confidence`.** The latter
  does not exist and raises `AttributeError`.
- **`language=None`** (auto-detect), then pass the detected code explicitly. The draft
  hardcoded `language="en"`, which transcribes a Russian or German source as English
  and is not verbatim by any definition (R1, R4.6).
- **`vad_filter=False` for the transcript pass.** VAD filtering deletes quiet asides,
  whispers, and short utterances from a transcript that R4.6 requires to be
  word-for-word. Run VAD *separately* as a silence signal.
- **`condition_on_previous_text=False`**, and chunk at detected silences ≥1 s into
  ≤10-minute blocks with known offsets, so a single malformed timestamp cannot shift
  the decoder's seek pointer and offset every subsequent segment for the rest of a
  90-minute file.

### 6.4 Forced alignment and the hallucination guard

Whisper's native word times come from cross-attention DTW and are routinely ±200–500 ms
— not good enough for R2.3, and worse under int8 quantisation.

1. Run WhisperX forced alignment (wav2vec2) over Whisper's text. Store `t_asr`,
   `t_forced`, and the residual. Gate `SC4` (§5.6) hard-fails if median residual
   > 150 ms.
2. **Hallucination guard.** On music-only and silent stretches — which is the entirety
   of the user's R1.3.a and R1.3.d archetypes — Whisper emits confident subtitle-credit
   text spanning the silence. That text is monotonic, high-confidence and
   time-covering, so it passes every naive check, enters the "verbatim transcript",
   gets bound to frames, and is handed to Claude as fact under R5. Guard with all three:
   - Run an independent VAD (silero) on the **un-filtered** audio and **flag** — never
     delete — any word whose interval is < 30% voiced. Deleting words would violate
     R4.6 and R8.2, and no gate would catch it (`G6` compares transcript *span*, not
     word count). Set `hallucination_suspect: true` on the `Word`, render it inline in
     §10.9, and count it in the Analysis Integrity block.
   - Flag n-gram repetition loops.
   - Flag segments with `compression_ratio > 2.4`.
3. Word text is stored **twice**: `text` verbatim (for R4.6) and `norm` — lowercased,
   punctuation- and whitespace-stripped — for matching. Whisper returns words with a
   **leading space** and attached punctuation, so the draft's
   `word.text.lower() in ['click','press',...]` never matched anything: every
   action-word feature, every semantic alignment, and every text-to-visual correlation
   silently produced nothing while the manifest reported the feature had run.

### 6.5 Diarization (`S6_DIARIZE`) — optional

`pyannote/speaker-diarization-community-1` — the default pipeline for the pinned
`pyannote.audio` 4.0.7. (3.1 is the legacy pipeline; it is separately gated on Hugging
Face, so naming one in the bootstrap and loading the other would fail on an unaccepted
gate and silently degrade to a single speaker. §2.5 gates the same model named here.)

Off by default. Enabled with `--diarize`, or automatically when `A2.multi_speaker` is
set. **That auto-trigger is why `S9a_FRAMING` runs before the audio stages in §4.2** —
`A2` must exist before `S6` decides whether to run. `A2.multi_speaker` is therefore
inferred from probe and frame evidence (multiple video tiles, call-UI chrome, container
metadata, two or more audio tracks), never from `speakers.json`, which does not exist
yet. Diarization is the single slowest local stage (§13.2) and adds nothing to a solo
screen recording.

- **Word→turn assignment** is by maximum temporal overlap. On ties, or when overlap is
  < 60%, emit `speaker: "UNKNOWN"` rather than guessing. Words inside overlapped-speech
  regions get `overlapped: true`.
- **If chunking for memory, never merge by label.** `SPEAKER_00` in chunk 2 is not the
  same person as `SPEAKER_00` in chunk 1. Overlap chunks by 30 s and stitch by speaker
  embedding similarity, or re-cluster embeddings globally. The draft's "merge results,
  accounting for time offsets" would attribute the wrong sentences to the wrong people
  for an entire recording while every check passed — corrupting R4.3 meeting points.
- Without an HF token (§2.5), skip: one unnamed speaker, and the output says so.

### 6.6 Non-speech audio (`S7_AUDIO_FEATURES`)

Silence intervals, music/speech segmentation, tempo, key, energy, and sound events
(tool noise, sanding, router, hammer — R5.2).

- **Every event MUST carry real timestamps derived from the model's own frame hop**
  (e.g. YAMNet emits one vector per 0.48 s; timestamp = hop index × hop length, then
  `to_master`). The draft computed a whole-file `np.mean(probabilities)` — no time
  axis at all — and then reported per-event `start_time`/`end_time` values that no code
  produced. Fabricated timestamps in the section whose purpose is timeline integrity,
  flowing into R5's "physically grounded durations, not imaginary numbers".
- Use `librosa.feature.tempo`. `librosa.beat.tempo` still works in the pinned 0.11.0
  but emits a `FutureWarning` — it moved in 0.10.0 and is slated for removal in 1.0.
  It returns an ndarray, so index before formatting.
- Process in 60-second windows via `librosa.stream`. A full-file `chroma_cqt` +
  `tempogram` on a 90-minute track is multi-GB of intermediates and 5–15 minutes
  (§13.3).
- **Sound-event classification (R5.2) has one sanctioned implementation:** YAMNet via
  `tensorflow-macos` + `tensorflow-hub`, which do publish arm64 wheels. It is a heavy
  dependency for one feature, so it is **optional**, installed via the
  `videoanalyser[soundevents]` extra and enabled with `--sound-events`. When it is not
  installed, tool-sound detection is skipped and the Analysis Integrity block records
  `Sound events: not analysed (extra not installed)`. R5.2's tool *identity* still comes
  from `C2` via transcript and vision; only the acoustic confirmation is lost.
- **Do not use `essentia` or `panns-inference`.** `essentia-d` and `pann` are not real
  package names, `panns_inference.GooglePANNs` is not an API, and essentia has no macOS
  arm64 wheels — a hard install failure on the exact target machine.

---

## 7. Visual pipeline

Dense scan and state segmentation are specified in §5.4 and §5.5. This section covers
what happens after states exist.

### 7.1 Representative frames (`S8_REPRESENTATIVES`)

One representative per visual state, extracted at the state's **actual decoded PTS**.

**Never record a requested time as a frame's timestamp.** The draft ran
`ffmpeg -ss 12.345 -vframes 1` and wrote `timestamp: 12.345` into the manifest. On a
VFR screen recording — which emits frames only when the picture changes — the frame
actually returned can be seconds earlier, because that is genuinely the picture at
12.345 s. The manifest then claims a frame at 12.345 s whose true PTS is 4.1 s, and the
Markdown shows it as the visual state for a sentence spoken at 12.345 s. Extract with
`-copyts` and record the emitted PTS, storing `requested_t`, `actual_pts`, and the delta.

Extraction is a **single decode pass** over a sorted timestamp list using a
`select='between(t,...)'` expression. Per-frame output seeking re-decodes the file from
frame 0 every time — hundreds of full-file decodes on a long video.

Write **JPEG q≈2** to the cache, not PNG. A 4K PNG is 8–20 MB; the draft's own
arithmetic reached 21.6 GB of intermediates for one 90-minute video, guarded by a 1 GB
free-space check. PNG is used only for frames promoted to the deliverable (§11).

Add `-vf scale=in_range=tv:out_range=pc` **only when the source is untagged**. swscale
already expands correctly when the stream carries a limited-range tag, and `out_range`
is meaningless for an RGB destination (RGB is full-range by definition) — so on a
properly tagged file the flag is a no-op. The load-bearing half is `in_range=tv` on
untagged sources, which is where the washed-out screenshots actually come from.

### 7.2 OCR

`ocrmac` (Apple Vision) over each representative frame. Store text **and bounding
boxes** — the boxes are required for step-field extraction (§10.4) and optional
annotation (§11), and the draft specified annotation while producing no coordinates
anywhere.

**OCR output is passed to the vision model as text.** Sending a full-resolution frame
and asking the model to re-read text already extracted locally for free is the single
largest avoidable line item in the API bill (§13.1).

---

## 8. The alignment engine (`S10_ALIGN`)

### 8.1 Two quantities, not one

The draft used a single `lead_lag_tolerance = 1.0 s` for two unrelated things. Split them:

- **Binding error** — the discrepancy between a word's time and the visual state it is
  bound to. Because states partition the timeline (§5.5) and both artifacts share one
  clock (§5.3), this is **0 by construction**. Assert exactly 0; the 50 ms figure
  quoted elsewhere absorbs floating-point representation error in the comparison only,
  and is not a tolerance for real misalignment.
- **Behavioural lead/lag** — how far a spoken instruction precedes or follows the
  action it describes. This is a real, interesting human quantity (people say "click
  here" a beat before clicking). **Measure and report its distribution; never use it as
  a correctness gate.** Defaults for search windows: −0.5 s lead, +1.5 s lag.

A 1.0 s tolerance is 17–25× looser than human perceptibility (EBU R37 puts acceptable
A/V offset near +40/−60 ms) and is precisely why the draft's sync checks could not detect a
400 ms error.

### 8.2 Binding by containment, not nearest-neighbour

Bind each word to the visual state whose half-open interval **contains** its midpoint
(`bisect` over state starts).

Nearest-neighbour over a sparse representative list is wrong in a way that produces
confident, plausible, incorrect output: representatives mark the *start* of states that
may last 30 s, so any word past the midpoint of a long static stretch snaps to the
**next** state. Concretely — a form is on screen 10 s→40 s; at t=32 s the narrator says
"now fill in your email"; nearest-neighbour returns the 40 s post-submit confirmation
page. The tutorial then illustrates "fill in your email" with a screenshot of the
confirmation screen. Containment makes this structurally impossible.

Containment also gives a real `UNALIGNED` outcome. Under `min()` every word always gets
some frame, so the draft's reported `"UNALIGNED": 0` was a structural certainty, not a
finding — and `min()` on an empty frame list raises `ValueError`.

### 8.3 Clause and step binding — the contract R4.8/R4.9 depend on

Word-level binding alone does not satisfy R4.8 ("a screenshot of every step explained").
Something must produce *step ↔ screenshot*. Define it:

1. **Clause segmentation:** boundaries at ASR segment ends, silences ≥ 400 ms, and
   speaker changes.
2. **Clause → states:** the states overlapping `[clause_start − lead, clause_end + lag]`;
   the dominant state is the one with the greatest overlap duration. Ties break toward
   the state whose OCR text shares content words with the clause.
3. **Candidate steps:** a candidate is a contiguous clause run sharing an action verb
   and a target, or an explicit enumeration in the narration ("first", "step two").
4. Emit a **step manifest** of candidates — the assembler's contract, schema in §4.3.

`S10` assigns every `step_id`. Persona `C1` may **merge, split, or label** candidates
but may not mint an ID; a split inherits the parent ID with a suffix (`step_04a`). This
is the single-owner rule of §9.11 made concrete — without it `S10` and `C1` would both
be segmenting the video, which is the exact defect §9.11 exists to prevent.

**Deictic words raise the visual-sync requirement, they do not lower it.** The draft
listed `here`, `there`, `look`, `see`, `notice`, `now`, `next`, `then` as
`needs_visual_sync = False` — exactly backwards. "Look *here*" and "*now* it turns
green" are the moments that most require the right frame.

### 8.4 Visual-change detection must be able to say "nothing happened"

When testing whether a word caused a visible change, require the change magnitude to
exceed a floor calibrated from the file's own inter-frame distance distribution
(95th percentile, with a hard minimum). Otherwise return `NO_CHANGE_DETECTED`.

The draft returned `argmax` unconditionally with no floor, so on a completely static
screen it reported the largest of three noise deltas as "the visual change caused by the
word 'click'", complete with a confident interpretation string.

---

## 9. The multi-persona analysis system (`S12_PERSONAS`)

R3 asks for many personas covering many perspectives. This section makes that a
structured system rather than a list of prompts.

### 9.1 One canonical category enum

Declared once, referenced everywhere. The draft had personas keyed on trigger strings
(`craft`, `meeting`, `tutorial`, `ai_generated`) that the categoriser could never emit,
so most personas could never fire — the style analyst (R4.11) only ran for
`music_inspiration`, and meeting points (R4.3) only for `video_conference`, meaning the
user's own "video call with screen share" archetype lost its main points entirely.

```
screen_share_tutorial | software_tutorial | workshop_tutorial | talking_head
| video_conference | music_inspiration | mixed | other
```

Plus independent boolean attributes. **These are not all produced by the same stage,
and that distinction is load-bearing** — three blocking gates read them:

| Attribute | Produced by | How |
|---|---|---|
| `has_screen_content` | `A2` (`S9a`) | Frame evidence: UI chrome, text density, straight edges |
| `has_physical_tools` | `A2` (`S9a`) | Frame evidence |
| `has_timelapse` | `A2` (`S9a`) | Frame evidence + probe frame rate |
| `multi_speaker` | `A2` (`S9a`) | Probe + frame evidence only — video tiles, call UI, ≥2 audio tracks. **Never** from `speakers.json`, which does not exist yet (§6.5) |
| `has_speech` | **`S5`** | Voiced-word fraction ≥ `has_speech_floor` |
| `has_music` | **`S7`** | Music-segment fraction ≥ `has_music_floor` |

`has_speech` and `has_music` are properties of the *audio* and cannot be inferred from
`probe.json` and a few video frames — a music-only Instagram clip has an audio stream,
so a frame-derived guess would read `has_speech: true`, fire `G6`, and quarantine
exactly the archetype §12.1's carve-out exists to protect. Gates read the `S5`/`S7`
values, never `A2`'s. Both floors are in §14.3.

**Routing is a table from (category, attributes) → persona set, with an explicit
default that activates everything.** No category may fall through to an empty set.

### 9.2 The roster

Grouped by pass. Every persona has one job; overlapping jobs were merged.

**Pass A — provenance and framing** (needs no vision beyond a few frames)
| ID | Persona | Job | R |
|---|---|---|---|
| A1 | Provenance analyst | Where this video came from: aspect ratio, UI chrome, watermarks, container metadata, encoder fingerprints | R3.1, R7.3 |
| A2 | Categoriser | Category enum + attributes + title | R3.2 |

**Pass B — grounded description** (the expensive pass; §13.1 governs its budget)
| ID | Persona | Job | R |
|---|---|---|---|
| B1 | Visual state describer | For each state: what is on screen, layout, what changed from the previous state | R2.1, R3.3 |
| B2 | Temporal structure analyst | Chapters, pacing, step boundaries, on-screen durations | R3.5 |
| B3 | Real-world duration analyst | Process time vs screen time; elisions; cure/dry/set times | R5.3, R5.4, R5.5 |

**Pass C — domain depth**
| ID | Persona | Job | R |
|---|---|---|---|
| C1 | UI/step analyst | Application, URL, exact element, interaction verb, before/after state | R4.9 |
| C2 | Tools & materials extractor | Tool identity, blade/bit/grit, settings, consumables | R4.10, R5.1, R5.2 |
| C3 | Technique analyst | What they are doing, why, and how | R4.10 |
| C4 | Safety & prerequisites analyst | PPE, hazards, skills and materials needed beforehand | R5 |
| C5 | Transferability analyst | Which parameters generalise to a different project, and which are specific | R5.6 |
| C6 | Aesthetic / style analyst | Grade, palette, filters/LUT, lens, editing rhythm, mood | R4.11 |
| C7 | Music & sound analyst | Genre, tempo, key, energy, sound design, diegetic tool sounds | R4.11 |
| C8 | Meeting points extractor | Decisions, action items, open questions | R4.3 |
| C9 | Differentiator analyst | What makes this video distinctive versus others of its kind | R3.4 |

**Pass D — audit** (adversarial; these may send work back)
| ID | Persona | Job | R |
|---|---|---|---|
| D1 | Grounding auditor | Every claim carries evidence and the correct claim class | R8 |
| D2 | Contradiction auditor | No two claims conflict; conflicts escalate rather than merge | R8.4 |
| D3 | Coverage auditor | No unexplained span of the timeline is unanalysed | R8.2 |
| D4 | Tutorial completeness auditor | Every step has fields, evidence, and a screenshot | R3.6, R4.8 |
| D5 | Screenshot↔audio verifier | Runs **after** export (§9.4): files exist, are referenced, are non-blank, match their timestamps | R3.7 |

**Pass E — assembly**
| ID | Persona | Job |
|---|---|---|
| E1 | Synthesizer | Writes the prose sections of the Markdown |

The transcript, timeline table and step tables are **not** written by E1 — see §9.3.

### 9.3 The verbatim transcript never enters an LLM context as content to reproduce

R4.6 is absolute. Three mechanisms in the draft guaranteed violation: the assembler had
a ~50 KB context budget (a 90-minute transcript is 75–90 KB alone), transcripts were
"pre-chunked so personas receive only relevant segments", and the transcript stage was
itself modelled as a generative persona that could "re-listen to unclear segments" —
which an LLM cannot do.

**Rule:** §10.9 is emitted by `render_transcript(transcript.json)`, a deterministic
string-templating function. The transcript is exempt from every context budget. The
same applies to the timeline table and the step tables — data goes through templating,
prose goes through the model.

Corollary: where a step quotes the narrator (§10.4), that quote MUST be an exact
substring slice of the transcript keyed by word IDs, and gate `G7` asserts
byte-equality after whitespace normalisation. Two independently generated copies of the
same speech is exactly where paraphrase leaks in.

### 9.4 Ordering

`A → B → C → D → E → export → D5`.

`D5` is the only persona that runs **after** assembly and export, because its job is to
verify files that do not exist until then. In the draft it ran before export and was
handed in-memory frame objects, so nothing ever verified the delivered PNGs — leaving
R3.7's "no broken links, no garbage" duty unowned.

The dependency graph is published once as a DAG in code. The draft carried a cycle
(two Pass-A personas each depended on the other), referenced two personas that did not exist,
and gave one persona three mutually incompatible positions.

### 9.5 Claim classes — including the one that makes R5 answerable

Every claim is exactly one of:

| Class | Meaning | Example |
|---|---|---|
| `fact` | Stated in the transcript or visible on screen. Carries evidence. | "He says the glue needs to sit overnight." |
| `inference` | Derived by reasoning. Carries evidence **and** basis. | "This is a router table, inferred from the fence and the visible bit." |
| `domain_estimate` | **Not** in the video; supplied from domain knowledge because the video leaves a needed quantity unstated. Carries `value_range`, `unit`, `source: "domain knowledge"`, `applies_because`. | "Typical PVA wood glue reaches full strength in 24–48 h under clamps; the video shows only initial set." |

`domain_estimate` exists because R5.3/R5.4 are otherwise unanswerable. The user's three
canonical cases — stain drying half a day, glue curing two days, steam bending 30
minutes — are exactly the quantities a video shows without stating. The draft forbade
the only persona that could supply them from doing so, and its conflict rules then
*discarded* the domain fact in favour of the transcript. A `domain_estimate` may never
overwrite a stated value; both are rendered side by side:

> Video says: "leave it overnight." Typical full cure: 24–48 h (domain estimate).

### 9.6 Grounding, enforced mechanically

- Timestamps are `HH:MM:SS.mmm` **everywhere**. The draft's schema used `MM:SS.mmm`,
  which cannot represent any video over 99 min 59 s — every claim in a 2-hour video
  would fail validation, trigger the retry loop, and flag the run for manual review
  after paying for it.
- Every claim is validated against the Pydantic `Claim` model. Evidence references MUST
  resolve to a real word ID, state index, or OCR box. Unresolvable evidence fails the
  claim; it is not a warning.
- Claim schemas MUST accept the fields personas actually emit — the draft set
  `additionalProperties: false` while its own canonical example carried an extra field,
  so the example failed the schema.
- `needs_visual` implies `backing_state is not None`, enforced in the same Pydantic
  validation. Unlike `visual_backup` — which cannot exist until `S13` has frozen the
  export set — `backing_state` is a state index the persona already has, so requiring it
  at emission time is satisfiable and does not reintroduce the drop-every-style-claim
  failure the `Claim` model's comment guards against. Without this rule the export
  mechanism dereferences a null and `G14` passes vacuously.
- Bounded remediation: at most **2** retries targeting only the failing claims. On
  exhaustion the claim is dropped and recorded in the coverage report, never silently
  kept.

### 9.7 Elision detection — how R5.5 is actually satisfied

Nothing in the draft ever computed the screen-time↔real-time ratio it defined a field
for. Detect elisions explicitly and emit an `elision_marker` on the timeline:

- OCR title cards matching `/later|next day|hours?|overnight|next morning|\d+\s*(min|hour|day)/i`
- Cut with a large lighting/shadow change (suggesting a different time of day)
- Motion-energy spike consistent with a speed ramp; `has_timelapse` from A2
- A narration cue ("after two days", "once it's dry")

Any step containing an elision marker MUST carry a non-null real-world duration or a
`domain_estimate`. This is what lets a downstream AI answer "how long will this take?"
with 2 days rather than 10 seconds.

### 9.8 Evidence must resolve, not merely exist

The draft's entire anti-hallucination mechanism was a schema that enforced the
**presence** of an `evidence_reference` string. A model can emit
`frame_00:02:15.500` for a frame that was never extracted and pass that check.

Implement `resolve_evidence(ref) -> Artifact | None` against a lookup table built from
`transcript.json`, `visual_states.json` and the OCR index. Any claim carrying an
unresolvable reference is **rejected before it reaches the assembler**. Additionally,
for a random 10% sample of `fact` claims, re-send the referenced artifact with a
yes/no verification question — presence-checking alone cannot satisfy R8.4.

### 9.9 Persona prompts are written out in full

The draft shipped a prompt *template* containing unfilled holes
(`{schema for this persona}`, `[List of what this persona must never do]`) and no
actual persona prompt text anywhere. Under R9.4 that is not executable.

**Deliverable:** `videoanalyser/prompts/<persona_id>.md`, one complete prompt per
persona, each stating role, inputs, the exact output JSON schema, the prohibitions, and
two worked examples. Milestone M5 (§16) is not complete until all of them exist.

### 9.10 Conflict resolution — never silently pick

When two grounded personas disagree, emit both with a `[CONFLICT]` marker and both
evidence references. Do **not** rank by self-reported confidence scores: they come from
the same model family, are uncalibrated, and are not comparable across personas.

The draft's own worked example resolved a conflict by writing "Video states overnight
(≈12–16 hours); full cure may require longer per product specs" — inventing numeric
precision the video never gave and citing "product specs" with no source. That is
exactly the imaginary number R5.3 forbids. Correct output:

> Video states "overnight" (`transcript[seg_045]`); no numeric duration given.
> Domain estimate: PVA glue reaches full strength in 24–48 h.

### 9.11 One canonical step spine

Four personas in the draft independently segmented the same video into steps with four
different ID schemes and no merge rule, so the same tutorial would appear enumerated
several incompatible ways. **`C1` owns the step spine.** Every other persona references
`step_id` from it and may not mint its own. The same applies to prerequisites, PPE, and
on-screen text, each of which had three competing owners.

---

## 10. The output Markdown — literal specification

`E1` fills the prose; everything marked *(templated)* is emitted deterministically from
JSON and never passes through a model (§9.3).

### 10.1 Section order

```
---  YAML front matter (templated)  ---
# <Title>
## Analysis Integrity          (templated)   <- §10.2, MANDATORY, always first
## General description                        R4.4
## Short summary                              R4.5
## Category and source                        R3.1, R3.2
## Key points                                 R3.4
## Meeting main points        (if applicable) R4.3
## Timeline               (templated)         R4.2
## Synchronized record    (templated)         R2.3  <- §10.3
## Steps                                      R4.8, R4.9, R4.10  <- §10.4
## Explained topics       (conditional)       R4.8               <- §10.5
## Visual walkthrough     (conditional)       R2.3, R4.7  <- §10.6
## Tools and materials                        R5.1, R5.2
## Timing and real-world durations            R5.3, R5.4, R5.5
## Applying this elsewhere                    R5.6
## Look, style and mood       (if applicable) R4.11
## Music and sound            (if applicable) R4.11
## Full transcript        (templated)         R4.6  <- §10.9
```

**"Conditional" is not a judgement call.** `G1` needs a computable predicate, so the
required-section set is a table keyed on the §9.1 category enum:

| Section | Required when |
|---|---|
| Meeting main points | `multi_speaker` **and** category ∈ {`video_conference`, `screen_share_tutorial`} |
| Steps | a non-empty step manifest exists |
| Explained topics | `B2` emitted at least one `explanation_span` (§10.5) |
| Visual walkthrough | at least one **significant transition** (defined in §10.6) falls outside every step, **or** any `needs_visual` claim forced an export (§10.6) — hosts those screenshots with their descriptions |
| Look, style and mood | `category ∈ {music_inspiration, mixed}` **or** `has_music` |
| Music and sound | `has_music` |
| Tools and materials | `has_physical_tools` **or** any tool claim exists |

Every other section is unconditionally required. `G1` reads this table.

R4.4, R4.5 and R5.4 had **no owner at all** in the draft — no persona was tasked with
writing a general description, a short summary, or a project-time estimate. They are
`E1`'s explicit responsibility, synthesised only from grounded persona claims.

### 10.2 The Analysis Integrity block — mandatory, never omitted

Because the user uploads this file to an AI (R5), a degraded run must not be
indistinguishable from a complete one.

```markdown
## Analysis Integrity

| Property | Value |
|---|---|
| Timeline coverage | 99.4% (no unexplained gap > 2 s) |
| Sync verification | PASS — cross-modal peak lag 12 ms |
| Words transcribed | 8,431 (0 dropped) |
| Visual states | 214 (0 unanalysed) |
| Screenshots | 23 exported, 23 referenced, 0 orphaned |
| Speaker labels | 2 speakers, 41 words UNKNOWN |
| Claims | 312 fact · 74 inference · 9 domain estimate |
| Stages skipped | none |
| Unresolved evidence | 0 |
```

If anything degraded, this block says so in plain language at the top of the file.
A run that dropped frames, skipped diarization, or exhausted its budget reports it here.

### 10.3 The synchronized record — for every video, not just tutorials

R2.3 is the user's central requirement and applies to all archetypes. Emit a
three-column table at utterance/state granularity, templated from `alignment.json`:

```markdown
| Time | On screen | Audio |
|---|---|---|
| 00:01:12.400 | Chrome, Drive home, "New" button highlighted | "so we click New up in the top left" |
| 00:01:15.900 | Upload menu open, "File upload" hovered | "and then File upload" |
```

The draft offered fine-grained pairing only inside its tutorial section, so the user's
Instagram, AI-inspiration and static-slide archetypes degraded to 30-second chapter
rows — for the requirement the user cared about most.

### 10.4 Steps — machine-checkable fields, not free prose

R4.9 wants "the website there, the button that they're pressing, the second step, the
screen, how it looks". R4.10 wants "what tool is it, what are you doing, why, how".
Free prose lets a step be written as "then they configure the settings", which loses
exactly what was asked for. Every step therefore carries **required key/value lines**:

```markdown
### Step 3 — Upload the file

- **Time:** 00:01:15.900 – 00:01:24.100 (8.2 s on screen)
- **Application/URL:** Google Drive — `https://drive.google.com/drive/my-drive`
- **Action:** Click the menu item labelled "File upload" in the "New" dropdown
- **Result:** The macOS file picker opens, defaulting to Recents
- **Why this step matters:** This is the only path that preserves the original file
  format; the drag-and-drop route re-encodes Google-native formats.
- **Watch out for:** If a folder is selected first, this menu item reads
  "Folder upload" instead and uploads the whole directory.

![The New dropdown open in Google Drive, with "File upload" highlighted in blue as the second item, above "Folder upload"](<../screenshots/Upload%20a%20file%20to%20Drive%20%7C%2028.07.2026%20%7C%20original%3A%20youtube%20video/03_click-file-upload.png>)

**What you see on screen:** The Drive interface with the left sidebar showing My Drive,
Computers and Shared with me. The blue "New" button at the top left has been clicked and
its dropdown is open, showing six items. The second, "File upload", is highlighted...

> **What the narrator says:** "and then File upload"
```

For physical/workshop content (R4.10) the field set is:

- **Tool:** the specific tool — "random orbital sander, 5-inch"
- **Consumable/setting:** "120 grit, then 180 grit" · "1/4-inch straight bit, 12,000 rpm"
- **Action:** verb + workpiece + manner
- **Parameters:** each as `name: value` — grit, speed, angle, passes, depth, feed
- **Why / Watch out for:** as above

Gate `G8` asserts every step has each required field, non-empty, and that **Action**
begins with a verb from the controlled list in §14.4. A vague step cannot be emitted.

**The required set depends on the content class**, because a software step legitimately
has no `tool` and a workshop step has no `application` — the `Step` model (§4.3) marks
both Optional for exactly that reason. `G8` reads this table:

| Field | Software step (`has_screen_content`) | Physical step (`has_physical_tools`) |
|---|---|---|
| `span`, `action_verb`, `target`, `result`, `why` | required | required |
| `screenshot_ids` (≥ 1) | required | required |
| `application` | required | — |
| `url` | required when the application is a website | — |
| `tool` | — | required |
| `consumable` | — | required when one is used |
| `parameters` | — | required, ≥ 1 entry |
| `watch_out` | optional | optional |

**`G8` runs in `S14`, after `S13` has backfilled `screenshot_ids`.** `S10` emits
`backing_states`; it cannot emit screenshot ids, which are not determined until the
export set is frozen and §3.3c's filename width is chosen. Enforcing `≥ 1
screenshot_ids` at `S10` would fail every step on every tutorial, burn both repair
rounds, and quarantine the run — the same trap the `Claim` model avoids for
`visual_backup`.

A step in a video with both classes must satisfy whichever class its own evidence
supports; if both, both. Read literally without this table, `G8` fails every step ever
emitted; read loosely, it enforces nothing.

### 10.5 Explained topics (R4.8)

R4.8 covers a two-minute explanation inside a seven-minute video that is not itself a
tutorial. Whenever a contiguous span is **classified as explanation**, emit an
**Explained topics** section using the same field set and the same ≥1-screenshot-per-step
rule. Without this, a talking-head video with one explained concept falls through every
section in the template.

**"Classified as explanation" is a computable predicate, not a judgement call** — `G1`
is blocking and depends on it. Persona `B2` owns the classification and emits
`explanation_spans` into the alignment artifact. A span qualifies when all hold:

- duration ≥ `explanation_min_seconds` (default 60);
- speech density ≥ `explanation_speech_density` (default 0.6 — the fraction of the span
  covered by word intervals), so a silent working shot does not qualify;
- **no** step-manifest candidate (§8.3) overlaps it — an explained procedure is already
  covered by §10.4, and this section exists for the non-procedural case.

Both thresholds live in §14.3.

### 10.6 Screenshot density, and the Visual walkthrough section

**≥ 1 screenshot per step** (R4.8, R4.9), and ≥ 1 per explained-topic step. This is a
floor, not a target. The draft suggested "6–10 screenshots for a 3–5 minute tutorial",
which actively pushes a dense 14-step UI walkthrough to drop half its steps.

**A "significant transition" is a defined quantity, not a judgement call** — `G1` is
blocking and `G3`/`G5` and §15.3's determinism criterion all depend on the resulting
export set. A state boundary is significant when **both** hold:

- the boundary ranks in the top `walkthrough_top_boundary_fraction` (default 0.05) of
  **state boundaries in this file**, scored by *the same detector that created it* —
  per-tile MAD for screen content, pHash Hamming distance for camera footage (§5.5);
  **and**
- the state it opens lasts at least `walkthrough_min_state_seconds` (default 3.0).

**Rank boundaries against boundaries, using one detector's own scale.** Do **not** reuse
§8.4's `change_floor_percentile`: that is a percentile over *whole-frame* inter-frame
distances computed per word, and it is not commensurable with either segmentation score.
Comparing a pHash Hamming distance (0–64 bits) to a MAD distribution (0–255) is
meaningless; comparing a single tile's MAD to a whole-frame MAD fails in both directions
— one tile changing hard scores ~1.25 whole-frame and is silently discarded, while on a
mostly-static recording the 95th percentile of whole-frame MADs sits in the codec-noise
floor and *every* boundary qualifies. Those two readings differ by roughly 20×, and the
resulting export set feeds `G1`, `G3`, `G5`, §13.5's cost model and §15.3's determinism
criterion.

**Screenshots at significant transitions that fall outside any step** are exported only
when §10.1's *Visual walkthrough* section is emitted, and they live there.

**A claim that needs visual backing also forces an export.** `G14` is blocking and
requires every `needs_visual` claim to resolve to an *exported* screenshot — so a rule
must exist that produces one. It does: the state named by that claim's `backing_state`
field (§4.3) is added to the export set, and emitting one forces the *Visual walkthrough* section on
even when no transition otherwise qualified. Without this, a talking-head or
music-inspiration video with no step manifest has style claims that are `needs_visual`
by definition and nothing to point at — `G14` blocks, the repair rounds cannot mint a
screenshot, and the run quarantines. This is the mechanism §17 credits for R4.7.

Nothing may be exported without a section that references it: `G3` treats an
unreferenced file as a blocking orphan, and `G5` requires alt text and a "What you see
on screen" block for *every* image — so an exported frame with no home fails two gates.

### 10.7 Alt text carries the content

Every image's alt text must independently describe what the image shows, in ≥ 60
characters, and must not match `^(screenshot|image|step \d+)`. This is enforced by gate
`G5`, not merely requested — it is the mechanism that keeps the file useful when
uploaded without its images (§10.8).

### 10.8 Making the file useful after upload (R5)

**When a `.md` is uploaded to a chat, it is ingested as plain text.** Nothing renders;
there is no broken-image placeholder. The model sees the literal
`![alt](<path>)` characters — so the alt text and surrounding prose are the only things
that survive, and the long percent-encoded path is wasted tokens.

Three delivery modes:

| Mode | Flag | What it produces |
|---|---|---|
| Linked (default) | — | `.md` + `screenshots/` folder. Correct locally. |
| Self-contained | `--self-contained` | A second `.md` with images re-encoded to WebP q80 at ≤ 1024 px and inlined as `data:` URIs. Hard cap 8 MB total; over the cap, only step-critical images are inlined and the rest are listed. |
| Bundle | `--bundle` | A `.zip` of the Markdown plus its screenshot folder. |

Base64-inlining full-size PNGs — the draft's suggestion, which it then declared out of
scope — yields a 6–65 MB Markdown file that cannot be uploaded anywhere.

### 10.9 The transcript

Templated (§9.3), verbatim (R4.6), with timestamps and speaker labels, including filler
words, false starts and repetitions. Never truncated, never summarised, never
spell-corrected, and exempt from every context budget.

---

## 11. Screenshots

- **Format:** PNG for UI/screen content (lossless — JPEG ringing on small text is the
  dominant OCR and vision failure mode, and R4.9 depends on reading button labels);
  WebP q90 permitted for photographic workshop frames.
- **Resolution:** native, capped at 2560 px on the long edge. Never upscale — a
  1024×576 source does not become readable by enlarging it.
- **Colour:** apply `scale=in_range=tv:out_range=pc` for limited-range sources, or
  every screenshot is visibly washed out.
- **No file-size floor.** The draft required 500 KB–2 MB and made it a pass condition;
  a flat-UI PNG legitimately lands at 100–300 KB. Cap at 10 MB, no floor.
- **Naming:** §3.3c. Deterministic and derived from **video position**, never a
  wall-clock capture time — the draft used a Unix timestamp, which changes every
  filename and every Markdown link on each re-run while claiming determinism as an
  acceptance criterion.
- **Annotation** (highlighting the exact element pressed) requires pixel coordinates.
  It is enabled only when OCR supplied a bounding box for the target element, sizes are
  expressed as percentages of image width, and it is behind `--annotate`. Never
  "annotate only the first of several" — that discards the detail R4.9 asks for.

---

## 12. QA gates (`S14_QA`)

### 12.1 The gates are code, not prose

`videoanalyser/qa.py` returns a list of violations. Each gate has an ID, a `severity`
(`blocking` | `warning`), and a test. The draft's gates ended in passive constructions
("flag as incomplete; require revision") with no actor, no retry bound, no exit code —
and the template hardcoded `qa_passed: true`.

| ID | Gate | Severity |
|---|---|---|
| G1 | Every required section present and non-empty | blocking |
| G2 | Every `![](...)` target resolves on disk after percent-decoding and NFC normalisation | blocking |
| G3 | Every exported screenshot is referenced ≥ 1 time (no orphans) | blocking |
| G4 | No two screenshots are byte-identical; perceptual near-duplicates flagged | warning |
| G5 | Every image has alt text ≥ 60 chars, non-generic, plus a "What you see on screen" block ≥ 40 words | blocking |
| G6 | *(only when `has_speech`)* `union(transcribed ∪ silence ∪ music ∪ non-speech)` ≥ 99% of the audio span, with no unexplained gap > 2 s | blocking |
| G7 | Every narrator quote is a byte-exact substring of the transcript after whitespace normalisation | blocking |
| G8 | Every step has the required fields **for its content class** (table below), non-empty, `Action` starting with a controlled verb (§14.4) | blocking |
| G9 | Every claim's evidence reference resolves (§9.8) | blocking |
| G10 | Timestamps monotonic within each table; adjacent chapter rows share boundaries | blocking |
| G11 | No sentinel tokens left in the document (`[CONFLICT]` unresolved, `{`…`}`, template placeholders) | blocking |
| G12 | Folder name matches the R7 regex for the active `sanitize_colon` setting | blocking |
| G13 | Prose sections pass spell-check — **excluding** the transcript and every narrator quote | warning |
| G14 | Every claim with `needs_visual` has a `visual_backup` that resolves to an exported screenshot | blocking |

G6 must be measured against `ffprobe` and VAD output, not against the document's own
front matter — the draft compared the transcript to a duration the same pipeline wrote,
so a run that truncated both passed.

**G6 has no "transcript span" term.** An earlier version compared the span from first to
last word against the container duration, which fails on any video with more than a
second of non-speech at the head or tail — a music intro, a title card, a silent outro,
closing b-roll. That is the common case, and it would have fired *while the union clause
was fully satisfied*, quarantining the run over speech that was never spoken. Truncation
is already caught by the union clause: a transcript short by 30 s leaves a 30-second
unexplained gap. G6 now mirrors `SC2` exactly — same set, same 99% threshold, same 2 s
gap rule — which is what "mirrors" was always meant to mean.

**G6 does not apply to silent or music-only videos.** Without that carve-out, an AI-generated silent clip (R1.3.d) has
no transcript span at all and a music-led Instagram clip (R1.3.a) has near-zero speech
*and* near-zero silence — both would quarantine, and the tool would never produce output
for two of its five mandatory archetypes.

G13 is scoped and non-blocking because a verbatim transcript of real speech is by
definition full of spell-check hits; the draft's unscoped gate would either never pass
or would induce an agent to "fix" the transcript and destroy R4.6.

G10 replaces two draft gates that contradicted each other — one required adjacent
chapter rows to share boundary values, the other forbade duplicate timestamps.

### 12.2 Placeholder detection is structural, not a deny-list

A fixed regex list (`TODO`, `TBD`, `...`) both false-positives on legitimate content
(the transcript's `[INAUDIBLE]` markers, ordinary ellipses) and is trivially evaded by
different filler. Assert instead: every section non-empty and above a per-section word
floor, every table cell non-empty, zero occurrences of the pipeline's own sentinels.

### 12.3 What happens on failure

- **Warnings** are recorded in the Analysis Integrity block; the run succeeds.
- **Blocking** failures trigger up to `max_repair_rounds` (default 2) targeted repairs
  of only the failing section.
- On exhaustion the file is written to `outputs/_quarantine/`, not `outputs/`, with
  `qa_passed: false`, a `## QA FAILURES` section, and a non-zero exit code.

A gate that lets the file ship anyway is decorative.

---

## 13. Cost, memory, runtime — the M1 reality

The draft claimed **~$0.65 per hour of video**. Under its own literal sampling rule
(1 fps to the vision model) a 90-minute video costs **$118–197**, and with persona
fan-out and remediation rounds, **$235–400**. This section is why.

### 13.1 Frame budget and model routing

**One frame budget, referenced everywhere.** The draft had five contradictory ones
(100, 50, 200–300, 1 fps, 6–10).

```
vision_frames = min(len(representatives), vision_frame_cap)   # vision_frame_cap default 300
```

**Every state is accounted for.** Because §5.5 caps state duration at 10 s,
`len(representatives) ≥ duration / 10` on every file, so a duration-derived budget would
always be smaller than the number of states — silently leaving states undescribed while
`D3` asserts full coverage. Instead:

- If `len(representatives) ≤ vision_frame_cap`, every state is described. This covers
  any video up to ~50 minutes of continuously-changing content.
- Above the cap, **temporally adjacent runs** of similar states are merged until the
  count fits. Merging is restricted to adjacent runs precisely so the result is still a
  partition: merging states 3 and 47 by descriptor similarity alone would produce a span
  overlapping everything between them and break `SC3`. Merge the adjacent pair with the
  smallest descriptor distance, repeat until the count fits. A merged state records
  `merged_from: [ids]` and is rendered as one entry; no state is dropped.
- **Owner:** `S8_REPRESENTATIVES` performs the merge, keyed on the `descriptor` field
  persisted in `visual_states.json` (§4.3). `visual_states.json` keeps the raw state
  list; `S10_ALIGN`, §10.3's table, and screenshot export all consume the **merged**
  list, so there is exactly one notion of "a state" downstream.
- Coverage therefore remains 100% by construction, and `D3` checks the merged list.

| Work | Model | Rationale |
|---|---|---|
| Per-state visual description (`B1`) — the bulk | `claude-haiku-4-5` | $1/$5 per MTok. ~5× cheaper than Opus for a descriptive task. |
| Domain personas (`C*`) | `claude-sonnet-5` | $3/$15. Judgement without Opus cost. |
| Audit + synthesis (`D*`, `E1`) — text only | `claude-opus-5` | $5/$25. Reasoning-heavy, no images, low volume. |

**Downscale frames to ≤ 1024 px on the long edge before encoding.** Image cost is
capped per model tier, so the saving differs by route — state both rather than quoting
one number:

| Route | Per-image ceiling | At 1024 px | Saving |
|---|---|---|---|
| `claude-haiku-4-5` (B1, the bulk) | ~1,600 tok | ~786 tok | ~2× |
| `claude-sonnet-5` / `claude-opus-5` | ~4,784 tok | ~786 tok | ~6× |

**786 tokens assumes a 16:9 (or 9:16) frame at 1024 px on the long edge.** Image cost is
`width × height / 750`, so a square or near-square UI crop at 1024 px costs ~1,398
tokens — 1.8× the budgeted figure. Budget per frame from actual dimensions, not from a
flat constant.

Downscaling is safe because the on-screen text has already been read locally by OCR
(§7.2) and is supplied to the model as text. Sending 4K frames so a paid model can re-read text you already
extracted for free is the single largest avoidable line item.

### 13.2 Runtime

Honest targets on an M1 (not Pro/Max), measured per stage and recorded in the run report:

| | 7-minute video | 90-minute video |
|---|---|---|
| Dense visual scan (hwaccel) | 20–40 s | 4–9 min |
| Transcribe + align | 1–2 min | 12–25 min |
| Diarize (optional) | 2–6 min | 15–45 min |
| OCR representatives | 20–60 s | 3–8 min |
| API (synchronous, the default) | 2–5 min | 20–50 min |
| **Total (no diarization)** | **~6–12 min** | **~50–100 min** |

The table assumes **synchronous** API calls (§13.4's default), H.264/HEVC input, and no
diarization. With `--batch` the API row is unbounded in latency — usually under an hour,
guaranteed only within 24 — which is why batching is opt-in and why §15.3's runtime
criterion is scoped to the synchronous path.

`-hwaccel videotoolbox` is **requested** on every decode, not assumed. The base M1's
VideoToolbox decodes H.264 and HEVC only — it has no VP9 or AV1 hardware path, and
ffmpeg silently falls back to software rather than erroring. Since R1.1 requires
`.webm` support, probe the codec at `S1`, record `hwaccel_used: bool` in `probe.json`,
and surface it in the Analysis Integrity block. **The table above applies to H.264/HEVC
sources; software-decoded VP9/AV1 runs roughly 3–6× slower**, and §15.3's runtime
criterion is scoped accordingly.

### 13.3 Memory

Budget **7–8 GB**, not 16 (§2.1). **Never hold two ML models resident.** Stages run
strictly sequentially with explicit teardown (`del model; gc.collect()`; for MLX,
`mx.clear_cache()`). Whisper large-v3 is ~3.1 GB wired; diarization 2–4 GB; OCR ~1.5 GB
— concurrently they exceed the budget, and macOS caps GPU-wired memory on a 16 GB M1 at
roughly 10.6 GB, a smaller pool than the draft assumed. Audio features stream in
60-second windows.

### 13.4 API efficiency — three multipliers the draft left on the table

1. **Prompt caching.** One `cache_control` breakpoint at the end of the shared block
   (schema + grounding rules + transcript + frame manifest). Reads cost ~0.1×; **writes
   cost 1.25× at the 5-minute TTL and 2× at the 1-hour TTL** — quote both, because the
   write multiplier is what decides whether caching pays. Break-even is 2 requests at
   5 min, 3 at 1 hour. Use the 1-hour TTL only for a phase whose fan-out will not
   complete inside 5 minutes.
   **Minimum cacheable prefix is model-dependent** — `claude-haiku-4-5` requires
   **4096 tokens**, and Haiku carries the bulk B1 traffic. A shorter prefix silently
   does not cache and reports `cache_creation_input_tokens: 0`. Do not declare a
   breakpoint on a phase whose shared prefix is under that floor.
2. **Prime before fan-out.** Concurrent requests sharing a prefix **all miss** — none
   has written the cache yet. Issue one cheap **synchronous, non-batch** priming request
   carrying the exact prefix, await the response, *then* fan out. With a 20k-token
   prefix and ten personas, at the **5-minute** write rate (1.25×): naive
   10 × 20k × 1.25 = 250k billed tokens, versus primed 20k × 1.25 + 9 × 20k × 0.1 = 43k.
   At the **1-hour** rate (2×) the same comparison is **400k versus 58k** — quote the
   multiplier you actually chose, or the saving is overstated. Priming cannot itself be
   part of a batch, because a batch is submitted atomically; issue it synchronously
   immediately before submission.
3. **Batch API.** This workload is entirely offline and non-interactive — the textbook
   fit. 50% off, up to 100k requests per batch, results retrievable for 29 days. Submit
   each phase as one batch. This also makes crash recovery free (§4.4).
   **Batching is opt-in (`--batch`), not the default.** Batches usually complete within
   an hour but are only *guaranteed* within 24 h, which is incompatible with §15.3's
   12-minute wall-clock criterion for a short video. Default to synchronous calls for
   interactive runs; use `--batch` for long videos and overnight directory runs, where
   halving the cost matters more than latency. §13.2's runtime table assumes synchronous.
   **Caching and batching also interact:** a batch outliving the cache TTL loses the
   cache mid-flight, so treat the discount as best-effort on batched phases and never
   build the budget on it.

Verify caching per phase: assert `cache_creation_input_tokens > 0` on the **priming**
call, and `cache_read_input_tokens > 0` on the **first fanned-out** call. Asserting a
read on the priming call is wrong — that call is the one doing the write.

### 13.5 Budget control

**What a run actually costs.** §13 exists to replace a number that was wrong by two
orders of magnitude, so it must state its own. At the prices in §13.1, a 300-frame cap,
1024 px frames, and a realistic persona fan-out:

| | 7-minute video | 90-minute video |
|---|---|---|
| B1 per-state description (Haiku, cached prefix) | ~$0.12 | ~$1.07 |
| Domain personas C1–C9 (Sonnet 5) | ~$0.35 | ~$3.24 |
| Audit + synthesis D1–D5, E1 (Opus 5) | ~$0.40 | ~$4.50 |
| **Subtotal** | **~$0.87** | **~$8.81** |
| With `--batch` (50% off) | ~$0.44 | ~$4.41 |
| Worst case, both repair rounds fired | ~$1.20 | ~$8.91 |

So a routine 90-minute video is **$4–9**, not $118–197. Two consequences the numbers
force: the frame cap bounds only the *image* line item (~$0.20 of the Haiku figure) —
the money is in persona fan-out over the transcript and state descriptions, so that is
where the budget lever belongs; and `confirm_threshold_usd` must clear the routine
long-video cost **on the default path**, or every 90-minute run blocks on a prompt.
Since §13.4 makes batching opt-in, the default 90-minute cost is the unbatched $8.81
(and up to $8.91 with both repair rounds), not the batched $4.41 — so the threshold is
**$10.00**, not $5.00 and certainly not $2.00.

Sonnet 5 also carries an introductory $2/$10 per MTok through 2026-08-31, so estimates
built on the $3/$15 list price are conservative until then.

- **Estimate before spending.** `--dry-run` prints projected frames, tokens and dollars
  per model, and exits. Above `confirm_threshold_usd` an interactive run requires
  confirmation.
- The check is **pre-call**: "would this call exceed the cap?" A post-hoc check always
  overshoots by at least one call.
- The cap is **global across a directory run**, not per video. Per-video caps let a
  folder of ten videos cost ten times the stated ceiling.
- **On exhaustion, degrade — never `exit 1` mid-video.** Assemble from everything
  cached, emit the Markdown with the shortfall recorded in the Analysis Integrity block,
  and exit non-zero. The draft's fatal abort spent the money and then threw away the
  results.

### 13.6 Disk

- JPEG q≈2 for cache frames; PNG only for exported screenshots (§11).
- Preflight requires `free_bytes > 3 × estimated_frame_bytes` — not a flat 1 GB, which
  passes moments before writing 100 GB.
- Delete the frame cache on success; keep manifests. `--cache-max-gb` (default 20) with
  LRU eviction.

### 13.7 Rate limits and retries

Read `anthropic-ratelimit-*` response headers and self-throttle. Honour `retry-after`.
Full-jitter exponential backoff to a 60 s cap, ≥ 8 attempts.

**Retry exactly these:** 408, 409, 429, all 5xx (including 529), and connection/timeout
errors. **Never retry** any other 4xx — 400, 401, 403, 404 are deterministic. Note that
429 *is* a 4xx, so "retry 429 but never 4xx" is self-contradictory and must not be
written that way.

**Do not stack retries.** The `anthropic` SDK already retries 408/409/429/5xx with
backoff (`max_retries=2` by default). If this pipeline wraps calls in `tenacity`,
construct the client with `max_retries=0`, or the two layers multiply and a sustained
rate limit produces far more attempts than intended.

**Never silently skip work.** The draft's rule — "retry 3 times, then skip frame" —
violates R8.2 outright. Dropped frames are recorded and surfaced in the Analysis
Integrity block; > 2% dropped is a non-zero exit.

### 13.8 No network call at startup

Validate the API key's *format* only. The draft called `models.list()` on every launch
and treated failure as fatal, which breaks `--dry-run`, breaks offline re-runs from
cache, and fails for any user without the second vendor's key. (That call consumes no
tokens — it is not billable — but it is still a network dependency at startup for no
benefit. Let the first real call surface an auth error.)

---

## 14. CLI and configuration

### 14.1 Invocation

```
videoanalyser <path>            # a file, or a directory (defaults to the workspace raw/)
videoanalyser --all             # everything in raw/
```

### 14.2 Flags

| Flag | Meaning |
|---|---|
| `--workspace PATH` | Override `~/Desktop/Video Analysis` |
| `--source TEXT` | Override Part 3 of the folder name (§3.3a) |
| `--resume` | Resume an interrupted video from its last complete stage |
| `--skip-existing` | Skip videos that already have an output file |
| `--reprocess` | Re-run and replace existing output, clearing the screenshot folder |
| `--diarize / --no-diarize` | Force speaker diarization on or off |
| `--budget USD` | Global spend cap for this invocation |
| `--dry-run` | Estimate cost and print the plan; make no API calls |
| `--self-contained` / `--bundle` | Delivery modes (§10.8) |
| `--annotate` | Draw element highlights where a bounding box exists |
| `--clean` / `--clean-all` | Cache management |
| `-v` / `-vv` | Verbosity |

Two flags with the same effect, or one flag with two meanings, are defects — the draft
had `--resume` defined three incompatible ways and `--markdown-only` documented as its
own opposite in the same file.

### 14.3 Configuration

**One file** (`~/.config/videoanalyser/config.yml`), **one schema**, expressed as a
Pydantic model that is the single source of truth for defaults *and* validation. The
draft specified config in three files with two disjoint schemas and duplicate keys
carrying different units — `scene_detection_threshold` was simultaneously 0.15 on a 0–1
scale and 27 on a 0–100 scale.

Precedence: CLI flag > config file > model default. Any config key nothing reads must be
deleted, not left as decoration.

**Every threshold named anywhere in this plan appears below with a literal default.**
A named-but-unvalued threshold is an unimplementable spec — §5.5's two delta floors
alone determine the state count, the screenshot count, and therefore the cost.

```yaml
workspace_root: "~/Desktop/Video Analysis"
sanitize_colon: false            # false = obey R7 literally (see §3.3)
default_source: null             # null = use the A1 provenance enum

visual:
  screen_edge_density_floor: 0.08   # Sobel-magnitude fraction; content_class (§5.2)
  screen_flatness_floor: 0.55       # fraction of pixels in top-16 quantised colours
  descriptor_size: 64            # NxN grayscale, values 0-255
  block_grid: 4                  # 4x4 = 16 tiles for screen content
  block_delta_floor: 6.0         # mean-abs-diff per tile, 0-255 scale; screen content
  anchor_delta_floor: 12.0       # cumulative drift from the group anchor
  phash_distance_floor: 8        # camera footage only; recalibrated per file
  max_state_seconds: 10.0
  change_floor_percentile: 95    # §8.4 calibration
  change_floor_minimum: 4.0      # hard floor under the percentile

audio:
  has_speech_floor: 0.02         # voiced-word fraction of the audio span
  has_music_floor: 0.30          # music-segment fraction of the audio span
  asr_sample_rate: 16000
  analysis_sample_rate: 48000
  silence_margin_db: 10.0        # threshold = measured noise floor + this
  min_silence_seconds: 0.4
  forced_align_residual_max_ms: 150
  voiced_fraction_floor: 0.30    # below this, flag hallucination_suspect

alignment:
  clause_silence_seconds: 0.4
  explanation_min_seconds: 60.0
  explanation_speech_density: 0.6
  lead_seconds: 0.5              # behavioural, reported not gated
  lag_seconds: 1.5
  binding_error_epsilon_ms: 50   # float error absorption only (§8.1)

sync:
  sc1_max_lag_ms: 50
  sc1_peak_prominence_floor: 0.25   # below this, SC1 reports INCONCLUSIVE
  sc2_min_coverage: 0.99
  sc2_max_unexplained_gap_s: 2.0
  sc6_resample_fraction: 0.05

vision:
  vision_frame_cap: 300
  walkthrough_min_state_seconds: 3.0
  walkthrough_top_boundary_fraction: 0.05   # of state boundaries, scored by the
                                            # detector that created them (§10.6)
  max_image_long_edge_px: 1024
  screenshot_long_edge_px: 2560

models:
  describe: "claude-haiku-4-5"
  domain:   "claude-sonnet-5"
  synthesise: "claude-opus-5"

budget:
  max_usd: 25.00                 # global for the invocation, not per video
  confirm_threshold_usd: 10.00   # clears the unbatched 90-min worst case (§13.5),
                                 # so routine long videos do not block on a prompt
  max_dropped_frame_fraction: 0.02

qa:
  max_repair_rounds: 2
  min_words:                     # per-section floors for G1/G11
    general_description: 60
    short_summary: 25
    key_points: 40
    step_why: 15
    screen_description: 40
  min_alt_text_chars: 60

cache:
  root: "~/Library/Caches/VideoAnalyser"
  max_gb: 20
```

### 14.4 Appendix A — the controlled action verbs

Gate `G8` is blocking and asserts that every step's **Action** begins with a verb from
this list. The list must therefore exist in the document, not merely be referred to.

*Software / UI:* `click`, `double-click`, `right-click`, `press`, `type`, `paste`,
`select`, `choose`, `drag`, `drop`, `scroll`, `hover`, `open`, `close`, `switch`,
`navigate`, `upload`, `download`, `save`, `rename`, `delete`, `enable`, `disable`,
`expand`, `collapse`, `search`, `filter`, `copy`, `confirm`, `cancel`.

*Workshop / physical:* `measure`, `mark`, `cut`, `rip`, `crosscut`, `drill`, `sand`,
`plane`, `joint`, `route`, `chisel`, `carve`, `clamp`, `glue`, `screw`, `nail`,
`assemble`, `disassemble`, `stain`, `seal`, `paint`, `finish`, `polish`, `steam`,
`bend`, `dry`, `cure`, `wipe`, `mix`, `heat`, `cool`, `align`, `level`, `test`.

Extending the list is a config change (`qa.extra_action_verbs`), not a code change. A
step whose action does not start with a listed verb fails `G8` — which is the mechanism
that makes "then they configure the settings" unemittable.

---

## 15. Test plan and acceptance

### 15.1 Fixtures are generated, not sourced

`tests/make_fixtures.py` synthesises every fixture with `ffmpeg lavfi` (`testsrc`,
`sine`) plus a TTS voice, so no test depends on the user finding content:

- CFR and **VFR** variants; the VFR fixture is generated with irregular PTS.
- A **B-frame** fixture (catches DTS-vs-PTS regressions).
- A fixture remuxed with `-itsoffset 0.4` (the offset round-trip, §5.3).
- Portrait, 4K, silent, music-only, two-speaker, non-English, 3-second, 2-hour.
- Adversarial titles: containing `/`, `:`, `(`, `)`, `#`, `%`, a newline, 260 characters,
  and NFD-composed accents.

### 15.2 Negative tests — every gate must be proven to fire

`tests/fixtures/broken/` contains a document per gate that must trip it:
`gap_in_timeline.md`, `dangling_image_link.md`, `orphan_screenshot/`,
`transcript_short_by_30s.md`, `vague_step.md`, `quote_not_in_transcript.md`,
`unresolvable_evidence.json`, `generic_alt_text.md`.

Each maps to its gate ID and expected exit code. **Without these, a `return True`
implementation of `qa.py` passes the entire suite** — which is exactly the state the
draft's test plan would have shipped in.

### 15.3 Acceptance criteria

The build is done when:

1. Every fixture in §15.1 processes without error, or fails with a clear diagnosis.
2. Every gate in §12.1 is proven to fire by a test in §15.2.
3. No `SC*` check reports FAIL on any fixture. `INCONCLUSIVE` is an accepted outcome
   exactly where §5.6 defines it (SC1 on speechless or uncorrelated content; SC4/SC5
   have no meaning without a transcript). The PCM `-itsoffset` fixture recovers 0.400 s
   exactly (see §5.3 on why the fixture must not use AAC).
4. The R7 folder name for a known input is byte-identical to the required format, and
   `os.listdir` returns it exactly.
5. Every `![](...)` in every generated document resolves.
6. A 7-minute H.264/HEVC tutorial, without diarization, completes in ≤ 12 minutes
   synchronously and under **$1.50** with default settings — the ceiling clears §13.5's
   worst case of ~$1.20 with both repair rounds fired, since `max_repair_rounds: 2` *is*
   the default. Software-decoded VP9/AV1 and `--diarize` runs are out of scope for this
   criterion (§13.2).
7. Every row of §17 is satisfied.
8. **Structural determinism:** the same video processed twice yields the same section
   set, step count, screenshot count and timestamps (±1 frame). Prose will differ — that
   is inherent to LLM synthesis, so byte-determinism must not be claimed as the draft did.

---

## 16. Build order

| M | Milestone | Done when |
|---|---|---|
| M1 | Skeleton: CLI, config, workspace, cache, stage ledger | `--dry-run` works on a real file |
| M2 | `S1`–`S3`: probe, dense scan, states | State list partitions the timeline on all fixtures |
| M3 | `S5`–`S7`: transcript, alignment, audio features | Word timestamps verified; `SC4` passes |
| M4 | `S8`, `S10`, `S11`: representatives, alignment, **sync proofs** | **`SC1`–`SC6` pass. Do not proceed until they do.** |
| M5 | `S9a`, `S9b`, `S12`: personas, prompts, grounding, evidence resolution | Every prompt file exists; `G9` passes |
| M6 | `S13`: assembly, screenshots, delivery modes | A real tutorial produces a usable document |
| M7 | `S13b`, `S14`: output verification, gates + negative tests | Every gate proven to fire |
| M8 | Cost/perf: batching, caching, budget, resume | 90-minute video within §13.2 and §13.5 |

M4 is the gate. Everything after it is worthless if the timeline is wrong, and a wrong
timeline is invisible in the output — it produces a confident, plausible, incorrect
document rather than an error.

---

## 17. Requirements traceability

| R | Requirement | Satisfied by |
|---|---|---|
| R0.1–0.4 | Local Python CLI on M1/16 GB | §2, §14 |
| R1.1–1.2 | All formats and resolutions | §5.2 |
| R1.3 | Five source archetypes | §1.1, §9.1 |
| R2.1 | Frame-by-frame analysis | §5.4, §5.5 |
| R2.2 | Audio analysis | §6 |
| R2.3 | Exact screen↔speech pairing | §8.2, §8.3, §10.3 |
| R2.4 | Perfect sync, losing nothing | §5.3, §5.6, §13.7 |
| R2.5 | Frame-accurate association | §8.1, §8.2 |
| R3.1–3.9 | Multi-persona analysis | §9.2 |
| R4.1 | All collected details | §10.1 |
| R4.2 | All timestamps | §10.3, §9.6 |
| R4.3 | Meeting main points | §9.2 (C8), §9.1 routing |
| R4.4 | General description | §10.1 (E1) |
| R4.5 | Short summary | §10.1 (E1) |
| R4.6 | Verbatim transcript | §9.3, §10.9, G6, G7, G13 |
| R4.7 | Screenshot where a claim needs visual backup | `Claim.visual_backup` (§4.3), gate G14 |
| R4.8 | Screenshot of every step / explained topic | §10.4, §10.5, §10.6 |
| R4.9 | Full UI step detail | §10.4, G8 |
| R4.10 | Tool, action, why, how | §10.4, §9.2 (C2, C3) |
| R4.11 | Style, mood, filters, music | §9.2 (C6, C7), §6.6 |
| R4.12 | Standalone tutorial | §10.4, §10.7, G5 |
| R5.1–5.2 | Tools and materials list | §9.2 (C2) |
| R5.3–5.4 | Real durations, time estimate | §9.5 `domain_estimate`, §9.7 |
| R5.5 | Process time vs screen time | §9.7 elision markers |
| R5.6 | Transferability | §9.2 (C5) |
| R6 | `Video Analysis` + three folders | §3.1 |
| R7 | Screenshot folder naming | §3.2, §3.3, §3.3a, G12 |
| R8.1–8.3 | Maximum detail, lose nothing, all perspectives | §9.2 roster, persona D3, §10.2, §13.7, SC2 |
| R8.4 | No garbage, no broken links | G2, G3, G4, G11, D5 |
| R9.1–9.3 | Process requirements — composition, grilling, notification | Discharged by `docs/GRILL_LOG.md`; not build items |
| R9.4 | Executable with no prior context | This document; §9.9; §14.3; §14.4 |

