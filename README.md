# VitalCheck

**A privacy-first health-habit app for people managing, or trying to prevent, type 2 diabetes and high blood pressure in India.**
Product case study, specs and a working interactive prototype.

> VitalCheck is for general awareness and wellness tracking only. It is not a medical device and does not diagnose, treat, cure or prevent any medical condition. Always consult a doctor. In an emergency call 112.

**Live prototype:** https://rohit384185-sudo.github.io/vitalcheck/prototype/ (open on a phone; it keeps your entries in your own browser only)

## Status, stated plainly

| Piece | Status |
|---|---|
| Market and competitor research, regulatory review | Done (Oct 2026), sources in [research-and-decisions.md](docs/research-and-decisions.md) |
| Product spec (v2) and engineering build prompt (v3) | Done: [product-spec-v2.md](docs/product-spec-v2.md), [claude-code-build-prompt-v3.md](docs/claude-code-build-prompt-v3.md) |
| Interactive prototype (this repo, `/prototype`) | Working: daily log, lab-report tips, day planner with optional voice reminders, Simple and Realistic looks |
| v1 web app (React, Express, SQLite) | Built earlier with Claude Code; source is not in this repo |
| v3 native Android app | **Not built yet.** The prompt to build it is ready |
| Clinician review of reference ranges, legal review of claims | **Pending**, listed as a launch blocker |

## The problem

India has roughly 101 million people with diabetes, 136 million with prediabetes and 315 million with hypertension (ICMR-INDIAB, Lancet Diabetes & Endocrinology, 2023). Most health apps lose nearly everyone quickly: one industry summary puts day-30 retention for fitness apps at about 3 to 4%, with manual data entry and slow onboarding as the main reasons.

## What I found

Five apps checked on their Play Store pages (one also on the App Store):

| App | Installs / rating | Centres on |
|---|---|---|
| BeatO | 1M+, 4.6 | Own glucometer, doctor consults, chatbot |
| Omron Connect | 1M+, 4.5 | Omron BP monitor sync; reviews mention logouts and lost connections |
| mySugr | 500K+, 4.6 | Glucose logbook and meter sync; reviewers ask for dark mode |
| sugar.fit | 100K+, 4.5 | Paid CGM and coaching programme |
| Fitterfly | 100K+, 4.2 | CGM, AI coach, nutrition; iOS app is 183 MB |

The gap is not basic tracking. It is a **free, no-hardware, no-login, on-device-private** app that also plans the day with reliable voice reminders in Indian languages, prepares the doctor visit and supports caregivers. Not checked: HealthifyMe, Dario, Welldoc, Twin Health.

## Product decisions and why

| Decision | Reason |
|---|---|
| **Awareness-only, with a configurable compliance level** (`full`, `standard`, `trackingOnly`) | India's CDSCO software guidance (21 Jul 2026) says a "wellness" label cannot cancel monitoring or prediction claims. A disclaimer alone is not protection, so interpretation features (range labels, risk score) can be dialled down without a rewrite. |
| Offline-first, no accounts, no analytics SDKs, encrypted local storage | Trust, and simpler compliance with India's DPDP Rules (notice and consent duties start 13 Nov 2026). Trade-off: no funnel analytics. |
| Day planner with reminders that always fire; voice toggle silences only the voice | Retention is the core problem. A reminder must never depend on a sound setting. |
| Notification budget (3 non-plan nudges a day), adaptive timing, back-off after ignored nudges | Avoid the nagging that makes people mute apps. Plan reminders are never throttled. |
| Habit tips with safety gates (no medicines, doses, supplements or calorie targets; hidden for kidney disease, pregnancy, urgent readings) | Useful general guidance (WHO, ADA, ICMR-NIN sourced) without crossing into advice. A test fails the build if banned terms appear. |
| No camera-based BP or glucose measurement | Apple App Review 1.4.1 forbids it and Google Play requires regulatory clearance. |
| Flutter with native Android services, not a web-view wrapper | Realistic gauges at 60 fps on cheap phones, Bluetooth, Health Connect, exact alarms with voice. |
| App size target of 60 to 70 MB from real content, never padding | Large downloads hurt installs on low-end phones; bundled OCR models, fonts, animations and a food database get there honestly. |

## Try the prototype

Switch between the **Simple** and **Realistic** looks with the button in the header. Add an HbA1c of 6.0 under Labs to see the range label and general habit cards, then tap "Add to my plan". Under Plan, use "Test reminder in 10 seconds" with voice on and off. Sample data is marked and can be cleared.

Limits of the prototype: reminders fire only while the page is open (reliable closed-app alarms need the Android app); no report scanning, Hindi, PDF export or app lock.

## Repository map

- `prototype/index.html`: self-contained interactive prototype (no build step, no network calls)
- `docs/research-and-decisions.md`: competitors, regulation, store rules, open decisions, sources
- `docs/product-spec-v2.md`: scope, risk-engine rules, planner and lab-guidance design
- `docs/claude-code-build-prompt-v3.md`: the engineering brief for the native app

## How it was made

I set the scope, constraints and priorities (awareness-only, privacy, realistic UI, planner, lab-based tips). Claude did the research synthesis, specs, prototype code and tests; Claude Code built the v1 app. The prototype's rules engine was checked with boundary-value tests and a 29-step automated walkthrough. I am a product manager, not a software engineer, and this repo is meant to show product thinking and AI-assisted delivery.

## Risks and open questions

1. Reference ranges and every piece of user-facing wording need clinician review before public release.
2. Intended-use classification under India's medical device rules needs a written legal opinion.
3. Bluetooth meter support is untested against real devices.
4. Business model is undecided (free core with optional paid family profiles, backup or reports is the leading idea).

## Metrics I would track

Day-30 retention; share of new users who log a first reading in the first session; readings logged per active week; plan items completed as a share of planned; reminder opt-out rate. Because the app is privacy-first, these would need an opt-in, anonymous counter or Play Console aggregates.

## Related work

- **[HydroMate](https://github.com/rohit384185-sudo/HydroMate1):** a voice-enabled hydration and medicine reminder app (React Native with Expo), with a product demo video in its README.
- **AI agents portfolio:** seven small agent projects (RAG over documents, PM metrics reviewer, ingredient label reader, insurance workflows and more), built on Groq's API with Llama models.
