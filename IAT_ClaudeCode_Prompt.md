# Claude Code Prompt: WFH vs In-Office IAT Web App

复制下面整段给 Claude Code。建议在一个新的空文件夹里启动。

---

## ✅ Current Status (last updated: 2026-05-02)

**All 5 steps are COMPLETE.** The app is fully built and tested.

**File locations:**
- App: `/Users/yuwang/Desktop/yw/Projects/1. WFH/Implicit Association Test/iat-app/IAT.html`
- Images: `WFH1.png`, `WFH2.png`, `Inoffice1.png`, `Inoffice2.png` (same folder)

**What's been implemented:**
- Steps 1–5 all done: full 7-block IAT, Order A/B random assignment, D-score (D6), two CSV downloads, 5-page survey, fullscreen with focus-loss handler, timeout handler
- WFH and In-Office stimuli: 8 text words + 2 images each (images appear ~20% of trials, randomly mixed with words)
- Positive and Negative stimuli: text only (8 words each)
- UI: stimulus absolutely centered on screen; key reminder at 1.3rem bold

**To share with participants or advisor:** zip the entire `iat-app` folder (all 5 files must stay together — the HTML needs the image files in the same folder to load correctly).

**Pending / open questions:**
- Consent text and welcome blurb still have `// TODO: researcher to fill in` placeholders — needs IRB-approved language
- May want more images per category (currently 2 each for WFH/In-Office)
- Hosting decision: currently local file only; Netlify Drop is the easiest path if a public URL is needed

---

## Project Context

I'm a researcher building a web-based Implicit Association Test (IAT) to measure implicit attitudes toward Work-From-Home (WFH) vs In-Office work. The app needs to:

1. Run the full 7-block IAT in a browser
2. Record reaction time and accuracy for every single trial
3. Counterbalance block order across participants (Order A vs Order B)
4. Administer a post-IAT survey
5. Export data as CSV files (see "Export" section below for the exact structure)
6. Use text stimuli initially, but the architecture must allow swapping to images later by changing only a config file

I want a **single-page static web app** (HTML + CSS + vanilla JS, no build step, no backend required for the experiment itself). For data collection, the simplest path is: at the end, the participant clicks a button and CSV files download to their machine — I'll collect them manually.

## IAT Design (Kleissner & Jahn, 2020 structure)

**Categories and stimuli (current mix of text + images):**

```
WFH (text):    remote work, work from home, telework, virtual meeting,
               Zoom work, home office, flexible work, remote job
WFH (images):  WFH1.png, WFH2.png

In-Office (text):   office work, in-office work, workplace, office meeting,
                    corporate office, office desk, in-person meeting, workplace office
In-Office (images): Inoffice1.png, Inoffice2.png

Positive (text only): productive, effective, efficient, professional, focused,
                      successful, reliable, motivated
Negative (text only): unproductive, distracted, inefficient, unprofessional, lazy,
                      disengaged, careless, unfocused
```

Images appear ~20% of trials for WFH/In-Office categories (2 images out of 10 items each), randomly mixed with text words. To add more images, append `{ type: 'image', content: 'filename.png' }` entries to the STIMULI config in `IAT.html` and place the image files in the same folder.

**Block structure (Order A — "compatible first"):**

| Block | Trials | Type | Left key (E) | Right key (I) |
|-------|--------|------|--------------|---------------|
| 1 | 24 | Practice | WFH | In-Office |
| 2 | 24 | Practice | Negative | Positive |
| 3 | 24 | Test (compatible) | WFH + Negative | In-Office + Positive |
| 4 | 48 | Test (compatible) | WFH + Negative | In-Office + Positive |
| 5 | 24 | Practice | In-Office | WFH |
| 6 | 24 | Test (incompatible) | In-Office + Negative | WFH + Positive |
| 7 | 48 | Test (incompatible) | In-Office + Negative | WFH + Positive |

**Order B — "incompatible first" (counterbalance):** swap blocks 1↔5, and swap the pairings in 3&4 with 6&7. Specifically: Block 1 has In-Office on left / WFH on right; Block 3&4 are incompatible (In-Office+Negative / WFH+Positive); Block 5 has WFH on left / In-Office on right; Block 6&7 are compatible (WFH+Negative / In-Office+Positive). **Randomly assign each participant to Order A or Order B with 50/50 probability.**

**Within each block:** trials are randomly ordered. For category-only blocks (1, 2, 5), draw equally from each of the 2 categories. For combined blocks (3, 4, 6, 7), draw equally from all 4 categories assigned to that block (so each combined block has equal counts of WFH, In-Office, Positive, Negative items, in random order).

## Trial Mechanics

- Show category labels at the top-left and top-right of the screen for the entire block (e.g., "WFH" top-left, "In-Office" top-right; or in combined blocks, "WFH / Negative" top-left in two lines).
- Show the stimulus (a word, centered, large font) at the start of each trial.
- Participant presses **E** for left, **I** for right.
- If correct: stimulus disappears, brief 250ms inter-trial interval, next trial.
- If incorrect: a red **X** appears below the stimulus and the participant must press the *correct* key to continue. Record both the time-to-first-response (this is the RT used for analysis) and whether the first response was an error.
- Hard timeout: if no response within 10000 ms, log the trial as a timeout (RT = null, error = true) and move on.

## Data to Record (internally, per trial)

For every trial, the program internally records:
- `participant_id` (generated at start, format: `P` + timestamp + 4 random digits)
- `order_condition` ("A" or "B")
- `block_number` (1–7)
- `block_label` (e.g., "Practice: WFH/InOffice", "Test: WFH+Neg / Office+Pos")
- `block_type` ("practice" | "compatible" | "incompatible")
- `trial_number_in_block` (1-indexed)
- `trial_id` (string like `"B1_T1"`, `"B1_T2"`, ..., `"B7_T48"` — used as column names in summary CSV)
- `stimulus` (the word shown)
- `stimulus_category` ("WFH" | "InOffice" | "Positive" | "Negative")
- `correct_key` ("E" or "I")
- `first_response_key`
- `first_response_correct` (boolean)
- `rt_ms` (time from stimulus onset to first response; null if timeout)
- `total_trial_time_ms` (time from onset until correct key was pressed)
- `timestamp_iso` (when the trial started)

## D-score (Greenwald, Nosek, & Banaji, 2003 — the "improved" algorithm, D6 variant)

Compute and store on the participant summary row:

1. Use only Blocks 3, 4, 6, 7 (the test blocks).
2. Drop trials with RT > 10000 ms.
3. Mark participant as `excluded_fast_responder = true` if more than 10% of their trials have RT < 300 ms (but still keep their data; just flag).
4. For error trials, replace RT with (block mean of correct trials) + 600 ms penalty.
5. Identify compatible blocks vs incompatible blocks based on `order_condition`:
   - Order A: compatible = Blocks 3+4, incompatible = Blocks 6+7
   - Order B: compatible = Blocks 6+7, incompatible = Blocks 3+4
6. Compute pooled SD across all included trials in test blocks (3,4,6,7 combined).
7. **D-score = (mean RT incompatible − mean RT compatible) / pooled SD.**
   Positive D = stronger association of WFH with Negative (i.e., anti-WFH bias).
8. Also compute D separately using only the 24-trial test blocks vs only the 48-trial test blocks, and store both as `D_score_24trial` and `D_score_48trial`.

## Survey (after IAT)

Present these questions on subsequent pages after the IAT finishes:

1. **Stereotype ratings (1–5 Likert, separately for WFH firms and In-Office firms):**
   Productive, Efficient, Focused, Professional, Reliable, Motivated.
   Show as a table with radio buttons.

2. **Overall favorability (0–10 numeric input or slider):**
   - "How favorably do you feel toward WFH firms?"
   - "How favorably do you feel toward in-office firms?"

3. **Comparative preference (single choice, 5 options):**
   Strongly prefer in-office / Moderately prefer in-office / No preference / Moderately prefer WFH / Strongly prefer WFH.

4. **Paper review scenario (single choice, 5 options as in the design doc).**

5. **Demographics:** age (number), gender (radio), primary field (text), career stage (radio: PhD student / Postdoc / Assistant Prof / Associate Prof / Full Prof / Other), current work mode (radio: In-office / Hybrid / WFH).

Store every survey answer as its own column on the participant summary row.

## Export — IMPORTANT: this is the exact structure I want

At the end of the survey, show a "Download data" button that triggers TWO file downloads to the participant's machine:

### File 1: Per-participant trial-level CSV — `IAT_<participant_id>.csv`

**Long format**, one row per trial. This is the detailed log for one participant. Columns (in this order):

```
participant_id, order_condition, block_number, block_label, block_type,
trial_number_in_block, trial_id, stimulus, stimulus_category,
correct_key, first_response_key, first_response_correct,
rt_ms, total_trial_time_ms, timestamp_iso
```

Total rows: 216 (24+24+24+48+24+24+48 trials).

### File 2: Single-participant summary CSV — `IAT_summary_<order>_<participant_id>.csv`

**Wide format**, ONE row per participant. The filename includes the order letter (e.g., `IAT_summary_A_P12345.csv` or `IAT_summary_B_P12345.csv`). I will manually concatenate all Order-A files into one master file later, and all Order-B files into another — keeping them separate avoids mixing semantically different blocks (because Block 3 means different things in the two orders).

Columns, in this order:

1. **Identifier and metadata columns:**
   - `participant_id`
   - `order_condition`
   - `start_time_iso`, `end_time_iso`, `duration_seconds`
   - `excluded_fast_responder` (boolean)

2. **D-score and aggregate columns:**
   - `D_score_overall`, `D_score_24trial`, `D_score_48trial`
   - `accuracy_block1` … `accuracy_block7` (proportion correct in each block)
   - `mean_RT_compatible`, `mean_RT_incompatible`

3. **Per-trial RT columns** (216 columns):
   - Format: `B<block>_T<trial>_RT`
   - Order: `B1_T1_RT, B1_T2_RT, ..., B1_T24_RT, B2_T1_RT, ..., B2_T24_RT, B3_T1_RT, ..., B7_T48_RT`
   - Value: the `rt_ms` from that trial (null/empty if timeout)

4. **Per-trial error columns** (216 columns):
   - Format: `B<block>_T<trial>_ERR`
   - Order: same as RT columns above (`B1_T1_ERR, B1_T2_ERR, ..., B7_T48_ERR`)
   - Value: 1 if `first_response_correct` was false, 0 if true, empty if timeout

5. **Survey answer columns** (one per question):
   - Stereotype ratings: `stereo_WFH_productive`, `stereo_WFH_efficient`, `stereo_WFH_focused`, `stereo_WFH_professional`, `stereo_WFH_reliable`, `stereo_WFH_motivated`, `stereo_office_productive`, … (12 columns total)
   - Favorability: `favor_WFH`, `favor_office`
   - `comparative_preference` (store as a string like "moderately_prefer_WFH")
   - `paper_review` (store as string)
   - `age`, `gender`, `primary_field`, `career_stage`, `current_work_mode`

Total columns ≈ 13 metadata/aggregate + 216 RT + 216 ERR + ~22 survey ≈ **~467 columns, 1 row.**

## Architecture Requirements

- **Single HTML file** (or HTML + one JS + one CSS, your call) — no build step, no npm.
- **Stimuli config in one place** — at the top of the JS, an object like:
  ```js
  const STIMULI = {
    WFH: [
      { type: 'text', content: 'remote work' },
      { type: 'text', content: 'work from home' },
      // ...
    ],
    InOffice: [ /* ... */ ],
    Positive: [ /* ... */ ],
    Negative: [ /* ... */ ],
  };
  ```
  And a render function `renderStimulus(item)` that handles both `type: 'text'` and `type: 'image'` (for images, render an `<img>` from `content` as a path/URL). Even though I'm only using text now, the image branch must work — write it and leave a comment showing how I'd swap a word to an image later.

- **No external dependencies** other than (optionally) a simple CSS reset. No React, no jQuery.
- **All timing must use `performance.now()`**, not `Date.now()`, for sub-millisecond precision.
- **Stimulus presentation timing:** make sure the stimulus is actually painted to the screen before starting the RT timer — use `requestAnimationFrame` to start the timer on the frame the stimulus first appears.
- **Prevent accidental key holds:** ignore `keydown` repeats (`event.repeat === true`).
- **Fullscreen / focus:** add a "Begin" button that requests fullscreen and the experiment runs in fullscreen. If the participant exits fullscreen mid-trial, pause and show a "please return to fullscreen" message.

## UI / Pages (in order)

1. **Welcome page**: study title, brief description (placeholder text — I'll edit it), consent checkbox, "Begin" button.
2. **Instructions page** for each block, shown before the block starts. Show the category labels and which key goes to which side. "Press SPACE to start."
3. **Block runs.** Top-left and top-right show the category labels persistently. Stimulus is absolutely centered on screen (do not rely on flex flow — use `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)`). Below the stimulus, show the key reminder "E = left  I = right" at `1.3rem`, bold, dark (`#333`).
4. **Between-block break screen** with progress ("Block 3 of 7 complete").
5. **Survey pages** (one section per page, "Next" button to advance, validation that all required fields are answered).
6. **Thank-you page** with the "Download data" button. After download, show "You may close this window."

## Build Order — ALL STEPS COMPLETE ✅

1. ~~**Step 1:** Welcome page, stimulus config, Block 1 practice, console.log output.~~ ✅
2. ~~**Step 2:** Full 7-block sequencing, Order A/B random assignment, instruction + break screens.~~ ✅
3. ~~**Step 3:** D-score (D6) computation, two CSV downloads (trial-level + summary with 216 RT + 216 ERR cols).~~ ✅
4. ~~**Step 4:** Full 5-page survey, survey answers appended to summary CSV.~~ ✅
5. ~~**Step 5:** Fullscreen, focus-loss handler, timeout handler.~~ ✅

## Things to ask me about before assuming

- Anything about the consent text or welcome blurb — leave a clearly-marked `// TODO: researcher to fill in` placeholder, don't invent IRB language.
- Visual design preferences — default to a clean, high-contrast, accessible look (white background, large dark text, sans-serif). I'll tweak later.

Let's start with Step 1.
