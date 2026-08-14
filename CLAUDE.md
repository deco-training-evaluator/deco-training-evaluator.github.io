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

- `index.html` — the entire front end. Single self-contained file: HTML, CSS, and JS inline. Seven tabs: **Dashboard** (certification roster) plus one per session type — **Training Day**, **Team On-Site Day**, **Conversion Call**, **Reference Checks**, **Stage 1 Interview**, **Stage 2 Interview**. The Reference Checks tab covers two session types (candidate and parent) behind one picker — see its `variants` array.
  - The tab row is generated from a single `SESSION_TABS` array. Each entry is either `status: 'live'` (Code.gs has a rubric for that exact `sessionType` string) or `status: 'preview'` (the tab exists and explains itself but does nothing).
  - **The two live tabs share one grading panel.** `showTab()` re-points `currentSessionType` at the panel rather than there being two copies of the upload form — so the manual-grade cap, the labels, the pill, and which Recent Submissions rows are shown all follow the active tab. Adding a live session type means adding an entry to `SESSION_TABS` and an entry to `SESSION_MAX_POSSIBLE`, nothing more on this side.
  - Because one panel is shared and an AI grade takes ~3 minutes, `gradeSessionWithAI` captures its session type at submit time and refuses to paint progress or results into a panel that has since switched tabs — it shows a "still grading in the background" note on the other tab instead and drops the finished grade into Recent Submissions.
  - **Don't flip a `'preview'` entry to `'live'` just by editing the string.** Code.gs rejects any `sessionType` it has no rubric for; it needs the rubric text, the section keys, the max score, and a redeploy first.
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
- **Passes that fail are survivable, but a degraded result must say so (V11).** A 429 or 5xx on one pass is retried (waiting the delay Google suggests, capped so backoff can't eat the 6-minute budget); a *daily* quota 429 is not retried, because waiting can't clear a daily cap. If a pass still fails, grading proceeds with whatever completed rather than throwing away passes already paid for — but the reviewer notes say how many actually ran, and **a single completed pass flags every section for human review**. That last part matters: `reconcileSection` can't detect disagreement it never saw, so one uncorroborated pass would otherwise produce a confident-looking score with nothing behind it.
- Because grading is 3 sequential calls, a long recording can approach Apps Script's 6-minute web app limit. If timeouts appear, either lower `GRADING_RUNS_PER_SESSION` to 2 or restructure as parallel `UrlFetchApp.fetchAll` calls.
- **Evidence timestamps (V10).** Every section returns `evidence_clips` — 1–3 `{at, quote}` moments pointing at the audio that drove its score — and the results card turns those into clickable jump points on an in-page player. The point is that a flagged section can be checked in about a minute instead of by re-listening to the whole session. These are an aid to a human's judgement, **not** citations. Don't present them as exact anywhere in the UI.
- **Measured timestamp accuracy — 2026-08-13, the only real measurement so far.** Jordan hand-checked 5 citations against a 2:22 test clip: off by **+4s, +4s, +3s, +24s, +23s**. Three conclusions, all of which shaped the current design:
  1. **Every error ran the same way — the AI cites EARLY, the true moment is always LATER.** Never once late. That's why `CLIP_PREROLL_SECONDS` is 3 and not the original 10: a large pre-roll moved the listener *away* from the moment in every measured case. It's also why the nudge buttons are weighted forward (−15 / +15 / +30).
  2. **The error is not drift and not a constant offset.** It's accurate to ~4s, then slips ~20s all at once and never recovers — both bad citations were after ~1:00, both off by the same amount, and the AI even put a 1:07 line *before* a 0:58 one. Suspected cause: a long silence or an edit in the audio that the model doesn't count. **No fixed offset or scale factor can correct this** — that was checked and rejected. The nudge controls exist precisely because the error isn't correctable.
  3. **Untested at length.** Real 60-minute sessions are full of dead air, so slips may happen repeatedly and stack. If a citation on a long recording is minutes out rather than seconds, the honest response is to stop rendering a clickable time and show only the quote, so a manager searches for words instead of trusting a number that lies. Re-measure before trusting timestamps on a full session.

## Known quirks worth knowing before you debug

- **Column S of the Grading Log holds all 3 raw runs concatenated**, separated by `----- next run -----` — not one clean JSON object. `index.html` has a `parseAiResultFromRow()` function that handles both that format and the older single-JSON format. Optional future cleanup: have `Code.gs` also write one reconciled JSON object so the front end doesn't need to rebuild it.
- **Because of that, reconciliation logic is deliberately duplicated in two places** — and has to stay in sync. A freshly graded session gets its reconciled sections from `Code.gs`; the same session re-opened from Recent Submissions gets them rebuilt in `index.html` from those raw runs. `parseClipSeconds` / `formatClipTime` / `mergeClips` and the `SCORE_SPREAD_FLAG_THRESHOLD` constant exist in both files. Change one, change the other, or a session will look different depending on how it was opened. (This bit before: the front end's rebuild used to collapse every non-unanimous section into a plain "insufficient evidence", so the `†` and Double-check flags silently vanished from any session re-opened from the log. Fixed in V10.)
- **Gemini quota and cost — this caused a confusing failure once.** The API key was on Gemini's **free tier: 20 `generate_content` requests per day** for `gemini-3.5-flash`. Consensus grading spends **3 per session**, so that's ~6 sessions a day, and a day of testing exhausts it. When it runs out, the page shows **"AI grading request failed: Failed to fetch"**, which looks like a network or CORS problem and is not — the real error (HTTP 429) is only visible in **Apps Script → Executions**. Always check that log before theorising. Cost on the paid tier: audio bills at 32 tokens/second, so a 60-minute session is ~115k tokens per pass and ~345k across 3; at $1.50/1M in and $9.00/1M out that's **roughly $0.63 a session**. Fixed by enabling billing on the API key's Google Cloud project (AI Studio → API keys → the project → Set up Billing).
- **Testing costs real money now.** Use a short clip (2–5 minutes) for anything that only exercises plumbing — whether timestamps render, whether the card parses, whether a deploy took. A 5-minute clip is ~1/12 the cost of an hour-long one and tests the mechanism just as well. Save full-length sessions for actual calibration work, where the grade itself is the point.
- **` | ` is a structural separator, not punctuation.** `notes_for_human_reviewer` is one long string that `index.html` splits on ` | ` to render bullets. Anything appended to it — including error text from a failed pass — must not contain ` | ` internally, or one note shatters into several nonsense bullets. This bit V11 on its first live run, when raw Google error JSON was appended after a ` | raw: ` marker.
- **Raw API responses belong in Apps Script → Executions, never on the results card.** `callGemini` logs the full failure response there; the message that reaches a manager is a plain sentence. When debugging, that log is the source of truth.
- **Apps Script Web App access must be set to "Anyone", not "Anyone within decorepair.com".** The domain-restricted setting breaks CORS when called via `fetch()` from GitHub Pages, even though the endpoint works fine when opened directly in a browser. This cost real debugging time once — don't change it back.
- **Sign-in can't be made permanent.** This is a client-only Google auth flow; tokens last ~1 hour and are silently renewed in the background, but that renewal depends on the browser's own Google session. Truly indefinite sign-in would need a backend holding a refresh token. Don't promise more than that.
- The progress bar during AI grading is an **estimate**, not real server progress — Apps Script is one blocking call with no progress feed. The UI says so explicitly. Keep it honest.
- OAuth client ("Training Tracker Web") is **User type: Internal**, so any @decorepair.com account can sign in with no verification screen. Authorized origins include both `https://deco-training-evaluator.github.io` (current) and `https://decomancer128.github.io` (the original personal-account URL). Adding a new domain means adding it here too, or sign-in silently breaks.

## Where things stand

Roadmap re-cut by Jordan on 2026-08-13 for the leadership PowerPoint, then re-cut again the same day into the version below — **this is the current one**. The shape changed in two ways worth knowing, because earlier notes in this repo may still reflect the old cut: *breadth now comes before authority* (open the tool up to every session type first, then earn the right to decide), and *the AI advises from Phase 2 onward* rather than waiting for a separate "recommendation" phase. Every phase is gated on the one before it actually working, not on a date.

**Phase 1 — done.** Site live. Upload an audio recording and get a reading back from the AI, end to end.

**Phase 2 — in progress (current). Get Training Day dialled in — as a recommendation.** Take real Training Day recordings, grade them with the AI, and put that side by side with the district manager's actual human grade from the "Training Evaluations" Smartsheet. Refine the rubric and prompt until the two line up. Findings go in the "DECO AI Grading — Calibration Decisions Log (Phase 2)" Sheet (`1o4VvjAgdQ6VNPNt4-kEiu_nH5GrFh4LHe4jVo5TRNzc`) — one row per finding, not raw scores. The UI work is already done here: full visual rebuild from DECO's brand book (exact palette `#ef7f23` / `#eeaf66` / `#f0e6d3` / `#a69a83`, real logo, Sora + Caviar Dreams fonts), click-to-expand results in Recent Submissions, readable section cards, a two-click delete, dashboard table fitting one view, numeric manual grade on the rubric's own 110 scale, a dedicated Certified column, the tab renamed "Training" with Team On-Site Day removed (superseded 2026-08-13 — it is now "Training Day", one of six session-type tabs; see Repo layout), and an honest estimated-progress indicator. On 2026-08-13 Jordan reviewed live output and asked for shorter, more digestible written feedback: the V9 prompt caps justifications at one sentence and strong/weak/focus entries at "label: one clause" (2–4 entries per list) while still requiring every covered/missed item to appear, and the results card bolds those labels, one-lines the footnotes, and breaks reviewer notes into bullets. Also on 2026-08-13, **evidence timestamps (V10)**: the card was flagging sections as doubtful without saying where to check them, so the flags went unchecked — now each section carries the AI's own "Listen at" jump points, a flagged section also shows what all 3 passes scored it (`9 · 6 · 3` reads very differently from `8 · 8 · 5`), and the recording plays in the page instead of only linking out to Drive. This is the piece that makes calibration tractable: when the AI and the Smartsheet disagree, you can hear what the AI heard instead of guessing why. **The output is explicitly a recommendation, not a verdict** — the district manager doing the grading uses it to inform their call, they don't defer to it. Every phase up to 5 keeps that framing; don't let the UI drift into sounding authoritative.

**Phase 3 — Open it up to the other avenues.** Same upload-audio-get-feedback loop, extended past Training Day to: **Team On-Site Day**, **Conversion Call**, **Reference Checks**, **Stage 1 Interview**, **Stage 2 Interview**. Still feedback, still not deciding. Two structural notes:
- **Reference Checks is one tab, not two.** The candidate reference check and the *parent* reference check share a tab, with a picker inside it for which one you're grading (Jordan's call, 2026-08-13 — a second tab would be clutter). They still need separate rubrics and separate `sessionType` strings in `Code.gs`; the `variants` array on that `SESSION_TABS` entry holds them.
- **Team On-Site Day is already wired** — `Code.gs` has always carried its rubric (50 points, 5 sections) and it has a live tab, so a recording can be graded today. What it hasn't had is calibration against the real human grades in the "Site Visit Evaluations" Smartsheet. Treat any Team On-Site Day score as uncalibrated until that happens.
- The other four are **preview tabs only** as of 2026-08-13: they explain themselves and show a dimmed mock of the form, and nothing uploads or scores. Rubrics exist (Jordan has them) but are deliberately not loaded yet. The tabs went in early for the leadership presentation, to show where this goes.

**Phase 4 — Continuous feedback, coaching, and fine-tuning.** The tool becomes something the team uses constantly rather than at certification moments: ongoing feedback and coaching across all those session types, with the volume of real examples that generates feeding back into fine-tuning the grading. This is where accuracy gets earned at scale — Phase 5 isn't reachable without it.

**Phase 5 — The AI starts making the decisions.** Once it's fine-tuned, it makes the actual call: certifying someone on a Training Day, and the equivalent decision on the other session types. Used for both certifications and continuous development. **This is the only phase where the AI decides anything** — everything before it is advisory, and that gate is the point of the whole roadmap.

**Bonus phase (may not happen) — grade the other side of the conversation.** So far everything grades the person *running* the session. This would grade the person on the receiving end: the **trainee** (needs-a-Day-3 / cut / ready-for-the-field), the **interviewee**, and the **employee being evaluated on a Team On-Site Day**. **Recommendation only — never the decision, at any level of accuracy**, unlike Phase 5. Rubrics still need writing, and Jordan hasn't committed to building it.

**Final phase — company-wide.** Expand across DECO, beyond BC.

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
