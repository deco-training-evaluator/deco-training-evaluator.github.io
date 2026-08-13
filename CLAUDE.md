# DECO Training Evaluator — project context

Read this first, every session. It replaces the memory that used to live in the "Training" Claude Project.

## Who you're working with

Jordan Evans (jordan.evans@decorepair.com), DECO Windshield Repair. **Not a developer** — no coding background, no command-line experience. Explain things in plain language, avoid jargon, and don't assume familiarity with git, terminals, or web tooling. When something needs to happen outside this repo (in Google Apps Script, Google Cloud Console, GitHub's website), give click-by-click steps.

Always write the company name as **DECO** or **DECO Windshield Repair** — in caps, never "Deco".

## What this app is

A web app that grades DECO field-training sessions. A manager uploads an audio recording of a training session; the app sends it to Google's Gemini API, which scores it against DECO's real training rubric and returns section-by-section scores, a total, a certified Yes/No, and written feedback. It also tracks where each field manager stands on their certification path.

**Live at:** https://deco-training-evaluator.github.io

Built as a hackathon project for DECO leadership (judged on Complete / Critical Thinking / Useful, 1–3 each).

## How it's wired together

```
index.html (this repo, served by GitHub Pages)
   │  Google sign-in (Google Identity Services), Sheets + Drive REST APIs
   │
   ├── reads/writes ──> Google Sheet "DECO Training Tracker - Roster"
   │                      1LAqMWl_sSv0NSvzh2Z4YefgxpAr2RZtjvepYvful66s
   ├── writes ────────> Google Sheet "DECO Training Tracker - Grading Log"
   │                      1TsfoRDTZa73ruDGC1uDaSJJ7WKr4STQRTNNbz7S2H2M
   ├── uploads audio ─> Drive folder "DECO Training Tracker - Audio Uploads"
   │                      1Gj4Z8GeNzThuHXTUY0chEEvQixg2Zr8z
   │
   └── POSTs to ──────> Google Apps Script Web App ("Training AI Grader")
                          bound to the Grading Log sheet; holds the Gemini
                          API key server-side in Script Properties
                             │
                             └──> Gemini API (audio + prompt in, JSON out)
                                  model: gemini-3.5-flash (GEMINI_MODEL property)
```

Apps Script Web App URL (the `AI_GRADING_ENDPOINT` constant in `index.html`):
`https://script.google.com/a/macros/decorepair.com/s/AKfycbyWTE5fjMMSr2pExK0EuyuExZhXT0Por7k2Hh6vXeSrXRbAlDT7qrOsErKz6W_SnY3I/exec`

The official *human* evaluations live in Smartsheet, not in this app: **"Training Evaluations"** (Training Day) and **"Site Visit Evaluations"** (Team On-Site Day). Those are the source of truth for calibrating the AI's accuracy.

## Repo layout

- `index.html` — the entire front end. Single self-contained file: HTML, CSS, and JS inline. Two tabs: **Dashboard** (certification roster) and **Training** (upload audio + grade a session).
- `apps-script/Code.gs` — the backend. **This is a copy.** The version that actually runs lives inside the Apps Script editor bound to the Grading Log sheet. Editing it here does nothing until it's pasted back in and redeployed (see below).
- `docs/` — the rubrics and reference data the Gemini prompt is built from. **If a rubric or a policy number changes, `docs/` and the prompt text inside `Code.gs` both have to be updated to match.**

## Deploying changes

**Front end (`index.html`):** commit and push to `main`. GitHub Pages redeploys automatically in 1–2 minutes. That's the whole process.

**Backend (`Code.gs`):** editing the file in this repo is not enough. It must be pasted into the Apps Script editor (Grading Log sheet → Extensions → Apps Script), saved, and then **Deploy → Manage deployments → pencil icon → New version → Deploy**. Skipping the New Version step leaves the old code running at the same URL. Label the version to match the file's header (currently V10).

## Rules that matter when changing grading logic

- **BC only.** Everything is scoped to British Columbia. Other provinces are the final phase and explicitly not active work — don't build toward them.
- **Replacements 101 is always graded in BC**, in every territory, never marked N/A. (Confirmed by Jordan; a region-based alternative was considered and rejected.)
- **Certification formulas** must match the Smartsheets exactly:
  - Training Day: total out of 110 across 11 sections; certified = total ≥ 85 **and** every section ≥ 6.
  - Team On-Site Day: total out of 50 across 5 sections; certified = total ≥ 38 **and** every section ≥ 6.
- **A section with no confident score counts as 0**, not as "can't determine." `certified` is always a real Yes/No.
- **Consensus grading:** every session is graded by 3 sequential Gemini calls that get reconciled (median per section). This costs ~3x tokens and ~3x time versus one call — the tradeoff is deliberate, because single calls produced wildly inconsistent scores. Don't "optimize" it back to one call.
- Because grading is 3 sequential calls, a long recording can approach Apps Script's 6-minute web app limit. If timeouts appear, either lower `GRADING_RUNS_PER_SESSION` to 2 or restructure as parallel `UrlFetchApp.fetchAll` calls.
- **Evidence timestamps (V10).** Every section returns `evidence_clips` — 1–3 `{at, quote}` moments pointing at the audio that drove its score — and the results card turns those into clickable jump points on an in-page player. The point is that a flagged section can be checked in about a minute instead of by re-listening to the whole session. These are an aid to a human's judgement, **not** citations: Gemini's audio timestamps land near the right moment, not on it, which is why the prompt asks for the *start* of an exchange and the player seeks 10 seconds early. Don't present them as exact anywhere in the UI.

## Known quirks worth knowing before you debug

- **Column S of the Grading Log holds all 3 raw runs concatenated**, separated by `----- next run -----` — not one clean JSON object. `index.html` has a `parseAiResultFromRow()` function that handles both that format and the older single-JSON format. Optional future cleanup: have `Code.gs` also write one reconciled JSON object so the front end doesn't need to rebuild it.
- **Because of that, reconciliation logic is deliberately duplicated in two places** — and has to stay in sync. A freshly graded session gets its reconciled sections from `Code.gs`; the same session re-opened from Recent Submissions gets them rebuilt in `index.html` from those raw runs. `parseClipSeconds` / `formatClipTime` / `mergeClips` and the `SCORE_SPREAD_FLAG_THRESHOLD` constant exist in both files. Change one, change the other, or a session will look different depending on how it was opened. (This bit before: the front end's rebuild used to collapse every non-unanimous section into a plain "insufficient evidence", so the `†` and Double-check flags silently vanished from any session re-opened from the log. Fixed in V10.)
- **Apps Script Web App access must be set to "Anyone", not "Anyone within decorepair.com".** The domain-restricted setting breaks CORS when called via `fetch()` from GitHub Pages, even though the endpoint works fine when opened directly in a browser. This cost real debugging time once — don't change it back.
- **Sign-in can't be made permanent.** This is a client-only Google auth flow; tokens last ~1 hour and are silently renewed in the background, but that renewal depends on the browser's own Google session. Truly indefinite sign-in would need a backend holding a refresh token. Don't promise more than that.
- The progress bar during AI grading is an **estimate**, not real server progress — Apps Script is one blocking call with no progress feed. The UI says so explicitly. Keep it honest.
- OAuth client ("Training Tracker Web") is **User type: Internal**, so any @decorepair.com account can sign in with no verification screen. Authorized origins include both `https://deco-training-evaluator.github.io` (current) and `https://decomancer128.github.io` (the original personal-account URL). Adding a new domain means adding it here too, or sign-in silently breaks.

## Where things stand

Roadmap re-cut by Jordan on 2026-08-13 for the leadership PowerPoint. The through-line is *trust earned in steps*: the AI reads, then it advises, then — only once it has proven itself against real human grades — it decides. Every phase after 2 is gated on the one before it actually being accurate, not on a date.

**Phase 1 — done.** Site live. Upload an audio recording of a training session and get a reading back from the AI, end to end.

**Phase 2 — in progress (current). Training Day accuracy.** Take real Training Day recordings, grade them with the AI, and put that side by side with the district manager's actual human grade from the "Training Evaluations" Smartsheet. Refine the rubric and prompt until the two line up. Findings go in the "DECO AI Grading — Calibration Decisions Log (Phase 2)" Sheet (`1o4VvjAgdQ6VNPNt4-kEiu_nH5GrFh4LHe4jVo5TRNzc`) — one row per finding, not raw scores. The UI work is already done here: full visual rebuild from DECO's brand book (exact palette `#ef7f23` / `#eeaf66` / `#f0e6d3` / `#a69a83`, real logo, Sora + Caviar Dreams fonts), click-to-expand results in Recent Submissions, readable section cards, a two-click delete, dashboard table fitting one view, numeric manual grade on the rubric's own 110 scale, a dedicated Certified column, the tab renamed "Training" with Team On-Site Day removed, and an honest estimated-progress indicator. On 2026-08-13 Jordan reviewed live output and asked for shorter, more digestible written feedback: the V9 prompt caps justifications at one sentence and strong/weak/focus entries at "label: one clause" (2–4 entries per list) while still requiring every covered/missed item to appear, and the results card bolds those labels, one-lines the footnotes, and breaks reviewer notes into bullets. Also on 2026-08-13, **evidence timestamps (V10)**: the card was flagging sections as doubtful without saying where to check them, so the flags went unchecked — now each section carries the AI's own "Listen at" jump points, a flagged section also shows what all 3 passes scored it (`9 · 6 · 3` reads very differently from `8 · 8 · 5`), and the recording plays in the page instead of only linking out to Drive. This is the piece that makes calibration tractable: when the AI and the Smartsheet disagree, you can hear what the AI heard instead of guessing why.

**Phase 3 — AI recommends, human decides.** Once the grading criteria are consistently good enough, the app gives a recommendation on a Training Day. It is decision *support*: a manager still makes the call.

**Phase 4 — AI decides on Training Day.** Keep feeding it examples and keep refining until its own rating is reliable enough to make the actual approved / not approved call on a training session, without a human re-grading it first. This is the bar Phase 3 is building toward — don't shortcut to it.

**Phase 5 — Team On-Site Days.** Same treatment applied to Team On-Site Day evaluations (50-point rubric, 5 sections).

**Phase 6 — Recruiting activities.** Same treatment applied to recruiting and interviews.

**Bonus phase (may not happen) — grade the trainee.** Grade the person being trained, not just the trainer, producing a needs-a-Day-3 / cut / ready-for-the-field recommendation. **Recommendation only — never the decision, at any level of accuracy.** Rubric still needs writing. Jordan hasn't committed to building this.

**Final phase — company-wide.** Roll it out across DECO, beyond BC.

## Open items

- **AI grading consistency is still not fully solved.** Consensus grading improved it but didn't eliminate drift. Treat any AI result near a pass/fail threshold as directional until real calibration data exists.
- Sections that are inherently visual (uniform, tent setup, tool organization, drill technique) can't be scored from audio alone. Expect `insufficient_evidence` there — that's a medium limitation, not a prompt bug.
- **Evidence timestamp accuracy is not yet validated against a real recording.** The mechanism is tested (merging, capping, the flagged-section cases, the past-the-end-of-file guard) but how *close* Gemini's cited times actually land is unknown until V10 runs on a real session. First real use should check a few clips against the audio: if they're consistently landing early/late, adjust `CLIP_PREROLL_SECONDS`; if they're landing in the wrong part of the session entirely — especially deep into a long recording — the timestamps aren't trustworthy and the UI should say so more loudly than it currently does.
- **Langley glass shop phone: (825) 962-7747** — given by Jordan 2026-08-13, now in `docs/bc-shop-info.md`, `docs/gemini-grading-prompt.md`, and the `Code.gs` reference data. Written in as a *soft* fact (a rep saying a different number is noted, not penalized) because 825 is an Alberta area code and BC numbers are 604/778/236 — possibly a call-routing number, possibly needs a re-check. Harden it once confirmed. **Code.gs still needs redeploying for this to reach live grading.**
- One reference-data gap left, non-blocking: the complex-chip price ceiling ($125) is an unconfirmed assumption.
- **Team access:** the Roster/Grading Log Sheets and the Audio Uploads Drive folder still need to be shared (Editor) with the rest of the team. This is the last real blocker to someone other than Jordan using the app end to end. Sign-in itself is already fine.
- Roster certification columns are updated by hand; auto-flipping them from AI results isn't built.

## Working preferences

- **Verify UI changes with a real browser** before saying they're done — render the page headless, screenshot it, check for console errors. This has caught real regressions on this project.
- **Don't overpromise.** Where a limitation is real (sign-in longevity, progress accuracy, audio-only grading), say so plainly in both the code and the conversation rather than papering over it.
- **Resist scope creep.** Leadership explicitly rewards judgment about what *not* to build. Two live-monitoring features were deliberately scoped and dropped once it turned out the existing workflow already covered the need.
- When something is edited outside a Claude session, note it here so this file stays a true picture of the project.
