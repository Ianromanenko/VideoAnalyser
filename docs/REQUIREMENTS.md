# VideoAnalyser — Requirements (source of truth)

Extracted from the user's original request. Every plan revision and every grill round
must be checked against this list. Nothing here may be silently dropped, narrowed, or
re-interpreted. Items marked **[HARD]** are literal, non-negotiable specifications.

## R0. Nature of the deliverable
- R0.1 A **local script**, run from the terminal on a macOS machine.
- R0.2 Language: **Python** (user's assumption, confirmed as correct).
- R0.3 Target hardware: **Apple MacBook, M1 chip, 16 GB RAM**. Everything must fit that box.
- R0.4 Invocation: point it at a video file; it does the rest.

## R1. Input coverage
- R1.1 **All video format types** (container + codec variety: mp4, mov, mkv, webm, avi, m4v, …).
- R1.2 **All resolutions** (vertical Instagram 1080x1920 … 4K screen recordings).
- R1.3 Known source archetypes the user actually feeds it:
  - R1.3.a Instagram / social video (mood, look, style, music).
  - R1.3.b **Video calls with screen share** — person talks while showing something.
  - R1.3.c iPhone / macOS **screen recordings** of tutorials.
  - R1.3.d AI-generated "inspiration" videos (specific look, style, music genre).
  - R1.3.e Woodworking / hand-tool / shop tutorials (user's hobby).

## R2. Core analysis requirement
- R2.1 Analyze the video **frame by frame**.
- R2.2 Analyze the **audio**.
- R2.3 **Pair exactly** what is on screen with what is said at that specific time.
- R2.4 **Perfect synchronization, without losing anything.** This is the central requirement.
- R2.5 Frame-accurate association of speech ↔ visual state, including moments where the
  speaker narrates a change happening on screen.

## R3. Multi-persona analysis
The user explicitly asked for **many different personas**, each analyzing from a different
perspective, so that maximum detail is extracted and nothing is lost. Named examples:
- R3.1 A persona that analyzes **the source** (where the video came from).
- R3.2 A persona that **categorizes** what is on the video.
- R3.3 A persona that processes **frame by frame**.
- R3.4 A persona that extracts **key points / what makes it different from others**.
- R3.5 A persona that looks at it from the **time perspective** (is it a tutorial, duration of steps).
- R3.6 A persona that **verifies tutorial completeness** — all steps and details collected.
- R3.7 A persona that **verifies screenshot↔audio matching**, no garbage, no broken links.
- R3.8 "…and so on and so on" — the persona set is expected to be broad, not minimal.
- R3.9 Goal of the persona set: **maximum extraction of details from all possible
  perspectives**, producing the "perfect complex markdown video details file".

## R4. Output file — content requirements
One **markdown** file per video. It must contain **all** of:
- R4.1 All collected details.
- R4.2 **All timestamps.**
- R4.3 If it is a meeting: **main points**.
- R4.4 **General description.**
- R4.5 **Short summary.**
- R4.6 **Word-by-word transcript** (verbatim, not paraphrased).
- R4.7 **Screenshots of the screen at specific points**, where a claim needs visual backup.
- R4.8 For an explained topic (e.g. 2 minutes of explanation inside a 7-minute video):
  a **text summary of the explanation together with a screenshot of every step explained**.
- R4.9 For an N-step tutorial (user's example: 5 steps to upload a file in a web system):
  the process **fully described without losing any detail** — the website shown, the button
  being pressed, how each screen looks, step by step, "as if it would be a tutorial".
- R4.10 For woodworking/tool tutorials: **what tool is it, what are they doing, why, how,
  steps, screenshots**.
- R4.11 For music/inspiration videos: **what style, what mood, what filters were used to
  achieve such effects**, music style on the background.
- R4.12 The file must serve as a **standalone tutorial** a reader could follow.

## R5. Downstream use — the file is fed to an AI (Claude)
The user will upload the produced markdown to Claude to reuse the knowledge in a project.
Therefore the file must let a downstream AI answer, without seeing the video:
- R5.1 **Which tool(s) do I need** to achieve this result → explicit tool list.
- R5.2 **Full list of tools and materials** (e.g. if wood must be stained).
- R5.3 **How long it takes** — real, physically grounded durations, not imaginary numbers.
  Explicit examples given by the user, all of which must be capturable:
  - staining → ~half a day of drying,
  - glue under clamps → ~2 days to fully cure,
  - steam-bending → ~30 min of steam so wood becomes bendy, then shape + fix + dry.
- R5.4 The AI must be able to **estimate project time** from the extracted timing facts.
- R5.5 Distinguish **on-screen elapsed time** from **real-world process time** (a 10-second
  cut can represent 2 days of glue curing) — the file must never conflate them.
- R5.6 Applicability: user wants to lift a technique (e.g. angle-grinder surface finishing)
  into a different project (coffee table top/sides) — so technique parameters must be
  captured generically enough to transfer (grit, speed, angle, passes, safety).

## R6. Folder structure **[HARD]**
A root folder named **`Video Analysis`** containing exactly three folders:
- R6.1 **`raw`** — the user drops input videos here.
- R6.2 **`outputs`** — the script saves all resulting markdown files here.
- R6.3 **`screenshots`** — all screenshots needed by the markdown files.

## R7. Screenshot folder naming **[HARD]**
Inside `screenshots`, **one separate folder per processed video**. Name format, per the
user's own example:

```
Claude skills and best practices | 28.07.2026 | original: instagram recording
```

Three parts:
- R7.1 Part 1 — the **actual video** the screenshots come from (its title/subject).
- R7.2 Part 2 — the **timestamp of when the video was processed** (date, `DD.MM.YYYY` per example).
- R7.3 Part 3 — the **source of origin** of the original video
  (Instagram recording, YouTube video, iPhone screen recording, other web source, …).
- R7.4 All screenshots inside must be **named accordingly** (self-describing names).

## R8. Quality bar
- R8.1 "Maximum effort", "save as much detail as it can".
- R8.2 "Not to lose anything."
- R8.3 "Underline all possible perspectives about that video."
- R8.4 No garbage entries, no broken links (explicit QA persona duty, R3.7).

## R9. Process requirements for producing THIS plan
- R9.1 The plan is composed using **maximum effort of the smallest available model**.
- R9.2 The plan is then **grilled** (adversarial gap-finding) — up to **20 interactions**,
  stopping early once the plan is solid, to avoid an infinite loop.
- R9.3 When grilling is done, **notify the user**; they will execute the final plan
  themselves via the `/goal` skill.
- R9.4 Therefore the final plan must be **executable by an agent with no access to this
  conversation** — fully self-contained.
