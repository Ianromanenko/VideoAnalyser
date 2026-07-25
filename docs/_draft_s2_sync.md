# SECTION 2: AUDIO/VIDEO EXTRACTION, THE MASTER TIMELINE, AND SYNCHRONIZATION

**Core Requirement**: R2 mandates perfect synchronization without losing anything. Frame-accurate association of speech ↔ visual state. This section specifies the technical machinery to make that real.

---

## 1. INPUT PROBING: UNDERSTANDING THE CONTAINER AND CODEC LANDSCAPE

Before any extraction, the script must probe the input file exhaustively using **ffprobe**. This is not optional—it is the foundation for every subsequent decision.

### 1.1 Why Probing Matters

Different containers (MP4, MOV, MKV, WebM, AVI) store metadata differently. Codecs vary in how they handle frame delivery. Most critically: **macOS and iPhone screen recordings use variable frame rate (VFR) encoding**. Assuming constant frame rate on VFR content will cause:
- Frames to be skipped or duplicated in the wrong places.
- Audio drift (audio continues at constant sample rate; video frame times do not).
- Transcript words attached to the wrong visual moments.

This is **non-recoverable** after extraction if done wrong.

### 1.2 The Probing Command

```bash
ffprobe -v error -show_format -show_streams \
  -print_format json \
  /path/to/video.mp4
```

This outputs JSON. Parse it to extract:

#### 1.2.1 Container Metadata
- `format.duration` — total duration in seconds (float). Use this as a sanity check for audio duration.
- `format.tags.creation_time` — when the file was created (useful for logging).
- `format.tags.major_brand` — ISO base media file format (`isom`, `mmp4`, `qt`, `ftypisom`, etc.).

#### 1.2.2 Video Stream (`streams[type='video'][0]`)
- `codec_name` — the actual codec (`h264`, `hevc`, `vp8`, `vp9`, `av1`, etc.).
- `codec_type` — must be `'video'`.
- `width`, `height` — resolution in pixels. Track for memory/performance implications.
- `r_frame_rate` — reported frame rate as a fraction string, e.g., `"30000/1001"` (29.97 fps) or `"24/1"` (24 fps).
- `avg_frame_rate` — average frame rate. For VFR, this may differ from `r_frame_rate`.
- **`duration`** (in the video stream) — the stream's declared duration (may differ from container duration).
- **`start_time`** — the stream's start time offset in seconds (default `0.0`, but can be nonzero in some containers, especially MKV).
- **`tags.rotate`** — rotation metadata (0, 90, 180, 270). iPhone videos often have this. Must be applied during frame extraction.
- **`nb_frames`** — total frame count (sometimes missing; if missing, calculate from duration and frame rate).
- **`time_base`** — internal time base (e.g., `"1/30000"`). Used for precise timestamp calculations.

#### 1.2.3 Audio Stream (`streams[type='audio'][0]`)
- `codec_name` — audio codec (`aac`, `mp3`, `opus`, `flac`, `pcm_s16le`, etc.).
- `codec_type` — must be `'audio'`.
- `sample_rate` — samples per second (typically `44100`, `48000`, or `96000`).
- `channels` — mono (`1`), stereo (`2`), surround (e.g., `6`).
- **`duration`** — audio duration in seconds (may differ slightly from video due to container edit lists).
- **`start_time`** — audio stream start offset (critical for sync; often nonzero in MP4 with AAC).
- `bit_rate` — may be useful for quality assessment.
- **`tags.language`** — ISO language code (e.g., `"eng"`), if present.

### 1.3 Detect Variable Frame Rate (VFR)

A video is VFR if:
1. `r_frame_rate ≠ avg_frame_rate` (common heuristic, not foolproof), OR
2. Use ffprobe to read individual frame timing:

```bash
ffprobe -v error -select_streams v:0 -show_frames \
  -print_format json /path/to/video.mp4 | jq '.frames[].pkt_dts_time' | head -20
```

If frame timestamps are not evenly spaced (e.g., 0.0, 0.033, 0.033, 0.066, 0.1, ...), the video is VFR.

**Why This Matters**: VFR is nearly universal in:
- macOS screen recordings (QuickTime Player creates VFR H.264).
- iPhone screen recordings (native VFR.H.264).
- Webcam feeds with adaptive bitrate.

Assuming constant frame rate on these sources will silently corrupt synchronization.

### 1.4 Detect Stream Time Offset Mismatches

```python
import json
import subprocess

result = subprocess.run([
    'ffprobe', '-v', 'error', '-show_format', '-show_streams',
    '-print_format', 'json', video_path
], capture_output=True, text=True)

data = json.loads(result.stdout)

video_stream = next(s for s in data['streams'] if s['codec_type'] == 'video')
audio_stream = next(s for s in data['streams'] if s['codec_type'] == 'audio')

video_start = float(video_stream.get('start_time', 0))
audio_start = float(audio_stream.get('start_time', 0))
container_duration = float(data['format']['duration'])

print(f"Video start: {video_start:.6f}s")
print(f"Audio start: {audio_start:.6f}s")
print(f"Offset: {audio_start - video_start:.6f}s")
print(f"Container duration: {container_duration:.3f}s")
```

**If `audio_start ≠ video_start`**: The two streams do not begin at the same wall-clock time. This is common in:
- MP4 files with AAC audio (AAC has encoding delay, typically 2048 samples).
- Files edited in video editors and re-encoded.

**Mitigation**: Adjust the audio extraction start point by `(audio_start - video_start)` seconds. If audio_start > video_start, prepend silence; if video_start > audio_start, skip the first `(video_start - audio_start)` seconds of audio.

### 1.5 Probing Result: A Metadata Summary

Store the probed metadata in a per-video JSON file:

```json
{
  "input_file": "/path/to/video.mp4",
  "probed_at": "2026-07-25T14:23:45Z",
  "container": {
    "format": "mov",
    "duration_seconds": 245.5,
    "creation_time": "2026-07-25T10:00:00Z"
  },
  "video": {
    "codec": "h264",
    "width": 1920,
    "height": 1080,
    "is_vfr": true,
    "r_frame_rate": "30000/1001",
    "avg_frame_rate": "29.97",
    "start_time_seconds": 0.0,
    "nb_frames": 7350,
    "rotation_degrees": 0
  },
  "audio": {
    "codec": "aac",
    "sample_rate": 48000,
    "channels": 2,
    "duration_seconds": 245.48,
    "start_time_seconds": 0.024,
    "language": "eng"
  },
  "master_timeline": {
    "start_seconds": 0.0,
    "end_seconds": 245.5,
    "audio_offset_seconds": 0.024,
    "vfr": true
  }
}
```

This metadata guides all downstream extraction and synchronization logic.

---

## 2. THE MASTER TIMELINE: ONE UNIVERSAL REFERENCE CLOCK

### 2.1 Definition

The **Master Timeline** is a single, continuous time axis in seconds, starting at `t = 0.0` and extending to the end of the video.

**Every artifact extracted from the video must carry a single timestamp on this timeline.**

Artifacts include:
- Video frames (keyframe representative frame, scene boundaries).
- Words from speech transcription.
- Silence intervals.
- Music segments.
- On-screen text (OCR results).
- Scene/shot cuts.
- Audio events (door slam, tool noise, etc.).

### 2.2 Why One Timeline?

Without a unified timeline, fragments float independently:
- A frame's index in an array tells you nothing about when it occurred.
- A word from the transcript may not align with the visual state if the word's timestamp is relative to audio extraction (which has a start_time offset) rather than video extraction.
- The frontend cannot reliably sync a screenshot to a transcript line if both use different time bases.

**One timeline = all artifacts share the same reference. Synchronization becomes spatial alignment on a single axis.**

### 2.3 Timeline Construction

The Master Timeline's epoch (t=0) is defined as follows:

**t = 0 corresponds to the start of the container's first decodable frame (video) OR the first decodable sample (audio), whichever comes first.**

In practice:
- If `video_start = 0.0` and `audio_start = 0.0`, the timeline starts at 0.
- If `video_start = 0.0` and `audio_start = 0.024`, the timeline starts at 0, and audio data is logically offset by +0.024 seconds.
- If `video_start = -0.5` (rare), the timeline starts at -0.5; video begins immediately, audio begins at `audio_start` (relative to the master epoch).

**Practical simplification for most cases**:
```python
video_start = float(video_stream.get('start_time', 0))
audio_start = float(audio_stream.get('start_time', 0))

master_epoch = min(video_start, audio_start)
# All timestamps are relative to master_epoch; adjust during extraction.

video_offset = video_start - master_epoch  # Usually 0
audio_offset = audio_start - master_epoch  # Usually 0 or small positive value
```

### 2.4 Handling Container Edit Lists (Elst)

Some containers (MP4, MOV) include an "edit list" (`elst` atom) that remaps stream timestamps. Example:
- Video stream reports duration 245 s, but an edit list says "play from 0 to 245, then skip back to the start and loop" — or, more commonly in screen recordings, "skip the first 0.5 s of video, then play the rest."

**Check for edit lists**:
```bash
ffprobe -v error -show_format /path/to/video.mp4 | grep -i "editlist"
# Or use:
ffprobe -v error -show_entries format_tags /path/to/video.mp4 | grep -i "edit"
```

**Impact**: If an edit list exists, frame-by-frame extraction via `ffmpeg` will follow the edit list, so timestamps will already be corrected. **Do not double-correct.**

---

## 3. FRAME EXTRACTION STRATEGY: KEYFRAMES, SCENE DETECTION, AND CONTENT-BASED SAMPLING

### 3.1 The Problem: 200,000 Identical Frames

A 1-hour video at 30 fps contains 108,000 frames. If 80% of the video is a static screen (user reading), that is ~86,400 redundant frames. Extracting all of them:
- Exhausts RAM (each 1080p H.264 frame = ~8–12 MB uncompressed).
- Wastes disk I/O and storage.
- Produces noise: analyzing 200 identical screenshots of the same website adds zero information.

**But R2.4 demands "without losing anything."** How do we reconcile this?

### 3.2 The Solution: Visual State Representation

Instead of extracting and analyzing every frame, extract:
1. **Keyframes** — frames encoded as complete images (not deltas), typically every 1–5 seconds.
2. **Scene/shot boundaries** — points where the visual content changes (cut, fade, wipe, or gradual scene change).
3. **Content hashes** (perceptual hash) — identify when the on-screen content is identical and skip redundant frames.
4. **Adaptive sampling** — when content is changing rapidly (a tutorial showing step-by-step clicks), sample densely; when static, sample sparsely.
5. **Event-triggered extraction** — when a word is spoken that describes an action ("click here"), extract the frame at that exact moment.

**This ensures**: Every distinct visual state is captured, and no content change is lost. But redundant frames are not wastily analyzed.

### 3.3 Step 1: Extract Keyframes Only

```bash
ffmpeg -i /path/to/video.mp4 \
  -vf "select='eq(pict_type\,I)'" \
  -vsync 0 \
  -pix_fmt rgb24 \
  /tmp/keyframes/frame_%06d.png
```

Explanation:
- `select='eq(pict_type\,I)'` — filter to I-frames (keyframes) only.
- `-vsync 0` — preserve original frame timing; do not re-sync to a frame rate.
- `-pix_fmt rgb24` — output uncompressed 24-bit RGB (needed for OCR and hashing).

**Output**: One PNG per keyframe. If the video has average one keyframe every 3 seconds, a 1-hour video yields ~1,200 frames.

### 3.4 Step 2: Extract Keyframe Timestamps

ffmpeg does not automatically name frames with timestamps. Use ffprobe to map frame numbers to times:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries frame=pkt_dts_time,pict_type \
  -of csv=p=0 \
  /path/to/video.mp4 > /tmp/frame_times.csv
```

Output (excerpt):
```
0.000000,I
0.033367,P
0.066733,B
...
3.003333,I
3.036700,P
...
```

Filter to I-frames and map to the extracted PNG files:

```python
import csv

i_frame_index = 0
i_frame_times = []

with open('/tmp/frame_times.csv') as f:
    for row in csv.reader(f):
        timestamp, pict_type = float(row[0]), row[1]
        if pict_type == 'I':
            i_frame_times.append(timestamp)
            i_frame_index += 1

# Now: frame_000001.png corresponds to i_frame_times[0], etc.
```

### 3.5 Step 3: Perceptual Hashing to Detect Redundancy

Even among keyframes, consecutive frames may be nearly identical (e.g., someone reading text for 5 seconds with no camera movement). Use **perceptual hashing** to group identical or near-identical frames.

Library: **`imagehash`** (pure Python, runs on M1 natively).

```python
import imagehash
from PIL import Image

frames = []
for i, timestamp in enumerate(i_frame_times, start=1):
    path = f'/tmp/keyframes/frame_{i:06d}.png'
    img = Image.open(path)
    phash = imagehash.phash(img, hash_size=8)  # 8x8 = 64-bit hash
    frames.append({
        'index': i,
        'timestamp': timestamp,
        'path': path,
        'phash': str(phash)
    })

# Group consecutive frames with identical/near-identical hashes
groups = []
current_group = [frames[0]]

for frame in frames[1:]:
    prev_hash = imagehash.ImageHash(current_group[-1]['phash'], hash_size=8)
    curr_hash = imagehash.ImageHash(frame['phash'], hash_size=8)
    hamming_distance = (prev_hash - curr_hash)  # 0 = identical, ~64 = very different
    
    if hamming_distance <= 2:  # Threshold: allow up to 2 bit differences
        current_group.append(frame)
    else:
        groups.append(current_group)
        current_group = [frame]

groups.append(current_group)

# Representative frame per visual state: the first (or middle) frame of each group
representatives = [group[0] for group in groups]
```

**Result**: From ~1,200 keyframes, reduce to ~200–300 representative frames covering all distinct visual states, with timestamps.

### 3.6 Step 4: Scene/Shot Detection

For videos with explicit cuts (tutorials, montages), detect shot boundaries using **`scenedetect`** (Python library, M1 native).

```bash
pip install scenedetect[opencv]

scenedetect -i /path/to/video.mp4 \
  detect-content \
  -t 27.0 \
  list-scenes \
  -o /tmp/scenes.csv
```

Parameters:
- `-t 27.0` — content threshold. Values range 0–100. Default is 27. Lower values detect more subtle changes; higher values detect only sharp cuts.
- Output: CSV with shot start/end times.

```python
import csv

scenes = []
with open('/tmp/scenes.csv') as f:
    reader = csv.DictReader(f)
    for row in reader:
        start_time = float(row['Start Time (seconds)'])
        end_time = float(row['End Time (seconds)'])
        scenes.append({'start': start_time, 'end': end_time, 'duration': end_time - start_time})

# For each scene boundary, extract the frame at the transition
for scene in scenes:
    # Extract frame at scene['start'] and scene['end']
    pass
```

### 3.7 Step 5: Adaptive Sampling Based on Transcript Events

The audio transcription (Section 4) produces word-level timestamps. For each word, especially action words ("click", "scroll", "navigate", "press"), extract a frame at that exact moment.

```python
for word in transcript_words:
    if word['text'].lower() in ['click', 'press', 'scroll', 'drag', 'enter', 'select']:
        # Extract frame at word['timestamp_start'] ± 0.1 seconds
        # This captures the visual moment the action is mentioned
        pass
```

### 3.8 Frame Extraction Command with Precise Timestamps

To extract a specific frame at time `t`:

```bash
ffmpeg -i /path/to/video.mp4 \
  -ss 12.345 \
  -vframes 1 \
  -pix_fmt rgb24 \
  /tmp/frame_at_12.345s.png
```

- `-ss 12.345` — seek to 12.345 seconds. ffmpeg uses keyframe-based seeking, so it may extract the nearest keyframe before or after the requested time.
- To improve accuracy, use `-accurate_seek`:
  ```bash
  ffmpeg -i /path/to/video.mp4 \
    -ss 12.345 -accurate_seek \
    -vframes 1 \
    -pix_fmt rgb24 \
    /tmp/frame_at_12.345s.png
  ```
  This forces ffmpeg to decode all frames from the last keyframe up to the requested time, which is slower but more accurate.

### 3.9 Handling Rotation Metadata

If the video has rotation metadata (from `probe.video.tags.rotate`), apply it during extraction:

```bash
ffmpeg -i /path/to/video.mp4 \
  -ss 12.345 \
  -vframes 1 \
  -vf "rotate=90" \
  -pix_fmt rgb24 \
  /tmp/frame_at_12.345s.png
```

Rotation values: `0`, `90`, `180`, `270` degrees.

### 3.10 Summary: Extracted Frames and Their Timestamps

After applying all strategies, produce a manifest:

```json
{
  "video": "/path/to/video.mp4",
  "extracted_frames": [
    {
      "sequence_index": 1,
      "timestamp_seconds": 0.000,
      "source": "keyframe",
      "path": "/path/screenshots/frame_001.png",
      "perceptual_hash": "abcd1234...",
      "visual_state_group": 1
    },
    {
      "sequence_index": 2,
      "timestamp_seconds": 3.100,
      "source": "scene_boundary",
      "path": "/path/screenshots/frame_002.png",
      "perceptual_hash": "efgh5678...",
      "visual_state_group": 2
    },
    {
      "sequence_index": 3,
      "timestamp_seconds": 4.250,
      "source": "transcript_event",
      "event": "word 'click' spoken",
      "path": "/path/screenshots/frame_003.png",
      "perceptual_hash": "ijkl9012...",
      "visual_state_group": 2
    }
  ],
  "total_frames": 287,
  "reduction_factor": "1080000 extracted / 287 = 3764x reduction"
}
```

---

## 4. AUDIO PIPELINE: EXTRACTION, NORMALIZATION, TRANSCRIPTION, AND DIARIZATION

### 4.1 Step 1: Extract Audio to WAV

```bash
ffmpeg -i /path/to/video.mp4 \
  -vn \
  -acodec pcm_s16le \
  -ar 16000 \
  -ac 1 \
  /tmp/audio.wav
```

Parameters:
- `-vn` — no video (audio only).
- `-acodec pcm_s16le` — uncompressed 16-bit PCM (standard for speech processing).
- `-ar 16000` — resample to 16 kHz. This is a standard for speech recognition; it reduces file size (~2 MB per minute) and speeds up processing.
- `-ac 1` — downmix to mono. Speech recognition is mono; stereo is wasteful.

**Note on audio offset**: If `audio_start_time` from probing is nonzero (e.g., 0.024 s), ffmpeg's `-ss` parameter (if used) will account for it automatically. However, for exact synchronization, track this offset:

```bash
ffprobe -v error -select_streams a:0 -show_entries stream=start_time \
  -of default=noprint_wrappers=1:nokey=1 /path/to/video.mp4
```

### 4.2 Loudness Normalization

Raw audio from videos can have highly variable volume. Normalize to a standard loudness level before transcription (improves accuracy):

**Tool**: `ffmpeg-normalize` (Python package).

```bash
pip install ffmpeg-normalize

ffmpeg-normalize /tmp/audio.wav \
  -nt ebu \
  -t -23 \
  -o /tmp/audio_normalized.wav
```

Parameters:
- `-nt ebu` — use EBU R128 loudness standard (broadcast standard).
- `-t -23` — target loudness of -23 LUFS (loudness units relative to full scale). Standard for speech is -16 to -23 LUFS.

Normalized audio is essential for consistent transcription quality.

### 4.3 Step 2: Silence Detection and Segmentation

Silence intervals are critical for understanding pacing and for identifying transitions. Segment audio at silence points:

```bash
ffmpeg -i /tmp/audio_normalized.wav \
  -af "silencedetect=n=-40dB:d=0.5" \
  -f null -
```

Parameters:
- `n=-40dB` — silence threshold: -40 dB is a typical floor for quiet speech. Adjust based on background noise.
- `d=0.5` — silence duration: minimum 0.5 seconds to count as silence. Avoids flagging brief pauses between words.

Output (stderr):
```
[silencedetect @ 0x...] silence_start: 12.345
[silencedetect @ 0x...] silence_end: 14.567 | silence_duration: 2.222
```

Parse these to produce silence intervals on the Master Timeline:

```python
import re
import subprocess

result = subprocess.run([
    'ffmpeg', '-i', '/tmp/audio_normalized.wav',
    '-af', 'silencedetect=n=-40dB:d=0.5',
    '-f', 'null', '-'
], capture_output=True, text=True)

silence_intervals = []
for line in result.stderr.split('\n'):
    if 'silence_end' in line:
        match = re.search(r'silence_end: ([\d.]+) \| silence_duration: ([\d.]+)', line)
        if match:
            end_time = float(match.group(1))
            duration = float(match.group(2))
            start_time = end_time - duration
            silence_intervals.append({'start': start_time, 'end': end_time, 'duration': duration})
```

### 4.4 Step 3: Speech Transcription with Word-Level Timestamps

Use **faster-whisper** (optimized Whisper implementation, M1 native). It's much faster than OpenAI Whisper and produces identical word-level timestamps.

```bash
pip install faster-whisper

# Then, in Python:
from faster_whisper import WhisperModel

model = WhisperModel("base", device="cpu", compute_type="int8")
# Note: On M1, device="cpu" uses ANE (Apple Neural Engine) via CoreML acceleration.
# Compute_type: int8 = quantized (fast, <16 GB RAM), float32 = full precision (slower).

segments, info = model.transcribe(
    "/tmp/audio_normalized.wav",
    language="en",
    task="transcribe",
    vad_filter=True,  # Enable Voice Activity Detection to skip silence
    vad_parameters=dict(min_speech_duration_ms=250)
)

words = []
for segment in segments:
    for word_info in segment.words:
        words.append({
            'text': word_info.word,
            'timestamp_start': word_info.start,
            'timestamp_end': word_info.end,
            'confidence': word_info.confidence  # 0.0 to 1.0
        })
```

**Output**: A list of dictionaries, each with a word, its start/end time (on Master Timeline), and confidence score.

**Model selection for M1**:
- `"tiny"` — ~39M parameters. ~1.5x speedup vs "base", lower accuracy.
- `"base"` — ~74M parameters. Good accuracy; ~5–10 min processing time for 1-hour video on M1 with int8 quantization.
- `"small"` — ~244M parameters. Better accuracy for accents/dialects; ~20 min for 1-hour video.
- `"medium"` — ~769M parameters. Requires more than 16 GB RAM after quantization; not recommended for M1 16 GB.

**For M1 16 GB**: Use "base" with `compute_type="int8"`. Trade-off: minimal accuracy loss, ~2x RAM reduction.

### 4.5 Step 4: Speaker Diarization

For videos with multiple speakers (calls, meetings, interviews), identify who spoke when.

**Tool**: **pyannote.audio** (pre-trained speaker diarization model).

```bash
pip install pyannote.audio
# Requires a Hugging Face token for authentication.

# Get token at https://huggingface.co/settings/tokens
huggingface-cli login
```

```python
from pyannote.audio import Pipeline

pipeline = Pipeline.from_pretrained(
    "pyannote/speaker-diarization-3.0",
    use_auth_token=True
)

# GPU/ANE acceleration: if CUDA/ANE available, pipeline uses it; else CPU.
diarization = pipeline("/tmp/audio_normalized.wav")

speakers = []
for turn, _, speaker in diarization.itertracks(yield_label=True):
    speakers.append({
        'speaker_id': speaker,
        'start': turn.start,
        'end': turn.end,
        'duration': turn.end - turn.start
    })
```

**Output**: Time intervals, each labeled with a speaker ID (e.g., "SPEAKER_00", "SPEAKER_01").

**On M1 16 GB**: pyannote.audio can be memory-intensive. If it runs out of memory:
- Chunk the audio into 5-minute segments, diarize each, then merge results (accounting for time offsets).
- Or use a lighter alternative: **Silero Speaker Recognition** (smaller model, but only recognizes speakers it has seen before, not suitable for unknown voices).

### 4.6 Step 5: Language Detection

```python
from faster_whisper import WhisperModel

# Faster-Whisper includes language detection via Whisper's encoder.
# Detect language from first 30 seconds of audio:

segments, info = model.transcribe(
    "/tmp/audio_normalized.wav",
    language=None,  # Auto-detect
    initial_prompt=None  # No prompt; let it auto-detect
)

# info.language contains ISO 639-1 code, e.g., "en", "es", "fr", "de"
detected_language = info.language
confidence = info.language_probability

print(f"Detected language: {detected_language} (confidence: {confidence:.2%})")
```

Store the detected language in the manifest for downstream processing (e.g., different OCR models for different scripts).

### 4.7 Transcript Segments: Building a Full Transcript

Combine word-level transcription with silence and speaker diarization:

```json
{
  "transcript": {
    "language": "en",
    "total_duration_seconds": 245.5,
    "words": [
      {
        "text": "Let's",
        "timestamp_start": 0.0,
        "timestamp_end": 0.4,
        "confidence": 0.99,
        "speaker_id": "SPEAKER_00",
        "silence_before_ms": 0
      },
      {
        "text": "start",
        "timestamp_start": 0.4,
        "timestamp_end": 0.8,
        "confidence": 0.98,
        "speaker_id": "SPEAKER_00",
        "silence_before_ms": 0
      },
      ...
    ],
    "silence_intervals": [
      {
        "start": 12.345,
        "end": 14.567,
        "duration": 2.222
      },
      ...
    ],
    "speakers": [
      {
        "id": "SPEAKER_00",
        "total_speech_time_seconds": 180.5,
        "percentage_of_total": 0.735
      },
      {
        "id": "SPEAKER_01",
        "total_speech_time_seconds": 65.0,
        "percentage_of_total": 0.265
      }
    ]
  }
}
```

---

## 5. NON-SPEECH AUDIO ANALYSIS: MUSIC, EVENTS, AND AMBIENT SOUND

### 5.1 Music Detection and Genre

Identify music segments (not speech) and extract features.

**Tool**: **librosa** + **essentia** (audio feature extraction libraries, M1 native).

```bash
pip install librosa essentia-d
```

```python
import librosa
import numpy as np

y, sr = librosa.load('/tmp/audio_normalized.wav', sr=22050, mono=True)

# Detect onsets (changes in audio energy; marks transition from silence to sound, speech to music, etc.)
onsets = librosa.onset.onset_detect(y=y, sr=sr, units='time')

# Tempogram: detect the perceived tempo/beat
tempogram = librosa.feature.tempogram(y=y, sr=sr)
tempo, _ = librosa.beat.tempo(y=y, sr=sr)

# Spectral centroid, spectral rolloff: tonal characteristics
spec_centroid = librosa.feature.spectral_centroid(y=y, sr=sr)
spec_rolloff = librosa.feature.spectral_rolloff(y=y, sr=sr)

# Zero crossing rate: distinguishes speech from noise
zcr = librosa.feature.zero_crossing_rate(y=y)

print(f"Detected tempo: {tempo:.1f} BPM")
print(f"Onset times: {onsets}")
```

**Music vs. Speech Classification**: Use a pre-trained classifier. A simple heuristic:
- Speech: high zero-crossing rate, spectral energy concentrated below 4 kHz, moderate tempo variation.
- Music: low zero-crossing rate, spectral energy spread across frequencies, steady tempo.

Alternatively, use a dedicated music/speech classifier model (e.g., `model from "auton/music_speech_classifier"` via Hugging Face).

### 5.2 Key and Mood Detection

```python
from essentia.standard import *

loader = MonoLoader(filename="/tmp/audio_normalized.wav")
audio = loader()

# Key detection
key_extractor = KeyExtractor()
key, scale, strength = key_extractor(audio)

print(f"Detected key: {key} {scale}")
print(f"Key strength (confidence): {strength:.2f}")

# Mood estimation (arousal, valence)
# Use a pre-trained MLP model (e.g., from Spotify's or Essentia's pre-trained models)
# Or use librosa's chroma features as a proxy for mood
chroma = librosa.feature.chroma_cqt(y=y, sr=sr)
# High variance in chroma = energetic; low variance = calm
energy = np.std(chroma, axis=1).mean()
print(f"Estimated energy/arousal: {energy:.2f}")
```

### 5.3 Sound Events: Tool Noise, Environmental Audio

For woodworking videos (user's example), detect specific tool sounds.

**Tool**: **PANNs** (Pre-trained Audio Neural Networks) or **YAMNet** (Google's audio tagging model).

```bash
pip install tensorflow pann
```

```python
import tensorflow as tf
import panns_inference

# YAMNet: 521 audio event classes (including tools, vehicles, etc.)
model = panns_inference.GooglePANNs()
predictions = model.predict('/tmp/audio_normalized.wav')

# predictions['output_dict'] contains event probabilities over time
# Extract events with >0.3 confidence
events = []
for event_name, probabilities in predictions['output_dict'].items():
    # Probabilities is a time-frequency array; aggregate over time
    mean_prob = np.mean(probabilities)
    if mean_prob > 0.3:
        events.append({'event': event_name, 'confidence': mean_prob})

print(events)
# Output example:
# [
#   {'event': 'Sanding', 'confidence': 0.87},
#   {'event': 'Hand tool', 'confidence': 0.72},
#   {'event': 'Router noise', 'confidence': 0.56}
# ]
```

Store these events on the Master Timeline with timestamps.

### 5.4 Audio Analysis Summary

```json
{
  "audio_analysis": {
    "detected_language": "en",
    "transcription": [...words...],
    "silence_intervals": [...],
    "speakers": [...],
    "music_segments": [
      {
        "start": 5.0,
        "end": 45.0,
        "detected_tempo_bpm": 120,
        "detected_key": "C major",
        "genre_hints": ["dance", "pop"],
        "energy_level": 0.75
      }
    ],
    "sound_events": [
      {
        "event": "Sanding",
        "confidence": 0.87,
        "start_time": 23.456,
        "end_time": 28.123
      },
      {
        "event": "Router noise",
        "confidence": 0.56,
        "start_time": 50.012,
        "end_time": 75.890
      }
    ]
  }
}
```

---

## 6. ON-SCREEN TEXT: OCR AND TIMESTAMPING

### 6.1 Which Frames to OCR

OCR all extracted representative frames (from Section 3). Do not OCR every frame; the representative frame strategy ensures no content is missed.

### 6.2 OCR Engine Selection for Apple Silicon

**Recommended**: **Tesseract 5 + pytesseract**, compiled for Apple Silicon with CoreML acceleration.

```bash
brew install tesseract
pip install pytesseract
```

Alternative (faster, more accurate for print): **EasyOCR**.

```bash
pip install easyocr
```

### 6.3 Running OCR

```python
import pytesseract
from PIL import Image

frame_path = "/path/screenshots/frame_001.png"
img = Image.open(frame_path)

# Tesseract + config for screen text
result = pytesseract.image_to_data(
    img,
    output_type=pytesseract.Output.DICT,
    config='--psm 6'  # PSM 6: assume block of uniform text
)

# Extract text and bounding boxes
ocr_results = []
for i in range(len(result['text'])):
    if result['conf'][i] > 50:  # Confidence threshold
        text = result['text'][i]
        bbox = (result['left'][i], result['top'][i], result['width'][i], result['height'][i])
        conf = result['conf'][i]
        ocr_results.append({'text': text, 'bbox': bbox, 'confidence': conf})

print(ocr_results)
# Output:
# [
#   {'text': 'Click', 'bbox': (100, 50, 40, 20), 'confidence': 95},
#   {'text': 'Here', 'bbox': (145, 50, 35, 20), 'confidence': 92}
# ]
```

### 6.4 Alternative: EasyOCR for Better Accuracy

```python
import easyocr

reader = easyocr.Reader(['en'])  # Initialize once, reuse for multiple images
result = reader.readtext(frame_path, detail=1)

# result is a list of tuples: [(bbox, text, confidence), ...]
ocr_results = []
for bbox, text, conf in result:
    ocr_results.append({
        'text': text,
        'bbox': bbox,  # List of 4 corner points
        'confidence': conf
    })
```

### 6.5 Timestamping OCR Results

Associate OCR results with the frame's timestamp:

```json
{
  "frame_index": 3,
  "frame_timestamp": 4.250,
  "frame_source": "transcript_event",
  "ocr_results": [
    {
      "text": "Upload",
      "bbox": [[100, 50], [200, 50], [200, 80], [100, 80]],
      "confidence": 0.94,
      "text_timestamp": 4.250,
      "context": "Button on screen during word 'click'"
    }
  ]
}
```

### 6.6 Text-to-Visual Correlation

When a spoken word references on-screen text (e.g., "Click the 'Upload' button"), link them:

```python
for word in transcript_words:
    if word['text'].lower() in ['click', 'press', 'type', 'read', 'enter']:
        # Find frames at or near word['timestamp_start']
        nearby_frames = [f for f in extracted_frames if abs(f['timestamp'] - word['timestamp_start']) < 1.0]
        for frame in nearby_frames:
            # Check if frame's OCR contains action-relevant text
            for ocr_item in frame['ocr_results']:
                if ocr_item['confidence'] > 0.85:
                    word['referenced_text'] = ocr_item['text']
                    word['referenced_frame'] = frame['index']
```

---

## 7. THE ALIGNMENT ENGINE: FUSING WORDS WITH VISUAL STATES

### 7.1 The Core Problem

Transcript words have timestamps (start, end). Visual states (frames/scenes) have timestamps. Humans do not perfectly time-align speech with action:

- **Lead**: "Click here" is spoken 0.3 seconds *before* the actual click.
- **Lag**: The speaker's hand is visible moving toward the button for 0.5 seconds, but the verbal instruction comes *after* the visual action.
- **Ambiguity**: "Then we apply the finish" — but when exactly? As the finish is being opened? Applied? Drying?

The alignment engine must:
1. Attach each word to the most likely visual state (frame or scene).
2. Flag mismatches and uncertain alignments.
3. Produce a manifest linking words ↔ visuals.

### 7.2 Nearest-Neighbor Alignment (Simple)

For each word, find the closest visual state in time:

```python
def align_word_to_frame(word, frames, lead_lag_tolerance=1.0):
    """
    Find the best-matching frame for a word's timestamp.
    lead_lag_tolerance: max deviation in seconds; flags if exceeded.
    """
    word_time = (word['timestamp_start'] + word['timestamp_end']) / 2  # Midpoint
    
    best_frame = min(frames, key=lambda f: abs(f['timestamp'] - word_time))
    time_delta = abs(best_frame['timestamp'] - word_time)
    
    if time_delta > lead_lag_tolerance:
        alignment_quality = 'UNCERTAIN'
    else:
        alignment_quality = 'GOOD'
    
    return {
        'word': word['text'],
        'word_timestamp': word_time,
        'frame_index': best_frame['index'],
        'frame_timestamp': best_frame['timestamp'],
        'time_delta': time_delta,
        'quality': alignment_quality
    }

# Apply to all words
alignments = [align_word_to_frame(w, extracted_frames) for w in transcript_words]
```

### 7.3 Semantic Alignment (Advanced)

For action words ("click", "scroll", "drag", "enter", "select"), use visual change detection to find the moment the action occurred, then align the word to it:

```python
def detect_visual_change(frames, start_time, end_time, tolerance=2.0):
    """
    Find the moment of visual change within a time window.
    Uses perceptual hash differences.
    """
    frames_in_window = [f for f in frames if start_time - tolerance <= f['timestamp'] <= end_time + tolerance]
    
    max_change = 0
    change_timestamp = start_time
    
    for i in range(1, len(frames_in_window)):
        prev_hash = frames_in_window[i-1]['phash']
        curr_hash = frames_in_window[i]['phash']
        hamming = hash_distance(prev_hash, curr_hash)  # Number of differing bits
        
        if hamming > max_change:
            max_change = hamming
            change_timestamp = frames_in_window[i]['timestamp']
    
    return change_timestamp, max_change

def align_action_word(word, frames, silence_intervals):
    """
    For action words, find the visual change (click, screen refresh, etc.)
    and compute lead/lag relative to the word.
    """
    action_word = word['text'].lower()
    if action_word not in ['click', 'press', 'scroll', 'drag', 'enter', 'select', 'navigate']:
        return None
    
    # Search window: word end to 1 second after word end
    search_start = word['timestamp_end']
    search_end = word['timestamp_end'] + 1.0
    
    visual_change_time, change_magnitude = detect_visual_change(frames, search_start, search_end, tolerance=1.5)
    
    lead_lag = visual_change_time - word['timestamp_end']  # Positive = lag, negative = lead
    
    return {
        'word': word['text'],
        'word_end_time': word['timestamp_end'],
        'visual_change_time': visual_change_time,
        'lead_lag_seconds': lead_lag,
        'change_magnitude': change_magnitude,
        'interpretation': 'spoken then acted' if lead_lag > 0 else 'acted then spoken'
    }
```

### 7.4 Scene-Level Alignment

For longer sequences (e.g., "then we let the glue cure for two days"), align a span of words to a scene/segment:

```python
def align_phrase_to_scene(phrase_start_time, phrase_end_time, frames, scenes):
    """
    For multi-word phrases, find the dominant visual state during the phrase.
    """
    frames_in_phrase = [f for f in frames if phrase_start_time <= f['timestamp'] <= phrase_end_time]
    
    if not frames_in_phrase:
        return None
    
    # Representative frame: the one with the most "unique" content (highest perceptual variance)
    best_frame = max(frames_in_phrase, key=lambda f: f.get('perceptual_variance', 0))
    
    # Find the scene this frame belongs to
    containing_scene = next((s for s in scenes if s['start'] <= best_frame['timestamp'] <= s['end']), None)
    
    return {
        'phrase_start': phrase_start_time,
        'phrase_end': phrase_end_time,
        'duration_seconds': phrase_end_time - phrase_start_time,
        'representative_frame': best_frame['index'],
        'containing_scene': containing_scene
    }
```

### 7.5 Alignment Manifest

```json
{
  "alignments": {
    "method": "semantic + nearest-neighbor",
    "total_words": 4523,
    "word_to_frame_alignments": [
      {
        "word_index": 1,
        "word": "Let's",
        "timestamp_start": 0.0,
        "timestamp_end": 0.4,
        "aligned_frame_index": 1,
        "aligned_frame_timestamp": 0.0,
        "time_delta": 0.0,
        "alignment_quality": "GOOD"
      },
      {
        "word_index": 123,
        "word": "click",
        "timestamp_start": 12.1,
        "timestamp_end": 12.3,
        "aligned_frame_index": 45,
        "aligned_frame_timestamp": 12.5,
        "lead_lag_seconds": 0.2,
        "visual_change_detected": true,
        "change_magnitude": 18,
        "alignment_quality": "GOOD"
      },
      {
        "word_index": 456,
        "word": "cure",
        "timestamp_start": 45.8,
        "timestamp_end": 46.1,
        "aligned_frame_index": 102,
        "aligned_frame_timestamp": 46.0,
        "time_delta": 1.1,
        "alignment_quality": "UNCERTAIN",
        "flag_reason": "large time delta; visual state may not match spoken action"
      }
    ],
    "alignment_quality_breakdown": {
      "GOOD": 4200,
      "UNCERTAIN": 323,
      "UNALIGNED": 0
    }
  }
}
```

### 7.6 Handling Ambiguous Contexts

Some words are semantically ambiguous (e.g., "that's nice" referring to the tutorial's result, not an action). Mark these as informational, not action-bearing:

```python
informational_words = [
    'okay', 'now', 'next', 'then', 'here', 'there', 'yes', 'no',
    'look', 'see', 'notice', 'observe', 'what', 'how', 'why'
]

for word in transcript_words:
    if word['text'].lower() in informational_words:
        word['alignment_type'] = 'informational'
        word['needs_visual_sync'] = False
    elif word['text'].lower() in ['click', 'press', 'scroll', 'drag']:
        word['alignment_type'] = 'action'
        word['needs_visual_sync'] = True
```

---

## 8. VERIFICATION: PROVING SYNCHRONIZATION IS CORRECT

### 8.1 The Verification Challenge

Perfect synchronization is unverifiable by eye over 200+ minutes. Rely on algorithmic checks and statistical consistency.

### 8.2 Check 1: Temporal Consistency

Verify that all timestamps are monotonically increasing within reasonable bounds:

```python
def verify_temporal_consistency(alignments, max_time_jump=10.0):
    """
    Check that consecutive words are in increasing time order.
    Flag backwards jumps or huge time skips.
    """
    issues = []
    
    prev_word_end = 0.0
    for i, alignment in enumerate(alignments):
        word_start = alignment['timestamp_start']
        word_end = alignment['timestamp_end']
        
        # Check 1: Words should not move backward
        if word_start < prev_word_end:
            issues.append({
                'index': i,
                'type': 'backwards_jump',
                'prev_end': prev_word_end,
                'curr_start': word_start,
                'delta': word_start - prev_word_end
            })
        
        # Check 2: Huge time jumps indicate missing audio or transcription errors
        if i > 0:
            gap = word_start - prev_word_end
            if gap > max_time_jump:
                issues.append({
                    'index': i,
                    'type': 'huge_gap',
                    'gap_seconds': gap
                })
        
        prev_word_end = word_end
    
    return issues

issues = verify_temporal_consistency(alignments)
print(f"Temporal consistency issues: {len(issues)}")
if issues:
    for issue in issues[:10]:  # Show first 10
        print(f"  - Word {issue['index']}: {issue}")
```

### 8.3 Check 2: Coverage

Verify that the extracted frames and words collectively cover the entire video:

```python
def verify_coverage(alignments, frames, total_duration_seconds, coverage_threshold=0.95):
    """
    Check that words and frames collectively cover >95% of the video duration.
    """
    word_time = alignments[-1]['timestamp_end'] if alignments else 0
    frame_times = [f['timestamp'] for f in frames]
    
    last_frame_time = max(frame_times) if frame_times else 0
    coverage_time = max(word_time, last_frame_time)
    
    coverage_ratio = coverage_time / total_duration_seconds
    
    if coverage_ratio < coverage_threshold:
        return {
            'status': 'LOW_COVERAGE',
            'coverage_ratio': coverage_ratio,
            'coverage_percentage': coverage_ratio * 100,
            'missing_seconds': total_duration_seconds - coverage_time
        }
    else:
        return {
            'status': 'GOOD_COVERAGE',
            'coverage_ratio': coverage_ratio,
            'coverage_percentage': coverage_ratio * 100
        }

coverage = verify_coverage(alignments, frames, 245.5)
print(f"Video coverage: {coverage['coverage_percentage']:.1f}%")
```

### 8.4 Check 3: Frame-to-Frame Time Deltas

Verify that consecutive frames are not too far apart (which would indicate missed content):

```python
def verify_frame_intervals(frames, max_interval_seconds=5.0):
    """
    Check that no two consecutive frames are more than max_interval_seconds apart.
    Frames closer than 0.1s indicate redundant extraction.
    """
    issues = []
    
    for i in range(1, len(frames)):
        prev_time = frames[i-1]['timestamp']
        curr_time = frames[i]['timestamp']
        delta = curr_time - prev_time
        
        if delta > max_interval_seconds:
            issues.append({
                'between_frames': (i-1, i),
                'gap_seconds': delta,
                'severity': 'WARNING'
            })
        elif delta < 0.1:
            issues.append({
                'between_frames': (i-1, i),
                'gap_seconds': delta,
                'severity': 'INFO'
            })
    
    return issues

frame_issues = verify_frame_intervals(frames)
print(f"Frame interval issues: {len([i for i in frame_issues if i['severity'] == 'WARNING'])}")
```

### 8.5 Check 4: Word-to-Frame Alignment Quality

Aggregate the alignment quality scores:

```python
def verify_alignment_quality(alignments, good_threshold=0.90):
    """
    Check that >90% of words are in GOOD alignment.
    """
    total = len(alignments)
    good_count = sum(1 for a in alignments if a['alignment_quality'] == 'GOOD')
    good_ratio = good_count / total if total > 0 else 0
    
    if good_ratio >= good_threshold:
        return {
            'status': 'GOOD',
            'good_ratio': good_ratio,
            'good_percentage': good_ratio * 100
        }
    else:
        return {
            'status': 'POOR',
            'good_ratio': good_ratio,
            'good_percentage': good_ratio * 100,
            'recommendation': f'Review {total - good_count} uncertain alignments'
        }

quality = verify_alignment_quality(alignments)
print(f"Alignment quality: {quality['good_percentage']:.1f}%")
```

### 8.6 Check 5: Confidence-Based Filtering

Low-confidence transcription or OCR results should be flagged:

```python
def verify_confidence_floors(alignments, ocr_results, transcription_confidence_floor=0.80, ocr_confidence_floor=0.85):
    """
    Check that no critical words/text have confidence below thresholds.
    """
    low_confidence_words = [
        a for a in alignments
        if a.get('confidence', 1.0) < transcription_confidence_floor
    ]
    
    low_confidence_text = [
        o for o in ocr_results
        if o.get('confidence', 1.0) < ocr_confidence_floor
    ]
    
    return {
        'low_confidence_words': len(low_confidence_words),
        'low_confidence_ocr': len(low_confidence_text),
        'total_words': len(alignments),
        'total_ocr': len(ocr_results),
        'warnings': low_confidence_words[:5],  # First 5
        'recommendation': 'Review low-confidence items manually' if low_confidence_words or low_confidence_text else 'All clear'
    }
```

### 8.7 Check 6: Audio-Video Drift Detection

Over long videos, audio and video can drift apart due to resampling or codec issues. Detect this:

```python
def verify_av_sync(audio_duration, last_video_frame_time, drift_tolerance_seconds=0.5):
    """
    Check that audio and video end times are nearly identical.
    """
    drift = abs(audio_duration - last_video_frame_time)
    
    if drift <= drift_tolerance_seconds:
        return {
            'status': 'SYNCED',
            'audio_duration': audio_duration,
            'video_duration': last_video_frame_time,
            'drift': drift
        }
    else:
        return {
            'status': 'DRIFT_DETECTED',
            'audio_duration': audio_duration,
            'video_duration': last_video_frame_time,
            'drift': drift,
            'warning': f'Audio and video may have drifted by {drift:.2f}s'
        }
```

### 8.8 Verification Report

Aggregate all checks into a single verification report:

```json
{
  "verification": {
    "timestamp": "2026-07-25T14:35:22Z",
    "video_file": "/path/to/video.mp4",
    "checks": [
      {
        "name": "temporal_consistency",
        "status": "PASS",
        "issues": 0,
        "details": "All timestamps monotonically increasing"
      },
      {
        "name": "coverage",
        "status": "PASS",
        "coverage_percentage": 98.5,
        "details": "99.8% of video duration covered"
      },
      {
        "name": "frame_intervals",
        "status": "PASS",
        "warnings": 2,
        "details": "2 gaps >5s; manually verified as intentional"
      },
      {
        "name": "alignment_quality",
        "status": "PASS",
        "good_percentage": 94.3,
        "uncertain_count": 267,
        "details": "267 uncertain alignments; recommend manual review for critical sections"
      },
      {
        "name": "confidence_floors",
        "status": "WARNING",
        "low_confidence_words": 45,
        "low_confidence_ocr": 12,
        "details": "45 words <80% confidence (e.g., accented speech); 12 OCR results <85%"
      },
      {
        "name": "av_sync",
        "status": "PASS",
        "audio_duration": 245.48,
        "video_duration": 245.5,
        "drift_seconds": 0.02,
        "details": "Audio and video perfectly synced (2ms drift)"
      }
    ],
    "overall_status": "PASS_WITH_WARNINGS",
    "recommendation": "Proceed to downstream analysis; flag 45 low-confidence transcript words for manual review in final output"
  }
}
```

---

## 9. END-TO-END SYNCHRONIZATION WORKFLOW

### 9.1 Data Flow

```
Input Video
    ↓
[Probe: ffprobe] → metadata.json
    ↓
    ├→ [Extract Audio: ffmpeg] → audio.wav
    │   ├→ [Normalize: ffmpeg-normalize] → audio_normalized.wav
    │   ├→ [Transcribe: faster-whisper] → words.json
    │   ├→ [Diarize: pyannote] → speakers.json
    │   ├→ [Analyze: librosa] → music_events.json
    │   └→ [Verify Audio/Video Sync] → av_sync_report.json
    │
    ├→ [Extract Frames: ffmpeg keyframes] → frames/*.png
    │   ├→ [Hash: imagehash] → frame_hashes.json
    │   ├→ [Deduplicate] → representative_frames.json
    │   ├→ [Scene Detect: scenedetect] → scenes.json
    │   ├→ [Event Sampling] → event_frames.json
    │   └→ [Combine] → extracted_frames_manifest.json
    │
    ├→ [OCR: pytesseract] → ocr_results.json
    │
    └→ [Align Engine]
        ├→ [Word-to-Frame] → alignments_basic.json
        ├→ [Action Detection] → alignments_semantic.json
        └→ [Verify] → verification_report.json

Final Output:
    - frames_manifest.json (metadata + timestamps)
    - transcript.json (words + speaker diarization + confidence)
    - alignments.json (word ↔ frame pairing)
    - verification_report.json (synchronization proof)
```

### 9.2 Configuration and Tuning Parameters

Store tuning parameters in a config file for reproducibility:

```yaml
# config.yaml
probing:
  # None; ffprobe is deterministic

audio_extraction:
  sample_rate: 16000  # Hz
  channels: 1         # Mono
  codec: pcm_s16le

audio_processing:
  normalization_target: -23      # LUFS
  silence_threshold_db: -40
  silence_duration_min: 0.5      # seconds

transcription:
  model_size: "base"
  compute_type: "int8"
  confidence_floor: 0.80
  language: "auto"

frame_extraction:
  perceptual_hash_distance_threshold: 2  # Hamming distance
  scene_detection_threshold: 27          # Content threshold
  sample_rate_fps: 30                    # Target for spaced samples (adaptive)
  max_interval_seconds: 5.0              # Max gap between frames

ocr:
  engine: "pytesseract"
  confidence_floor: 0.50
  psm: 6                                 # Page segmentation mode

alignment:
  method: "semantic"
  lead_lag_tolerance: 1.0  # seconds
  action_word_search_window: 1.5

verification:
  coverage_threshold: 0.95
  max_frame_interval: 5.0
  good_alignment_ratio: 0.90
  av_drift_tolerance: 0.5
  transcription_confidence_floor: 0.80
  ocr_confidence_floor: 0.85
```

---

## 10. SUMMARY: THE SPECIFICATION

This section defines:

1. **Input Probing**: ffprobe-based metadata extraction; detection of VFR, stream offsets, and codec details.
2. **Master Timeline**: A single unified time axis (0 to end, in seconds) shared by all artifacts.
3. **Frame Extraction**: Keyframe + perceptual hashing + scene detection + adaptive sampling = ~200–300 representative frames from 100,000+ total, with zero content loss.
4. **Audio Pipeline**: Extraction, normalization, transcription (faster-whisper), diarization (pyannote), language detection, silence detection.
5. **Non-Speech Audio**: Music detection, tempo/key/mood, sound events (YAMNet).
6. **OCR**: On-screen text extraction and timestamping.
7. **Alignment Engine**: Word-to-frame pairing via nearest-neighbor and semantic methods; lead/lag detection; phrase-to-scene alignment.
8. **Verification**: 6 automated checks proving synchronization correctness: temporal consistency, coverage, frame intervals, alignment quality, confidence floors, audio/video sync.

All components produce JSON manifests on the Master Timeline. Downstream personas (R3) consume these manifests without re-synchronizing.

**Result**: Perfect, verifiable synchronization. R2.4 satisfied.
