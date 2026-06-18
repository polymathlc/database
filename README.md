# Polymath WordVault — the teacher‑in‑a‑box

A single‑file web app (`index.html`) that turns the Singapore‑syllabus material **you**
feed it into a teaching assistant that can stand in for you. It runs entirely in the
browser on Firebase (Auth + Firestore) and Gemini (via Firebase AI Logic). No build step,
no server — open the file or host it as a static page.

> Everything you save is kept **word for word**. The AI only *splits, labels, OCRs,
> explains and marks* — it never rewrites your stored text or answer keys.

---

## What it does (mapped to the goal)

The student interface is **one search bar**. Type any question (or upload an image of it) and
press Enter; the AI answers using a single unified library — it draws keywords from the
**syllabus** first and falls back to the **question bank** to complete the answer, then replies
with an answer + explanation phrased for the student. The syllabus knowledge base and the
question bank are managed together by the teacher in one **Library**.

| You wanted… | Where it lives |
|---|---|
| One **unified** knowledge base + question bank | Teacher **📚 Library** holds both (syllabus notes/PDFs *and* OCR'd questions with answer keys); the student **🔎 Ask** searches across both. |
| Students get **just one search bar** | The whole student UI is the Ask bar — ask any question, optionally attach an image. No tabs, filters or other buttons. |
| **AI OCR** of PDFs & screenshots | Every page/screenshot is rendered to an image and read by Gemini vision — works on **scanned papers with no text layer**. |
| Convert each question's **text + picture → descriptive text + answer key** | Ingestion captures `questionText` (verbatim OCR), an `imageDescription` of any figure, and an `answerKey` + `approvedKeywords`. |
| Take the answer **word for word from the answer key in the last pages** | Set **"Answer-key pages at end"** when ingesting a PDF; those pages are transcribed verbatim and each question's answer is copied from them word for word. |
| Edit any stored answer | Each question in the Library has an ✏️ button to edit its answer key and approved keywords inline. |
| Ask → **answer in the essence of the keywords**, tailored | Ask consults the syllabus then the question bank; the reply carries the essence of both sources' key terms but is phrased to directly answer the student. |
| **Mark** student answers + give feedback | **✅ Mark** (teacher) — paste/upload the question and the student's answer; the AI crafts the correct answer from the syllabus + question bank, then scores it with feedback. |
| **Remember** wrong answers so marking stays consistent | **🚩 Mistakes** (teacher only) — upload a student's wrong answer, comment on the mistake, and save it. Marking then flags the same mistake and reuses your comment on similar answers. |

---

## Roles

Sign‑in is with Google. Your account (`ADMIN_EMAIL` in `index.html`) is the **Teacher** and sees
the full nav: **Ask**, **Library** (load syllabus + OCR/edit the question bank), **Mark**, and
**Mistakes** (your private bank of marked student answers). Everyone else is a **Student** and
sees **only the Ask search bar** — nothing else. The Mistakes bank is never readable by students;
it holds individual students' wrong answers and is loaded only for the teacher.

---

## Data model (Firestore collections)

All documents carry an `owner` (the writer's `uid`), an `ownerEmail` (the writer's lowercased
email, used to share the teacher's library with students) and `createdAtMs`.

- `wordvault_sources` — the full original text of everything ingested (your permanent record).
- `wordvault_entries` — verbatim knowledge‑base passages: `text, title, subject, topic, type, keywords[], verified, sourceId`.
- `wordvault_questions` — the question bank:
  `questionText, imageDescription, subject, topic, type, level, keywords[],`
  `answerKey, approvedKeywords[], markingPoints[], thumb, sourceId`.
  The `answerKey` and `approvedKeywords` can be edited in place from the Question Bank.
- `wordvault_mistakes` — the teacher's bank of marked student answers (teacher‑only):
  `questionText, imageDescription, studentAnswer, teacherComment, correctAnswer,`
  `mistakeTags[], subject, topic, keywords[], questionId, thumb`.
  Retrieved at marking time so the AI flags the same mistakes and reuses the teacher's comments.
- `wordvault_attempts` — a log of Ask/Mark interactions (non‑fatal if it fails).

Question thumbnails are downscaled JPEGs stored inline so each doc stays well under
Firestore's 1 MB limit. Full‑resolution images are only used in‑memory at OCR time.

---

## Setup

1. **Firebase** — the app uses the shared Polymath project (`firebaseConfig` in `index.html`).
   Enable **Google sign‑in**, **Firestore**, and **Firebase AI Logic (Gemini)** for that project.
2. **App Check (recommended)** — paste your reCAPTCHA v3 site key into `RECAPTCHA_SITE_KEY`
   to protect Gemini quota. If left as the placeholder, App Check is skipped.
3. **Host it** — open `index.html` locally, or serve the repo as a static site
   (e.g. GitHub Pages). Sign in with the admin Google account to begin loading the syllabus.

### Firestore security rules (shared class library)

The teacher's **library** (syllabus + question bank) is shared with every signed‑in student so
Ask answers are grounded in your material. Sharing is **automatic in the app** — every library doc
is stamped with the writer's email (`ownerEmail`) — but Firestore's security rules are the real
boundary, so you must publish a rule once in the Firebase console for students to read your library.
Replace `chungzhikai@gmail.com` below with your `ADMIN_EMAIL` if it differs.

> ⚠️ **Firestore rules are shared by the whole database, not per‑app.** Every app using the same
> (default) Firestore database in this project obeys the *same* published ruleset. If you share this
> project with other apps, **do not** paste the full ruleset below over your existing rules — it would
> govern their collections too. Use the *additive* snippet instead.

**Recommended (project shared with other apps) — add, don't replace.** Keep your current rules and
add these blocks inside your existing `match /databases/{database}/documents { … }`. Firestore grants
access if *any* matching rule allows it, and a literal‑path `match` only ever *adds* permission, so
this touches no other collection and removes no existing access:

```
// Let any signed-in student READ the teacher's WordVault library.
match /wordvault_sources/{doc} {
  allow read: if request.auth != null
    && resource.data.ownerEmail == 'chungzhikai@gmail.com';
}
match /wordvault_entries/{doc} {
  allow read: if request.auth != null
    && resource.data.ownerEmail == 'chungzhikai@gmail.com';
}
match /wordvault_questions/{doc} {
  allow read: if request.auth != null
    && resource.data.ownerEmail == 'chungzhikai@gmail.com';
}
```

This assumes your existing rules already let the teacher *write* the `wordvault_*` collections (they
must, since uploads work today). It adds only the student cross‑account *read*.

**Alternative (WordVault has the project to itself) — complete ruleset.** Only use this if no other
app shares the database, since it replaces everything:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /{col}/{doc} {
      // Read your own docs; ALSO read the teacher's shared library by email.
      allow read: if request.auth != null
        && (resource.data.owner == request.auth.uid
            || (col in ['wordvault_sources','wordvault_entries','wordvault_questions']
                && resource.data.ownerEmail == 'chungzhikai@gmail.com'));
      // You may only write/own your own docs.
      allow write: if request.auth != null
        && resource.data.owner == request.auth.uid;
      allow create: if request.auth != null
        && request.resource.data.owner == request.auth.uid;
    }
  }
}
```

Either way, only the three library collections are shared. `wordvault_mistakes` (a student's wrong
answers) and `wordvault_attempts` are **not** listed, so they stay strictly owner‑only — students can
never read them. Students still write their own `wordvault_attempts`; only the teacher writes the
library.

**Existing uploads** made before this feature are migrated automatically: the next time the teacher
signs in, the app stamps `ownerEmail` onto any library docs that lack it (a one‑time, no‑op‑afterwards
backfill), so you don't need to re‑upload anything.

> Want a private, single‑teacher setup with **no** sharing instead? Simply don't add the rule above —
> students then see only their own data and Ask falls back to the model's general knowledge.

---

## Day‑to‑day workflow

1. Teacher → **Library**: load the syllabus (paste or upload digital PDFs) so answers have
   verbatim passages to draw keywords from.
2. Teacher → **Library**: ingest question papers (screenshots/scans). If the answer key is in
   the last pages, set **"Answer-key pages at end"** so each answer is taken word for word from
   it. Review, then save. Edit any answer later with the ✏️ button.
3. Students just **Ask**: type a question (or upload an image) in the single search bar and
   press Enter. Teachers can also **Mark** a student's answer.
4. Teacher → **Mistakes**: when a student gets something wrong, upload/paste the question and
   the wrong answer, write your comment (optionally let *“✨ Draft what's wrong”* pre‑fill it for
   you to edit), and **Save**. From then on, marking a similar answer flags the same mistake and
   reuses your comment. You can also hit *“🚩 Save as a marked mistake”* straight from a Mark result.

---

## Writing help on every field

Every text box in the app — questions, answers, comments, the syllabus paste box, the
Ask/search bars, the optional hints and the inline answer‑key editors — carries two small
helpers in its corner:

- **✨ Improve** — sends just that box's text to Gemini to fix **grammar, spelling and
  phrasing only**. Your meaning, facts, numbers, names and answer keys are never changed,
  and the box keeps roughly the same length; tap **↩** to undo and restore your original.
- **🎤 Speak** — records your microphone and transcribes it with Gemini, inserting the words
  at the cursor. Requires microphone permission and a secure page (https or localhost); the
  button hides itself where recording isn't supported.

The live **quick‑filter** boxes are deliberately left plain (they filter as you type, so
there's nothing to polish). Nothing extra is stored — both helpers reuse the same Firebase
AI Logic / Gemini stack the rest of the app already runs on.

---

## Notes & limits

- OCR/answer quality depends on image clarity and on how much syllabus material you've
  loaded. The tutor is deliberately **grounded** — if it lacks the material it says so
  rather than inventing an answer.
- **Finding stored material is fuzzy and forgiving.** Retrieval (Ask, Mark and the
  question/mistake banks) matches on word **forms and misspellings**, not exact text — so a
  saved *“frictional force”* note is still found by *“friction”*, *“explain friction”* or even
  the typo *“fictional force”*. It cascades through phrase → word → character‑trigram
  signals and hands the closest candidates to Gemini to judge, so relevant notes are no
  longer missed because the wording differed.
- PDF ingestion is capped at 40 pages per file and processed in small page‑chunks.
- The model is configured in `GEMINI_MODEL`.
</content>
</invoke>
