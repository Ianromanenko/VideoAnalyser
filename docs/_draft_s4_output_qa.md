# Section 4: Output Markdown Specification, Screenshots, QA, and Testing

*Maximum-effort implementation specification for markdown document generation, screenshot management, and comprehensive quality assurance.*

---

## 1. Complete Markdown Document Template Specification

Every output markdown file must follow this exact structure, section by section, in order. This template accommodates all video archetypes (tutorials, screen shares, woodworking, music/inspiration, meetings).

### 1.1 Front-Matter Metadata Block

```yaml
---
title: "{Video Title}"
source: "{Source of Origin}"
processed_date: "{DD.MM.YYYY}"
processed_time: "{HH:MM:SS}"
video_duration: "{HH:MM:SS}"
total_speakers: {n}
category: "{Category}"
tags: [{list_of_tags}]
transcript_status: "complete"
screenshot_count: {n}
tools_listed: {n}
materials_listed: {n}
qa_passed: true
qa_check_timestamp: "{YYYY-MM-DD HH:MM:SS UTC}"
schema_version: "1.0"
---
```

**Rationale for each field:**
- `title`: The video's original title or inferred title from content.
- `source`: Instagram recording, YouTube video, iPhone screen recording, Zoom call, professional video, other web source.
- `processed_date`: Date the analysis was conducted (DD.MM.YYYY format, per R7.2).
- `processed_time`: Exact time analysis completed (for reproducibility).
- `video_duration`: Total length of the source video (HH:MM:SS).
- `total_speakers`: Count of distinct speakers (for meeting transcripts and multi-voice content).
- `category`: Primary classification (Tutorial, Meeting, Screen Share, Music/Inspiration, Woodworking, Other).
- `tags`: Searchable keywords (e.g., ["web", "upload", "tutorial", "UI"], or ["woodworking", "finishing", "technique"]).
- `transcript_status`: "complete" if the entire video has been transcribed; "partial" if audio is missing or inaudible.
- `screenshot_count`: Total number of screenshots embedded in the document.
- `tools_listed`: Count of unique tools or software mentioned.
- `materials_listed`: Count of physical or digital materials required.
- `qa_passed`: Boolean indicating whether all QA checks passed.
- `qa_check_timestamp`: When the final QA validation was performed.
- `schema_version`: Version of this template specification (for forward compatibility).

**Example:**
```yaml
---
title: "Upload a Document to Google Drive in 5 Steps"
source: "YouTube video"
processed_date: "25.07.2026"
processed_time: "14:32:45"
video_duration: "03:47"
total_speakers: 1
category: "Tutorial"
tags: ["Google Drive", "cloud storage", "document upload", "step-by-step", "web tutorial"]
transcript_status: "complete"
screenshot_count: 8
tools_listed: 1
materials_listed: 0
qa_passed: true
qa_check_timestamp: "2026-07-25 14:33:12 UTC"
schema_version: "1.0"
---
```

### 1.2 General Description

**Heading:** `## General Description`

A paragraph (3–5 sentences) providing context about the video. Answer: What is this video about? Who made it? What is its purpose? Write in neutral, informative tone.

**Requirements:**
- Must be accessible to someone who has never seen the video.
- Must not paraphrase or summarize; instead, describe the video's subject matter, intent, and scope.
- Must mention the original platform or source if discernible.
- Must state whether the video is a tutorial, demonstration, documentation, or other type.

**Example — Screen Share Tutorial:**
```
## General Description

This is a screen-capture tutorial recorded on an Apple MacBook, showing how to upload a document to Google Drive using a web browser. The narrator is a professional trainer speaking in English. The video was originally published on YouTube as part of a "Google Workspace for Teams" training series. The screen shows a Google Drive interface, a file manager, and step-by-step mouse movements and keyboard actions.
```

**Example — Woodworking Tutorial:**
```
## General Description

This is a hands-on woodworking tutorial filmed in a workshop, demonstrating the process of hand-planing a walnut board to final thickness and flatness. The video captures the woodworker's techniques, tool selection, and quality-checking methods. The camera angle alternates between close-ups of hand positions and wider shots of the workbench. The audio includes the sound of the plane cutting wood, narration explaining each step, and ambient workshop noise.
```

### 1.3 Short Summary

**Heading:** `## Short Summary`

A single paragraph (2–3 sentences) that distills the video's essence for quick reference.

**Requirements:**
- Must capture the core takeaway in under 100 words.
- Must be suitable for a search result or document index.
- Must include the primary action or learning outcome.

**Example — Screen Share Tutorial:**
```
## Short Summary

This tutorial walks through uploading a file to Google Drive in five simple steps: opening Google Drive, clicking the "New" button, selecting "File Upload", choosing a file from your computer, and confirming the upload. Total runtime is 3 minutes 47 seconds.
```

**Example — Woodworking Tutorial:**
```
## Short Summary

A woodworker demonstrates hand-planing a walnut board to thickness and flatness, including tool setup, body mechanics, quality checks with a straightedge, and achieving a museum-quality finish. The process takes approximately 20 minutes of active work.
```

### 1.4 Category and Source of Origin

**Heading:** `## Category & Source`

Two simple fields presenting the video's classification and provenance.

**Requirements:**
- `Category`: One of the following: Tutorial, Meeting, Screen Share, Music/Inspiration, Woodworking, Vlog, Social Media, Educational, Documentary, Other.
- `Source`: The actual platform or origin (Instagram, YouTube, Zoom, iPhone Screen Recording, Professional Studio, User-Submitted, Other).
- Must be plaintext, not markdown formatting, for easy parsing.

**Example:**
```
## Category & Source

**Category:** Tutorial  
**Source:** YouTube video  
**Original URL (if available):** https://www.youtube.com/watch?v=example
```

### 1.5 Key Points

**Heading:** `## Key Points`

A bulleted list of the most important takeaways, learnings, or highlights from the video. This section must be browsable and scannable.

**Requirements:**
- Minimum 3 key points, maximum 12 (depending on video length and complexity).
- Each point should be atomic (addressing one idea).
- Each point should be actionable or informative, not vague.
- Optionally include the timestamp where the point is discussed (in format `[MM:SS]` or `[HH:MM:SS]`).

**Example — Screen Share Tutorial:**
```
## Key Points

- **Step 1: Open Google Drive** [00:12] — Navigate to drive.google.com and sign in with your Google account.
- **Step 2: Click "New" button** [00:45] — Look for the blue "New" button on the left sidebar; it opens a dropdown menu.
- **Step 3: Select "File Upload"** [01:08] — Choose "File Upload" from the dropdown to browse your computer.
- **Step 4: Choose your file** [01:35] — Navigate to your file and select it; the upload begins immediately.
- **Step 5: Confirm and organize** [03:10] — The file appears in your Drive; you can then move it to a folder and rename it.
- **Pro tip:** Drag-and-drop is faster — you can drag files directly from Finder onto the Google Drive window [03:35].
```

**Example — Woodworking Tutorial:**
```
## Key Points

- **Tool preparation** [00:30] — The hand plane must be sharp and properly tuned; plane iron angle affects finish quality.
- **Grain direction matters** [02:15] — Planing against the grain causes tearout; the woodworker adapts direction based on grain direction.
- **Body mechanics** [03:45] — Proper stance and arm extension control cut depth and surface finish; posture prevents fatigue.
- **Quality checking** [08:20] — A straightedge reveals high spots; multiple passes are needed to achieve flatness.
- **Final finish** [18:50] — Even after hand-planing, a 150-grit sand ensures smoothness before staining or oiling.
- **Time to completion** [20:00] — Planing a typical board takes 15–25 minutes, depending on starting condition and desired finish.
```

### 1.6 Meeting Main Points (Conditional)

**Heading:** `## Meeting Main Points` (Included only if the video is a meeting, recorded call, or collaborative discussion)

If the video is a meeting (Zoom, Slack, Teams, in-person recorded discussion):
- **Attendees:** List of participants (names and roles if mentioned).
- **Meeting Objective:** Primary purpose or agenda.
- **Decisions Made:** Any agreed-upon actions or conclusions.
- **Action Items:** Explicit task assignments with assigned owners.
- **Unresolved Issues:** Questions or topics to revisit.

**Example:**
```
## Meeting Main Points

**Attendees:** Alice Chen (Project Lead), Bob Thompson (Developer), Carol Johnson (Designer)

**Meeting Objective:** Sprint planning for Q3 release; discuss technical debt and timeline.

**Decisions Made:**
- Prioritize mobile responsiveness (Bob will lead).
- Defer database migration to next sprint (technical risk too high now).
- Allocate 20% of sprint capacity to bug fixes.

**Action Items:**
- Bob: Create detailed mobile testing plan by Wednesday.
- Carol: Deliver final UI mockups by Friday.
- Alice: Update stakeholder roadmap by Monday.

**Unresolved Issues:**
- Third-party API deprecation date — requires vendor confirmation.
- Performance baseline for new server setup — awaiting infrastructure team's measurements.
```

### 1.7 Chapter / Timeline Table

**Heading:** `## Timeline & Chapters`

A table mapping time ranges to the content shown or discussed at those moments. This table must be present for all videos and serves as a navigation aid.

**Requirements:**
- Three columns: Start Time, End Time, Chapter/Action.
- Times must be in HH:MM:SS format (or MM:SS for videos under 1 hour).
- Each row represents a logical segment of the video (a topic, a step, a scene change, a speaker change).
- Segments should be 30 seconds to 2 minutes long (adjust based on content density).
- The last row's end time must equal the video's total duration.
- Chapter names must be descriptive and consistent with subsequent detailed sections.

**Example — Screen Share Tutorial (3:47 total):**
```
## Timeline & Chapters

| Start | End | Chapter |
|-------|-----|---------|
| 00:00 | 00:12 | Intro and navigation to Google Drive |
| 00:12 | 00:45 | Opening Google Drive and explaining the interface |
| 00:45 | 01:08 | Clicking the "New" button and understanding the menu |
| 01:08 | 01:35 | Selecting "File Upload" and browsing the file system |
| 01:35 | 03:10 | Uploading the file and monitoring progress |
| 03:10 | 03:47 | Post-upload organization, naming, and folder placement |
```

**Example — Woodworking Tutorial (22:15 total):**
```
## Timeline & Chapters

| Start | End | Chapter |
|-------|-----|---------|
| 00:00 | 00:30 | Intro: workbench setup and tool introduction |
| 00:30 | 03:00 | Hand plane tuning and sharpness check |
| 03:00 | 08:30 | First planing pass: grain direction assessment and execution |
| 08:30 | 12:00 | Flatness checking with straightedge and identifying high spots |
| 12:00 | 17:00 | Second planing pass: correcting high spots |
| 17:00 | 20:15 | Final quality check and sanding with 150-grit paper |
| 20:15 | 22:15 | Outro: wood is ready for staining; recommended next steps |
```

### 1.8 Step-by-Step Tutorial Reconstruction with Embedded Screenshots

**Heading:** `## Detailed Tutorial: [Tutorial Title]`

This section is the heart of the document. It provides a complete, frame-by-frame reconstruction of any tutorial, demonstrated procedure, or multi-step process shown in the video. This section **must be present and comprehensive for tutorials and how-to videos**. For other video types, adapt the structure as needed (e.g., "Detailed Walkthrough" for screen shares, "Technique Demonstration" for woodworking).

**Requirements:**
- Break the process into numbered steps (one step per major action or screen state).
- For each step:
  - **Step title** (imperative: "Upload the file", not "Uploading the file").
  - **Timestamp** (when this step begins and ends in the video).
  - **What you see on screen:** A rich, detailed textual description of the visual state (for R5 downstream AI use, and for accessibility when images don't load).
  - **What the narrator says:** The exact verbatim transcript for this step.
  - **Embedded screenshot:** An image showing the key moment of this step (in markdown image syntax, with a caption).
  - **Why this step matters:** A sentence or two explaining the purpose or consequence of this step.
  - **Potential pitfalls:** Optional warning or common mistakes to avoid.

**Example — Screen Share Tutorial:**
```
## Detailed Tutorial: Upload a Document to Google Drive

### Step 1: Navigate to Google Drive and Sign In
**Timestamp:** [00:12 – 00:45]

**What you see on screen:**
A MacBook Safari browser window showing the Google Drive login page (drive.google.com). The page displays the Google logo, a "Sign in" button, and a note about accessing files from anywhere. The Safari address bar clearly shows "drive.google.com".

**What the narrator says:**
"We're going to upload a document to Google Drive. First, open your web browser — Chrome, Safari, Firefox, any browser works — and navigate to drive.google.com. If you're already signed in to Google, you'll see your Drive immediately. If not, click 'Sign in' and enter your Google account credentials."

**Embedded screenshot:**
![Step 1: Google Drive login page, showing the URL drive.google.com and the "Sign in" button in the center of the page](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/01_google_drive_login_1721901165.png)

**Why this step matters:**
You must access Google Drive in a web browser to upload files. Signing in ensures the file is saved to your personal Drive storage, not someone else's.

**Potential pitfalls:**
- If you're using a work or school Google account, you may need to grant additional permissions before accessing Drive.
- Bookmark drive.google.com for faster access in the future.

---

### Step 2: Locate and Click the "New" Button
**Timestamp:** [00:45 – 01:08]

**What you see on screen:**
The Google Drive interface is now fully loaded. The left sidebar contains navigation items: "My Drive", "Shared with me", "Starred", and "Recent". At the top of the left sidebar is a blue button labeled "New" with a plus sign icon. The main area shows thumbnails of previously uploaded files and folders.

**What the narrator says:**
"Once you're in Google Drive, look at the left sidebar. You'll see a blue button that says 'New' with a plus sign. This is your entry point for creating and uploading files. Click this button to see the menu options."

**Embedded screenshot:**
![Step 2: Google Drive main interface, showing the left sidebar with the blue "New" button highlighted. The main area displays folders like "Documents", "Photos", and "Projects".](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/02_new_button_location_1721901178.png)

**Why this step matters:**
The "New" button is the gateway to uploading or creating files in Google Drive. This is the standard workflow for all upload operations.

**Potential pitfalls:**
- The button location may vary slightly depending on screen resolution or whether your Drive is in compact mode.
- On mobile browsers, the "New" button may be in a different location (usually at the bottom of the screen).

---

### Step 3: Select "File Upload" from the Dropdown Menu
**Timestamp:** [01:08 – 01:35]

**What you see on screen:**
The user clicks the blue "New" button, and a dropdown menu appears below it. The menu has four options: "Folder", "File upload", "Folder upload", and "Google Docs", "Google Sheets", "Google Slides" (depending on Drive subscription). The cursor is positioned over "File upload".

**What the narrator says:**
"A dropdown menu will appear. You have several options here: you can create a new folder, upload a single file, upload an entire folder, or create a new Google Doc, Sheet, or Slide. For uploading an existing file from your computer, we want 'File upload'. Click on that."

**Embedded screenshot:**
![Step 3: The "New" button dropdown menu is open, showing options including "Folder", "File upload", "Folder upload", and other options. The cursor is hovering over "File upload".](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/03_file_upload_menu_1721901192.png)

**Why this step matters:**
The dropdown menu provides different upload methods. "File upload" is for individual files, while "Folder upload" allows you to upload an entire directory structure.

**Potential pitfalls:**
- Do not select "Folder" (this creates an empty folder, not an upload mechanism).
- If you have large files (over 5 GB), you may need to use Google Drive for Desktop instead of this web interface.

---

### Step 4: Browse and Select Your File
**Timestamp:** [01:35 – 03:10]

**What you see on screen:**
Clicking "File upload" opens a native macOS file browser dialog. The dialog title is "Choose a file to upload". The left sidebar shows common folders: "Desktop", "Documents", "Downloads". In the main area, there are several files displayed: "Q3_Report.pdf", "Budget_2026.xlsx", "Presentation_Draft.pptx", and a folder called "Archive". The user navigates to the Documents folder and selects "Q3_Report.pdf". The file is highlighted in blue. At the bottom right of the dialog is a blue "Open" button.

**What the narrator says:**
"A file browser window opens. Navigate to the location where your file is stored. In this case, I'm going to select a file from the Documents folder. Here's the Q3 Report as a PDF. I'll click on it to select it, and then I'll click the 'Open' button to confirm the selection. Once you click 'Open', the upload begins immediately. You don't need to do anything else."

**Embedded screenshot:**
![Step 4a: macOS file browser dialog, showing the Documents folder with files listed including "Q3_Report.pdf" (highlighted), "Budget_2026.xlsx", and "Presentation_Draft.pptx".](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/04a_file_browser_select_1721901210.png)

![Step 4b: After clicking Open, the browser transitions back to Google Drive. A progress bar appears in the bottom right corner showing "Q3_Report.pdf uploading... 45%".](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/04b_file_uploading_progress_1721901225.png)

**Why this step matters:**
Selecting and opening the file triggers the upload to Google servers. The progress bar confirms the transfer is in progress.

**Potential pitfalls:**
- Large files (over 100 MB) may take several minutes to upload, depending on your internet speed.
- If your internet connection is interrupted, the upload will fail and must be restarted.
- The file must be closed on your computer (not locked by another application) for the upload to succeed.

---

### Step 5: Confirm Upload and Organize the File
**Timestamp:** [03:10 – 03:47]

**What you see on screen:**
The upload completes. The progress bar disappears. The Google Drive interface now shows the file "Q3_Report.pdf" as a new entry in the file list. The file icon is a small PDF icon, the file name is displayed to the right, the file size "2.4 MB" is shown, and the modified date "Today" appears on the right. The file is now permanently stored in Google Drive. At the top of the screen, a green notification appears briefly: "Q3_Report.pdf uploaded successfully."

**What the narrator says:**
"The upload is complete. You'll see a confirmation message: 'Q3_Report.pdf uploaded successfully.' The file now appears in your Drive. You can see the file name, its size, and when it was uploaded. From here, you can organize it further: right-click on the file to rename it, move it to a folder, share it with others, or download it back to your computer if needed."

**Embedded screenshot:**
![Step 5: Google Drive interface showing the successfully uploaded "Q3_Report.pdf" file in the file list, with the green confirmation message "uploaded successfully" visible at the top of the page.](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/05_upload_complete_confirmation_1721901242.png)

**Why this step matters:**
Once the file is in Google Drive, it is automatically backed up to the cloud and accessible from any device. You can now share it, organize it, or perform further actions on it.

**Potential pitfalls:**
- Do not close your browser or turn off your computer before the upload completes.
- The file is stored in "My Drive" by default; to organize it, move it to a specific folder (create folders as needed).

---

### Pro Tip: Drag-and-Drop Upload
**Timestamp:** [03:35 – 03:47]

**What you see on screen:**
An alternative method is demonstrated. The user opens a Finder window on the left side of the screen, showing the Documents folder. The Google Drive window is open on the right side of the screen. The user drags a file ("Budget_2026.xlsx") from the Finder window and drops it onto the Google Drive window. The file immediately begins uploading.

**What the narrator says:**
"There's a faster way if you prefer. You can simply drag a file from your Finder (or File Explorer on Windows) directly onto the Google Drive window. The upload starts immediately. This method works for multiple files too — just select several files and drag them all at once. They'll all upload in parallel."

**Embedded screenshot:**
![Pro Tip: Drag-and-drop upload demonstrating a file being dragged from Finder (left side) to the Google Drive window (right side).](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/06_drag_and_drop_alternative_1721901258.png)

**Why this step matters:**
Drag-and-drop is faster and more intuitive for frequent uploads. For power users, this can save significant time.

---
```

**Example — Woodworking Tutorial:**
```
## Detailed Technique: Hand-Planing a Walnut Board to Final Thickness

### Step 1: Workbench Setup and Plane Inspection
**Timestamp:** [00:00 – 00:30]

**What you see on screen:**
A woodworking shop setting. A sturdy wooden workbench is positioned in the center of the frame. On the benchtop, a walnut board approximately 24 inches long, 6 inches wide, and 1.25 inches thick is clamped in a wooden vise at one end of the bench. Next to the vise, a hand plane (a #5 Jack Plane, identifiable by its size and shape) is resting on the bench. The plane's sole (bottom surface) is visible, showing the blade opening. Wood shavings and dust cover the benchtop. Natural light from large windows illuminates the workspace.

**What the narrator says:**
"Today we're going to hand-plane this walnut board from the rough-sawn state to a perfectly flat, finished thickness. This is a #5 Jack Plane — a medium-length plane ideal for stock preparation. Before we start, let me show you the setup. The board is secured firmly in the vise; you don't want any movement while planing. Here's the plane. Feel the sole — it should be smooth and flat. The blade should be sharp. We can test sharpness by shaving an arm — if it catches hair, it's sharp enough."

**Embedded screenshot:**
![Step 1: Overview of the woodworking workbench with the walnut board clamped in the vise, the #5 Jack Plane on the bench to the left, and wood shavings on the benchtop.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/01_workbench_setup_1721901500.png)

**Why this step matters:**
A secure workbench setup is essential for safety and quality. Proper tool inspection prevents delays and ensures a smooth finish.

**Potential pitfalls:**
- If the board is loose in the vise, it will vibrate and bounce, causing tearout and an uneven surface.
- If the plane blade is dull, it will crush wood fibers instead of slicing them cleanly, making more work for you.

---

### Step 2: Assessing Grain Direction and Setting Up the First Pass
**Timestamp:** [00:30 – 03:00]

**What you see on screen:**
The woodworker runs a finger along the edge of the walnut board, showing the grain direction (the "grain slope" or "slope of the grain"). The board shows growth rings; the woodworker traces the slope from one end of the board to the other. On one end, the grain slopes from left to right at roughly a 20-degree angle. The woodworker positions himself at the left end of the board, with the plane held at a slight diagonal angle (skewed) to the direction of wood travel. The plane's body is perpendicular to the board's edge but the stroke direction is diagonal.

**What the narrator says:**
"Look at the grain. Run your finger along the edge and feel the slope. Here, the grain rises to the right, which means we need to plane from left to right, against the slope. This prevents tearout. Also notice I'm skewing the plane — holding it at an angle rather than perfectly perpendicular to the board. This reduces the force on the blade and produces a finer shaving. The blade angle, combined with the skew, is the difference between a smooth surface and a splintery mess."

**Embedded screenshot:**
![Step 2a: Close-up of the grain slope on the board edge, with the woodworker's finger pointing to the direction.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/02a_grain_direction_inspection_1721901520.png)

![Step 2b: The woodworker positioned at the planing position, showing body posture, hand placement, and the skewed plane angle (roughly 30 degrees to the board's length).](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/02b_planing_stance_and_angle_1721901535.png)

**Why this step matters:**
Understanding grain direction prevents tearout (splintered fibers) and produces a superior surface. Skewing the plane reduces cutting force and improves surface quality.

**Potential pitfalls:**
- Planing with the grain (downslope) causes tearout. If you discover tearout, stop and flip the board.
- Holding the plane perpendicular (no skew) requires more force and can cause blade chatter.

---

### Step 3: Taking the First Planing Pass
**Timestamp:** [03:00 – 08:30]

**What you see on screen:**
The woodworker makes the first pass down the length of the board. The plane moves from left to right along the walnut board. With each stroke, a thin, continuous shaving is produced and curls out of the plane's throat (opening). The shaving is roughly the thickness of a piece of paper, light honey-gold in color, indicating a clean cut. The woodworker's arms are extended smoothly; there's no jerky motion. The sound of the plane cutting is a clear "whoosh" sound, rhythmic and unhurried. After the first pass covers the entire surface, the woodworker stops and inspects the board.

**What the narrator says:**
"Now we take our first pass. I'm starting here at the left end, applying pressure downward and forward. The plane glides smoothly across the wood. Hear that sound? That's a sharp plane cutting cleanly. Each shaving should be thin and continuous. If the shaving is thick and full of splinters, or if the plane is chattering, stop — the blade might be dull or the skew angle wrong. I'm taking several passes, each time covering the entire length. After the first pass, you can see the change in the surface — it's getting smoother and flatter."

**Embedded screenshot:**
![Step 3a: The plane in mid-stroke, showing the shaving curling out of the throat, the plane angle, and the woodworker's arm position.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/03a_plane_in_motion_shaving_1721901560.png)

![Step 3b: After multiple passes, the board's surface is visibly smoother, with a uniform golden-brown sheen. Sawdust and shavings are visible around the board and on the benchtop.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/03b_board_after_initial_passes_1721901590.png)

**Why this step matters:**
The first pass removes the roughest material and establishes a baseline surface. Multiple passes in the same direction gradually flatten the board.

**Potential pitfalls:**
- Do not rush. Planing is a rhythmic, deliberate process. Hurrying causes mistakes and leaves an uneven surface.
- If you feel the plane grabbing or jumping, stop immediately; the blade angle may be wrong.

---

### Step 4: Flatness Checking with a Straightedge
**Timestamp:** [08:30 – 12:00]

**What you see on screen:**
The woodworker retrieves a straightedge — a long, flat metal or wooden bar (approximately 24 inches long, 2 inches wide, 0.5 inches thick) — and places it on the board's surface. The straightedge is placed diagonally across the board, then along the length, then along the width, checking for gaps between the straightedge and the board surface. When placed along the length, a small gap is visible near the middle of the board, indicating a "cup" (the center is lower than the ends). The woodworker marks this high area with a pencil for the next pass.

**What the narrator says:**
"After several passes, we check flatness with a straightedge. This is critical. Place the straightedge across the board in different directions — length, width, and diagonals. If there's a gap between the straightedge and the board, that's a high spot. Mark it with a pencil. Here, I can see the ends are higher than the middle. This tells me the board is slightly cupped. I need to focus the next few passes on bringing down these high spots."

**Embedded screenshot:**
![Step 4a: The straightedge placed along the length of the board, with a visible gap (approximately 1/16 inch) between the straightedge and the board's surface in the center area.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/04a_straightedge_check_cupping_1721901620.png)

![Step 4b: Close-up of the high spots marked with pencil lines, showing where further planing is needed.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/04b_marked_high_spots_1721901635.png)

**Why this step matters:**
A straightedge check is the only reliable method to verify flatness. This prevents over-planing one end and under-planing the other.

**Potential pitfalls:**
- Do not rely on visual inspection alone; the human eye easily mistakes a slightly cupped board for a flat one.
- If high spots are not addressed early, you may plane the entire board and still have a cupped surface.

---

### Step 5: Targeted Planing on High Spots
**Timestamp:** [12:00 – 17:00]

**What you see on screen:**
The woodworker focuses on the marked high-spot areas, taking lighter passes specifically over those zones. The grain direction has been re-assessed, and planing is now concentrated on the ends of the board where the pencil marks are visible. After several focused passes, the pencil marks begin to disappear as the high spots are removed. The woodworker pauses, retrieves the straightedge again, and checks the surface. The gap is now much smaller.

**What the narrator says:**
"Now I'm going to focus on bringing down these high spots. I'll take lighter passes, concentrating on the marked areas. Watch how the pencil marks disappear as the high spots come down. After a few passes, I'll check again with the straightedge. Each check tells me how much more work is needed. This is where patience pays off. Rushing and over-planing is the most common mistake."

**Embedded screenshot:**
![Step 5a: The woodworker taking targeted passes on a high-spot area, with the plane visible in mid-stroke over the marked region.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/05a_targeted_planing_high_spots_1721901660.png)

![Step 5b: Straightedge check after targeted planing, showing the gap is now reduced to approximately 1/32 inch (much improved).](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/05b_straightedge_improved_flatness_1721901680.png)

**Why this step matters:**
Targeted planing corrects cupping and other flatness issues without wasting material or time.

**Potential pitfalls:**
- Planing too aggressively on high spots can create new low spots.
- Frequent straightedge checks (every 2–3 passes) are essential to avoid over-correction.

---

### Step 6: Final Planing and Surface Preparation for Finishing
**Timestamp:** [17:00 – 20:15]

**What you see on screen:**
The board is now nearly flat. The woodworker performs a few light passes over the entire surface, using a lower cut depth (the blade is set to produce thinner shavings, perhaps 0.003 to 0.005 inches). The surface becomes increasingly glossy and smooth. A final straightedge check confirms flatness within 0.015 inches (acceptable for fine woodworking). The woodworker then retrieves 150-grit sandpaper and a sanding block. Sanding is performed in the direction of the grain, with light pressure, removing the planing marks and any slight irregularities.

**What the narrator says:**
"The board is nearly flat now. These final light passes refine the surface and remove any remaining irregularities. You'll notice the sound changes — it's even smoother now. After the plane passes, I'll sand with 150-grit paper to remove the planing marks and prepare the surface for finishing. The goal is a smooth surface that's ready for stain or oil. After sanding, the wood feels silky to the touch."

**Embedded screenshot:**
![Step 6a: Final light planing passes, showing the plane set to a shallow cut and the smooth, glossy surface of the walnut board.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/06a_final_light_passes_1721901710.png)

![Step 6b: Sanding the board with a 150-grit sanding block in the grain direction, showing the fine dust and the smooth surface.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/06b_sanding_150_grit_1721901730.png)

**Why this step matters:**
Final planing and sanding produce a museum-quality surface. The wood is now ready for finishing (staining, oiling, or other treatments).

**Potential pitfalls:**
- Do not sand before planing; sanding clogged with planing marks wastes paper and time.
- Always sand in the grain direction to avoid cross-grain scratches.

---

### Step 7: Inspection and Project Completion
**Timestamp:** [20:15 – 22:15]

**What you see on screen:**
The woodworker inspects the board by running a hand across the surface, tilting it to the light to check for any remaining imperfections. The surface is visibly smooth and flat. The grain pattern of the walnut is now visible and beautiful, with rich color variations. The woodworker measures the final thickness with a digital caliper: approximately 0/8 inches (0.875 inches), confirming the target thickness was achieved. The woodworker then explains the next steps for the wood (staining and oiling), but notes that the planing work is complete.

**What the narrator says:**
"The board is finished. The surface is smooth, flat, and ready for any further work. The grain is now visible, the color is rich, and the wood is prepared for staining or oiling. This process — from rough-sawn to finished thickness — took about 20 minutes of active planing and sanding. If you're building a larger project, you'd repeat this process for each board, then move on to assembling and finishing your project. Hand-planing gives you complete control and produces superior results compared to machine planing. It's a skill that pays dividends for the rest of your woodworking career."

**Embedded screenshot:**
![Step 7: The finished walnut board, with smooth, flat surface and visible grain pattern, photographed in natural light. The surface shows the warm, honey-brown color of walnut wood.](../screenshots/Hand-Planing%20a%20Walnut%20Board%20%7C%2025.07.2026%20%7C%20original%3A%20workshop%20recording/07_finished_board_inspection_1721901760.png)

**Why this step matters:**
Inspection confirms the work meets quality standards. Proper measurement confirms target thickness is achieved.

**Potential pitfalls:**
- Do not skip inspection; what looks flat may have imperfections visible in certain lighting.
- If the final thickness is not correct, additional passes are needed (or material must be removed in other ways).

---
```

### 1.9 Tools and Materials List

**Heading:** `## Tools & Materials Required`

A comprehensive, annotated list of all tools, software, materials, and equipment mentioned or used in the video. This section addresses R5.1 and R5.2 (tools and materials needed for a downstream AI to understand what is required).

**Requirements:**
- Organize by category: Hardware/Software, Tools, Materials, Optional/Nice-to-Have.
- For each item:
  - **Name and description** (what it is, and a brief explanation of its purpose in the context of the video).
  - **Brand/model (if specific):** The exact brand, model number, or version mentioned or identifiable in the video.
  - **Quantity:** How many of this item are needed or used.
  - **Alternatives:** Any substitutes mentioned or implied.
  - **Cost (optional, if mentioned):** Approximate cost or cost range, for budget planning.
- If the video does not apply (e.g., a music/inspiration video), adapt this section to list software, plugins, filters, etc.

**Example — Screen Share Tutorial:**
```
## Tools & Materials Required

### Software / Online Services
| Tool | Description | Version/Brand | Quantity | Alternatives | Notes |
|------|-------------|---------------|----------|--------------|-------|
| Web Browser | Required to access Google Drive via the web | Chrome, Safari, Firefox, Edge (any modern browser) | 1 | None (required) | Free; pre-installed on macOS |
| Google Account | Required for authentication to Google Drive | Any Google Account (personal or workplace) | 1 | None (required) | Free if personal account; may be provided by employer |
| Google Drive | Cloud storage and file management service | Google Drive (web-based) | 1 subscription | Dropbox, OneDrive, iCloud | Free tier includes 15 GB storage |

### No physical materials required
This tutorial uses only a computer, internet connection, and a digital file (any file type).
```

**Example — Woodworking Tutorial:**
```
## Tools & Materials Required

### Hand Tools
| Tool | Description | Brand/Model | Quantity | Alternatives | Notes |
|------|-------------|------------|----------|--------------|-------|
| Hand Plane (Jack Plane) | Primary tool for surfacing and flattening | Lie-Nielsen #5 (shown in video) or equivalent; also Stanley Bailey #5 | 1 | Power planer, belt sander (lower quality finish) | Shown plane costs ~$350; vintage planes ($20–100) also work if tuned |
| Straightedge | Precision measurement tool for checking flatness | 24-inch aluminum straightedge (brand not specified in video) | 1 | 36-inch straightedge, wooden straightedge | Critical for quality control; inexpensive (~$15–30) |
| Pencil or marking knife | Marking high spots for planing | Standard pencil or carpenter's marking knife | 1–2 | Any marking tool | Used for visible marking on wood |
| Digital or dial caliper | Measuring final thickness | Digital caliper (brand not specified) | 1 | Tape measure (less precise) | Optional; used to confirm target thickness |
| Sanding block | Backing for sandpaper | Wood or plastic sanding block (brand not specified) | 1 | Hand-sanding pad, sanding sponge | Keeps pressure even across the surface |

### Abrasives
| Material | Description | Grit | Quantity | Notes |
|----------|-------------|------|----------|-------|
| Sandpaper | For final surface preparation after planing | 150-grit | 2–3 sheets (9" × 11" standard) | Used after planing to remove blade marks and final smoothing |

### Materials
| Material | Description | Quantity | Notes |
|----------|-------------|----------|-------|
| Walnut board | The workpiece being planed | 1 board (~24" × 6" × 1.25") | Starting thickness ~1.25"; target thickness ~0.875"; must be sound (no large knots or defects) |

### Workspace
| Item | Description | Notes |
|------|-------------|-------|
| Workbench | Sturdy flat work surface | Minimum 4 feet long, 24 inches deep; shown bench appears to be traditional hardwood construction (~$300–1,500) |
| Woodworking vise | Secures the workpiece | Shown vise appears to be a leg vise or bench vise (~$150–400) |
| Lighting | Adequate to see grain direction and surface quality | Natural window light shown; task lighting recommended (~$30–100) |

### Optional / Nice-to-Have
| Item | Purpose | Notes |
|------|---------|-------|
| Spokeshave (curved blade) | Alternative tool for edge rounding | Not used in this video; useful for woodworking projects involving curved edges |
| Tormek sharpening system | Tool sharpening | Mentioned in context of blade maintenance; cost ~$1,200; simpler sharpening methods available |
| Finish (stain, oil, or wax) | Final wood finishing | Mentioned as next steps after planing; not part of this video's scope |
```

### 1.10 Timeline and Real-World Timing

**Heading:** `## Real-World Timing & Duration Analysis`

A detailed breakdown of time, addressing R5.3, R5.4, and R5.5 (real-world grounded durations, distinguishing on-screen time from actual elapsed time).

**Requirements:**
- Distinguish clearly between video runtime and real-world process time.
- For each step or major action:
  - Video duration (how long it takes in the video, which may include edits/cuts).
  - Real-world duration (how long it actually takes in a real scenario).
  - A note explaining any discrepancy (cuts, time-lapses, or accelerated footage).
- Must include margins for learning curve (first attempt vs. experienced practitioner).
- Must explicitly address pauses, setup, and cleanup time.

**Example — Screen Share Tutorial:**
```
## Real-World Timing & Duration Analysis

### Video Runtime vs. Real-World Time

**Total Video Duration:** 3 minutes 47 seconds

**Real-World Time to Complete This Task:** 2–5 minutes (depending on experience)

**Breakdown:**

| Step | Video Duration | Real-World Duration | Notes |
|------|----------------|-------------------|-------|
| Step 1: Navigate and sign in | 00:33 | 30 seconds – 2 minutes | If already signed in, ~10 seconds. If not logged in and unfamiliar with Google, may take longer. |
| Step 2: Locate "New" button | 00:23 | 5 seconds | Once you know where it is, very quick. First-time users may search for 10–20 seconds. |
| Step 3: Select "File Upload" | 00:27 | 5 seconds | Straightforward once the menu is open. |
| Step 4: Browse and select file | 01:35 | 1–2 minutes | Depends on file location and familiarity with computer file system. First time: 2–3 minutes. Familiar user: 30 seconds. |
| Step 5: Confirm and organize | 00:37 | 30 seconds – 5 minutes | Upload speed depends on file size and internet speed. A 2.4 MB file over broadband: 10–30 seconds. A 500 MB file over slow connection: several minutes. |
| Pro Tip: Drag-and-drop | 00:12 | 10 seconds per file | Faster for repeat uploads. Overhead is minimal. |

**Total Realistic Time for First Attempt:** 3–8 minutes (including orientation, looking for buttons, selecting file, waiting for upload)

**Total Realistic Time for Experienced User:** 1–2 minutes

### Time Considerations for Project Planning

- **Learning Curve:** First-time users should allow 5–10 minutes for the full task.
- **File Size Impact:** Very large files (over 1 GB) may require 5–10 minutes or more to upload, depending on connection speed.
- **Internet Speed:** If your internet connection is slow (under 10 Mbps), upload times increase significantly.
- **Multiple Files:** If uploading multiple files, each file can be uploaded in parallel (drag all at once), saving time compared to uploading one-by-one.
- **Setup Time:** Before the first upload, you must create a Google Account and configure Drive, which adds 5–15 minutes (one-time, not repeated for subsequent uploads).

### Timing Examples from Real-World Scenarios

| Scenario | Setup Time | Active Upload Time | Total | Notes |
|----------|-----------|------------------|-------|-------|
| First-time user uploading one document | 5 min | 3 min | 8 min | Includes learning the interface |
| Experienced user uploading one document | 0 min | 1 min | 1 min | Already knows the process |
| Batch uploading 10 documents (typical business task) | 0 min | 5–10 min | 5–10 min | Files upload in parallel; total time is longest single file |
| Uploading a large video file (500 MB) over 25 Mbps connection | 0 min | 3–5 min | 3–5 min | Rough estimate: 500 MB ÷ 25 Mbps ≈ 3 min upload |
```

**Example — Woodworking Tutorial:**
```
## Real-World Timing & Duration Analysis

### Video Runtime vs. Real-World Time

**Total Video Duration:** 22 minutes 15 seconds

**Real-World Time to Complete This Task:** 20–45 minutes (first time), 15–25 minutes (experienced woodworker)

**Breakdown:**

| Step | Video Duration | Real-World Duration | Notes |
|------|----------------|-------------------|-------|
| Step 1: Setup and tool inspection | 00:30 | 5–10 minutes | Setting up the workbench, securing the board in the vise, checking the plane, and confirming the blade is sharp. First-time setup may take longer. |
| Step 2: Assess grain direction | 00:30 | 2–5 minutes | For experienced woodworkers, this is quick. Beginners may need to inspect the board more carefully. |
| Step 3: First planing pass | (included in video montage, appears to be ~5 min edited to show multiple passes) | 10–15 minutes | Taking 4–6 passes along the full length. Each pass takes 1–2 minutes. The video shows this sped up or condensed. |
| Step 4: Flatness check with straightedge | 03:30 | 2–3 minutes | Placing the straightedge in multiple directions and assessing high spots. |
| Step 5: Targeted planing on high spots | (video shows multiple passes, ~5 min edited) | 5–10 minutes | Focused planing on marked areas. May require 2–4 additional passes. |
| Step 6: Final planing and sanding | 03:15 | 5–10 minutes | Light planing passes, then sanding with 150-grit paper. Sanding can take 5 minutes for a single pass; multiple passes may be needed. |
| Step 7: Inspection | 02:00 | 2–3 minutes | Final flatness check, measurement, and assessment. |

**Total Realistic Time for First Attempt:** 30–50 minutes (includes learning, tool setup, extra checks)

**Total Realistic Time for Experienced Woodworker:** 15–25 minutes

### Time Considerations for Project Planning

**Important Distinction: On-Screen Time vs. Real-World Time**

The video is edited and condensed. Multiple planing passes that take 10–15 minutes in real time may be shown in 30 seconds of video footage (time-lapse or montage). When planning a real project, you must account for:

- **Planing Time:** ~2 minutes per pass per board (including direction changes and inspection).
- **Grain Direction Assessment:** Often requires planing a small test area first, adding 5–10 minutes.
- **Flatness Checking:** Frequent straightedge checks slow down the process but prevent mistakes; budget 2–3 minutes per check.
- **Tearout Recovery:** If tearout occurs (wood grain splinters), you may need to:
  - Reverse the board direction and try planing from the opposite end (+10–15 minutes).
  - Replace the damaged area or accept cosmetic damage (+0 to 30 minutes depending on severity).

### Real-World Timing Table for Different Board Scenarios

| Board Condition | Estimated Thickness Reduction | Expected Planing Time | Notes |
|-----------------|-------------------------------|----------------------|-------|
| Rough-sawn, significant cupping | 0.375" reduction | 25–40 minutes | Requires multiple corrective passes; frequent flatness checks |
| Rough-sawn, relatively flat | 0.25" reduction | 15–25 minutes | Shown scenario; assumes board is relatively straight initially |
| Planed once before, minor surface irregularities | 0.125" reduction | 8–15 minutes | Only refinement passes needed; minimal high-spot correction |
| Final finishing (last 1/32" thickness reduction) | 0.031" reduction | 5–10 minutes | Light passes; focus on smoothness, not flatness |

### Drying and Finishing Time (Post-Planing)

The video ends after planing and sanding. If finishing with stain or oil:

- **Oil Finish (e.g., tung oil, linseed oil):** First coat takes 10–15 minutes to apply; requires 4–8 hours to dry between coats. Typically 2–3 coats are needed (total elapsed time: 1–2 days if applying one coat per day).
- **Stain:** 10–15 minutes to apply; 2–4 hours to dry (varies by product). Stain is often followed by a topcoat (sealer or polyurethane).
- **Drying in humidity:** High humidity slows drying significantly. In a dry climate or with good air circulation, drying is faster.

**Note:** Post-finishing drying time is NOT included in the 20–45 minute estimate above; it is elapsed time, not active work time.

### Timing Considerations for Project Scaling

If you're planing multiple boards (e.g., for a tabletop):

- **Single board:** 20–40 minutes
- **Two boards:** 40–80 minutes (not double, due to efficiency; once you understand the grain, subsequent boards are faster)
- **Four boards for a tabletop:** 80–150 minutes (1.5 – 2.5 hours), accounting for increasing setup overhead and matching grain patterns

### First-Time Learner Padding

If you're new to hand-planing:
- Add 30–50% to all time estimates above.
- Allow extra time for tool setup, sharpening, and recovery from mistakes.
- First board: 30–50 minutes is realistic. Second board: 20–30 minutes. By the fifth board, you'll be closer to 15–20 minutes.
```

### 1.11 Aesthetic and Stylistic Analysis

**Heading:** `## Aesthetic & Style Analysis`

For videos that emphasize visual style, mood, filters, or production aesthetics (music/inspiration videos, vlogs, cinematography-focused content), this section captures those details. This addresses R4.11.

**Requirements:**
- Describe the overall mood, tone, and visual aesthetic.
- List color grading or filters applied (if identifiable).
- Describe camera angles, lighting, and cinematography choices.
- Note transitions and visual effects (if present).
- Describe the pacing and editing style.
- Explain how these choices contribute to the viewer's experience.
- Adaptation: For non-aesthetic videos (tutorials, meetings), this section may be brief or omitted.

**Example:**
```
## Aesthetic & Style Analysis

### Overall Mood and Tone
The video conveys a **calm, methodical, educational mood**. There is no sense of urgency or drama. The pacing is deliberate and reflective, inviting the viewer to absorb each step without rush. The woodworker speaks in a steady, measured tone, emphasizing patience and precision.

### Visual Aesthetic
- **Color Palette:** Warm, natural wood tones (golden-brown walnut, honey-colored shavings). The workshop lighting is natural and warm (no harsh shadows or cool fluorescent lighting).
- **Lighting:** Predominantly natural light from large windows, supplemented by task lighting on the workbench. This creates an inviting, comfortable atmosphere.
- **Camera Work:** Mostly medium and close-up shots; wide shots are used to establish context (workbench overview). Handheld camera work conveys an intimate, instructional feeling.

### Filters and Color Grading
- **Minimal color grading:** The video appears to have very little post-processing. Colors are natural and true-to-life.
- **Slight warmth:** A subtle warm color cast (perhaps a slight increase in orange/yellow) enhances the welcoming, craft-focused aesthetic.
- **No artificial effects:** No fades, wipes, or transitions; cuts are clean and direct (edit-to-edit).

### Editing and Pacing
- **Edit style:** Quick, practical cuts. Each step flows to the next without unnecessary padding.
- **Speed variations:** Real-time demonstration of planing (so viewer can hear the plane sound), but some passes are time-compressed or shown in montage (multiple passes edited together).
- **Sound design:** Emphasis on natural sounds (plane cutting, wood grain being discussed, ambient workshop noise). Minimal background music; when present, it's very subtle and non-distracting.

### How These Choices Support the Content
The calm, warm, natural aesthetic reinforces the message that hand-planing is a **meditative, precise craft**, not a rushed or industrial process. The viewer is positioned as an apprentice learning from a patient master. The natural light and warm wood tones create emotional resonance and encourage the viewer to try the technique themselves.
```

### 1.12 Audio and Music Analysis

**Heading:** `## Audio & Music Analysis`

A detailed breakdown of the audio track, including narration, background music, ambient sounds, and any audio effects. This section addresses the audio analysis component of R2.1 (audio analysis).

**Requirements:**
- **Narration:** Speaker(s), language, tone, technical terminology used, pacing of speech.
- **Background Music:** Genre, tempo, mood, volume level relative to narration (if present).
- **Ambient Sounds:** Environmental sounds (workshop noise, computer interface beeps, page turns, etc.).
- **Sound Effects:** Any intentional audio effects or enhancements.
- **Audio Quality:** Microphone type, background noise level, clarity.
- **Silence and Pauses:** Moments where sound is absent or minimal, and their effect.

**Example — Screen Share Tutorial:**
```
## Audio & Music Analysis

### Narration
- **Speaker:** One speaker, male, native English speaker (American accent).
- **Tone:** Friendly, instructional, patient. The narrator explains concepts clearly without condescension.
- **Technical Language:** Uses common terms (Google Drive, browser, file, upload) without jargon. Explains terms like "dropdown menu" and "progress bar" for clarity.
- **Speech Pace:** Moderate, deliberate pace (~140 words per minute). Allows time for viewers to follow along visually.
- **Terminology Highlights:**
  - "Google Drive" (repeated multiple times)
  - "New button" (emphasized as the entry point)
  - "Drag-and-drop" (technical term, explained and demonstrated)
  - "Progress bar" (UI element, explained)

### Background Music
- **Present:** No background music. The video relies entirely on the narrator's voice and natural UI sounds.
- **Silence:** Brief pauses between steps allow the narrator's voice to be the focus.

### Ambient / Interface Sounds
- **Mouse clicks:** Audible when the user clicks the "New" button, "File upload", "Open" button, etc. Each click is distinct and helps orient the viewer to screen interactions.
- **File browser sounds:** The macOS file browser is silent; no audible feedback for file selection.
- **Upload sounds:** No audible feedback for the upload progress. The progress bar is visual only.

### Audio Quality
- **Microphone:** Appears to be a desktop condenser microphone or lavalier mic. Clear, warm tone with minimal background noise.
- **Background Noise:** Negligible; no computer fan noise, room noise, or keyboard clicks audible.
- **Dynamic Range:** Consistent volume throughout; no sudden spikes or dips.
- **Processing:** Likely mild EQ and compression to enhance clarity; no distortion or artifacts.

### Silence and Pauses
- **Intentional pauses:** Between steps, the narrator pauses briefly (1–2 seconds) to allow visual processing. This is effective for tutorial pacing.
- **During screen interactions:** The narrator goes silent while clicking or selecting, allowing the sound of the interaction to be heard (mouse click). This creates a sense of real-time action.

### Timestamp-by-Timestamp Audio Annotation
See Section 1.13 (Full Transcript) for a detailed timestamp-based breakdown of what is spoken and when.
```

**Example — Woodworking Tutorial:**
```
## Audio & Music Analysis

### Narration
- **Speaker:** One speaker, male, experienced woodworker, American accent (possibly regional).
- **Tone:** Authoritative yet approachable. The narrator shares knowledge from years of experience. Encouraging and safety-conscious.
- **Technical Language:** Uses proper woodworking terminology (grain direction, tearout, skew, cupping, straightedge, etc.). Explains terms for beginners without oversimplifying for experts.
- **Speech Pace:** Slower and more deliberate than the Google Drive tutorial (~120 words per minute). This reflects the contemplative, hands-on nature of the work.
- **Terminology Highlights:**
  - "Grain direction" (foundational concept)
  - "Tearout" (common problem, repeated)
  - "Skew" (technique, explained with visual demo)
  - "Cupping" (flatness defect, explained with visual check)
  - "Straightedge" (critical tool)

### Background Music
- **Present:** No background music. The focus is on natural sound.
- **Approach:** Audio is entirely diegetic (natural to the scene). This authenticity is valuable for craft instruction.

### Ambient / Natural Sounds
- **Plane cutting wood:** The primary sound during planing. The "whoosh" of the plane blade is clear and rhythmic. The quality of this sound (sharp, smooth, no chatter) serves as auditory feedback; beginners can learn to recognize good sound vs. poor technique by ear.
- **Wood shavings:** Slight rustling sound as shavings curl out and fall.
- **Workbench interactions:** Subtle sounds of the straightedge being placed, pencil marking, water bottle (implied, not heard), etc.
- **Breathing and body movement:** Subtle, natural sounds of the woodworker at work.
- **Workshop ambient:** Very subtle background noise (air movement, distant ambient) that creates a sense of place without distraction.

### Audio Quality
- **Microphone:** Appears to be a lapel or contact microphone positioned near the woodworker's body or on the workbench. This captures narration and tool sounds with equal clarity.
- **Background Noise:** Minimal; the workshop is reasonably quiet. No major machinery or external noise.
- **Dynamic Range:** Wide, natural range. Planing sounds are prominent; narration is clear and intelligible.
- **Processing:** Minimal processing evident. The audio feels natural and unmanipulated.

### Silence and Pauses
- **Intentional silence:** During active planing, the narrator often goes silent to let the tool sounds dominate. This is a sophisticated choice; it lets viewers hear what good planing sounds like.
- **Pauses for thought:** Occasional pauses when the narrator is checking the work (straightedge checks) create a reflective mood.
- **Rhythm:** The rhythm of plane strokes, pauses, checks, and narration creates a meditative cadence.

### Timestamp-by-Timestamp Audio Annotation
See Section 1.13 (Full Transcript) for a detailed breakdown of narration and sound events by timestamp.
```

### 1.13 Full Word-by-Word Transcript with Timestamps and Speaker Labels

**Heading:** `## Full Transcript`

A complete, verbatim transcript of all spoken audio in the video, with timestamps and speaker labels. This is mandatory per R4.6. The transcript must be precise and complete; it is the basis for claiming "all details captured."

**Requirements:**
- Format: `[HH:MM:SS] Speaker: "Exact quote"` or `[MM:SS]` for videos under 1 hour.
- Include every spoken word, including filler words ("um", "uh", "like"), interjections, and repetitions.
- Do NOT paraphrase or clean up the transcript; it must be verbatim and comprehensive, meeting R4.6 ("Word-by-word transcript, verbatim, not paraphrased").
- Include non-verbal sounds (if significant): `[SOUND: plane cutting]`, `[PAUSE 2 seconds]`, `[MOUSE CLICK]`, `[LAUGHTER]`.
- If there are multiple speakers, label each clearly.
- If any audio is inaudible, note it: `[INAUDIBLE]`.
- If there is no audio for a segment, note it: `[NO AUDIO]`.

**Example — Screen Share Tutorial (partial):**
```
## Full Transcript

### Audio Transcript with Timestamps

[00:00 – 00:12] **Narrator:** "We're going to upload a document to Google Drive. First, open your web browser — Chrome, Safari, Firefox, any browser works — and navigate to drive.google.com."

[00:12 – 00:25] **[SOUND: Mouse click, browser address bar]**

[00:25] **Narrator:** "If you're already signed in to Google, you'll see your Drive immediately."

[00:30] **[SOUND: Page loading, brief delay]**

[00:35] **Narrator:** "If not, click 'Sign in' and enter your Google account credentials."

[00:45 – 01:08] **[SOUND: Mouse click, "New" button being clicked]**

[00:50] **Narrator:** "Once you're in Google Drive, look at the left sidebar. You'll see a blue button that says 'New' with a plus sign. This is your entry point for creating and uploading files. Click this button to see the menu options."

[01:08 – 01:20] **[SOUND: Dropdown menu appears, mouse hover over "File upload" option]**

[01:20] **Narrator:** "A dropdown menu will appear. You have several options here: you can create a new folder, upload a single file, upload an entire folder, or create a new Google Doc, Sheet, or Slide."

[01:35] **Narrator:** "For uploading an existing file from your computer, we want 'File upload'. Click on that."

[01:45 – 02:00] **[SOUND: File browser dialog opens; macOS file browser interface sounds]**

[02:00] **Narrator:** "A file browser window opens. Navigate to the location where your file is stored."

[02:15] **Narrator:** "In this case, I'm going to select a file from the Documents folder. Here's the Q3 Report as a PDF. I'll click on it to select it, and then I'll click the 'Open' button to confirm the selection."

[02:30 – 02:45] **[SOUND: File selection, "Open" button click]**

[02:45] **Narrator:** "Once you click 'Open', the upload begins immediately. You don't need to do anything else."

[02:50 – 03:10] **[SOUND: Upload progress indicator; subtle UI sounds]**

[03:00] **Narrator:** "The upload is in progress. You can see a progress bar showing the upload percentage. For a small file like this PDF, it should complete within seconds."

[03:10 – 03:20] **[SOUND: Upload complete; success notification sound (quiet chime)]**

[03:20] **Narrator:** "The upload is complete. You'll see a confirmation message: 'Q3_Report.pdf uploaded successfully.' The file now appears in your Drive. You can see the file name, its size, and when it was uploaded."

[03:35] **Narrator:** "From here, you can organize it further: right-click on the file to rename it, move it to a folder, share it with others, or download it back to your computer if needed."

[03:47 – 04:00] **[SOUND: Demo of drag-and-drop feature; file being dragged and dropped into Google Drive window]**

[04:00] **Narrator:** "There's a faster way if you prefer. You can simply drag a file from your Finder — or File Explorer on Windows — directly onto the Google Drive window. The upload starts immediately. This method works for multiple files too — just select several files and drag them all at once. They'll all upload in parallel."

[04:15] **Narrator:** "Thanks for watching. Please subscribe for more productivity tips."

[04:20] **[END OF VIDEO]**
```

**Example — Woodworking Tutorial (partial, detailed):**
```
## Full Transcript

### Audio Transcript with Timestamps

[00:00] **[SOUND: Workshop ambient, workbench creaking as user moves]**

[00:05] **Woodworker (Host):** "Today we're going to hand-plane this walnut board from the rough-sawn state to a perfectly flat, finished thickness."

[00:15] **[SOUND: Hand plane being placed on workbench]**

[00:20] **Host:** "This is a number 5 Jack Plane — a medium-length plane ideal for stock preparation."

[00:30] **[SOUND: Plane sole being tapped lightly]**

[00:32] **Host:** "Before we start, let me show you the setup. The board is secured firmly in the vise; you don't want any movement while planing."

[00:40] **[SOUND: Hand tapping the walnut board in the vise — solid, no movement]**

[00:45] **Host:** "Here's the plane. Feel the sole — it should be smooth and flat."

[00:50] **[SOUND: Fingernail or metal object dragging across the plane sole]**

[00:55] **Host:** "The blade should be sharp. We can test sharpness by shaving an arm — if it catches hair, it's sharp enough."

[01:05] **[SOUND: Blade hair test being performed; subtle sound of hair being cut]**

[01:10] **Host:** "Good. Blade is sharp."

[01:15] **[SOUND: Plane being lifted and repositioned]**

[01:20] **Host:** "Look at the grain."

[01:25] **[SOUND: Fingernail or ruler dragging along the board edge to show grain direction]**

[01:30] **Host:** "Run your finger along the edge and feel the slope. Here, the grain rises to the right, which means we need to plane from left to right, against the slope. This prevents tearout."

[01:45] **Host:** "Also notice I'm skewing the plane — holding it at an angle rather than perfectly perpendicular to the board. This reduces the force on the blade and produces a finer shaving. The blade angle, combined with the skew, is the difference between a smooth surface and a splintery mess."

[02:00] **[SOUND: Plane being positioned at a skew angle; woodworker adjusting stance]**

[02:10] **Host:** "Now we take our first pass."

[02:15] **[SOUND: Plane engaging wood; distinct "whoosh" sound of blade cutting; continuous shaving sound]**

[02:20 – 02:45] **[SOUND: Continuous planing sound, rhythmic strokes; no narration]**

[02:50] **Host:** "I'm starting here at the left end, applying pressure downward and forward."

[03:00] **[SOUND: Plane gliding smoothly; consistent cutting sound]**

[03:10] **Host:** "The plane glides smoothly across the wood."

[03:15] **[SOUND: Audible shaving sound; plane blade engagement]**

[03:20] **Host:** "Hear that sound? That's a sharp plane cutting cleanly. Each shaving should be thin and continuous."

[03:30 – 03:45] **[SOUND: Another planing stroke; plane blade sound is clear and smooth]**

[03:50] **Host:** "If the shaving is thick and full of splinters, or if the plane is chattering, stop — the blade might be dull or the skew angle wrong."

[04:05] **[SOUND: Multiple planing strokes, edited together (time-compressed montage, but audio is representative)]**

[04:15] **Host:** "I'm taking several passes, each time covering the entire length."

[04:30 – 04:50] **[SOUND: Continuing planing sounds; multiple passes implied by audio editing]**

[05:00] **Host:** "After the first pass, you can see the change in the surface — it's getting smoother and flatter."

[05:10] **[SOUND: Plane being set down]**

[05:15] **[SOUND: Straightedge being retrieved and placed on board]**

[05:20] **Host:** "After several passes, we check flatness with a straightedge. This is critical."

[05:35] **Host:** "Place the straightedge across the board in different directions — length, width, and diagonals."

[05:50] **[SOUND: Straightedge being moved and positioned]**

[06:00] **Host:** "If there's a gap between the straightedge and the board, that's a high spot. Mark it with a pencil."

[06:10] **[SOUND: Pencil marking, faint graphite sound]**

[06:20] **Host:** "Here, I can see the ends are higher than the middle. This tells me the board is slightly cupped. I need to focus the next few passes on bringing down these high spots."

[06:40 – 08:30] **[SOUND: Multiple planing passes (montage); time-compressed but representative audio showing consistent planing sounds]**

[08:35] **[SOUND: Plane being set down]**

[08:40] **Host:** "Now I'm going to focus on bringing down these high spots. I'll take lighter passes, concentrating on the marked areas."

[08:55] **[SOUND: Lighter planing sounds, shallower cuts, still continuous]**

[09:10] **Host:** "Watch how the pencil marks disappear as the high spots come down."

[09:20 – 09:50] **[SOUND: Multiple lighter planing passes]**

[10:00] **Host:** "After a few passes, I'll check again with the straightedge. Each check tells me how much more work is needed. This is where patience pays off. Rushing and over-planing is the most common mistake."

[10:20] **[SOUND: Straightedge being placed; gap assessment]**

[10:35] **Host:** "Better. The gap is now much smaller. I need just a couple more passes on this end."

[10:50 – 12:00] **[SOUND: Continued targeted planing on high-spot areas]**

[12:05] **[SOUND: Plane being set down; final check]**

[12:10] **Host:** "The board is nearly flat now. These final light passes refine the surface and remove any remaining irregularities."

[12:25 – 17:00] **[SOUND: Final light planing passes; plane sounds are very smooth and quiet, indicating shallow cuts and sharp blade]**

[17:05] **[SOUND: Plane being set down; sanding block being retrieved]**

[17:15] **Host:** "You'll notice the sound changes — it's even smoother now."

[17:30] **[SOUND: 150-grit sandpaper being placed on sanding block]**

[17:40] **Host:** "After the plane passes, I'll sand with 150-grit paper to remove the planing marks and prepare the surface for finishing."

[17:55 – 20:00] **[SOUND: Sanding sounds; sandpaper against wood; fine, whisking sound]**

[20:10] **Host:** "The goal is a smooth surface that's ready for stain or oil. After sanding, the wood feels silky to the touch."

[20:25] **[SOUND: Hand running across sanded surface; very smooth, tactile sound implied]**

[20:40] **[SOUND: Board being removed from vise]**

[20:50] **Host:** "The board is finished. The surface is smooth, flat, and ready for any further work."

[21:00] **[SOUND: Digital caliper measuring final thickness; faint beeping]**

[21:10] **Host:** "The final thickness is approximately seven-eighths of an inch, confirming the target thickness was achieved."

[21:25] **Host:** "This process — from rough-sawn to finished thickness — took about twenty minutes of active planing and sanding. If you're building a larger project, you'd repeat this process for each board, then move on to assembling and finishing your project."

[21:50] **Host:** "Hand-planing gives you complete control and produces superior results compared to machine planing. It's a skill that pays dividends for the rest of your woodworking career."

[22:10] **[SOUND: Workshop ambient, camera movement suggesting end of recording]**

[22:15] **[END OF VIDEO]**
```

---

## 2. Screenshot Specification

### 2.1 When Screenshots Are Taken (Triggers)

Screenshots must be captured at the following moments to ensure comprehensive visual coverage:

1. **Step Initiation:** At the beginning of each major step (the moment before the action, showing the "before" state).
2. **Action Moment:** During or immediately after a key action (the climactic moment: button press, item selection, measurement check, tool engagement).
3. **State Transition:** After an action completes, showing the "after" state (new screen, new position, new visual feedback).
4. **Quality Check Moments:** Whenever a verification or measurement occurs (straightedge check, visual inspection, progress bar, confirmation message).
5. **High-Complexity Sections:** For sections with multiple steps within 30 seconds, more frequent screenshots may be needed (every 10–15 seconds).
6. **Alternative Methods:** When an alternative or pro-tip method is shown, capture the key frames of that alternative (e.g., drag-and-drop vs. button-based upload).
7. **Error or Exception Handling:** If the video shows a mistake or recovery, capture both the error state and the correction.

**Guideline:** For a typical 3–5 minute tutorial, 6–10 screenshots are expected. For a 20+ minute tutorial, 12–25 screenshots may be appropriate, depending on complexity.

### 2.2 Image Format and Quality

**Format:** PNG (lossless, preserves detail, suitable for screenshots)

**Resolution and Downscaling Policy:**

- **Capture Resolution:** Capture at the native resolution of the screen/video being recorded (e.g., if the video shows a 1920×1080 screen, capture at that resolution).
- **Downscaling Policy:** If the native resolution exceeds 2560 pixels in width:
  - Downscale to 2560×1440 (maximum dimension) to keep file sizes manageable (~1–3 MB per image).
  - Use high-quality bicubic or Lanczos downsampling (not nearest-neighbor).
- **For smaller videos/recordings:** Keep at native resolution if it's 1920×1080 or smaller.
- **File Size Target:** Each screenshot should be 500 KB – 2 MB (PNG, after optimization).

**Quality Optimization:**

- Use PNG compression level 6–8 (standard tools like `pngquant` or built-in compression are acceptable).
- Do NOT strip metadata; keep embedded color profile (sRGB).
- Do NOT use PNG interlacing (Adam7) for web use; this is for static documents.

### 2.3 Annotation Policy

**Annotation:** Yes, annotate screenshots to highlight the action being demonstrated.

**Annotation Style:**

- Use a semi-transparent red circle (or arrow) to highlight the interactive element (button, menu item, file being selected).
- Line weight: 2–3 pixels for visibility.
- Circle diameter: Large enough to clearly encompass the element (~30–50 pixels), but not so large as to obscure neighboring UI.
- Opacity: 60–70% (semi-transparent, so background is still visible).
- Arrow annotations (if needed): Use a red arrow (2–3 pixels) pointing from empty space to the element. Offset the arrow start point to avoid obscuring the UI.

**Example Annotation:**
- For "click the New button", draw a red circle around the "New" button, centered on the button icon.
- For "select the Q3_Report.pdf file", draw a red box (semi-transparent) around the file row in the file list.
- For "drag this file to Google Drive", draw a red arrow from the Finder icon to the Drive window.

**Tools:** Annotate using Python (PIL/Pillow), ImageMagick, or manual design tools (design in mockup afterwards). If using code, an annotation module can be part of the script pipeline.

**When NOT to Annotate:**

- If the video narration explicitly points out the element by name and location, a minimal annotation is acceptable.
- For overview shots (showing the full interface), annotations may be omitted if the narration is clear.
- Avoid over-annotating; if a section has 3 different buttons being clicked in sequence, annotate only the first to avoid clutter.

### 2.4 Screenshot File Naming Convention

**Naming Format:**
```
[StepNumber]_[DescriptiveName]_[Timestamp].png
```

**Components:**

- **StepNumber:** Zero-padded 2-digit number (01, 02, 03, ... 99) indicating the step or sequence in the video. Corresponds to sections in the "Detailed Tutorial" section.
- **DescriptiveName:** A short, self-describing name (2–4 words, separated by underscores). Examples:
  - `google_drive_login`
  - `new_button_location`
  - `file_upload_menu`
  - `file_browser_select`
  - `upload_progress_bar`
  - `upload_complete_confirmation`
  - `drag_and_drop_alternative`
  - `grain_direction_inspection`
  - `planing_stance_and_angle`
  - `straightedge_check_cupping`
- **Timestamp:** A 10-digit Unix timestamp (seconds since 1970-01-01 UTC) indicating when the screenshot was captured (for deduplication and audit trail). Examples: `1721901165`, `1721901178`.

**Examples:**
```
01_google_drive_login_1721901165.png
02_new_button_location_1721901178.png
03_file_upload_menu_1721901192.png
04a_file_browser_select_1721901210.png
04b_file_uploading_progress_1721901225.png
05_upload_complete_confirmation_1721901242.png
06_drag_and_drop_alternative_1721901258.png
```

**Rationale:**
- Step numbers allow easy sorting and reference (step 01 is always first).
- Descriptive names make files searchable and self-documenting (no need to open each file to understand its content).
- Timestamps prevent collisions and provide a reproducibility trail (when the analysis was performed).

### 2.5 Screenshot-to-Markdown Referencing

**Markdown Image Syntax:**

```markdown
![Alt text: description of the image](path/to/image.png)
```

**Path Construction:**

The markdown file is located in `/outputs/[VideoTitle]_[Date].md`  
Screenshots are located in `/screenshots/[VideoFolderName]/*.png`

**Relative Path from Markdown to Screenshots:**

```
From: /outputs/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video.md
To:   /screenshots/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video/01_google_drive_login_1721901165.png

Relative path: ../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/01_google_drive_login_1721901165.png
```

**URL Encoding for Spaces and Special Characters:**

When folder or file names contain spaces or special characters (as per R7 folder naming), they must be properly URL-encoded in markdown links:
- Space → `%20`
- Pipe (`|`) → `%7C`
- Colon (`:`) → `%3A`

**Example Markdown Link:**

```markdown
![Step 1: Google Drive login page, showing the URL drive.google.com and the "Sign in" button in the center of the page](../screenshots/Upload%20a%20Document%20to%20Google%20Drive%20in%205%20Steps%20%7C%2025.07.2026%20%7C%20original%3A%20YouTube%20video/01_google_drive_login_1721901165.png)
```

**Alt Text Requirements:**

Every screenshot must have descriptive alt text (the part between `![` and `]`). The alt text must:
- Describe what is shown in the screenshot in detail (as if reading to someone who cannot see the image).
- Be specific (not generic "screenshot" or "button click").
- Provide enough information that a reader could understand the step without seeing the image.
- Example: `"Step 1: Google Drive login page, showing the URL drive.google.com and the "Sign in" button in the center of the page"`

---

## 3. Link and Reference Contract: Handling Offline Markdown

### 3.1 Problem Statement

The markdown file is generated with image references pointing to local screenshot files. However, per R5, "the file is fed to an AI (Claude) to reuse the knowledge in a project." When a user uploads the markdown to Claude without the accompanying screenshots directory, the image links will break. The downstream AI cannot see the images, but should still be able to understand and use the content.

### 3.2 Solution: Rich Textual Descriptions + Sidecar Metadata

**Strategy:**

1. **Every screenshot has an embedded, detailed textual description** (the alt text and the surrounding narrative text in the markdown).
2. **The markdown is self-documenting:** Even without images, the text describes every step in sufficient detail that a downstream AI can understand the procedure and reconstruct it.
3. **A sidecar JSON file** (see Section 6) contains metadata about each screenshot (timestamp, step, description, content hash) so automated tools can verify image correspondence.
4. **Graceful degradation:** If an image link breaks when uploaded to Claude, the text description remains intact.

### 3.3 Relative Path Handling in Uploaded Markdown

When the markdown file is uploaded to Claude (without accompanying screenshots):

1. **Image links will appear as broken references** in Claude's UI (showing `[Screenshot of ...]` as placeholder text).
2. **The alt text becomes the fallback:** Claude will read the alt text and the surrounding narrative.
3. **The text description is authoritative:** Because rich, detailed text descriptions are embedded in the markdown narrative (not just in alt text), the content is preserved.

**Example:** If a user uploads the markdown without screenshots, Claude will see:

```
### Step 1: Navigate to Google Drive and Sign In
...
**What you see on screen:**
A MacBook Safari browser window showing the Google Drive login page (drive.google.com). 
The page displays the Google logo, a "Sign in" button, and a note about accessing files 
from anywhere. The Safari address bar clearly shows "drive.google.com".

![Step 1: Google Drive login page...](../screenshots/...01_google_drive_login_1721901165.png)
```

Even if the image link is broken, Claude reads the "What you see on screen" description and can work from that.

### 3.4 Relative Path Specification

**Path resolution rules:**

1. **When both markdown and screenshots are present locally** (typical case):
   - Relative paths using `../screenshots/[FolderName]/...` work correctly.
   - Links resolve to the physical file.

2. **When markdown is uploaded to Claude without screenshots**:
   - Relative paths become inaccessible (Claude cannot resolve `../screenshots/...`).
   - Image renders as a broken link.
   - **Fallback:** The narrative text (e.g., "What you see on screen:") becomes the primary information source.

3. **If a user wants to upload markdown with embedded images**:
   - Convert relative paths to absolute data URIs or base64-encoded image data.
   - This is outside the scope of this spec (it's a post-processing step).

### 3.5 Broken Link Detection and QA

The QA process (Section 5) includes a check to ensure:
- All `![...](...)` links reference files that actually exist in the screenshots directory.
- No orphan screenshots exist (screenshots without corresponding markdown references).
- This verification happens **before** the file is declared complete.

---

## 4. Rich Textual Descriptions (Accessibility & Downstream AI Use)

### 4.1 Standard for Textual Screenshot Description

Every screenshot in the markdown must be accompanied by a rich, detailed textual description, written in the "What you see on screen:" section of each step. This description must be comprehensive enough that:

1. A reader who cannot see the image understands what it shows.
2. A downstream AI (Claude) can reason about the content and provide guidance based on the text alone.
3. The description is searchable and indexable (for documentation systems).

### 4.2 What to Describe

For each screenshot, the textual description must cover:

- **UI layout:** The spatial arrangement of elements (left sidebar, right panel, central content area).
- **Specific UI components:** Names and labels of buttons, menus, fields, and windows (e.g., "the blue 'New' button in the left sidebar").
- **Visual state:** Colors, highlighting, active states, disabled states (e.g., "the 'Upload' button is grayed out").
- **Content and data:** File names, text in fields, visible data (e.g., "the file list shows Q3_Report.pdf, Budget_2026.xlsx, and Presentation_Draft.pptx").
- **Viewport and size indicators:** Window title, address bar URL, zoom level, etc. (e.g., "Safari browser window; address bar shows 'drive.google.com'").
- **Actionable elements:** Things the user can interact with and their appearance (e.g., "the 'Open' button is in the bottom right corner of the dialog").
- **Feedback or status indicators:** Progress bars, loading spinners, success messages, error messages (e.g., "a green notification appears briefly: 'Q3_Report.pdf uploaded successfully'").

### 4.3 Example Descriptions (Already Shown in Section 1.8)

(See Step-by-Step Tutorial examples in Section 1.8 for detailed "What you see on screen:" descriptions.)

### 4.4 Length and Depth Guidelines

- **For complex UI screenshots:** 2–4 sentences (100–200 words).
- **For simple screenshots (single button or menu):** 1–2 sentences (30–80 words).
- **Target:** Descriptive enough that a reader mentally reconstructs the image.

---

## 5. QA Gates and Quality Assurance Checklist

Before a markdown file is declared finished and ready for delivery, it must pass all of the following QA checks. These checks prevent garbage entries, broken links, incomplete transcripts, and other defects.

### 5.1 Timeline Coverage Check

**Goal:** Verify that the video is covered from start to finish; no gaps or omissions in timeline.

**Procedure:**
- The "Timeline & Chapters" table (Section 1.7) must span from 00:00 to the video's total duration.
- Each row's end time must equal the next row's start time (no gaps).
- The final row's end time must match the video's duration (exact, or within 1 second).
- The sum of all chapter durations must equal the total video duration.

**Pass Criteria:** Timeline is continuous, complete, and accurate.

**Failure Outcome:** Flag as incomplete; require revision.

### 5.2 Broken Link Check

**Goal:** Verify that all markdown image links point to files that exist.

**Procedure:**
- Parse the markdown file for all `![...](path)` image references.
- For each reference, verify the file exists at the resolved path.
- Check that the relative path is correctly constructed (no typos, correct URL encoding).
- Verify file extensions match (`.png`).

**Pass Criteria:** All image links resolve to existing files; no 404s.

**Failure Outcome:** List broken links; require correction before file is released.

### 5.3 Orphan Screenshot Check

**Goal:** Verify that all screenshots in the screenshots directory are referenced in the markdown.

**Procedure:**
- List all `.png` files in the screenshot directory for this video.
- For each file, search the markdown for a reference to that filename.
- Flag any file that is not referenced.

**Pass Criteria:** Every screenshot is referenced at least once in the markdown.

**Failure Outcome:** Warn about unused screenshots; may delete them or add references (usually delete if truly unused).

### 5.4 Duplicate Screenshot Check

**Goal:** Verify that no screenshot is used more than once in the markdown (or if reused, it's intentional).

**Procedure:**
- Parse the markdown for all image references.
- For each screenshot file, count how many times it's referenced in the markdown.
- Flag duplicates (references to the same file in different parts of the markdown).

**Pass Criteria:** No accidental duplicates. (Intentional reuse is acceptable if documented in a comment or note.)

**Failure Outcome:** Flag duplicates; require justification or removal.

### 5.5 Timestamp Monotonicity Check

**Goal:** Verify that timestamps in the transcript and chapter table are in increasing order and accurate.

**Procedure:**
- Extract all timestamps from the "Timeline & Chapters" table.
- Extract all timestamps from the "Full Transcript" section.
- Verify they are in increasing order (t1 < t2 < t3 ..., no duplicates, no backward time travel).
- Verify timestamps in the transcript align with the chapters (e.g., if a chapter covers [00:45 – 01:08], transcript entries should also fall within that range or align correctly).

**Pass Criteria:** Timestamps are monotonically increasing; no gaps or inconsistencies.

**Failure Outcome:** Flag timestamp errors; require correction.

### 5.6 Transcript Completeness vs. Audio Duration

**Goal:** Verify that the transcript covers the entire audio duration of the video.

**Procedure:**
- The video's duration is listed in the front-matter metadata (`video_duration`).
- The transcript should span from 00:00 to `video_duration`, with no unexplained gaps (silent sections are acceptable and should be noted as `[SILENCE]` or `[PAUSE X seconds]`).
- If there are segments of the video with no transcription, flag them and require explanation or transcription.

**Pass Criteria:** Transcript is complete; all audio is accounted for (either transcribed or explicitly noted as silence/inaudible).

**Failure Outcome:** If audio is missing, mark as incomplete and require re-transcription or explicit note of what is missing (e.g., "audio 15:30 – 16:00 is inaudible; background noise too loud").

### 5.7 Claim-to-Evidence Mapping

**Goal:** Verify that every significant claim in the document has supporting evidence (screenshot, transcript quote, timestamp).

**Procedure:**
- For every claim in the "Key Points" section (Section 1.5), verify it has a corresponding timestamp.
- For every procedural step (Section 1.8), verify it references at least one screenshot and includes a transcript quote.
- For every timing estimate (Section 1.10), verify it's based on observed video data or explicitly marked as an estimate.
- For tools and materials (Section 1.9), verify they're mentioned in the transcript or visible in a screenshot.

**Pass Criteria:** Every major claim is traceable to evidence (screenshot, timestamp, or transcript).

**Failure Outcome:** Flag unsupported claims; require evidence or removal.

### 5.8 No Placeholder Text

**Goal:** Verify that no placeholder, draft, or incomplete text remains in the document.

**Procedure:**
- Search for common placeholder patterns: `[TODO]`, `[FIXME]`, `[XXX]`, `...`, `TBD`, "More to come", "Will update", etc.
- Search for incomplete sentences or sections (e.g., "The user then..." without completion).
- Verify all sections are complete and substantive (not empty or containing only a single word).

**Pass Criteria:** No placeholder text; all sections are complete and publishable.

**Failure Outcome:** Flag and require completion.

### 5.9 Spelling and Grammar Check

**Goal:** Verify that the document is free of major spelling and grammar errors.

**Procedure:**
- Use an automated spell checker (e.g., `aspell`, `hunspell`) to identify misspellings.
- Manually review for grammar (subject-verb agreement, tense consistency).
- Review for technical accuracy (terminology is used correctly).

**Pass Criteria:** Spell-check passes; no obvious grammar errors; technical terms are accurate.

**Failure Outcome:** Flag errors; require correction.

### 5.10 QA Sign-Off

After all checks pass, the QA process completes with:

1. **QA timestamp:** Record the date and time when QA completed (in the front-matter metadata: `qa_check_timestamp`).
2. **QA flag:** Set `qa_passed: true` in the front-matter metadata.
3. **QA report:** Optionally, generate a brief QA report listing the checks performed and their results.

---

## 6. Machine-Readable Sidecar File (JSON Metadata)

### 6.1 Purpose and Rationale

Alongside each markdown output file, a machine-readable JSON sidecar file is generated. This file contains structured metadata about the video analysis, the screenshots, and the document itself. This sidecar is useful for:

1. **Indexing and Search:** Downstream tools can index the metadata for full-text search or faceted search (by category, source, tools, etc.).
2. **Automated QA:** Tools can verify link integrity, screenshot correspondence, and other properties automatically.
3. **Analytics:** Tools can track which videos have been processed, which categories are most common, which tools are most frequently used, etc.
4. **Batch Operations:** Scripts can process multiple video analyses in bulk, reading metadata to understand structure and dependencies.
5. **Accessibility:** Screen reader and accessibility tools can use metadata to enhance user experience.

### 6.2 JSON Schema and Structure

```json
{
  "metadata": {
    "schema_version": "1.0",
    "generated_at": "2026-07-25T14:33:12Z",
    "generator": "VideoAnalyser v1.0"
  },
  "document": {
    "title": "Upload a Document to Google Drive in 5 Steps",
    "source": "YouTube video",
    "processed_date": "25.07.2026",
    "processed_time": "14:32:45",
    "video_duration_seconds": 227,
    "total_speakers": 1,
    "category": "Tutorial",
    "tags": ["Google Drive", "cloud storage", "document upload", "step-by-step", "web tutorial"],
    "transcript_status": "complete",
    "markdown_file": "outputs/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video.md",
    "screenshot_folder": "screenshots/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video/"
  },
  "statistics": {
    "screenshot_count": 8,
    "tools_count": 3,
    "materials_count": 0,
    "steps_count": 5,
    "transcript_words": 1247,
    "transcript_speakers": ["Narrator"]
  },
  "screenshots": [
    {
      "filename": "01_google_drive_login_1721901165.png",
      "step": 1,
      "timestamp": "00:12",
      "description": "Google Drive login page, showing the URL drive.google.com and the Sign in button",
      "file_hash": "sha256:abc123...",
      "file_size_bytes": 124567
    },
    {
      "filename": "02_new_button_location_1721901178.png",
      "step": 2,
      "timestamp": "00:45",
      "description": "Google Drive main interface, showing the left sidebar with the blue New button",
      "file_hash": "sha256:def456...",
      "file_size_bytes": 156789
    }
    // ... more screenshots
  ],
  "tools": [
    {
      "name": "Google Drive",
      "type": "Online Service",
      "description": "Cloud storage and file management service",
      "version": "Web-based",
      "cost": "Free (15 GB storage tier)"
    },
    {
      "name": "Web Browser",
      "type": "Software",
      "description": "Required to access Google Drive via the web",
      "version": "Any modern browser (Chrome, Safari, Firefox, Edge)",
      "cost": "Free"
    }
    // ... more tools
  ],
  "materials": [
    // (empty or populated if materials are listed)
  ],
  "timing": {
    "video_duration_seconds": 227,
    "real_world_duration_seconds": {
      "min": 120,
      "max": 300,
      "typical": 180
    },
    "breakdown": [
      {
        "step": 1,
        "chapter": "Navigate and sign in",
        "video_duration_seconds": 33,
        "real_world_seconds": {"min": 30, "max": 120}
      }
      // ... more breakdown
    ]
  },
  "quality_assurance": {
    "qa_passed": true,
    "qa_check_timestamp": "2026-07-25T14:33:12Z",
    "checks": {
      "timeline_coverage": "PASS",
      "broken_links": "PASS",
      "orphan_screenshots": "PASS",
      "duplicate_screenshots": "PASS",
      "timestamp_monotonicity": "PASS",
      "transcript_completeness": "PASS",
      "no_placeholder_text": "PASS"
    }
  }
}
```

### 6.3 Sidecar File Naming and Location

**Naming:** `[VideoTitle]_[Date]_metadata.json`

**Location:** Same directory as the markdown file (in `/outputs/`).

**Example:**
```
outputs/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video.md
outputs/Upload a Document to Google Drive in 5 Steps | 25.07.2026 | original: YouTube video_metadata.json
```

---

## 7. Test Plan: Comprehensive Test Matrix

A comprehensive test plan ensures the script handles all video types, resolutions, orientations, categories, and edge cases mentioned in R1 and R3.

### 7.1 Test Matrix: Video Formats and Codecs

**Purpose:** Verify the script can ingest, analyze, and process a wide variety of input video formats.

| Test ID | Format | Codec | Resolution | Duration | Source Archetype | Expected Outcome | Notes |
|---------|--------|-------|-----------|----------|------------------|------------------|-------|
| T1.1 | MP4 | H.264 | 1920×1080 | 5 min | Tutorial (screen share) | ✓ Markdown generated; audio transcribed; screenshots taken | Standard web format |
| T1.2 | MOV | H.264 | 1920×1080 | 5 min | iPhone screen recording | ✓ Markdown generated; audio transcribed; screenshots taken | Common on macOS |
| T1.3 | MKV | H.265 / HEVC | 2560×1440 | 10 min | 4K screen recording | ✓ Markdown generated; screenshots downscaled to 2560px width; audio transcribed | High-res codec test |
| T1.4 | WEBM | VP9 | 1280×720 | 3 min | Web video | ✓ Markdown generated; audio transcribed; screenshots taken | Alternative web codec |
| T1.5 | AVI | MPEG-4 Part 2 | 1024×768 | 2 min | Older tutorial | ✓ Markdown generated; audio transcribed; screenshots taken | Legacy format |
| T1.6 | M4V | H.264 | 1280×720 | 4 min | iTunes-compatible | ✓ Markdown generated; audio transcribed; screenshots taken | Apple container |
| T1.7 | MP4 | AV1 | 3840×2160 | 8 min | Modern 4K video | ✓ Markdown generated; screenshots downscaled; audio transcribed | Newest codec |

### 7.2 Test Matrix: Video Resolutions and Orientations

**Purpose:** Verify the script handles vertical, horizontal, and unusual aspect ratios.

| Test ID | Resolution | Aspect Ratio | Orientation | Source Archetype | Expected Outcome |
|---------|-----------|--------------|-------------|------------------|------------------|
| T2.1 | 1920×1080 | 16:9 | Horizontal | Standard | ✓ Normal processing |
| T2.2 | 1080×1920 | 9:16 | Vertical | Instagram video | ✓ Normal processing; vertical screenshots |
| T2.3 | 1920×1200 | 16:10 | Horizontal | Widescreen | ✓ Normal processing |
| T2.4 | 3840×2160 | 16:9 | Horizontal | 4K | ✓ Downscale to 2560×1440; quality maintained |
| T2.5 | 1024×576 | 16:9 | Horizontal | Low-res | ✓ Scale up if needed for readability; or keep as-is |
| T2.6 | 720×720 | 1:1 | Square | Social media | ✓ Normal processing; unique aspect ratio noted |

### 7.3 Test Matrix: Video Categories (Archetypes from R1.3)

**Purpose:** Verify the script correctly analyzes different video types and generates appropriate markdown.

| Test ID | Category | Source | Characteristics | Expected Markdown Features |
|---------|----------|--------|-------------------|---------------------------|
| T3.1 | Instagram / Social Video | Instagram | Short (30 sec – 2 min), music, mood focus, no tutorial | Aesthetic analysis; music analysis; mood/style section prominent |
| T3.2 | Video Call with Screen Share | Zoom | Multiple speakers, talking head + screen, Q&A elements | Transcript labels multiple speakers; screen content analyzed; timestamps mark speaker changes |
| T3.3 | iPhone Screen Recording | macOS/iPhone | Tutorial style, app demo, clear narration, 3–10 min | Step-by-step tutorial section; tools list (app names); timing accurate to real-world steps |
| T3.4 | AI-Generated Inspiration Video | YouTube | Specific look, effects, music, no narration or minimal narration | Aesthetic/filter analysis prominent; music analysis detailed; no transcript (or transcript is music titles) |
| T3.5 | Woodworking / Tool Tutorial | YouTube | Hands-on, tools visible, narrated, 10–30 min | Tools & materials list detailed; photos of tools; technique explanation thorough; timing realistic for craft work |

### 7.4 Test Matrix: Edge Cases

**Purpose:** Verify the script handles unusual, difficult, or problematic scenarios gracefully.

| Test ID | Edge Case | Characteristics | Expected Outcome | Error Handling |
|---------|-----------|-----------------|------------------|-----------------|
| T4.1 | No Audio | Video with no audio track or silent video | Script detects no audio; transcript marked "no audio"; visual analysis proceeds; markdown generated without transcript | ✓ Graceful degradation; document is valid but transcript section is empty/marked "not applicable" |
| T4.2 | Music Only | Video with music, no speech | Script detects no speech; transcript marked "music only"; audio analysis focuses on music details | ✓ Document focuses on music/aesthetic analysis; no speech transcript |
| T4.3 | Multiple Speakers, Overlapping | Meeting with 3+ speakers, some overlapping dialogue | Script attempts to label speakers; overlapping sections marked `[OVERLAPPING AUDIO]`; transcript is complete but labeled | ✓ Transcript is complete; overlaps are noted; document is usable |
| T4.4 | Non-English Speech | Video in Spanish, Mandarin, or other language | Script detects language; attempts transcription if STT supports it; falls back to description | ✓ Document generated; transcript marked with language; fallback description provided |
| T4.5 | Very Long Video (2+ hours) | Tutorial or conference recording over 2 hours | Script processes entire video; generates comprehensive markdown; may split into multiple sections | ✓ Markdown is long but navigable (table of contents, clear chapter structure) |
| T4.6 | Very Short Video (< 10 seconds) | Ultra-short clip, GIF-like | Script processes; minimal chapters; 1–2 screenshots | ✓ Markdown is brief but complete; QA passes; document is valid |
| T4.7 | Static Slide with Long Narration | PowerPoint or slide deck read aloud, 5 min | Script detects static image; audio analysis focuses on narration; visual analysis notes static content | ✓ Document focuses on transcript and narration analysis; visual section notes "static content"; minimal screenshot variation |
| T4.8 | Screen Recording with Rapid Scrolling | Tutorial showing list or document, rapid scrolling | Script captures multiple snapshots during scroll; or notes rapid motion | ✓ Multiple screenshots show scroll states; or single screenshot with note "rapid scrolling at timestamp X" |
| T4.9 | Inaudible Sections | Video with background noise, muffled audio, poor quality | Script attempts transcription; sections marked `[INAUDIBLE]`; description of audio quality noted | ✓ Transcript is partial; `[INAUDIBLE]` marks are clear; document notes audio quality issues |
| T4.10 | Corrupted or Damaged Video | File is corrupt, codec unsupported, playback fails | Script detects error; logs message; exits gracefully with error code | ✓ User is informed of the problem; no partial/invalid markdown is generated |

### 7.5 Expected Outcomes for Each Test

For each test, the expected observable outcome is:

1. **Markdown file is generated** (or error is logged if input is invalid).
2. **Markdown file contains all mandatory sections** (metadata, description, transcript, screenshots, tools, timing).
3. **Screenshots are present** (at expected locations, properly named).
4. **QA checks pass** (timeline coverage, no broken links, etc.).
5. **Transcript is complete** (or explicitly marked as "no audio", "partial", etc.).
6. **Timing data is realistic and grounded** (not imaginary numbers; based on observed video).
7. **No placeholder text remains** in the generated markdown.
8. **File is suitable for upload to Claude** (markdown is complete and self-contained).

---

## 8. Acceptance Criteria: Concrete Checklist

The script satisfies the user's request if and only if it meets ALL of the following criteria:

### Core Functionality

- [ ] **Input:** Script accepts a video file path as a command-line argument.
- [ ] **Output Location:** Script saves markdown to `/outputs/`, screenshots to `/screenshots/`.
- [ ] **Markdown Generation:** For each input video, one markdown file is generated per the template (Section 1).
- [ ] **Folder Naming:** Screenshot folders follow R7 naming convention: `{VideoTitle} | {DD.MM.YYYY} | original: {Source}`.
- [ ] **Screenshot Generation:** Screenshots are captured at key moments per Section 2.1; file names follow convention (Section 2.4); all annotated per Section 2.3.
- [ ] **Relative Paths:** Image links in markdown use correct relative paths and URL encoding (Section 3.4).

### Content Requirements (per R4)

- [ ] **Front-Matter Metadata:** Complete YAML metadata block (Section 1.1).
- [ ] **General Description:** 3–5 sentences describing the video (Section 1.2).
- [ ] **Short Summary:** 1–2 sentence distillation (Section 1.3).
- [ ] **Category & Source:** Clear classification and origin statement (Section 1.4).
- [ ] **Key Points:** Bulleted list with timestamps (Section 1.5).
- [ ] **Meeting Main Points (if applicable):** Attendees, decisions, action items, unresolved issues (Section 1.6).
- [ ] **Timeline & Chapters Table:** Continuous coverage from start to end, with chapter names (Section 1.7).
- [ ] **Detailed Tutorial Section:** Full step-by-step reconstruction with embedded screenshots and narration quotes (Section 1.8).
- [ ] **Tools & Materials List:** Comprehensive, annotated, with alternatives and costs (Section 1.9).
- [ ] **Real-World Timing:** Breakdown of video duration vs. real-world time; timing tables for project planning (Section 1.10).
- [ ] **Aesthetic Analysis (if applicable):** Mood, color grading, camera work, editing style (Section 1.11).
- [ ] **Audio Analysis:** Narration, background music, ambient sounds, quality assessment (Section 1.12).
- [ ] **Full Transcript:** Verbatim transcript with timestamps and speaker labels, no paraphrasing (Section 1.13).

### Downstream AI Usability (per R5)

- [ ] **R5.1 Tools List:** Every tool mentioned in the video is listed with description.
- [ ] **R5.2 Materials List:** Every material is listed with quantity and alternatives.
- [ ] **R5.3 Realistic Timing:** Durations are grounded in observed video, not imaginary; examples from user (drying times, curing times) are captured.
- [ ] **R5.4 Project Time Estimation:** The document provides enough timing detail that an AI can estimate total project duration.
- [ ] **R5.5 On-Screen vs. Real-World Time:** Document explicitly distinguishes between edited video time and actual elapsed time (e.g., time-lapses, cuts).
- [ ] **R5.6 Generic Technique Parameters:** Technique parameters are described generically enough to transfer to other projects (grit, speed, angle, passes, safety).

### Screenshot Requirements

- [ ] **Format & Quality:** PNG files, optimized, 500 KB – 2 MB each (Section 2.2).
- [ ] **Resolution:** Native resolution or downscaled to 2560px max width (Section 2.2).
- [ ] **Annotation:** Key interactive elements are marked with semi-transparent red circles/arrows (Section 2.3).
- [ ] **Naming:** All files follow convention `[StepNumber]_[DescriptiveName]_[Timestamp].png` (Section 2.4).
- [ ] **Descriptions:** Every screenshot has rich textual description in "What you see on screen:" section (Section 4).
- [ ] **References:** Every screenshot is referenced in the markdown (Section 2.5).
- [ ] **No Orphans:** No unused screenshots in the directory (Section 5.3).

### Quality Assurance

- [ ] **QA Gates Passed:** Timeline coverage, broken links, orphan check, duplicate check, timestamp monotonicity, transcript completeness, no placeholder text (Section 5).
- [ ] **QA Flag:** Front-matter metadata includes `qa_passed: true` and timestamp (Section 5.10).
- [ ] **No Garbage:** No incomplete sections, no placeholder text `[TODO]`, `[FIXME]`, etc. (Section 5.8).
- [ ] **Spell & Grammar:** Document is spell-checked and grammatically correct (Section 5.9).
- [ ] **Claims Traceable:** Every major claim is supported by screenshot, timestamp, or transcript quote (Section 5.7).

### Sidecar and Metadata

- [ ] **JSON Sidecar:** A `.json` metadata file is generated alongside the markdown (Section 6).
- [ ] **Schema Valid:** JSON conforms to the schema in Section 6.2.
- [ ] **QA Data Included:** JSON includes QA check results (Section 6.3).

### Edge Cases and Robustness

- [ ] **Multi-Format Support:** Script handles MP4, MOV, MKV, WEBM, AVI, M4V (Test Matrix 7.1).
- [ ] **Multi-Resolution Support:** Script handles vertical, horizontal, 4K, low-res videos (Test Matrix 7.2).
- [ ] **Category Handling:** Script adapts markdown for Tutorial, Meeting, Screen Share, Music/Inspiration, Woodworking (Test Matrix 7.3).
- [ ] **Edge Cases:** Script handles no audio, music only, multiple speakers, non-English speech, very long/short videos, static slides, rapid scrolling, inaudible sections (Test Matrix 7.4).
- [ ] **Error Handling:** Script detects corrupted files and logs errors gracefully (no partial markdown generated).

### Performance and Scalability

- [ ] **Reasonable Runtime:** Script completes analysis of a 5–20 minute video in reasonable time (5–15 minutes on M1 MacBook with 16 GB RAM, per R0.3).
- [ ] **M1 Compatibility:** Script runs on M1 MacBook (ARM-based); no x86-only dependencies.
- [ ] **Memory Usage:** Script does not exceed available RAM (target: under 8 GB for typical videos).

### File Outputs and Logging

- [ ] **Output Paths Correct:** Markdown saved to `/outputs/`, screenshots to `/screenshots/`.
- [ ] **File Naming Consistent:** All files follow naming conventions; no ambiguity or collisions.
- [ ] **Logging:** Script logs progress (video processed, screenshots taken, markdown generated, QA checks, errors).
- [ ] **Reproducibility:** Output files are deterministic; running on the same video produces consistent results.

### Integration with User Workflow

- [ ] **Standalone Execution:** User can run the script with `python script.py input_video.mp4` (or similar).
- [ ] **Markdown Uploadable:** Generated markdown can be copied and pasted into Claude with full content preservation.
- [ ] **Offline Usability:** Markdown is readable and useful even without embedded images (text descriptions are sufficient).
- [ ] **Manual Edits Possible:** User can manually edit the markdown (e.g., fix transcription errors, adjust timing, add notes) without breaking QA.

### Documentation and Maintenance

- [ ] **Script is Documented:** Code includes docstrings, comments explaining the algorithm and multi-persona approach.
- [ ] **README Provided:** Instructions for running the script, dependencies, known limitations.
- [ ] **Logging is Clear:** Error messages and warnings are informative enough for user to diagnose issues.

---

**End of Section 4 Specification**

This comprehensive specification document defines the complete output markdown template, screenshot management strategy, QA gates, test plan, and acceptance criteria for the VideoAnalyser tool. It ensures that the delivered markdown files are comprehensive, accurate, reproducible, and suitable for both human readers and downstream AI systems.
