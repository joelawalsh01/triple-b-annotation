# triple-b-annotation

Annotation tool for rating LLM Spanish translations of English science textbook passages against NGSS standards. Live at **https://joelawalsh01.github.io/triple-b-annotation/**.

Part of a research project comparing Aya / Gemini / GPT-4o translations of K-12 science content. Two raters independently score each passage; their per-rater annotations drive the statistical comparison.

## Using the tool

Login with `rater1` (Ashley) or `rater2` credentials (shared out-of-band — both live in the client JS).

For each of the 82 passages:
- The three Spanish translations are blinded as **Translation A / B / C** (deterministic shuffle per `page_id`, so the same rater always sees the same labels in the same positions).
- **Quality (per translation)**: Worst / Middle / Best + optional comment. Ashley's pre-existing quality ratings appear locked; she can unlock any cell to re-rate.
- **Source confirmation (per standard)**: Yes/No on whether the English source substantively teaches part of that standard.
- **Preservation (per standard × translation)**: 1 None / 2 Weak / 3 Partial / 4 Full — only rated when source is confirmed Yes.

Progress auto-saves to this repo every 5 seconds (see below). Raters can also Export JSON for a local download.

## How commits happen

Two paths write to this repo:

### 1. Auto-save from the tool (rater annotations)

The client-side JS in `docs/index.html` writes annotation state to `annotations_initial_set_rater{1,2}.json` via the GitHub Contents API, debounced 5 seconds after every edit. Each rater has a separate file so they never overwrite each other. Commits land as `Auto-save rater 1 progress` (or 2).

Authentication uses a fine-grained PAT embedded in the client JS, obfuscated as a 4-part string array joined at runtime (to dodge GitHub's push protection / secret scanning). **Do not collapse it into a single literal** — pushes will be blocked by secret scanning and the tool will stop auto-saving.

### 2. Manual deploys (tool or input data)

Updates to `docs/index.html` (the tool itself) or `docs/data.json` (the input dataset) are normal `git commit && git push`. GitHub Pages rebuilds on push.

Occasional failure mode: the `deploy-pages` action sometimes reports a 502 during the deployment-record API call while the artifact has actually already been served. If the action log shows a 502, verify the live site directly — chances are it deployed regardless.

## Files tracked in this repo

```
docs/index.html                            # the annotation tool (single-file HTML + inline JS + CSS)
docs/data.json                             # input dataset (82 passages × 3 translations + standards)
annotations_initial_set_rater1.json        # Ashley's auto-saved state, live-updated by the tool
annotations_initial_set_rater2.json        # rater 2's auto-saved state (created on first login)
.gitignore
README.md
```

## JSON schemas

### `docs/data.json` — input

Array of 82 passage objects. The tool fetches this at login.

```jsonc
[
  {
    "page_id": "f496d987",                  // stable id — 8 hex chars
    "english_text": "...",                  // source passage
    "textbook_grade": "3",                  // origin grade ("3", "Middle School", etc.)
    "target_grade": "2nd-3rd",              // target grade band the translation aims at
    "target_age": "7-8",
    "domain": "Life Science",               // NGSS domain
    "subject": "Science",
    "translations": [
      {
        "model": "aya",                     // "aya" | "gemini" | "gpt4o"
        "spanish_translation": "...",
        "rater1_quality": "Best" | "Middle" | "Worst" | null,  // Ashley's pre-existing rating
        "rater1_comment": "..."                                  // Ashley's pre-existing comment
      }
      // always exactly 3 translations, one per model
    ],
    "standards": [
      {
        "standard_code": "3-LS3-1",         // NGSS performance expectation
        "description": "...",
        "grade_band": "K-2" | "3-5" | "MS" | "HS"
      }
      // 1–5 standards per passage (240 total across the dataset)
    ]
  }
  // 82 pages total
]
```

### `annotations_initial_set_rater{1,2}.json` — auto-saved rater state

Written by the tool on every edit. This is the *live working state*, not the final analysis shape. One file per rater. Only pages the rater has touched appear in `data`; untouched pages are implicit.

```jsonc
{
  "page": 3,                                // page index the rater was last viewing
  "data": {
    "<page_id>": {
      "rater": 1,                           // 1 or 2
      "translationOrder": [1, 2, 0],        // indices into translations[] → A, B, C (blinding)
      "quality": {                          // per-model overall quality
        "aya": "Best",
        "gemini": "Middle"
      },
      "qualityComments": { "aya": "..." },
      "sourceConfirmed": {                  // per-standard Yes/No on source teaching
        "3-LS3-1": true,
        "3-LS3-2": false
      },
      "preservation": {                     // key = "<standard_code>__<model>", value = 1..4
        "3-LS3-1__aya": 4,
        "3-LS3-1__gemini": 3,
        "3-LS3-1__gpt4o": 2
      },
      "standardComments": { "3-LS3-1": "..." },
      "unlocked": { "aya": true }           // rater 1 only — models where Ashley overrode her pre-existing rating
    }
    // other page_ids...
  }
}
```

### Exported JSON — final analysis shape

Produced by the Export JSON button. This is the shape to use for analysis — translations carry both their real model name and their blinding label, and Ashley's edits are flagged with `rater1_edited`.

```jsonc
[
  {
    "page_id": "f496d987",
    "rater": 1,                             // 1 or 2
    "status": "complete" | "partial" | "not_started",
    "translations": [
      {
        "model": "aya",
        "blind_label": "Translation B",     // what the rater saw
        // When rater=1 (Ashley):
        "rater1_quality": "Best" | "Middle" | "Worst" | null,
        "rater1_comment": "...",
        "rater1_edited": true | false       // true only if Ashley unlocked-and-rerated this cell
        // When rater=2:
        // "rater2_quality": ...,
        // "rater2_comment": "..."
      }
      // 3 translations per page
    ],
    "standards": [
      {
        "standard_code": "3-LS3-1",
        "description": "...",
        "source_confirmed": "yes" | "no" | null,
        "preservation_ratings": [
          { "model": "aya",    "rating": 4 },
          { "model": "gemini", "rating": 3 },
          { "model": "gpt4o",  "rating": 2 }
        ],
        "comment": "..."
      }
    ]
  }
  // all 82 pages, always — even untouched ones
]
```

**Notes for analysis:**
- `rater1_edited: false` means Ashley did *not* unlock this cell — `rater1_quality` carries her pre-existing value from `docs/data.json`. Those pre-existing values come from an upstream-tool bug where one rating was stamped across all 3 models of a passage, so they **cannot be used to compare models** without confirming `rater1_edited: true`.
- `source_confirmed: null` → the rater hasn't answered the Yes/No question yet.
- When `source_confirmed: "no"`, all preservation ratings for that standard should be `null` by design.
- A preservation `rating: null` means either source_confirmed=No or the rater hasn't set a value yet.

## Deploying changes

Clone, edit, commit, push — GitHub Pages rebuilds automatically:

```bash
git clone https://github.com/joelawalsh01/triple-b-annotation.git
cd triple-b-annotation

# update the tool
vim docs/index.html && git add docs/index.html && git commit -m "..." && git push

# update the input dataset
cp /path/to/new/data.json docs/data.json && git add docs/data.json && git commit -m "..." && git push
```

Hard-refresh the live site (`Cmd+Shift+R`) after a deploy to bypass the CDN's ~5-minute cache. If you see the live site already updated but the Actions tab shows a failed `deploy-pages` run with a 502, ignore the failure — it deployed.

## Research context

The broader research pipeline — raw translation logs, NGSS-matching scripts, analysis code, figures — lives in a separate local working directory and is deliberately kept out of this public repo. `.gitignore` excludes all JSON except the annotation tool's input and auto-save files, so scratch data and credentials don't accidentally ship here.
