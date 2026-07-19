# TriageAI

**AI-assisted emergency triage & resource allocation for rural primary health centers.**

Built for Idea2Impact 2026 — Theme 3: Crisis Management, HealthTech & Emergency Response.

## The problem

Rural PHCs (Primary Health Centers) in India routinely triage patients on a
first-come-first-served basis, because there's no formal triage tool suited to
low-connectivity, low-resource settings. During mass-casualty events — road
accidents on highways, floods, disease outbreaks — this means:

- A patient with a minor complaint can be seen before someone in
  hemorrhagic shock, simply because they arrived first.
- Ambulances and beds get allocated by whoever calls loudest, not by clinical
  need or by which facility can actually take the patient.
- Frontline staff (ANMs, ASHA workers) aren't always trained to formally score
  severity — they're working off instinct under pressure.

## What this actually does

TriageAI is **not** a symptom-checker chatbot. It's three real, auditable
systems chained together:

### 1. Symptom red-flag extractor — `src/lib/symptomParser.js`
A lexicon-based NLP pass over the patient's free-text complaint (English +
common Hindi/Telugu transliterations). Matches clinical red-flag phrases
("can't breathe", "heavy bleeding", "face droop"), handles negation ("no chest
pain" won't fire), and outputs structured flags with clinical-system tags and
severity weights. No model call, no API key required — fully deterministic
and inspectable.

### 2. Clinical acuity scoring engine — `src/lib/triageEngine.js`
A 0–100 **Acuity Score**, built from vital-sign banding adapted from published
early-warning score logic (respiratory rate, pulse, systolic BP, capillary
refill, AVPU consciousness scale, age risk) plus the weighted output of the
symptom parser. Bucketed into START-style triage categories: **RED
(immediate) / YELLOW (urgent) / GREEN (minor) / BLACK (expectant)**. Every
point on the score is traceable — the UI shows the full breakdown per
patient, so a clinician can see exactly *why* a score was assigned and
override it if needed.

### 3. Resource allocation optimizer — `src/lib/resourceAllocator.js`
A priority-ordered greedy assignment algorithm. Given the ranked patient
queue and a network of facilities (each with typed capacity — ICU beds,
general beds, OPD slots — and real coordinates), it matches each patient to
the best available facility using haversine distance and live capacity
headroom, computes an ETA at realistic rural ambulance speed, and flags when
a RED patient needs ambulance dispatch. If no facility has capacity, the
patient is explicitly flagged for escalation rather than silently dropped.

**All three run entirely client-side. No patient data is transmitted
anywhere — the app works fully offline once loaded**, which matters for
low-connectivity PHCs.

## Tech stack

- React 19 + Vite
- Plain CSS (CSS custom properties, no framework) — see `src/App.css` /
  `src/index.css`
- IBM Plex Sans / IBM Plex Mono for a legible, instrument-panel feel suited
  to a clinical tool used under stress
- Zero backend, zero external API dependency for core functionality — a
  deliberate choice so this deploys as a static site to any free host and
  keeps working when the PHC's internet doesn't

## Setup

```bash
npm install
npm run dev      # local dev server
npm run build     # production build -> dist/
npm run preview   # preview the production build locally
```

Requires Node.js 18+.

## Deploying (for submission)

The `dist/` output from `npm run build` is a fully static site — deploy it
anywhere:

- **Netlify (drag-and-drop):** run `npm run build`, then drag the `dist`
  folder onto [app.netlify.com/drop](https://app.netlify.com/drop).
- **Vercel:** `npx vercel --prod` from the project root, or import the GitHub
  repo at [vercel.com/new](https://vercel.com/new).
- **GitHub Pages:** push `dist/` to a `gh-pages` branch, or use the
  `gh-pages` npm package.

No environment variables or secrets are required.

## Project structure

```
src/
  lib/
    symptomParser.js      # red-flag NLP extraction
    triageEngine.js        # clinical acuity scoring
    resourceAllocator.js   # facility assignment optimizer
  data/
    facilities.js          # sample rural facility network (replace with real data)
  components/               # UI
  App.jsx                    # wires intake -> scoring -> allocation together
```

## Using real data

`src/data/facilities.js` documents the schema for a facility record. Swap the
sample Telangana-district network for a real PHC/CHC/District Hospital
dataset (available from state health department open-data portals) and the
distance/allocation logic works unchanged.

## Honest limitations (also worth saying out loud to judges)

- The scoring thresholds and weights are adapted from published triage
  literature but are **not clinically validated** — this is a hackathon
  prototype, not a certified medical device, and should never replace
  clinical judgment.
- The symptom lexicon currently covers common English + transliterated
  Hindi/Telugu phrasing; a production version would need broader multilingual
  coverage and ideally voice input for low-literacy users.
- The allocator is a real-time greedy matcher, not a global optimum solver —
  intentional, since a real-time system needs sub-second decisions per new
  patient, but worth naming as a design tradeoff.
