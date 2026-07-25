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
