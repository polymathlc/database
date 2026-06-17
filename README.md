# Polymath WordVault — the teacher‑in‑a‑box

A single‑file web app (`index.html`) that turns the Singapore‑syllabus material **you**
feed it into a teaching assistant that can stand in for you. It runs entirely in the
browser on Firebase (Auth + Firestore) and Gemini (via Firebase AI Logic). No build step,
no server — open the file or host it as a static page.

> Everything you save is kept **word for word**. The AI only *splits, labels, OCRs,
> explains and marks* — it never rewrites your stored text or answer keys.

---

## What it does (mapped to the goal)

| You wanted… | Where it lives |
|---|---|
| Learn the syllabus textbook **word for word** | **📚 Knowledge Base** — paste text or upload digital PDFs; AI splits into verbatim entries that are verified against the original source. |
| Feed **PDFs and screenshots** | **❓ Question Bank → Ingest** accepts images *and* PDFs (and pasted screenshots). |
| **AI OCR the entire PDF** | Every page (or screenshot) is rendered to an image and read by Gemini vision — works even on **scanned papers with no text layer**. |
| Convert each question's **text + picture → descriptive text + answer key** | Ingestion captures `questionText` (verbatim OCR), an `imageDescription` of any figure/diagram, and an `answerKey` + `approvedKeywords` + `markingPoints`. |
| Take the answer **word for word from the answer key in the last pages** | Set **"Answer-key pages at end"** when ingesting a PDF; those pages are transcribed verbatim and each question's answer is copied from them word for word (the AI only drafts an answer when no key is supplied). |
| Edit any stored answer | In the **Question Bank**, each question has an ✏️ button to edit its answer key and approved keywords inline; changes are saved to Firestore. |
| Students upload a question image → **how to do it, in approved keywords** | **🎓 Ask** — photograph/type a question; the tutor matches it to the bank + knowledge base and explains the method, **grounded only in your stored material** and weaving in your approved keywords. |
| **Mark** student answers + give feedback | **✅ Mark** — upload/type a question and the student's answer; get a score, marks, points met/missed, and feedback in your voice. |

---

## Roles

Sign‑in is with Google. Your account (`ADMIN_EMAIL` in `index.html`) is the **Teacher** and
sees every tool. Everyone else is a **Student** and sees a read‑only Knowledge Base plus
**Ask** and **Mark**. Ingesting and editing the Question Bank are teacher‑only.

---

## Data model (Firestore collections)

All documents carry an `owner` (the writer's `uid`) and `createdAtMs`.

- `wordvault_sources` — the full original text of everything ingested (your permanent record).
- `wordvault_entries` — verbatim knowledge‑base passages: `text, title, subject, topic, type, keywords[], verified, sourceId`.
- `wordvault_questions` — the question bank:
  `questionText, imageDescription, subject, topic, type, level, keywords[],`
  `answerKey, approvedKeywords[], markingPoints[], thumb, sourceId`.
  The `answerKey` and `approvedKeywords` can be edited in place from the Question Bank.
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

### Firestore security rules (single teacher — default)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /{col}/{doc} {
      allow read, write: if request.auth != null
        && resource.data.owner == request.auth.uid;
      allow create: if request.auth != null
        && request.resource.data.owner == request.auth.uid;
    }
  }
}
```

### Optional — a shared class library for students

To let students read the teacher's bank/knowledge base:

1. Put the **teacher's `uid`** in `SHARED_LIBRARY_UID` (top of the `<script>` in `index.html`).
2. Allow signed‑in students to *read* (not write) the teacher's docs:

```
match /{col}/{doc} {
  allow read: if request.auth != null
    && (resource.data.owner == request.auth.uid
        || resource.data.owner == "TEACHER_UID_HERE");
  allow write: if request.auth != null
    && resource.data.owner == request.auth.uid;
  allow create: if request.auth != null
    && request.resource.data.owner == request.auth.uid;
}
```

Students still write their own `wordvault_attempts`; only the teacher writes the library.

---

## Day‑to‑day workflow

1. **Load the syllabus** (Knowledge Base → paste or upload digital PDFs) so the tutor has
   verbatim passages to ground its explanations in.
2. **Ingest questions** (Question Bank) from screenshots/scans. If the paper's answer key is
   in its last pages, set **"Answer-key pages at end"** so each answer is taken word for word
   from that key. Review the OCR, answer key and approved keywords, then save.
3. **Edit** any answer later with the ✏️ button on a question card.
4. Students **Ask** ("how do I do this?") and **Mark** ("is my answer right?").

---

## Notes & limits

- OCR/answer quality depends on image clarity and on how much syllabus material you've
  loaded. The tutor is deliberately **grounded** — if it lacks the material it says so
  rather than inventing an answer.
- PDF ingestion is capped at 40 pages per file and processed in small page‑chunks.
- The model is configured in `GEMINI_MODEL`.
</content>
</invoke>
