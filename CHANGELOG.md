# Changelog

All notable changes to **Polymath WordVault** are listed here, newest first. The
running version is shown at the **bottom-left** of the app — click it to see this
history in‑app and whether a newer version is available.

This project uses [semantic versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`).

> Keeping versions in sync: when you ship a change, bump `APP_VERSION` **and** add an
> entry to `CHANGELOG` (both in `index.html`), update `version.json`, and add the same
> entry here. The hosted `version.json` is what the app's "up to date?" check reads.

## [1.5.0] — 2026-06-18
- Added a **version badge** at the bottom-left and this **version history**. Clicking the
  badge opens the changelog in‑app, and when the app is hosted it checks `version.json`
  and tells you whether you're **up to date** or an **update is available** (refresh).

## [1.4.0] — 2026-06-18
- **Teacher:** every Ask answer now shows **📑 Rules behind this answer** — the stored notes
  it was grounded in — so you can see why it replied that way.
- **Teacher:** **✏️ Edit & refine** an answer; the AI reconciles your correction with those
  notes and saves it as a **verified** knowledge note, so the system keeps improving.

## [1.3.0] — 2026-06-18
- New **🗂️ Flashcards** deck for every user — save a card straight from an Ask answer (the AI
  keeps it short) or write your own; edit, delete and filter your deck.
- **Practice with spaced repetition** (SM‑2) scheduled on the **Ebbinghaus forgetting curve**:
  cards resurface right as you're about to forget them, most‑decayed first.

## [1.2.0] — 2026-06-18
- **Smarter, more reliable search.** Retrieval now tolerates word forms and misspellings — a
  saved "frictional force" note is found by "friction", "explain friction" or even the typo
  "fictional force" — so stored material is no longer missed because the wording differed.

## [1.1.0] — 2026-06-18
- Every text field gained two helpers: **✨ Improve** (AI tidies grammar & phrasing, never the
  meaning) and **🎤 Speak** (voice input transcribed by Gemini).

## [1.0.0] — 2026-06-17
- Initial **Polymath WordVault**: one **Ask** search bar, the teacher **Library** (syllabus +
  question bank), **Mark**, and the **Mistakes** bank — grounded in your material and kept
  word‑for‑word.
