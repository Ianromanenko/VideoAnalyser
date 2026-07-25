# Grill Log

Adversarial review of `PLAN.md`, per requirement R9.2 (up to 20 interactions, early stop
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
3. A finding is only "fixed" when `PLAN.md` actually changed.
4. The loop stops early when a round produces no finding that changes the plan's substance.
5. Hard ceiling: 20 rounds.

## Round index

| Round | Lenses | Findings | Fixed | Rejected | Status |
|------|--------|----------|-------|----------|--------|
| 0 | Direct review of composed draft (dependency/API facts) | 12 | 12 | 0 | closed |
| 1 | 5 independent lenses on the full draft set | 126 | 126 | 0 | closed |
| 2 | 2 lenses on the finished `PLAN.md`, plus a scripted self-audit | 50 | 50 | 0 | closed |
| 3 | Regression check on the patched plan | 10 | 10 | 0 | closed |
| 4 | Convergence check | 7 | 7 | 0 | closed |
| 5 | Convergence re-check | 5 | 5 | 0 | closed |
| 6 | Convergence re-check | 3 | 3 | 0 | closed |
| 7 | Convergence re-check | 2 | 2 | 0 | closed |
| 8 | Final sign-off check | _in progress_ | | | |

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
   single frame budget, 1024 px downscaling, model routing by task, prompt caching,
   cache priming before fan-out, and opt-in batching. (`PLAN.md` §13.) The corrected
   figure was worked out properly in round 2 — see that section for the final numbers;
   the estimate recorded here at the time was itself too optimistic.

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


---

## Round 3 — regression check on the patched plan

Round 2 produced roughly thirty edits in one pass. Large patches are where new defects
come from, so round 3 was scoped narrowly: **blocking findings only, capped at ten**,
with an explicit instruction that an honest short list beats a padded one.

It returned exactly ten, all real and all introduced or left open by the round-2 patch.
The pattern is worth recording: every one is a *seam* — two sections that were each
correct in isolation and disagreed with one another.

1. **Two sections named different diarization pipelines** (§2.5 said `community-1`,
   §6.5 still said `3.1`). Each is separately gated on Hugging Face, so the bootstrap
   would have had the user accept one model and the pipeline would then load the other,
   fail the gate, and take the documented degrade path to a single unnamed speaker —
   losing R1.3.b speaker attribution with no diagnosis.

2. **The state-merging rule I added in round 2 broke the invariant it was meant to
   preserve.** It merged "by descriptor similarity", but similarity is not adjacency:
   merging states 3 and 47 produces a span overlapping every state between them, so
   `SC3`'s partition assertion fails on any video over the cap — which is every video
   above ~50 minutes. Now restricted to temporally adjacent runs, with `S8` named as the
   owner and an explicit statement of which stages read the raw list and which read the
   merged one.

3. **Gate `G6` would have quarantined every silent and music-only video** — two of the
   five mandatory archetypes. A silent clip has no transcript span at all; a music-led
   clip has near-zero speech *and* near-zero detected silence. `SC2` already had the
   carve-out; `G6` did not.

4. **The confirmation threshold was justified against the wrong number.** The text
   argued it must clear the routine 90-minute cost, then set it to $5.00 — which clears
   only the *batched* $4.41, while batching is opt-in, so the default path is $8.81.
   Every 90-minute run would have blocked on exactly the prompt the paragraph existed to
   prevent. Raised to $10.00.

5. **The runtime table was labelled "API (batched)" while a later section asserted the
   table assumes synchronous calls** — and the 12-minute acceptance criterion sits
   directly on that row.

6. **Enabling the documented `sanitize_colon` option guaranteed a blocking failure**,
   because `G12`'s regex hard-required `original: `. A supported configuration would
   have quarantined every run.

7. **"Classified as explanation" gated a blocking check and was never defined** — no
   owner, no signal, no threshold, no field to carry it.

8. **Transition screenshots had nowhere to live.** `§10.6` mandated one per significant
   state transition, but no section hosted them, so every one exported would have been a
   blocking `G3` orphan.

9. **`G8`'s required-field set ignored that software and physical steps have disjoint
   mandatory fields** — a software step has no `tool`, a workshop step no `application`.
   Read literally the gate failed every step; read loosely it enforced nothing.

10. **The automatic diarization trigger read a persona output produced by a later
    stage** — the same cycle class the round-2 stage split was supposed to eliminate.
    Fixed by moving `S9a_FRAMING` ahead of the audio stages and stating that
    `A2.multi_speaker` is inferred from probe and frame evidence, never from
    `speakers.json`.

The reviewer also independently re-verified the Anthropic API claims — model IDs,
all three prices, the introductory Sonnet rate, the 4096-token Haiku cache minimum, the
1.25×/2× write and 0.1× read multipliers, the 50% batch discount, 29-day retention, and
the per-image token ceilings — and found no errors.

**Trend:** 126 findings → 46 → 10. Each round's findings are narrower and more local
than the last, which is what convergence looks like.


---

## Round 4 — convergence check

Scoped to blocking findings only, capped at eight, with an explicit instruction that
"no blocking issues remain" was an acceptable and valuable answer. It returned **seven**
and a verdict of **NOT CONVERGED** — so the instruction did not simply license a rubber
stamp.

The reviewer confirmed the round-3 patch held where it was applied: `§4.2` is acyclic,
the adjacent-only merge does preserve `SC3`'s partition invariant, `G6` and `SC2` now
agree on the explained-coverage set, and the cost, threshold and acceptance numbers are
mutually consistent. But moving `S9a_FRAMING` earlier — the round-3 fix for the
diarization cycle — created a new ownership defect, and two blocking gates still had no
computable predicate.

1. **The `S9a` move fixed one attribute and left two behind.** Round 3 correctly ruled
   that `A2.multi_speaker` must be inferred from probe and frame evidence rather than
   from `speakers.json`. But `has_speech` and `has_music` are also `A2` attributes, and
   they are properties of the *audio*, produced by `S5`/`S7` — which now run after
   `S9a`. A music-only Instagram clip has an audio stream, so a frame-derived
   `has_speech` reads true, `G6` fires, and the clip quarantines: precisely the failure
   round 3's `G6` carve-out was written to prevent. Now sourced from `S5`/`S7` with
   floors in §14.3, and the attribute table states which stage owns each.

2. **"Significant state transition" was never defined**, yet it triggers a conditional
   section that `G1` blocks on and determines the whole exported-screenshot set. The two
   natural readings differ by two orders of magnitude — reuse the segmentation floor and
   every non-step boundary qualifies; reuse the change-detection percentile and almost
   none do. Now a named threshold pair.

3. **`G14` was a blocking gate with no producer.** It requires every `needs_visual`
   claim to resolve to an *exported* screenshot, but the export set was closed to steps
   and walkthrough transitions. A talking-head or music-inspiration video with no step
   manifest has style claims that are `needs_visual` by definition and nothing to point
   at. Added the export rule that makes R4.7 real rather than merely asserted in §17.

4. **`S9a`'s "a few frames" had no producer and no rotation rule.** `S3` streams its
   descriptors and never stores them; `S8` runs much later. Worse, §5.2 establishes that
   PyAV does not apply the display matrix — so an un-rotated portrait iPhone capture
   would be judged landscape, `A1` would mis-classify provenance, and §3.3a would bake
   that error into the R7 folder name and every link in the document. `S1` now produces
   the framing frames explicitly, rotation applied.

5. **`S10` and `S11` were missing their `S7` edge.** Silence, music and non-speech
   intervals live in `audio_features.json`, which neither stage listed as an input —
   making `SC2` uncomputable as specified. The subtler consequence is caching: §4.4
   skips a stage when its input hash matches, so changing the silence threshold would
   re-run `S7` and leave `S10` cached, shipping boundaries computed under the old value.

6. **The merge key was data the plan explicitly never persists.** §13.1 merges "the
   adjacent pair with the smallest descriptor distance", but `VisualState` had no
   descriptor field and §5.4 says the series is never stored — and `S8` is independently
   resumable, so it may start with nothing in memory. Two engineers would have invented
   two different keys and produced different screenshot sets from the same input.
   `descriptor` is now a persisted field (~2 MB for a 90-minute video).

7. **Two sections disagreed about `sanitize_colon`.** §3.3a said the `original:` prefix
   is *never* sanitised; §3.3 said the flag rewrites it. Followed literally the namer
   writes one form while `G12` demands the other — every run under a supported
   configuration quarantines.

**Trend:** 126 → 46 → 10 → 7. Still converging, and the findings are now uniformly
local edits rather than structural rework — but four rounds in, each patch is still
producing a smaller crop of seam defects, which is the argument for one more pass rather
than stopping here.


---

## Round 5 — convergence re-check

Five blocking findings, verdict **NOT CONVERGED** again. The reviewer explicitly cleared
three of the round-4 fixes (attribute ownership, the stage graph's acyclicity, and
`sanitize_colon` agreement in both settings), which is useful signal: those patches held.
The five that did not:

1. **"Significant transition" compared three incommensurable scales.** The definition
   added in round 4 ranked boundaries by §8.4's `change_floor_percentile` — a percentile
   over *whole-frame* inter-frame distances — while boundaries are actually created by
   either a *per-tile* MAD (screen content) or a *pHash Hamming distance* (camera
   footage). Comparing Hamming bits to a MAD distribution is meaningless; comparing one
   tile's score to a whole-frame distribution fails in both directions at once. The two
   readings differ by ~20× in export count. Now ranked against other boundaries using
   the score the boundary's own detector produced.

2. **`G6` still carried a "transcript span" term** — the exact metric `SC2` was rewritten
   to reject. Any video with more than a second of non-speech at the head or tail (music
   intro, title card, silent outro) would fail it *while the union clause read 100%*.
   Deleted; `G6` now mirrors `SC2` literally — same set, same 99%, same 2 s gap rule.

3. **`Claim.visual_backup` was required at a point where no screenshot id can exist.**
   Personas emit in `S12`; screenshots are named in `S13`, and §3.3c fixes the filename
   width from a final count that itself depends on which claims are `needs_visual`. So
   the id was not merely unknown at emission — it was undetermined. Validation would have
   dropped every style claim on talking-head and music-inspiration content, deleting
   R4.7's only mechanism while `G14` passed vacuously over an empty set. Personas now
   emit `needs_visual` + `backing_state`; `S13` backfills `visual_backup` after freezing
   the export set.

4. **Two sections disagreed on what `frames.parquet` contains** — §4.2 said the dense
   scan produces it, §5.4 said the descriptor series is "never stored". Segmentation
   needs the full series; the merge key needs a persisted descriptor. Resolved by
   **fusing the scan and segmentation into one stage**, which removes the artifact
   ambiguity rather than papering over it: descriptors live in a rolling buffer, a
   compact ~140-byte-per-frame feature table persists for resume, and only *states* keep
   a full descriptor (~2 MB per video). `VisualState.descriptor` now has exactly one
   writer.

5. **Round 4 added a third decode site without adding it to the rotation rule.** §5.2 is
   written per decode site precisely because PyAV and the ffmpeg CLI disagree about the
   display matrix. The new framing-frame extraction was given a positive instruction to
   apply rotation while going through the CLI, which autorotates — so it would rotate
   twice, on the only frames `A1` and `A2` ever see, with `SC6` sampling representatives
   rather than framing frames and therefore unable to catch it.

**Trend:** 126 → 46 → 10 → 7 → 5.


---

## Round 6 — convergence re-check

Three blocking findings, verdict **NOT CONVERGED**. The reviewer cleared the S3/S4
fusion, the descriptor story, `G6`'s rewrite and `G14`'s validation point — and
explicitly listed four things it had decided *not* to report as non-blocking, with
reasons. That restraint is what makes the three it did report worth acting on.

1. **The screen-vs-camera content class had no producer `S3` could read.** §5.5 said the
   descriptor choice is "decided at probe time", but nothing in §5.2 computed it, and
   the plan's only screen-vs-camera signal — `A2.has_screen_content` — comes from `S9a`,
   which the stage table declares *parallel* with `S3`. The round-5 fusion closed the
   last stage where segmentation could have picked it up later. This is the label that
   selects between per-tile MAD and pHash, and §5.5 states the consequence of choosing
   wrong in its own words: on a UI, pHash deduplicates the change away and "directly
   defeats R4.9". `S1` now computes `content_class` from the framing frames, with an
   explicit asymmetry — `mixed` takes the screen branch, because guessing camera on
   screen content is unrecoverable while the reverse only over-segments, which the
   anchor rule already bounds. R1.3.b, a call with camera tiles *and* a shared screen,
   is exactly the `mixed` case.

2. **The rotation rule claimed to be exhaustive and missed a fourth decode site.** `SC6`
   re-extracts frames inside `S11`, and its decoder was unspecified — the natural
   implementation reuses `S3`'s PyAV helper, where "apply nothing" is the wrong rule.
   `SC6` is the very check §5.2 nominates as the one that would catch an orientation
   mismatch, so getting it wrong would hard-fail every rotated capture at `M4`, a gate
   the build is told not to proceed past, while pointing the engineer at the wrong
   culprit.

3. **`Claim.backing_state` was written and never read.** §10.6 instead invoked "the
   claim's dominant visual state (§8.2)" — but §8.2 defines containment binding for
   *words*, and "dominant state" is defined in §8.3 for *clauses*. Neither is defined for
   a claim. So the rule that must produce a screenshot for every `needs_visual` claim
   dereferenced an undefined quantity while the field built for it sat dead, and three
   plausible implementations would disagree about which moment backs the claim — a
   silent R4.7 defect that `G14` cannot catch, because `G14` only checks that *some*
   export resolves.

**Trend:** 126 → 46 → 10 → 7 → 5 → 3.

---

## Round 7 — convergence re-check

Two blocking findings, both text-level contradictions, with the reviewer stating it
would sign off once they were applied. It also listed five things it had considered and
deliberately declined to report, with reasons — the restraint that makes the remaining
two credible.

1. **The stage table ordered the exact double-rotation §5.2 forbids.** Round 6 added
   `SC6` to the rotation enumeration and got every site right — but `S1`'s row in §4.2
   still read "rotation applied per `probe.rotation`", while §5.2 lists `S1` as an
   apply-nothing site because it goes through the autorotating ffmpeg CLI. Two normative
   statements, directly contradictory, and the stage table is the implementation
   checklist an engineer works from. At 90°/270° the framing frames come out transposed
   and the aspect assertion turns it into a crash; at 180° the assertion passes and the
   frames are simply upside-down — which §5.2 itself says nothing can catch, because
   those five frames are all `A1` and `A2` ever see.

2. **`Step.screenshot_ids` had the chicken-and-egg problem round 6 fixed for claims.**
   The field is declared `>= 1` and required by blocking gate `G8`, and its declared
   producer is `S10` — which runs *earlier* than the persona stage where §4.3's own
   comment establishes that screenshot ids are "not yet determined". Enforced at
   emission, every step on every tutorial fails validation, burns both repair rounds and
   quarantines; unenforced, two owners write two different id namespaces and `G8` may
   validate the wrong one. `step_id`, `backing_state` and `visual_backup` all carried
   explicit ownership annotations; this one carried none. Fixed with the same pattern:
   `S10` emits `backing_states`, `S13` backfills `screenshot_ids`, and `G8`/`D4` validate
   after the backfill.

**Trend:** 126 → 46 → 10 → 7 → 5 → 3 → 2.
