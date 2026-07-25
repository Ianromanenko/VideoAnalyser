# SECTION 3: Multi-Persona Analysis System

**Purpose**: Extract maximum detail from video from all possible perspectives (R3, R5) without losing anything (R2.4) and without hallucinating detail (R5.3, R5.6).

The VideoAnalyser system uses a roster of specialized analysis agents (personas), each with a narrow purpose, exact input/output contracts, and verifiable grounding rules. No persona invents; all claims must be anchored to evidence or explicitly marked as inference.

---

## 3.1 Core Principle: Grounding and Evidence

Every factual claim made by any persona must satisfy ONE of these conditions:

1. **Grounded**: Accompanied by a `timestamp` (MM:SS.mmm format) and an `evidence_reference` that points to:
   - A transcript span (e.g., `transcript[1204:1218]`)
   - A frame ID (e.g., `frame_00:02:15.500`)
   - OCR-extracted text with frame reference (e.g., `ocr@frame_00:01:30.200`)
   - Audio metadata or waveform analysis (e.g., `audio_frequency@00:03:45`)

2. **Inference**: Explicitly labeled `{"claim": "...", "type": "inference", "confidence": 0.85, "basis": "..."}` when a persona must synthesize across evidence but is not stating a direct observation.

**Enforcement mechanism**: Every JSON output from a persona is validated by a schema that enforces the presence of `timestamp`, `evidence_reference`, and `type` fields on all factual claims. Claims without these fields cause the orchestrator to return the output to the persona for remediation.

---

## 3.2 Complete Persona Roster

A total of **19 specialized personas**, grouped by analysis domain. Each runs asynchronously but may depend on outputs from prior personas.

### 3.2.1 PERSONA A1: Source/Provenance Analyst

**Name**: Source Identifier

**Purpose**: Determine the origin and capture method of the video with high confidence, based on technical markers, UI chrome, metadata, and visual characteristics.

**Inputs**:
- Raw video file (container, codec, resolution, frame rate, aspect ratio)
- Video metadata (exif, creation date, camera model if present)
- First 30 frames (visual inspection of UI chrome, watermarks, logos)
- Audio track metadata (sample rate, channels, codec)

**Output JSON Schema**:
```json
{
  "persona": "source_identifier",
  "timestamp": "MM:SS.mmm",
  "source_category": "string (enum: instagram|youtube|iphone_screen_recording|macos_screen_recording|zoom_or_meet_call|dslr_footage|other_web_source|ai_generated)",
  "confidence": 0.0-1.0,
  "markers": [
    {
      "marker_type": "string (enum: aspect_ratio|resolution|ui_chrome|watermark|metadata|frame_rate|color_space)",
      "observed_value": "string",
      "evidence_reference": "string (frame_id or metadata_field)",
      "timestamp": "MM:SS.mmm or 'metadata'",
      "weight_in_decision": 0.0-1.0
    }
  ],
  "final_assessment": "string (e.g., 'Portrait Instagram video shot on iPhone 14 Pro')",
  "uncertainty_flags": ["string"]
}
```

**What it must NOT do**:
- Speculate about the intent or content of the video.
- Analyze the visual composition or aesthetic.
- Extract transcript or spoken content.
- Identify people or judge the quality of the video.

**Failure Mode**: Unable to determine source with confidence ≥0.70. Persona returns top 2-3 most likely candidates with confidence scores and explicitly states ambiguity.

**Runs**: Always (first pass).

**Retries**: Max 1 (no retry; if confidence <0.70, mark as ambiguous and proceed).

---

### 3.2.2 PERSONA A2: Content Categorizer

**Name**: Content Category Classifier

**Purpose**: Label the video's primary content type(s) to determine downstream persona activation. Controls routing logic.

**Inputs**:
- Source category (from A1)
- First 10 seconds of audio (to detect voice, music, silence, environmental noise)
- First 30 frames and frames at 25%, 50%, 75%, 100% (scene composition)
- Video duration and aspect ratio

**Output JSON Schema**:
```json
{
  "persona": "content_categorizer",
  "primary_categories": [
    {
      "category": "string (enum: screen_share_tutorial|talking_head|video_conference|music_inspiration|woodworking_tutorial|software_tutorial|mixed_media|unstructured|other)",
      "confidence": 0.0-1.0,
      "reasoning": "string",
      "evidence_reference": "string"
    }
  ],
  "secondary_attributes": [
    "string (enum: has_ui|has_spoken_narration|has_background_music|has_title_cards|has_editing_transitions|has_text_overlay|is_timelapsed)"
  ],
  "inferred_structure": "string (e.g., 'intro → steps → conclusion', 'unstructured free-form')",
  "estimated_step_count": "number or null (e.g., 5 for tutorial)",
  "routing_recommendations": {
    "activate_personas": ["A3", "A4", "A6", "B2", "B4", "C1"],
    "skip_personas": ["B5", "C3"]
  }
}
```

**What it must NOT do**:
- Extract detailed content (leave to frame-by-frame analyst).
- Judge quality or entertainment value.
- Identify specific people.
- Make assumptions about the end user's intent in creating the video.

**Failure Mode**: Unable to confidently classify. Persona marks as "mixed_media" and activates all downstream personas.

**Runs**: Always (second pass, after A1).

**Retries**: Max 1.

---

### 3.2.3 PERSONA A3: Verbatim Transcript Custodian

**Name**: Speech Transcriber and Verifier

**Purpose**: Produce a word-for-word, time-aligned transcript of all spoken content, with zero paraphrasing or summary. Speech must be accurately captured including filler words, mistakes, repairs, and false starts. Every sentence is timestamped.

**Inputs**:
- Full audio track from video
- Source category (from A1; affects language model if multilingual)
- Content category (from A2; affects terminology domain)

**Output JSON Schema**:
```json
{
  "persona": "transcript_custodian",
  "transcript_segments": [
    {
      "segment_id": "string (e.g., 'seg_001')",
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "speaker": "string (enum: 'narrator|participant_1|participant_2|...|background_voice|unknown')",
      "text": "string (verbatim, no edits)",
      "confidence": 0.0-1.0,
      "flags": [
        "string (enum: filler_word|false_start|correction|overlapping_speech|unclear_audio)"
      ],
      "phonetic_marker": "string or null (for unclear words, IPA or best-guess)"
    }
  ],
  "total_duration_seconds": "number",
  "coverage": {
    "silence_fraction": "0.0-1.0",
    "speech_fraction": "0.0-1.0",
    "music_only_fraction": "0.0-1.0",
    "overlapping_speech_fraction": "0.0-1.0"
  },
  "language_detected": "string (enum: en|es|fr|de|etc)",
  "audio_quality_issues": [
    {
      "issue": "string (e.g., 'background_hum', 'clipping', 'dropout')",
      "timestamp": "MM:SS.mmm",
      "affected_text": "string",
      "impact_on_transcript": "string"
    }
  ]
}
```

**What it must NOT do**:
- Paraphrase or summarize.
- Correct grammar or fix false starts (preserve verbatim).
- Interpret meanings or explain concepts.
- Filter out filler words.
- Add punctuation that wasn't audible (e.g., question marks based on context).

**Failure Mode**: Confidence <0.80 on any segment. Persona flags the segment as "requires_manual_review" and includes best-guess plus alternatives.

**Runs**: Always (parallel with A2 is acceptable; depends on A1 for language hint).

**Retries**: Max 2 (persona may re-listen to unclear segments).

---

### 3.2.4 PERSONA B1: Frame-by-Frame Visual Analyst

**Name**: Visual Content Extractor

**Purpose**: Describe every visually significant moment: scene changes, objects appearing/disappearing, on-screen text, people, activities, scene composition. Must be frame-accurate and reference the exact frame ID.

**Inputs**:
- Full video (or key frames sampled at 1 fps, with full-resolution fallback)
- Transcript with timestamps (from A3)
- Content category (from A2; affects detail level)
- Source category (from A1; affects expected UI chrome)

**Output JSON Schema**:
```json
{
  "persona": "frame_analyst",
  "key_frames": [
    {
      "frame_id": "string (format: 'frame_HH:MM:SS.mmm')",
      "timestamp": "MM:SS.mmm",
      "description": "string (2-3 sentences describing visual state)",
      "scene_category": "string (enum: title_card|tutorial_step|full_screen_app|partial_screen_share|person_talking|transition|black_screen|other)",
      "visual_elements": [
        {
          "element": "string (e.g., 'button labeled "Submit"', 'hand holding chisel', 'color gradient blue→red')",
          "location": "string (e.g., 'top-right corner', 'center', 'lower-left 10%')",
          "evidence_reference": "frame_id"
        }
      ],
      "on_screen_text": [
        {
          "text": "string",
          "font_size_relative": "small|medium|large",
          "color": "string (descriptive or hex if clearly visible)",
          "location": "string",
          "evidence_reference": "frame_id"
        }
      ],
      "concurrent_transcript_span": "string (e.g., 'transcript[seg_012:seg_015]')",
      "transitions_from_prior": "string or null (e.g., 'cut', 'fade', 'wipe')"
    }
  ],
  "scene_changes": [
    {
      "timestamp": "MM:SS.mmm",
      "transition_type": "string (enum: cut|fade|dissolve|wipe|pan|zoom|other)",
      "from_scene": "string",
      "to_scene": "string",
      "evidence_reference": "frame_id"
    }
  ],
  "visual_coverage_fraction": 0.0-1.0,
  "notes": "string (e.g., 'Screen is mostly black 03:45—04:02; unclear if intentional or technical issue')"
}
```

**What it must NOT do**:
- Describe what is being said (leave to transcript).
- Interpret meaning or teach concepts.
- Identify people by name (describe position/clothing instead).
- Make claims about quality or aesthetic judgment (leave to A5).
- Assume or infer action beyond what is visibly shown.

**Failure Mode**: Video contains many fast cuts or motion blur. Persona reduces frame sample rate and explicitly flags regions where confidence drops below 0.70.

**Runs**: Always (can run parallel with A3; depends on A1/A2).

**Retries**: Max 1 (if coverage_fraction <0.80, re-sample with denser frame rate).

---

### 3.2.5 PERSONA B2: Real-World Duration Analyst

**Name**: Process Time Extractor

**Purpose**: Distinguish screen time from real-world elapsed time. For tutorials involving physical processes (drying, curing, steaming, baking, etc.), extract how long the actual process takes, not how long the video shows. Critical for R5.3, R5.5, R5.6.

**Inputs**:
- Transcript with timestamps (from A3)
- Visual frames showing process start and end states (from B1)
- Content category (from A2)
- Inferred step count (from A2)

**Output JSON Schema**:
```json
{
  "persona": "duration_analyst",
  "screen_elapsed_time": "number (seconds; actual video duration)",
  "process_timeline": [
    {
      "step_id": "string (e.g., 'step_1_sand')",
      "step_description": "string",
      "on_screen_duration": "number (seconds)",
      "real_world_duration": {
        "value": "number or null (seconds)",
        "unit": "string (enum: seconds|minutes|hours|days)",
        "grounded": true|false,
        "evidence": "string (e.g., 'transcript[seg_045]: \"then let it dry for two days\"')",
        "reasoning": "string (how duration was determined)"
      },
      "screen_time_to_real_time_ratio": "number or null (e.g., 2.5 if 10s on screen = 25s real)",
      "process_type": "string (enum: active_work|passive_wait|drying|curing|heating|cooling|other)",
      "visual_markers": [
        {
          "marker": "string (e.g., 'paint is visibly wet')",
          "timestamp": "MM:SS.mmm",
          "evidence_reference": "frame_id"
        }
      ]
    }
  ],
  "total_real_world_time": {
    "value": "number",
    "unit": "string (enum: hours|days)",
    "estimate_confidence": 0.0-1.0
  },
  "caveats": [
    "string (e.g., 'Drying time varies by humidity and temperature; video does not specify conditions')"
  ]
}
```

**What it must NOT do**:
- Guess if not stated explicitly or inferable from visual state change.
- Assume standard drying/curing times not mentioned in the video.
- Conflate "video shows X seconds" with "the process takes X seconds."
- Ignore off-screen time.

**Failure Mode**: Process time not mentioned or visually inferrable. Persona returns `null` for `real_world_duration` and flags as "requires_manual_annotation."

**Runs**: If content_category includes `tutorial` OR `woodworking` OR `craft` (determined by A2 routing).

**Retries**: Max 1 (second pass may re-search transcript more carefully).

---

### 3.2.6 PERSONA B3: Key-Point and Differentiator Analyst

**Name**: Key Insight Extractor

**Purpose**: Identify moments that are novel, surprising, or particularly instructive. What makes this video worth watching? What are the non-obvious takeaways?

**Inputs**:
- Transcript (from A3)
- Visual frames (from B1)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "key_insight_extractor",
  "key_moments": [
    {
      "timestamp": "MM:SS.mmm",
      "insight": "string (1-2 sentences; what is novel or important here)",
      "type": "string (enum: technique|tip|warning|unexpected_result|counterintuitive|efficiency_gain|error_correction|setup_detail)",
      "evidence_reference": "string (transcript_span or frame_id or both)",
      "why_notable": "string (explanation of significance)",
      "contrast_with_prior": "string or null (e.g., 'This contradicts the standard approach shown at 01:23')"
    }
  ],
  "differentiators": [
    {
      "differentiator": "string (e.g., 'Uses steel wool instead of sandpaper for finishing')",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "string",
      "impact": "string (how does this change the outcome or efficiency?)"
    }
  ],
  "overall_theme": "string (one sentence capturing the core theme or innovation)"
}
```

**What it must NOT do**:
- State opinions about quality or entertainment value.
- Assume the viewer will find something interesting (report what is objectively notable).
- Invent insights not present in the video.

**Failure Mode**: Video has no clear novel points. Persona returns empty arrays and notes "Video is standard/routine."

**Runs**: Always (parallel with B1/B2).

**Retries**: Max 1.

---

### 3.2.7 PERSONA B4: Temporal Structure Analyst

**Name**: Pacing and Structure Mapper

**Purpose**: Identify the narrative/procedural structure of the video: chapters, pacing, transitions between topics. How is the content organized?

**Inputs**:
- Transcript (from A3)
- Scene changes and visual markers (from B1)
- Content category (from A2)
- Key moments (from B3)

**Output JSON Schema**:
```json
{
  "persona": "structure_analyst",
  "chapters": [
    {
      "chapter_id": "string (e.g., 'intro', 'step_1', 'conclusion')",
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "duration_seconds": "number",
      "title": "string (inferred from content, not stated explicitly)",
      "transcript_span": "string (e.g., 'seg_001:seg_010')",
      "visual_state": "string (description of what's on screen during this chapter)",
      "purpose": "string (enum: introduction|explanation|demonstration|transition|summary|conclusion)"
    }
  ],
  "pacing_analysis": {
    "average_chapter_duration": "number (seconds)",
    "longest_chapter": "string (chapter_id)",
    "shortest_chapter": "string (chapter_id)",
    "pacing_rhythm": "string (enum: steady|accelerating|decelerating|variable)"
  },
  "transitions": [
    {
      "from_chapter": "string",
      "to_chapter": "string",
      "timestamp": "MM:SS.mmm",
      "transition_style": "string (enum: abrupt|gradual|narrative_bridge|technical_cut)"
    }
  ],
  "overall_structure": "string (e.g., 'Linear step-by-step tutorial with intro/outro')"
}
```

**What it must NOT do**:
- Evaluate whether the pacing is good or bad.
- Assume chapter boundaries if not marked in the video.
- Summarize content (describe structure only).

**Failure Mode**: Video is unstructured or continuous. Persona divides into equal-length segments and notes "No clear chapter markers detected."

**Runs**: Always (parallel with B1/B2/B3).

**Retries**: Max 1.

---

### 3.2.8 PERSONA C1: UI/Step Analyst for Software Tutorials

**Name**: Software UI and Procedure Tracer

**Purpose**: For screen-share or software-tutorial videos, trace the exact buttons clicked, screen states, menus opened, form fields filled. Enough detail that a reader could reproduce the steps without watching the video.

**Inputs**:
- Visual frames (from B1)
- Transcript (from A3)
- Content category (from A2; only runs if category includes "software_tutorial" or "screen_share_tutorial")

**Output JSON Schema**:
```json
{
  "persona": "ui_step_analyst",
  "ui_context": {
    "application": "string (e.g., 'Google Drive web interface')",
    "url": "string or null",
    "application_version": "string or null",
    "browser": "string or null (e.g., 'Chrome, Version 128')"
  },
  "procedure_steps": [
    {
      "step_number": "number",
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "action": "string (e.g., 'Click button labeled \"New\"")",
      "target_ui_element": {
        "element_type": "string (enum: button|menu|text_field|dropdown|checkbox|link|other)",
        "label": "string",
        "location": "string (e.g., 'top-right corner')",
        "state_before": "string (e.g., 'enabled, blue')",
        "state_after": "string (e.g., 'disabled, greyed out')"
      },
      "screen_state_before": "string (frame_id reference)",
      "screen_state_after": "string (frame_id reference)",
      "result": "string (e.g., 'Dialog box opens')",
      "transcript_excerpt": "string (what narrator said during this step)"
    }
  ],
  "form_fills": [
    {
      "field_label": "string",
      "field_type": "string (enum: text|email|number|date|dropdown|checkbox|other)",
      "value_entered": "string",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "frame_id",
      "screen_capture_after": "string (frame_id)"
    }
  ],
  "error_states": [
    {
      "timestamp": "MM:SS.mmm",
      "error_description": "string",
      "recovery_action": "string",
      "evidence_reference": "frame_id"
    }
  ],
  "prerequisites": [
    "string (e.g., 'Logged into Google account', 'File must be readable by others')"
  ]
}
```

**What it must NOT do**:
- Explain the UI (describe only what is shown).
- Judge whether the UI is intuitive or confusing.
- Generalize across different versions or browsers without evidence.

**Failure Mode**: Screen is unclear, text is too small, or actions are ambiguous. Persona flags with confidence <0.70 and requests zoom/clarification.

**Runs**: If content_category includes "software_tutorial" OR "screen_share_tutorial" (from A2 routing).

**Retries**: Max 2.

---

### 3.2.9 PERSONA C2: Tools, Materials, and Setup Extractor

**Name**: Tools and Materials Inventory

**Purpose**: Extract a comprehensive list of tools, materials, supplies, settings, and prerequisites needed to replicate the work shown. Focus on concrete, transferable details (grit, speed, angle, settings). Critical for R5.2, R5.6.

**Inputs**:
- Transcript (from A3)
- Visual frames (from B1), especially close-ups of tools, materials, labels
- Content category (from A2)
- UI/step analyst output (from C1) if applicable

**Output JSON Schema**:
```json
{
  "persona": "tools_materials_extractor",
  "tools": [
    {
      "tool_name": "string (e.g., 'oscillating sander')",
      "specific_model": "string or null (e.g., 'Dewalt DWE6423K')",
      "brand": "string or null",
      "purpose": "string (e.g., 'Initial surface preparation')",
      "timestamp_visible": "MM:SS.mmm",
      "evidence_reference": "frame_id",
      "alternatives": [
        "string (e.g., 'Hand sanding, palm sander, belt sander')"
      ],
      "must_have": true|false
    }
  ],
  "materials": [
    {
      "material": "string (e.g., 'walnut wood')",
      "specification": "string (e.g., '8/4 rough-sawn', '3/4\" thick', 'S4S, kiln-dried')",
      "quantity": "string (e.g., '1 bd-ft', '6 pieces')",
      "source": "string or null (e.g., 'hardwood specialty supplier')",
      "timestamp_visible": "MM:SS.mmm",
      "evidence_reference": "frame_id or transcript",
      "must_have": true|false
    }
  ],
  "consumables": [
    {
      "consumable": "string (e.g., 'sandpaper')",
      "grit": "string or null (e.g., '120, 180, 220')",
      "type": "string or null (e.g., 'open-coat')",
      "quantity": "string or null",
      "evidence_reference": "frame_id or transcript"
    }
  ],
  "settings_and_parameters": [
    {
      "parameter": "string (e.g., 'sanding speed')",
      "value": "string (e.g., '5000 RPM')",
      "unit": "string",
      "evidence_reference": "frame_id or transcript",
      "critical_to_outcome": true|false
    }
  ],
  "safety_equipment": [
    {
      "equipment": "string (e.g., 'dust mask')",
      "type": "string or null (e.g., 'N95')",
      "evidence_reference": "frame_id or transcript"
    }
  ],
  "prerequisites": [
    {
      "prerequisite": "string (e.g., 'Wood must be acclimated to workshop humidity')",
      "evidence_reference": "transcript_span"
    }
  ],
  "cost_notes": "string or null (if materials/tools cost or sourcing is discussed)"
}
```

**What it must NOT do**:
- Invent tools or materials not mentioned or visible.
- Recommend alternatives not shown in the video.
- Provide generic supply lists (only extract what is in the video).
- Estimate costs if not stated.

**Failure Mode**: Tools/materials not clearly visible or named. Persona flags as "inference_needed" and includes best-guess with low confidence.

**Runs**: If content_category includes "woodworking_tutorial", "craft", "diy", or "software_tutorial" (from A2 routing).

**Retries**: Max 2 (second pass may listen more carefully to any spoken names or brand mentions).

---

### 3.2.10 PERSONA C3: Safety and Prerequisites Analyst

**Name**: Safety and Prerequisite Auditor

**Purpose**: Extract safety warnings, hazards, personal protective equipment (PPE) required, and prerequisite knowledge or skills. Prevent user injury or failed attempts.

**Inputs**:
- Transcript (from A3)
- Visual frames showing unsafe or cautious behavior (from B1)
- Tools/materials (from C2)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "safety_auditor",
  "explicit_warnings": [
    {
      "warning": "string (e.g., 'Do not touch the blade while the machine is running')",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "transcript_span",
      "severity": "string (enum: critical|high|medium|low)",
      "related_tool": "string or null"
    }
  ],
  "observed_hazards": [
    {
      "hazard": "string (e.g., 'Sharp blade')",
      "timestamp_visible": "MM:SS.mmm",
      "evidence_reference": "frame_id",
      "severity": "string (enum: critical|high|medium|low)",
      "mitigation": "string (what is shown in the video or implied)",
      "grounded": true|false
    }
  ],
  "required_ppe": [
    {
      "item": "string (e.g., 'safety glasses')",
      "explicitly_shown": true|false,
      "explicitly_stated": true|false,
      "evidence_reference": "frame_id or transcript"
    }
  ],
  "prerequisites_knowledge": [
    {
      "prerequisite": "string (e.g., 'Familiarity with power tools')",
      "evidence_reference": "transcript_span or 'inferred_from_complexity'",
      "grounded": true|false
    }
  ],
  "prerequisites_physical": [
    {
      "prerequisite": "string (e.g., 'Access to workshop with dust collection')",
      "evidence_reference": "transcript_span or frame_id"
    }
  ],
  "common_mistakes_or_pitfalls": [
    {
      "mistake": "string",
      "consequence": "string",
      "evidence_reference": "frame_id or transcript or 'inferred_from_process'"
    }
  ]
}
```

**What it must NOT do**:
- Add generic safety advice not mentioned in the video.
- Assume prior knowledge.
- Omit hazards observed in the video simply because they are not explicitly warned.

**Failure Mode**: No hazards or safety concerns evident. Persona returns empty arrays for warnings/hazards but includes any stated prerequisites.

**Runs**: If content_category includes "woodworking_tutorial", "craft", "diy", "tool_use" (from A2 routing).

**Retries**: Max 1.

---

### 3.2.11 PERSONA C4: Transferability and Generalization Analyst

**Name**: Technique Transferability Mapper

**Purpose**: Extract technique parameters and principles in a way that they can be applied to a different project. Not just "stain the wood," but "apply stain with x% dilution, wait Y minutes before wiping, in Z° humidity." Critical for R5.6.

**Inputs**:
- Transcript (from A3)
- Visual frames with close-ups of techniques (from B1)
- Tools/materials/settings (from C2)
- Process timeline (from B2)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "transferability_analyst",
  "techniques_extracted": [
    {
      "technique_name": "string (e.g., 'surface sanding')",
      "application_context": "string (e.g., 'walnut tabletop refinishing')",
      "generic_principle": "string (what is the core technique independent of the specific project?)",
      "critical_parameters": [
        {
          "parameter": "string (e.g., 'grit progression')",
          "value_observed": "string (e.g., '80 → 120 → 180 → 220')",
          "evidence_reference": "transcript or frame",
          "why_critical": "string (e.g., 'Coarser grits remove material; finer grits create finish-ready surface')",
          "variation_tolerance": "string or null (e.g., 'Can substitute 150 for 120 if material is soft')"
        }
      ],
      "non_critical_variations": [
        "string (e.g., 'Brand of sandpaper may vary')"
      ],
      "environmental_dependencies": [
        "string (e.g., 'Humidity above 60% increases drying time')"
      ]
    }
  ],
  "adaptation_guidance": [
    {
      "scenario": "string (e.g., 'Applying this technique to a different wood species')",
      "guidance": "string (what would need to adjust?)",
      "evidence": "string or null"
    }
  ],
  "analogous_processes": [
    {
      "analogous_process": "string (e.g., 'If not using walnut, similar hardwoods are maple, oak, ash')",
      "evidence_reference": "transcript or 'inferred_from_technique'"
    }
  ]
}
```

**What it must NOT do**:
- Recommend specific alternatives not hinted at in the video.
- Assume transferability without evidence.
- Over-generalize ("this works on any wood" if the video only shows walnut).

**Failure Mode**: Video technique is highly specific to one material/tool. Persona notes "low transferability" and explains why.

**Runs**: If content_category includes "tutorial" or "woodworking" or "craft" (from A2 routing).

**Retries**: Max 1.

---

### 3.2.12 PERSONA D1: Music and Sound Design Analyst

**Name**: Audio and Music Extractor

**Purpose**: Identify all music, sound effects, ambient sounds. Extract genre, mood, timing, and purpose. Critical for inspiration videos (R1.3.d) and music-driven content.

**Inputs**:
- Full audio track (from A3)
- Visual frames (from B1)
- Transcript (from A3)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "music_analyst",
  "music_tracks": [
    {
      "segment_id": "string",
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "genre": "string (e.g., 'lo-fi hip-hop', 'ambient', 'drone')",
      "mood": "string (e.g., 'relaxing', 'energetic', 'melancholic')",
      "tempo_bpm": "number or null",
      "instrumentation": "string (e.g., 'lo-fi beats, pad synth, vinyl crackle')",
      "visible_evidence": "frame_id or null (if music video or instrument is shown)",
      "purpose_in_video": "string (enum: background|main_focus|transition|climax|emphasis)"
    }
  ],
  "sound_effects": [
    {
      "sound": "string (e.g., 'click of mouse', 'sound of wood being sanded')",
      "timestamp": "MM:SS.mmm",
      "natural_or_added": "string (enum: natural|likely_foley|likely_library|unknown)",
      "evidence_reference": "frame_id or transcript"
    }
  ],
  "ambient_sounds": [
    {
      "sound": "string (e.g., 'workshop background hum', 'quiet traffic')",
      "prominence": "string (enum: prominent|subtle|barely_noticeable)",
      "evidence_reference": "timestamp_range"
    }
  ],
  "silence_and_pauses": [
    {
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "duration_seconds": "number",
      "context": "string (e.g., 'Pause between spoken sentences', 'Waiting for glue to dry')"
    }
  ],
  "overall_sonic_identity": "string (e.g., 'AI-generated beats with ambient pad, minimal human speech')"
}
```

**What it must NOT do**:
- Identify artist or song name by copyright inference only (only state if visible/stated).
- Judge whether the music is good or fits the video.
- Assume music licensing or rights.

**Failure Mode**: Music is copyrighted/unidentified. Persona describes it only by genre and characteristics.

**Runs**: If content_category includes "music_inspiration", "ai_generated", or has background music (from A2 routing).

**Retries**: Max 1.

---

### 3.2.13 PERSONA D2: Aesthetic, Mood, and Visual Style Analyst

**Name**: Visual Style and Mood Extractor

**Purpose**: For inspiration videos and mood-driven content, extract color grading, filters, lens characteristics, editing rhythm, visual effects. Capture the "look" of the video.

**Inputs**:
- Visual frames across the entire video (from B1)
- Music and sound (from D1)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "aesthetic_analyst",
  "color_grading": {
    "overall_tone": "string (e.g., 'warm, desaturated, sepia-shifted')",
    "color_palette": [
      "string (e.g., 'warm browns', 'cool blues', 'muted pastels')"
    ],
    "evidence": "frame_id"
  },
  "filters_and_effects": [
    {
      "filter_type": "string (enum: vignette|grain|blur|color_shift|distortion|other)",
      "intensity": "string (enum: subtle|moderate|heavy)",
      "evidence_reference": "frame_id"
    }
  ],
  "lens_and_capture": {
    "apparent_lens_type": "string or null (e.g., 'wide-angle', 'macro', 'telephoto')",
    "depth_of_field": "string (enum: shallow|moderate|deep|variable)",
    "motion_style": "string (enum: static|slow_pan|handheld|dynamic|timelapsed)",
    "evidence": "frame_id"
  },
  "editing_rhythm": {
    "cut_frequency": "string (enum: static_long_takes|moderate_cuts|fast_cuts)",
    "transition_style": "string (enum: hard_cuts|fades|crossfades|creative_transitions)",
    "pacing_feel": "string (e.g., 'meditative', 'energetic', 'deliberate')"
  },
  "overall_mood": "string (e.g., 'calm and contemplative with warm nostalgic tones')",
  "inspiration_vectors": [
    "string (e.g., 'Cinematography style: indie film', 'Color grade: 80s VHS', 'Editing: music video')"
  ]
}
```

**What it must NOT do**:
- Judge quality or taste.
- Assume creative intent beyond what is visible.
- Name specific camera models without evidence.

**Failure Mode**: Video has no consistent aesthetic. Persona notes "no coherent visual style" or "multiple distinct aesthetic zones."

**Runs**: If content_category includes "music_inspiration", "ai_generated", "instagram", or "aesthetic_driven" (from A2 routing).

**Retries**: Max 1.

---

### 3.2.14 PERSONA D3: Meeting Main-Points Extractor

**Name**: Meeting Minutes and Key Discussion Points Extractor

**Purpose**: For video conference, meeting, or discussion-based videos, extract agenda, key decisions, action items, and main points.

**Inputs**:
- Transcript (from A3)
- Visual frames showing slides or shared screens (from B1)
- Content category (from A2)

**Output JSON Schema**:
```json
{
  "persona": "meeting_extractor",
  "meeting_metadata": {
    "inferred_title": "string or null",
    "inferred_date": "string or null",
    "inferred_participants": ["string (names if identifiable, or roles)"],
    "duration": "number (seconds)",
    "source": "string (video conference platform name if detectable)"
  },
  "agenda": [
    {
      "item_number": "number",
      "topic": "string",
      "start_time": "MM:SS.mmm",
      "transcript_span": "string"
    }
  ],
  "key_points": [
    {
      "topic": "string",
      "point": "string",
      "timestamp": "MM:SS.mmm",
      "speaker": "string or null",
      "evidence_reference": "transcript_span"
    }
  ],
  "decisions": [
    {
      "decision": "string",
      "timestamp": "MM:SS.mmm",
      "consensus": true|false,
      "evidence_reference": "transcript_span"
    }
  ],
  "action_items": [
    {
      "action": "string",
      "assigned_to": "string or null",
      "deadline": "string or null",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "transcript_span"
    }
  ],
  "unresolved_questions": [
    {
      "question": "string",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "transcript_span"
    }
  ]
}
```

**What it must NOT do**:
- Invent attendees or organizational context not stated.
- Assume meeting purpose or importance.
- Interpret side comments as key points without clear emphasis.

**Failure Mode**: Meeting has no clear structure or points. Persona segments by speaker/topic and notes "unstructured discussion."

**Runs**: If content_category includes "video_conference", "meeting", or "discussion" (from A2 routing).

**Retries**: Max 1.

---

### 3.2.15 PERSONA E1: Tutorial Completeness Auditor

**Name**: Tutorial Completeness Verifier

**Purpose**: For tutorial-type videos, verify that all steps are present and no gaps exist. Compare claimed step count with actual coverage.

**Inputs**:
- Transcript (from A3)
- Visual frames (from B1)
- Temporal structure (from B4)
- UI/step analysis (from C1)
- Process timeline (from B2)

**Output JSON Schema**:
```json
{
  "persona": "completeness_auditor",
  "claimed_steps": "number or null (if stated in video)",
  "observed_steps": "number",
  "step_list": [
    {
      "step_number": "number",
      "title": "string",
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "coverage_quality": "string (enum: complete|partial|inferred|missing)",
      "confidence": 0.0-1.0,
      "evidence_reference": "transcript_span or frame_id"
    }
  ],
  "gaps": [
    {
      "gap_description": "string (e.g., 'Jump from step 3 to step 5; step 4 not shown')",
      "affected_time_range": "MM:SS.mmm—MM:SS.mmm",
      "severity": "string (enum: critical|high|medium|low)",
      "reason": "string (enum: cut_from_video|implied_by_narrator|unclear)"
    }
  ],
  "completeness_verdict": {
    "is_complete": true|false,
    "percentage_coverage": 0.0-1.0,
    "confidence": 0.0-1.0
  },
  "notes": "string or null"
}
```

**What it must NOT do**:
- Assume steps not shown were intended.
- Demand perfection (some tutorials are intentionally abbreviated).
- Hallucinate missing steps.

**Failure Mode**: Coverage is incomplete. Persona flags gaps and marks confidence low.

**Runs**: If content_category includes "tutorial" or "how_to" (from A2 routing).

**Retries**: Max 1.

---

### 3.2.16 PERSONA E2: Screenshot↔Audio Match Verifier

**Name**: Visual-Audio Consistency Auditor

**Purpose**: Ensure that what is on screen matches what is being said. Flag moments where narrator talks about something not visible, or where there is a mismatch.

**Inputs**:
- Transcript with timestamps (from A3)
- Visual frames with timestamps (from B1)
- Concurrent_transcript_span from B1 (already aligned)

**Output JSON Schema**:
```json
{
  "persona": "visual_audio_consistency_auditor",
  "consistent_segments": [
    {
      "timestamp": "MM:SS.mmm",
      "description": "string (e.g., 'Narrator says \"click the Save button\" and blue Save button is visible in top-right')",
      "evidence": "transcript_span and frame_id"
    }
  ],
  "mismatches": [
    {
      "timestamp": "MM:SS.mmm",
      "issue_type": "string (enum: visual_lag|visual_ahead|referencing_off_screen|unclear_reference|contradiction)",
      "description": "string (e.g., 'Narrator refers to \"the dropdown menu\" but menu is not visible on screen at this time')",
      "narration": "string (what was said)",
      "visual_state": "string (what is actually on screen)",
      "transcript_reference": "string",
      "frame_reference": "string",
      "impact": "string (enum: confusing|critical_for_understanding|minor|breaks_instructions)"
    }
  ],
  "quality_issues": [
    {
      "issue": "string (enum: text_unreadable|image_too_small|movement_too_fast|audio_unclear_at_sync_point)",
      "timestamp": "MM:SS.mmm",
      "evidence_reference": "frame_id or transcript"
    }
  ],
  "overall_sync_quality": {
    "is_synchronized": true|false,
    "confidence": 0.0-1.0,
    "notes": "string or null"
  }
}
```

**What it must NOT do**:
- Assume the narrator is wrong (report discrepancy neutrally).
- Ignore intentional off-screen references if they are clear in context.

**Failure Mode**: Sync issues detected. Persona flags severity and location.

**Runs**: Always (can run in parallel with E1).

**Retries**: Max 1.

---

### 3.2.17 PERSONA E3: Contradiction and Hallucination Auditor

**Name**: Internal Consistency and Accuracy Checker

**Purpose**: Flag contradictions within the video itself (narrator says different things at different times), inconsistencies with visible facts, and inferences that lack grounding.

**Inputs**:
- All prior persona outputs (A1–E2)
- Transcript (from A3)
- Visual evidence (from B1)

**Output JSON Schema**:
```json
{
  "persona": "contradiction_auditor",
  "contradictions": [
    {
      "claim_1": {
        "statement": "string",
        "timestamp": "MM:SS.mmm",
        "evidence_reference": "transcript_span"
      },
      "claim_2": {
        "statement": "string",
        "timestamp": "MM:SS.mmm",
        "evidence_reference": "transcript_span or frame_id"
      },
      "conflict_description": "string",
      "severity": "string (enum: critical|high|medium|low)",
      "possible_resolution": "string or null"
    }
  ],
  "ungrounded_claims": [
    {
      "claim": "string",
      "timestamp": "MM:SS.mmm",
      "source_persona": "string (e.g., 'tools_materials_extractor')",
      "why_ungrounded": "string",
      "evidence_available": true|false
    }
  ],
  "inference_vs_fact_review": [
    {
      "statement": "string (from a persona output)",
      "source_persona": "string",
      "marked_as": "string (enum: fact|inference)",
      "should_be": "string (enum: fact|inference|unclear)",
      "correction_needed": true|false
    }
  ],
  "auditor_verdict": {
    "overall_reliability": "string (enum: high_confidence|moderate_confidence|low_confidence|requires_review)",
    "critical_issues": "number",
    "recommendations": ["string"]
  }
}
```

**What it must NOT do**:
- Invent contradictions that don't exist.
- Assume the video is authoritative on factual claims (e.g., if it says "epoxy dries in 2 hours" but product specs say 24 hours, flag as ungrounded).
- Penalize intentional repetition or reformulation.

**Failure Mode**: No issues found. Returns empty arrays.

**Runs**: After all A–E personas complete (sequential, QA checkpoint).

**Retries**: Sends ungrounded claims back to originating personas for remediation; up to max 2 rounds.

---

### 3.2.18 PERSONA E4: Coverage Auditor

**Name**: Timeline Coverage Inspector

**Purpose**: Verify that the entire video timeline is accounted for in visual analysis. Flag dark spots, unexplained gaps, or regions where coverage is unclear.

**Inputs**:
- Key frames (from B1)
- Scene changes (from B1)
- Temporal structure (from B4)
- Visual coverage_fraction metric (from B1)

**Output JSON Schema**:
```json
{
  "persona": "coverage_auditor",
  "total_video_duration": "number (seconds)",
  "analyzed_duration": "number (seconds)",
  "coverage_percentage": 0.0-1.0,
  "covered_segments": [
    {
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "description": "string",
      "confidence": 0.0-1.0
    }
  ],
  "gaps": [
    {
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "duration_seconds": "number",
      "reason": "string (enum: black_screen|unclear|no_frames_extracted|motion_blur|other)",
      "severity": "string (enum: critical|high|medium|low)"
    }
  ],
  "unclear_regions": [
    {
      "start_time": "MM:SS.mmm",
      "end_time": "MM:SS.mmm",
      "issue": "string",
      "confidence_in_analysis": 0.0-1.0
    }
  ],
  "recommendations": [
    "string (e.g., 'Re-sample frames at 2 fps in region 03:15—03:45 for clarity')"
  ]
}
```

**What it must NOT do**:
- Demand coverage of black screens or irrelevant moments.
- Penalize intentional artistic pauses.

**Failure Mode**: Coverage is incomplete. Persona identifies gaps and recommends re-analysis or manual inspection.

**Runs**: After B1, B4 complete (sequential QA checkpoint).

**Retries**: If gaps are detected, triggers a re-run of B1 with denser frame sampling in affected regions.

---

### 3.2.19 PERSONA F1: Final Assembler and QA Editor

**Name**: Output Synthesizer and Final Review

**Purpose**: Combine all persona outputs into a coherent, structured markdown file. Resolve any remaining conflicts. Ensure grounding is complete. Format for readability and downstream AI consumption (R5).

**Inputs**:
- All persona outputs (A1–E4)
- Original video file (metadata check)
- Transcript (from A3)

**Output**:
A markdown file (single, no JSON schema) following the structure defined in R4:
- R4.1: All collected details
- R4.2: All timestamps
- R4.3: Meeting main points (if applicable)
- R4.4: General description
- R4.5: Short summary
- R4.6: Word-by-word transcript
- R4.7: Screenshots
- R4.8–4.9: Explanations and tutorials with screenshots
- R4.10: Tool tutorials (if applicable)
- R4.11: Music/inspiration details (if applicable)
- R4.12: Standalone tutorial format

**What it must NOT do**:
- Invent new content.
- Resolve conflicts by choosing one persona's answer without flagging (mark conflicts explicitly if unresolvable).
- Lose any grounding information.
- Add editorial tone or opinions.

**Failure Mode**: Output contains unresolved conflicts or ungrounded claims. Persona marks sections as "requires_manual_review" and documents the issue.

**Runs**: Last pass (after all QA checkpoints).

**Retries**: Max 2 (back to E3 auditor if critical issues remain).

---

## 3.3 Orchestration and Routing

### 3.3.1 Execution Graph

```
Phase 1 (Parallel):
├─ A1 (Source Identifier) ─────┐
├─ A3 (Transcript Custodian) ──┤
└─ (Video load & metadata) ────┤
                               │
                               ▼
Phase 2 (Sequential):
                  A2 (Categorizer)  ← reads A1, A3, video metadata
                       │
                       ├─ routing_recommendations.activate_personas
                       │
                       ▼
Phase 3 (Parallel, conditional on A2 routing):
├─ B1 (Frame Analyst) ─────────────────────┐
├─ B2 (Duration Analyst) ────────────────┬─┤ (if tutorial)
├─ B3 (Key Insights) ─────────────────┬──┤
├─ B4 (Structure Mapper) ────────────┤  │
├─ C1 (UI/Step Analyst) ────────────┤  │ (if software_tutorial)
├─ C2 (Tools/Materials) ────────────┤  │ (if diy/craft)
├─ C3 (Safety Auditor) ─────────────┤  │ (if diy/craft)
├─ C4 (Transferability) ────────────┤  │ (if tutorial)
├─ D1 (Music Analyst) ──────────────┤  │ (if music_inspiration)
└─ D2 (Aesthetic Analyst) ─────────┘  │ (if inspiration)
                                        │ (if meeting)
                    D3 (Meeting Extractor)
                                       │
                                       ▼
Phase 4 (Sequential, QA):
├─ E1 (Completeness Auditor) ┐       (if tutorial)
├─ E2 (Visual-Audio Auditor) ├─────→ E3 (Contradiction Auditor)
├─ E4 (Coverage Auditor) ────┘
         │
         ├─ Ungrounded claims/conflicts detected?
         │  ├─ Yes → Send back to originating personas (max 2 rounds)
         │  └─ No → Proceed
         │
         ▼
Phase 5 (Sequential, Final):
└─ F1 (Synthesizer + QA Editor) ──→ Output markdown file
```

### 3.3.2 Conditional Routing Logic

```python
# Pseudocode for A2 routing output
routing_map = {
    "screen_share_tutorial": activate=[A3, B1, B3, B4, C1, C2, C4, E1, E2, E4, F1],
    "talking_head": activate=[A3, B1, B3, B4, E2, F1],
    "video_conference": activate=[A3, B1, D3, E2, F1],
    "music_inspiration": activate=[A3, B1, D1, D2, E2, F1],
    "woodworking_tutorial": activate=[A3, B1, B2, B3, B4, C2, C3, C4, E1, E2, E4, F1],
    "software_tutorial": activate=[A3, B1, B3, B4, C1, C2, E1, E2, E4, F1],
    "mixed_media": activate=[all] # All personas
}
```

### 3.3.3 Number of Passes

- **Pass 1**: A1, A3 (parallel)
- **Pass 2**: A2
- **Pass 3**: Conditional personas (B1–D3) run in parallel
- **Pass 4**: E1–E4 run sequentially as QA checkpoints
- **Pass 5**: F1 synthesizes final output
- **Remediation passes** (0–2 additional): E3 sends ungrounded claims back to originating personas for re-grounding

---

## 3.4 Grounding and Evidence: Enforcement Mechanism

### 3.4.1 Validation Schema

Every persona output is validated against a base JSON schema that enforces grounding:

```json
{
  "definitions": {
    "grounded_claim": {
      "type": "object",
      "properties": {
        "claim": {"type": "string"},
        "type": {"enum": ["fact", "inference"]},
        "timestamp": {"type": "string", "pattern": "^[0-9]{2}:[0-9]{2}\\.[0-9]{3}$"},
        "evidence_reference": {
          "type": "string",
          "pattern": "^(transcript\\[seg_[0-9]+:[0-9]+\\]|frame_[0-9]{2}:[0-9]{2}:[0-9]{2}\\.[0-9]{3}|ocr@frame_[0-9]{2}:[0-9]{2}:[0-9]{2}\\.[0-9]{3}|audio_frequency@[0-9]{2}:[0-9]{2}:[0-9]{2}|metadata_[a-z_]+)$"
        },
        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
      },
      "required": ["claim", "type", "timestamp", "evidence_reference"],
      "additionalProperties": false
    }
  }
}
```

**Process**:
1. Persona produces JSON output.
2. Orchestrator validates against persona-specific schema (which includes the base grounded_claim definition).
3. If any claim fails validation (missing timestamp, evidence_reference, or improper format):
   - Orchestrator returns a structured error to the persona.
   - Persona is re-queried with context: "These claims require grounding: [list]. Add timestamp, evidence_reference, and type (fact or inference) to each."
   - Max 2 retry rounds before flagging as "requires_manual_review."

### 3.4.2 Evidence Reference Format

All evidence must use one of these formats:

| Type | Format | Example | Persona Source |
|------|--------|---------|-----------------|
| Transcript span | `transcript[seg_NNN:seg_MMM]` | `transcript[seg_045:seg_048]` | A3 |
| Frame ID | `frame_HH:MM:SS.mmm` | `frame_00:02:15.500` | B1 |
| OCR text | `ocr@frame_HH:MM:SS.mmm` | `ocr@frame_00:01:30.200` | B1 (derived) |
| Audio analysis | `audio_ANALYSIS_TYPE@HH:MM:SS.mmm` | `audio_frequency@00:03:45.000` | D1 |
| Metadata | `metadata_FIELD` | `metadata_creation_date` | A1 |
| Inferred (no direct evidence) | `inferred_from_REASONING` | `inferred_from_process_description` | (any) |

Inferred claims MUST be explicitly labeled with type: "inference" and include a confidence score <1.0.

---

## 3.5 Conflict Resolution

When two personas disagree about a fact, the following resolution strategy applies:

### 3.5.1 Resolution Hierarchy

1. **Frame-level visual evidence (B1)** beats inferred claims.
   - *Example*: B1 says "button is blue"; C1 says "button is grey" → B1 wins (visual source).

2. **Transcript (A3)** beats inferred or implicit claims.
   - *Example*: A3 says narrator says "leave for 2 hours"; B2 infers "must be at least 4 hours based on process" → A3 wins (explicit statement).

3. **Grounded claim** beats ungrounded claim.
   - *Example*: C2 says "sandpaper grit is 120" with frame reference; B3 infers "likely 150 based on tool type" → C2 wins.

4. **Higher confidence score** (when both grounded).
   - *Example*: Both have frame references; B1 has confidence 0.95, B3 has confidence 0.70 → B1 wins.

5. **If still tied**: Flag in output as a conflict and include both claims with their evidence, allowing downstream user or F1 to decide.

### 3.5.2 Example Conflict Resolution

**Conflict**: 
- B2 (Duration Analyst): "Glue curing process takes 24 hours (transcript: 'leave it overnight')"
- C4 (Transferability Analyst): "Typical wood glue cures in 48 hours for full strength; overnight is initial set only"

**Resolution**:
- B2's claim is grounded in transcript (direct statement), confidence 0.95.
- C4's claim is inference from domain knowledge, confidence 0.70.
- **Outcome**: B2 wins. F1 includes both: "Video states overnight (≈12–16 hours); note that full cure may require longer per product specs."

---

## 3.6 Prompt Design and Context Budget

Each persona receives a carefully scoped prompt to avoid hallucination and excessive token consumption.

### 3.6.1 Context Budget Per Persona

| Persona | Max Context | What It Reads | What It Outputs |
|---------|-------------|---------------|-----------------|
| A1 | ~2 KB | Metadata, 30 frames, container/codec info | JSON (~1 KB) |
| A2 | ~3 KB | A1 output, first 10s audio spectrogram, 5 key frames | JSON (~1.5 KB) with routing |
| A3 | ~unlimited | Full audio track | Full transcript JSON (~varies with duration) |
| B1 | ~8 KB | Key frames (1 fps) with transcript alignment, visual markers | JSON (~varies, ~5–10 KB typical) |
| B2 | ~5 KB | Process transcript snippets, visual markers for state changes | JSON (~1.5 KB) |
| B3 | ~4 KB | Transcript key passages, visual moments | JSON (~2 KB) |
| B4 | ~5 KB | Transcript, scene changes, key moments | JSON (~2 KB) |
| C1 | ~6 KB | UI frames, local transcript excerpts, prior C1 context if applicable | JSON (~3–4 KB) |
| C2 | ~6 KB | Transcript excerpts (tool/material mentions), zoom frames, prior C2 context | JSON (~2–3 KB) |
| C3 | ~4 KB | Transcript safety mentions, hazard frames, prior C3 context | JSON (~2 KB) |
| C4 | ~5 KB | Technique descriptions, transcript, prior C2/C4 context | JSON (~2 KB) |
| D1 | ~4 KB | Audio clip excerpts (spectrogram + waveform), timestamps | JSON (~1.5 KB) |
| D2 | ~5 KB | Color-sampled frames, editing cuts, prior D2 context | JSON (~2 KB) |
| D3 | ~6 KB | Transcript excerpts (topic/decision sentences), slide frames | JSON (~2 KB) |
| E1 | ~8 KB | B4 chapters, transcript claim count, visual coverage | JSON (~1.5 KB) |
| E2 | ~6 KB | Concurrent_transcript_span + visual markers from B1 | JSON (~2 KB) |
| E3 | ~20 KB | All prior persona outputs, contradiction matrix | JSON (~2–3 KB) |
| E4 | ~6 KB | B1 key_frames, temporal structure, coverage metrics | JSON (~1.5 KB) |
| F1 | ~50 KB | All persona outputs, transcript, selected frames | Markdown (~10–50 KB) |

### 3.6.2 Per-Persona Prompt Template

Each persona is prompted with this structure:

```
You are [PERSONA_NAME].
Your purpose: [PURPOSE]

INPUTS YOU WILL RECEIVE:
- [List of specific inputs with max size]

OUTPUT CONTRACT:
You must produce ONLY valid JSON matching this schema:
{schema for this persona}

GROUNDING RULES:
- Every factual claim requires a timestamp (MM:SS.mmm format) and evidence_reference.
- Claims without evidence_reference must be labeled type: "inference" with confidence <1.0.
- Do NOT invent detail. If you cannot find evidence, return empty arrays or null.

DO NOT:
- [List of what this persona must never do]

FAILURE MODE:
If you cannot complete the analysis (e.g., confidence <0.70), explicitly flag it in the output:
{
  "confidence": 0.65,
  "failure_reason": "string"
}

Proceed.
```

### 3.6.3 Token Optimization

- Personas receive frame arrays as **base64-encoded JPEG** (not raw RGB arrays).
- Transcript is **pre-chunked** into segments; personas receive only relevant segments by timestamp range.
- Large video metadata is **summarized** (resolution, codec, frame rate, duration) rather than raw file info.
- Redundant context (e.g., entire transcript when only one phrase is needed) is **omitted by default**; persona must explicitly request it if needed.

---

## 3.7 Cross-Checking and QA Pass

A rigorous multi-stage QA process ensures no hallucination, no lost detail, and complete grounding.

### 3.7.1 QA Checkpoints

**Checkpoint 1** (After Phase 3, before E):
- E3 (Contradiction Auditor) reads all A–D persona outputs.
- Identifies ungrounded claims and internal contradictions.
- Returns structured report: "These claims lack evidence: [list]."
- **Action**: Originating personas are re-queried with specific localization: "Re-examine 02:15—02:45. Justify your claim about [X] with timestamp and evidence_reference."

**Checkpoint 2** (During Phase 4):
- E1, E2, E4 run as sequential audits of tutorial completeness, sync quality, and coverage.
- Any failures are **escalated** to F1 (marked as "requires_manual_review").

**Checkpoint 3** (During Phase 5, F1):
- F1 reads all outputs and assembles markdown.
- If unresolved conflicts remain, F1 includes both versions and flags: **[CONFLICT]** marker in markdown.
- If grounding is incomplete, F1 flags: **[REQUIRES EVIDENCE]** marker.

### 3.7.2 Retry Logic

- **Tier 1 retry** (Persona re-queried with additional context): Max 2 rounds per persona per artifact.
- **Tier 2 retry** (Different persona consulted for same question): Max 1 round (e.g., if B1 failed to identify a tool, ask C2 to check for tool mentions in transcript).
- **Max total retries**: 3 rounds across all personas before flagging for manual review.

### 3.7.3 Escalation Criteria

An output is escalated to manual review if:
- Confidence <0.70 after 2 retry rounds.
- Evidence_reference does not resolve (format valid but evidence not found).
- Contradiction cannot be resolved by hierarchy.
- Coverage_fraction <0.80 in coverage auditor.
- Ungrounded claim persists after re-query.

---

## 3.8 Data Flow and Storage

### 3.8.1 Intermediate Artifacts

All persona outputs are stored as **JSON files** in a temporary directory:

```
analysis_run_ID/
├─ a1_source_identifier.json
├─ a2_categorizer.json
├─ a3_transcript_custodian.json
├─ b1_frame_analyst.json
├─ b2_duration_analyst.json
├─ ... (all personas)
├─ e3_contradiction_auditor.json
├─ e4_coverage_auditor.json
└─ f1_final_output.md
```

At end of run, only `f1_final_output.md` is moved to `outputs/` folder per R6.2.
JSON intermediate files are **retained for 7 days** to allow debugging/review, then deleted.

### 3.8.2 Screenshot References

Personas B1, C1, C2 reference frames and screenshots. By F1 assembly time, selected frames are:
- Exported as `.png` (lossless).
- Named per user example (R7): "Step 3 - Sanding surface@frame_00:01:30"
- Moved to `screenshots/{folder_name_per_R7}/` folder.
- Embedded in markdown with relative path: `![Step 3](../screenshots/folder_name/image.png)`

---

## 3.9 Personas Summary Table

| ID | Name | Category | Runs | Depends On | Output Size | Grounding |
|----|------|----------|------|------------|-------------|-----------|
| A1 | Source Identifier | Metadata | Always | Video file | ~1 KB | Metadata only |
| A2 | Content Categorizer | Classification | Always | A1, A3, video | ~1.5 KB | Observation only |
| A3 | Transcript Custodian | Audio | Always | Audio track | Varies | All claims grounded |
| B1 | Frame Analyst | Visual | Always | Video, A3 | 5–10 KB | Frame-grounded |
| B2 | Duration Analyst | Process | Conditional | A3, B1 | ~1.5 KB | Grounded or null |
| B3 | Key Insights | Analysis | Always | A3, B1 | ~2 KB | Grounded |
| B4 | Structure Mapper | Temporal | Always | A3, B1, B3 | ~2 KB | Grounded |
| C1 | UI/Step Analyst | UI | Conditional | B1, A3 | 3–4 KB | Frame-grounded |
| C2 | Tools/Materials | Inventory | Conditional | A3, B1 | 2–3 KB | Grounded or null |
| C3 | Safety Auditor | Safety | Conditional | A3, B1, C2 | ~2 KB | Grounded or observed |
| C4 | Transferability Analyst | Generalization | Conditional | A3, B1, C2, B2 | ~2 KB | Grounded |
| D1 | Music Analyst | Audio | Conditional | A3, B1 | ~1.5 KB | Audio-grounded |
| D2 | Aesthetic Analyst | Visual | Conditional | B1 | ~2 KB | Frame-grounded |
| D3 | Meeting Extractor | Discussion | Conditional | A3, B1 | ~2 KB | Grounded |
| E1 | Completeness Auditor | QA | Conditional | B4, A3, B1 | ~1.5 KB | Grounded |
| E2 | Sync Auditor | QA | Always | A3, B1 | ~2 KB | Grounded |
| E3 | Contradiction Auditor | QA | Sequential | All A–D | 2–3 KB | Meta-grounded |
| E4 | Coverage Auditor | QA | Sequential | B1, B4 | ~1.5 KB | Grounded |
| F1 | Synthesizer | Output | Final | All | 10–50 KB | (Compiled) |

---

## 3.10 Error Handling and Fallbacks

If a persona cannot complete (e.g., video corrupted, audio missing):

1. **A3 (Transcript) fails**: 
   - Mark as "audio_unavailable."
   - Continue; A2 routing marks "meeting_extractor" and "music_analyst" as skipped.
   - Other personas proceed without transcript (reduced confidence).

2. **B1 (Frame Analyst) fails**:
   - Mark as "video_unavailable."
   - Skip all visual personas (C1, D2, E2, E4).
   - Continue with audio-only personas (A3, D1, D3).

3. **A2 routing produces "mixed_media" or "unknown"**:
   - Activate ALL personas to maximize coverage.

4. **E3 detects >5 unresolvable contradictions**:
   - F1 outputs markdown with **[CONFLICT]** sections and recommendation for manual review.

---

## 3.11 Summary and Rationale

This persona system achieves R3, R2.4, R5.3, and R5.6 by:

1. **Narrow purpose per persona** → no hallucination, high precision.
2. **Mandatory grounding** → every claim is traceable to evidence or explicitly marked inference.
3. **Conditional activation** → no wasted analysis on irrelevant personas.
4. **Multi-stage QA** → contradictions and gaps caught and remediated before output.
5. **Structured JSON + markdown synthesis** → clean data flow, no information loss.
6. **Explicit evidence references** → downstream AI (Claude) can verify claims by reviewing video/frames/transcript.

The result is a "perfect complex markdown video details file" (R3.9) that is simultaneously:
- **Complete** (all perspectives analyzed).
- **Grounded** (all claims traceable).
- **Transferable** (techniques parameterized for reuse).
- **Standalone** (reader can follow without video).
- **Downstream-ready** (Claude can extract project time, tool lists, applicability).

