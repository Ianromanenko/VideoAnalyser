# Grill Log

Adversarial review of `docs/PLAN.md`, per requirement R9.2 (up to 20 interactions, early stop
once the plan holds up).

**Note:** the `grillme` skill is not installed in this environment (checked: no user skill,
no project skill, no plugin, nothing on disk). The loop below is an explicit implementation
of the same idea — independent adversarial reviewers whose only job is to find gaps,
mismatches and problems, followed by a revision pass. Every round is logged here with the
findings and what was done about each one, so the reasoning is auditable and the user can
re-run a real `grillme` pass later against the same document.

## Rules of the loop

1. Each round spawns independent reviewers with **distinct lenses**. Reviewers are told to
   find problems, not to praise.
2. Every finding is recorded below with a verdict: **FIXED**, **REJECTED** (with reason), or
   **DEFERRED** (with reason).
3. A finding is only "fixed" when `docs/PLAN.md` actually changed.
4. The loop stops early when a round produces no finding that changes the plan's substance.
5. Hard ceiling: 20 rounds.

## Round index

| Round | Lenses | Findings | Fixed | Rejected | Status |
|------|--------|----------|-------|----------|--------|
| 0 | Direct review of composed draft (dependency/API facts) | 12 | 12 | 0 | closed |
| 1 | 5 independent lenses on the full draft set | 126 | 126 | 0 | closed |
| 2 | 2 lenses on the finished `PLAN.md`, plus a scripted self-audit | 50 | 50 | 0 | closed |
| 3 | Regression check on the patched plan | _in progress_ | | | |

---

## Round 0 — direct review of the composed draft

Reviewed `_draft_s1_architecture.md` directly and verified every third-party dependency
and model identifier against live sources (PyPI JSON API, Anthropic API reference).
The small model produced a plausible-looking dependency table that does not survive
contact with reality.

| # | Finding | Verdict |
|---|---------|---------|
| 0.1 | **`imagetext-recognition` is not a real package.** PyPI returns 404. A `pip install -r requirements.txt` fails on the very first run. | FIXED — replaced with `ocrmac` 1.0.1 (Apple Vision OCR, native Apple Silicon) with `rapidocr-onnxruntime` 1.4.4 as the cross-platform fallback. |
| 0.2 | **Frame analysis routed to OpenAI `gpt-4-vision`.** The user's stated downstream consumer is Claude, and this silently introduces a second vendor, a second API key, and a second billing account for no benefit. | FIXED — all vision and synthesis calls go to the Anthropic API. |
| 0.3 | **`anthropic>=0.7.0`** — that SDK generation predates every API this plan uses (`output_config`, adaptive thinking, structured outputs). | FIXED — pinned `anthropic>=0.120.0` (current). |
| 0.4 | **"Claude 3.5 Sonnet" named as the analysis model.** Retired on 2025-10-28; requests 404. | FIXED — `claude-opus-5`, `claude-sonnet-5`, `claude-haiku-4-5` with an explicit routing table. |
| 0.5 | **Vanilla `openai/whisper` chosen for transcription.** Pure-PyTorch; slowest option on an M1 and no word-level timestamps by default — which R2.3/R2.4 depend on absolutely. | FIXED — `mlx-whisper` 0.4.3 (Apple-Silicon GPU via MLX) as the default, `faster-whisper` 1.2.1 as fallback, `whisperx` 3.8.6 for forced word-level alignment. |
| 0.6 | **No speaker diarization dependency** despite R1.3.b (video calls) and R4.6 (speaker-labelled transcript). | FIXED — `pyannote.audio` 4.0.7, with its gated-model/HF-token requirement documented as a prerequisite rather than discovered at runtime. |
| 0.7 | **Scene detection hand-rolled from OpenCV histograms.** Reinvents a solved problem and will misfire on the fades and cross-dissolves common in the user's "inspiration video" category. | FIXED — `scenedetect` 0.7.1 (`ContentDetector` + `AdaptiveDetector`). |
| 0.8 | **No perceptual-hash dependency** despite the plan's own claim to deduplicate near-identical frames. | FIXED — `ImageHash` 4.3.2. |
| 0.9 | **`hashlib` and `json` listed as pip dependencies.** Both are stdlib; listing them in `requirements.txt` is at best noise and at worst an install error. | FIXED — removed. |
| 0.10 | **Bootstrap creates `raw/`, `outputs/`, `screenshots/` at the repo root** — but R6 requires them inside a folder named `Video Analysis`, and R0.3/R6 place that on the user's Mac, not inside the source checkout. Straight spec violation. | FIXED — `~/Desktop/Video Analysis/{raw,outputs,screenshots}` is the canonical location, overridable, with the code checkout kept separate. |
| 0.11 | **No `av`/PyAV dependency** for frame-accurate presentation-timestamp reads; the draft assumes it can compute timestamps from a nominal frame rate. That assumption is false for the user's single most important input class (macOS/iOS screen recordings are variable-frame-rate). | FIXED — `av` 18.0.0, and all timestamps read from container PTS. |
| 0.12 | **Python 3.11 justified as "current"** with reasoning that is stale as of 2026. Harmless, but the pin needs a real constraint. | FIXED — pin driven by an actual dependency ceiling (`whisperx` requires `>=3.10,<3.14`); plan targets 3.12. |

**Round 0 conclusion:** the composed draft is usable as raw material but its dependency
and model layers were unreliable. Every version number in the final plan is now one
resolved against a live index rather than recalled.

---

## Round 1 — five independent adversarial lenses

Five reviewers, each given a distinct lens and instructed to find problems only. None
saw another's output. Combined: **126 findings**, all of which informed `PLAN.md`.

| Lens | Findings | What it was asked to attack |
|---|---|---|
| Synchronization correctness | 29 | VFR, PTS/DTS, drift, alignment, the R2.4 claim |
| Requirements compliance | 20 | Every R-number, checked one by one |
| M1 feasibility and cost | 19 | 16 GB unified memory, runtime, dollars, rate limits |
| Executability (R9.4) | 53 | Contradictions, undefined interfaces, silent-failure surfaces |
| Output spec and QA | 29 | The delivered artifact and whether the gates are real |

### The findings that mattered most

Ordered by how badly they would have hurt, not by which section they came from.

1. **The claimed cost was wrong by two orders of magnitude.** The draft stated
   "~$0.65 per hour of video". Under its own literal sampling rule (1 fps to a vision
   model at up to 4,784 tokens per image), a 90-minute video costs **$118–197**, and
   **$235–400** once persona fan-out and remediation rounds are counted. Fixed by a
   single frame budget, 1024 px downscaling (6× token reduction), model routing by task,
   prompt caching, cache priming before fan-out, and the Batch API — together bringing
   the 90-minute case to roughly **$12–18**. (`PLAN.md` §13.)

2. **The audio offset was computed, stored, and never applied.** The draft derived
   `audio_start − video_start`, wrote it into a manifest, then extracted audio with a
   command that normalises start to zero — and asserted downstream that ASR timestamps
   were already on the master timeline. Every word, silence, speaker turn and audio
   feature in the entire output would have been uniformly shifted. Fixed by a single
   `to_master()` conversion at one boundary, plus an `-itsoffset` round-trip test.

3. **Frames were matched to timestamps by an unkeyed positional join.** One `ffmpeg`
   run wrote `frame_%06d.png`; a separate `ffprobe` run listed I-frame times; the two
   were joined *by position*. A single dropped or duplicated frame shifts every
   subsequent timestamp by a full keyframe interval, silently, for the rest of the file
   — and no check in the draft could detect it. Fixed by a single decode pass reading
   PTS directly from the container.

4. **None of the six "synchronization proofs" could detect a constant A/V offset.**
   Inject a uniform +400 ms shift and all six still pass — they were self-consistency
   checks. The verification section nonetheless concluded "R2.4 satisfied". Replaced
   with a cross-modal correlation check with a 50 ms bound, interval-union coverage, and
   a forced-alignment residual gate.

5. **Every screenshot link in every output file would have been broken.** The required
   R7 folder name contains spaces, pipe characters and a colon; the draft emitted
   `![Step 3](../screenshots/Title | date | original: source/img.png)`. Unescaped spaces
   terminate the target, pipes break enclosing tables, and a bare token before a colon
   parses as a URI scheme. Fixed with per-segment percent-encoding inside angle-bracket
   destinations, plus a gate that re-resolves every link on disk.

6. **The verbatim transcript (R4.6) was routed through mechanisms that guarantee
   paraphrase** — a ~50 KB assembler context budget against a 75–90 KB transcript,
   explicit pre-chunking so personas "receive only relevant segments", and a transcript
   stage modelled as a generative persona that could "re-listen to unclear segments"
   (which an LLM cannot do). Fixed by emitting the transcript through deterministic
   templating, exempt from every context budget, with a byte-equality gate on quotes.

7. **The anti-hallucination mechanism checked only that an evidence string was
   *present*, never that it *resolved*.** A model could cite a frame that was never
   extracted and pass validation. Fixed with `resolve_evidence()` plus sampled
   re-verification.

8. **R5.3/R5.4 were unanswerable by construction.** The only persona able to supply
   real-world process times was forbidden from doing so, and the conflict rules then
   discarded domain knowledge in favour of the transcript. The user's three canonical
   cases — stain drying half a day, glue curing two days, steam bending 30 minutes —
   are exactly the quantities a video shows without stating. Fixed with a third claim
   class, `domain_estimate`, rendered alongside the stated value, never overwriting it.

9. **Nothing detected that on-screen time had been compressed (R5.5).** A field for the
   screen-time↔real-time ratio existed; no code computed it. A 30× timelapse would be
   reported at its on-screen duration. Fixed with explicit elision detection.

10. **The QA gates were decorative.** Every gate ended in passive voice with no actor,
    no retry bound and no exit code, while the output template hardcoded
    `qa_passed: true`. The spell-check gate would have forced an agent to "correct" the
    verbatim transcript, destroying R4.6. Fixed with executable gates, severities,
    bounded repair, and quarantine on failure.

11. **Steps were free prose,** so "then they configure the settings" was emittable —
    losing precisely the detail R4.9 and R4.10 ask for. Fixed with required key/value
    fields and a controlled action verb, enforced by a gate.

12. **`-vf "rotate=90"`** — that filter takes **radians**, so this rotates by ~5157°
    and crops; it also double-rotates video ffmpeg already auto-oriented. Every iPhone
    portrait video, an explicitly named input class, would have been mangled. Fixed by
    doing nothing and letting ffmpeg autorotate.

13. **A timestamp regex of `MM:SS.mmm`** could not represent any video over 99 min 59 s
    — every claim in a 2-hour video would fail validation and trigger the retry loop
    after the money was spent.

14. **Perceptual hashing was the wrong descriptor for screen recordings.** pHash is
    designed to be invariant to small high-frequency change; on a UI, that change *is*
    the content. Typing into a field, opening a dropdown, ticking a checkbox all
    deduplicate away — defeating R4.9 directly.

15. **Nearest-neighbour frame binding produced confidently wrong screenshots.** Because
    representatives mark the *start* of states lasting up to 30 s, a word past a state's
    midpoint snaps to the *next* state — illustrating "fill in your email" with the
    post-submit confirmation page. Fixed by modelling states as a partition and binding
    by containment, making the error zero by construction.

16. **Requirements with no owner at all:** R4.4 (general description), R4.5 (short
    summary), R5.4 (project time estimate), and the selection of which frames become
    screenshots. Each now has a named owner.

17. **Silent-failure surfaces** — stale cache never invalidated (the hash field did not
    exist in the schema it was compared against), `/tmp` collisions pairing one video's
    frames with another's audio, every video overwriting the previous output file,
    action-word matching that never fired because ASR returns words with leading spaces
    and punctuation, and sound-event detection that aggregated away its time axis and
    then reported per-event timestamps no code produced.

### Findings deliberately not adopted

- **"Delete diarization entirely."** Rejected — R1.3.b (video calls) and R4.6
  (speaker-labelled transcript) need it. Made optional and off by default instead, since
  it is the slowest local stage and useless for solo screen recordings.
- **"Drop the colon from the R7 folder name."** Rejected — R7 is a literal, [HARD]
  requirement and the user supplied the exact string. The Finder rendering is documented
  and a config switch is offered, but the default obeys the spec.

---

## Round 2 — grilling the finished plan

Two lenses, this time reading `PLAN.md` rather than the drafts: one requirements and
executability audit, one technical fact-check with shell and network access that
verified claims by measurement rather than by reading.

**46 findings.** A self-audit of internal cross-references, run in parallel, found four
more that I had introduced while writing.

### From the plan audit (16)

The three that would have broken the build outright:

1. **The dense-scan command could not produce timestamps at all.** It piped
   `-f rawvideo` to PyAV. rawvideo is a headerless byte stream — no container, no
   `time_base`, no PTS. The fact-checker measured it: a 5-second, 30 fps source came
   back described as 25 fps and 6 seconds, with fabricated uniformly-spaced timestamps.
   That is the *same* silent whole-file corruption the section was written to prevent,
   and it would have passed the monotonicity and partition checks because a fabricated
   series is perfectly self-consistent. Replaced with in-process PyAV decoding.

2. **The stage table contained a real dependency cycle** — `S9_NAMING` consumed personas
   A1/A2, while the persona stage consumed "everything above", including `S9`. The same
   defect class the plan congratulates itself for removing elsewhere. Split into
   `S9a_FRAMING` → `S9b_NAMING` → `S12_PERSONAS` → `S13` → `S13b_VERIFY_OUT`.

3. **The frame budget was arithmetically incapable of its own job.** States are capped
   at 10 s, so a timeline always holds ≥ `duration/10` states; the budget granted
   `duration/15` frames — strictly fewer, for every video length. A 90-minute video
   could never describe more than 46% of its states while the integrity block asserted
   "0 unanalysed". Replaced with a cap plus explicit state *merging*, so coverage stays
   100% by construction.

Also: the plan promised "every threshold named in this plan appears here with its
default" and then omitted the two delta floors that govern state segmentation — and the
controlled verb list that blocking gate `G8` depends on. Both are now written out
(§14.3, §14.4). Cache guidance quoted only the read discount while mandating the TTL
with the *higher write* multiplier, and asserted the wrong `usage` field (the priming
call writes the cache; asserting a read on it fails on every correct run).

### From the technical fact-check (30)

This pass ran commands rather than trusting recall, and overturned several claims I had
stated confidently:

| Claim in the plan | Measured reality |
|---|---|
| "Modern ffmpeg auto-applies the display matrix" → rotation needs no handling | True for the ffmpeg **CLI**, **false for PyAV**. Since the dense scan is PyAV and export is the CLI, rotated iPhone video would produce a descriptor series transposed relative to the screenshots — on two of the five mandatory archetypes, failing `SC6` with no diagnosis. |
| "macOS normalises filenames toward NFD" | **APFS preserves** normalisation and is normalisation-*insensitive*; NFC and NFD spellings collide as the same directory. HFS+ forced NFD. The duplicate-folder failure described was impossible. |
| "The macOS Desktop is iCloud-synced by default" | Opt-in, not default. The conclusion (keep the cache in `~/Library/Caches`) survives; the stated reason did not. |
| "APFS limits a component to 255 **bytes**" | 255 UTF-8 **characters** in practice. Truncating by bytes cuts non-ASCII titles up to 4× early. |
| "ffprobe emits `N/A` for `start_time`, `duration`, `nb_frames`" | With the JSON writer the plan mandates, those keys are **omitted**; `N/A` appears only in the flat writer. `start_time` is present and numeric. The error to absorb is `KeyError`, not `ValueError`. |
| "`librosa.beat.tempo` removed in ≥0.10.1" | Still present in the pinned 0.11.0, emitting a `FutureWarning`; moved in 0.10.0, removal slated for 1.0. |
| "whisper large-v3 ≈ 1.5 GB" | ~3.1 GB at fp16 — 1,550 M parameters. The disk pre-check would have under-reserved by 2×. |
| Cross-check scene boundaries with `scenedetect` over the descriptor series | `ContentDetector` calls `cvtColor(BGR2HSV)` and requires 3-channel input; it raises on a grayscale descriptor. Not implementable without a second decode. |
| "`-itsoffset 0.4` … the pipeline must report 0.4 s" | With AAC the offset snaps to the 1024-sample packet grid and lands at **0.376 s**. The mandatory test would fail by 24 ms through no fault of the pipeline. Fixture must use PCM. |
| `pyannote/speaker-diarization-3.1` | Legacy for the pinned 4.0.7, whose default is `speaker-diarization-community-1` — specifically better at speaker counting, the exact failure mode §6.5 guards. |
| "Accept Python 3.11–3.13" | `rapidocr-onnxruntime` caps at `<3.13`; the interpreter check would pass and `pip install` would then fail. |
| "retry 429/529 only; never retry 4xx" | 429 **is** 4xx. Self-contradictory as written. |
| "~4,784 tokens per image → 6× saving" | That ceiling belongs to the high-resolution tier. `claude-haiku-4-5` — which carries the bulk of the frames — caps near 1,600, so the real saving on the dominant route is ~2×. |

Two further structural points worth recording: batching was mandated while the
acceptance criteria demanded a 12-minute wall clock, but batch turnaround is only
*guaranteed* within 24 hours — batching is now opt-in. And §13 spent its length
refuting a wrong cost figure without ever stating a corrected one; it now gives the
worked number (**~$4–9 for a 90-minute video**, against the draft's $118–197) and the
confirmation threshold was raised so routine long videos do not block on a prompt.

### From my own cross-reference audit (4)

Found by scripting the document rather than reading it:

- **Identifier collision:** the sync checks were `C1`–`C6` and the domain personas were
  `C1`–`C9`. "C1–C6 pass" in the acceptance criteria and milestones was ambiguous.
  Renamed the sync checks `SC1`–`SC6`.
- Two `§n.n` cross-references pointed at sections that do not exist.
- One sentence implied a persona in the roster that is not in it.

### Confirmed correct

Worth recording, because a review that only reports errors gives no signal about what
was checked: every package and version in §2.4 exists as pinned; `mlx-whisper` really is
Metal-backed and `faster-whisper`/CTranslate2 really is CPU-only on Apple Silicon;
`ocrmac` really wraps Apple's Vision framework; `word.probability` is the correct
attribute; the `rotate` filter really takes radians; `best_effort_timestamp` really is
distinct from `pkt_dts_time`; the colon really is legal on APFS and really renders as
`/` in Finder; the three model IDs and all three prices are current; the Batch API is
50% off with 29-day retrieval; cache reads really are ~0.1× and writes 1.25×/2×;
concurrent identical-prefix requests really do all miss; and 64×64 grayscale really is
exactly 4096 bytes per frame.
