# Chrome Form Filler — Project Plan

## Vision (as stated)

- A local LLM (via **Ollama**) processes a mixture of inputs — resume(s), notes,
  past form fills — into a single **profile**: a structured, editable store of
  "everything about me" (personal info, work history, education, skills,
  common application answers, etc.).
- A Chrome extension adds a **Fill** button. Clicking it reads the current
  page's form, matches each field to something in the profile, and fills the
  form immediately.
- The hard part is **not** extraction — it's that every site names/labels its
  fields differently, so a fixed field-name mapping won't work. Matching has
  to be done per-page, at fill time.
- v1 target forms: **job applications** first (Greenhouse, Lever, Workday,
  LinkedIn Easy Apply style), generalizing to arbitrary forms later.
- Field matching strategy for v1: send the page's field signals + the
  profile schema to the local LLM on every fill, and let it map them live.
  Optimize later (caching, heuristics) only if needed.

## Prior art (checked before designing this)

Several existing open-source projects cover parts of this space — worth
skimming for ideas/pitfalls, not for reuse (all are separate repos, no
license/dependency implications for us by referencing them):

- [gabrielborgesdm/ai-auto-filler-extension](https://github.com/gabrielborgesdm/ai-auto-filler-extension) —
  closest architectural match: Manifest V3 extension + Ollama, sends the
  selected input's DOM context to a local model and predicts a value.
- [sainikhil1605/ApplyEase](https://github.com/sainikhil1605/ApplyEase) —
  React extension + FastAPI backend + local LLM (Ollama/LM Studio), resume↔JD
  matching plus autofill. Closest to our "backend + extension" split.
- [chandrasuda/ai-autofill](https://github.com/chandrasuda/ai-autofill) —
  local LLM autofill for grant applications; similar "upload docs → answer
  form questions" flow.
- [EasyApp-RPI/EasyApp](https://github.com/EasyApp-RPI/EasyApp),
  [bdcorps/easy-job-application-filler-extension](https://github.com/bdcorps/easy-job-application-filler-extension),
  [BrandonS8/Resume-Filler](https://github.com/BrandonS8/Resume-Filler) —
  simpler, mostly rule/snippet-based fillers (no LLM field matching); useful
  for seeing what raw field-detection heuristics look like in practice.

Takeaways for our design:
- Everyone converges on the same two-sided problem: (1) build a structured
  profile, (2) map arbitrary DOM fields → profile keys per page. Nobody has
  a magic trick for (2) beyond "ask the LLM with good context" or "cache
  per-domain after a human confirms once" — which matches your instinct.
- The backend-server pattern (extension → localhost API → Ollama) is the
  most common and simplest for Manifest V3 (avoids native-messaging setup).

## Architecture decision

```
┌─────────────────────┐      fetch (localhost)      ┌──────────────────────┐      HTTP      ┌────────┐
│ Chrome Extension     │ ───────────────────────────▶│ Local backend server │ ─────────────▶ │ Ollama │
│ (content script +    │◀─────────────────────────── │ (FastAPI, Python)    │◀──────────────  │        │
│  popup + bg worker)  │      JSON (mapping/fill)     │  owns the profile DB │                └────────┘
└─────────────────────┘                              └──────────────────────┘
```

- **Backend**: Python + FastAPI (fits this repo's existing stack/skills from
  the course). Owns the profile store (SQLite) and all Ollama calls. Runs on
  `127.0.0.1:8765` (fixed local port).
- **Extension**: Manifest V3. `content script` scans the page's form and
  extracts field signals (label, name, id, placeholder, aria-*, nearby text,
  section heading). `popup` has the Fill button + profile quick-view.
  `background` service worker relays messages and talks to the backend via
  `fetch`. Needs `host_permissions` for `http://127.0.0.1:8765/*` and CORS
  enabled on the backend for the extension's origin.
- **Why not native messaging**: more setup (a registered native host
  manifest, per-OS install step) for no real benefit over a localhost HTTP
  server, since we're already running a local server for the DB.
- **Why not push all logic into the extension**: profile storage, PDF/DOCX
  parsing, and prompting Ollama are all much easier in Python; the extension
  stays a thin client (DOM I/O + UI).

## Profile schema (draft, v1)

A single JSON document per user, versioned, editable field-by-field:

```jsonc
{
  "personal": { "first_name": "", "last_name": "", "email": "", "phone": "",
                "address": {...}, "linkedin": "", "github": "", "website": "" },
  "work_history": [ { "company": "", "title": "", "start": "", "end": "",
                       "location": "", "description": "" } ],
  "education": [ { "school": "", "degree": "", "field": "", "start": "", "end": "" } ],
  "skills": ["..."],
  "documents": { "resume_path": "", "resume_text": "" },
  "qa": [ { "question": "Why do you want to work here?", "answer": "...",
             "tags": ["motivation"] } ],   // reusable canned answers
  "custom_fields": { "any_key_user_added": "value" }
}
```

`qa` + `custom_fields` are the escape hatch for the "I don't know the field
names in advance" problem — the LLM can match a weird site-specific question
into an existing `qa` entry by meaning, not by key name, and the user can add
new ones directly.

## Staged plan, with pass/fail criteria per stage

Each stage should be independently demoable and testable before moving on.

### Stage 0 — Architecture spike (this document)
**Done when:** architecture, schema, and stack are written down and you've
signed off on them. *(current stage)*

### Stage 1 — Local backend skeleton + Ollama connectivity
Build the FastAPI server with a `/health` endpoint and one endpoint that
sends a prompt to Ollama and returns the response.
**Pass:** `curl localhost:8765/health` → `200 ok`; a test script sends a
sample resume paragraph + "extract name and email as JSON" and gets back
valid JSON from the local model.

### Stage 2 — Profile extraction & storage
Ingest a resume (PDF/DOCX/txt) and free-form notes; LLM extracts into the
schema above; store in SQLite; expose CRUD endpoints (`GET/PUT /profile`).
**Pass:** feeding a real sample resume produces a schema-valid profile you
manually verify is *substantially correct* (name/email/work history/education
match the source); editing a field via the API persists across a server
restart.

### Stage 3 — Extension skeleton
Manifest V3 extension: popup with a "Fill" button, background worker,
content script that only logs "I'm injected" for now.
**Pass:** loads unpacked in Chrome, popup opens, clicking Fill logs a message
end-to-end from popup → background → content script → console, on an
arbitrary page.

### Stage 4 — Field detection (content script)
Scan the DOM for fillable fields (input/select/textarea/radio/checkbox) and
extract signal text (label, name, id, placeholder, aria-label, nearby text).
**Pass:** on 4–5 real test pages (a Greenhouse job app, a Lever job app, a
LinkedIn Easy Apply modal, one generic contact form) the script enumerates
≥90% of visibly fillable fields with usable signal text — checked by eye
against the rendered page.

### Stage 5 — LLM field-to-profile matching
Send the field-signal list + profile schema to the backend; prompt Ollama to
propose, per field, either a profile value or "no confident match", with a
confidence score.
**Pass:** on the same test pages from Stage 4, ≥80% of fields that *do* have
a matching profile value get a correct proposed value; fields without a
sensible match are correctly flagged rather than filled with something wrong
(measured by manual comparison against ground truth you define per page).

### Stage 6 — Actual DOM filling + review UI
Content script writes matched values into the DOM (dispatching proper
`input`/`change` events so React/Vue-controlled forms register the value),
visually highlights filled vs. unmatched fields, and lets you review/edit
before or after filling.
**Pass:** on the Stage 4/5 test pages, filled values are visibly present *and*
accepted by the site's own validation (e.g., you can advance to the next
form step); unmatched fields are clearly marked for manual entry; you can
correct a wrong value and have it stick.

### Stage 7 — End-to-end reliability & UX polish
Error handling (Ollama not running, backend unreachable), settings (model
choice, timeout, backend port), resume-file-upload fields, dropdown/select
value matching to the nearest valid option, radio/checkbox groups.
**Pass:** you can go through 5 real job application forms end-to-end (click
Fill, review, submit) without the extension crashing or requiring you to
fight it — normal human review of a few fields is fine, silent breakage is
not.

### Stage 8 (stretch) — Learn-and-cache per domain
First fill on a domain uses live LLM matching + your corrections; the
corrected mapping is cached per-domain so subsequent fills on that same site
skip the LLM call.
**Pass:** second fill on a previously-visited domain produces the same
correct result with no network call to the backend's Ollama endpoint
(verified via logs), and is noticeably faster.

### Stage 9 (stretch) — Generalize beyond job applications
Broaden schema/matching to shipping/checkout/signup forms.
**Pass:** works acceptably (per Stage 6 bar) on 3 non-job-application forms
you didn't design the schema around.

## Open questions for later stages (not blocking Stage 1)

- Which Ollama model to default to (needs to be decent at structured
  JSON output + reasonably fast for interactive use)?
- How to handle multi-step/paginated application forms (fill happens once
  per page, or does the extension need to track a session across steps)?
- Where does the profile literally live on disk, and do we want any
  encryption at rest given it's PII (name, address, work history)?
- File-upload fields (resume/cover letter) — auto-attach the stored resume
  file, or always leave those for manual upload?

## Immediate next step

Build **Stage 1** (backend skeleton + Ollama health check) and demo it
before moving to Stage 2. Say the word and I'll start.
