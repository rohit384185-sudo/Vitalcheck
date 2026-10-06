# VitalCheck product spec (v2 working document)

> Working document written with Claude for the product owner, so "you" means the owner. It is **superseded by the v3 build prompt and the research notes where they differ** (for example v3 uses Flutter instead of a web-view wrapper). Kept in full to show how the scope developed. Reference ranges are general awareness defaults and still need clinician review.


> Prepared 1 Oct 2026. Based on the v1 build summary (React 18 + TS, Recharts, date-fns, Express + SQLite; daily log of sleep / exercise / food / BP / glucose; 5 weekly charts; 7-day risk flags).
> VitalCheck is a lifestyle tracking and awareness tool. It is **not** a medical device and must never diagnose.

---

## PART 1 — Hard analysis of v1

### 1.1 What v1 gets right
- Correct core idea: low-friction daily logging plus plain-language flags plus "see a doctor" framing.
- Right domain focus (diabetes and hypertension drive a huge share of lifestyle-disease burden in India).
- Sensible chart set; data stays local; responsive layout.

### 1.2 Critical weaknesses (ranked by severity)

**A. Clinical logic is too thin, and in places misleading (highest risk)**
1. **Glucose has no context.** A "fasting" chart and a single ≥200 threshold ignore *when* the reading was taken. 140 mg/dL is normal after a meal and high when fasting. Without a context tag (fasting / before meal / 2h after meal / bedtime / random), flags are unreliable.
2. **One reading per day.** BP varies hour to hour and guidelines use averages of repeated readings (morning and evening, 2 readings each). One number per day gives noisy, wrongly alarming or wrongly reassuring trends.
3. **Missing emergency tiers.** No "hypertensive crisis" tier (≥180/120 → seek care now, especially with symptoms like chest pain, severe headache, vision change, weakness) and no **hypoglycaemia** tier (<70 mg/dL → treat now). Hypoglycaemia is the more dangerous miss for diabetics.
4. **Thresholds are hard-coded and unsourced.** BP stages differ between guidelines (ACC/AHA: 130/80 = Stage 1; ESC / ISH / Indian practice: 140/90). They need to live in one clearly documented config, with a "reviewed by" note.
5. **No personalisation.** Targets differ for a known diabetic, someone with CKD, elderly users, pregnancy. v1 treats everyone as one profile.
6. **Food quality is one dropdown.** "Good/Fair/Poor" is subjective and not actionable. It can't tell the user *what* to change.

**B. Product gaps (why it is not yet a "must-have")**
1. **No reminders.** A tracker people forget to open is dead in 2 weeks. This is the #1 retention failure.
2. **No habit loop.** No streaks, goals, or rewards. No "what should I do today?" answer.
3. **No medication tracking.** For diabetes / BP patients, adherence is the biggest lever and the first thing a doctor asks about.
4. **No doctor-ready output.** The real moment of value is the clinic visit. There is no PDF / shareable report.
5. **No insight layer.** Charts show *what* happened, never *why*. ("Your BP is ~8 mmHg higher on days after <6h sleep" is the kind of insight that makes people stay.)
6. **No weight / waist / HbA1c / lab records.** Missing the metrics that define long-term risk.
7. **No edit/history browsing** beyond a date picker; no way to correct a wrong entry cleanly (v1 only has Delete).
8. **No symptoms log.** Dizziness, thirst, blurred vision, frequent urination are useful context for flags.

**C. Technical / architecture issues for a phone app**
1. **Express + SQLite server is the wrong shape for a phone app.** A server means the app is useless offline and "local data" is only true if the server runs on the user's own machine. For a phone, data should live **on-device** (IndexedDB now, SQLite via Capacitor later).
2. **No authentication or app lock.** Health data with zero protection. Anyone holding the unlocked phone or browser sees everything.
3. **No encryption at rest.** SQLite file is plain.
4. **Not installable / not offline.** No PWA manifest, service worker, or notifications.
5. **Test data was committed to the repo** (the final commit included sample logs). Health data, even test data, must not live in git. Add DB files to `.gitignore` and use seed scripts instead.
6. **No automated tests** for the risk engine, which is the one component where bugs are safety-relevant.
7. **Date/timezone risk.** Bedtime-after-midnight sleep logs and "which day does this belong to" are classic bugs; needs a defined rule (a sleep session belongs to the *wake* date) and tests. User is in IST (UTC+5:30).
8. **Accessibility** not mentioned: font scaling, contrast, screen-reader labels. Many users of this app will be 45+.

**D. Compliance and trust (Play Store reality)**
- Google Play requires a **Health apps declaration**, a privacy policy, and an accurate Data Safety form. Needs an in-app disclaimer ("not a medical device, not a diagnosis") and an emergency-numbers screen (India: 112 national emergency, 108 ambulance in many states).
- India's DPDP Act: explicit consent, clear purpose, easy deletion and export of data. Build "Export my data" and "Delete everything" in v2.

### 1.3 Strategic recommendation
**Pivot the architecture, not the idea:** make it an **offline-first PWA** (all data on the device, encrypted, installable, with local notifications) and wrap it with **Capacitor** for Android. Keep the Express backend only as an *optional* future sync layer. This matches your plan of web first, then Android, and it is the strongest privacy story.

---

## PART 2 — Feature roadmap (what makes it a must-have)

### Tier 0 — Foundation fixes (do first)
| # | Feature | Why |
|---|---|---|
| 0.1 | Offline-first storage (Dexie / IndexedDB), repository layer so storage can swap to SQLite later | Phone-ready, private |
| 0.2 | Multiple timestamped readings/day with **context tags** (glucose: fasting / pre-meal / 2h post-meal / bedtime / random; BP: morning / evening / arm / seated) | Makes flags clinically meaningful |
| 0.3 | **Risk engine v2**: 5-tier severity (Normal, Elevated, High, Very High, Urgent) + hypoglycaemia + crisis rules, config-driven and unit-tested | Safety |
| 0.4 | Edit / delete / undo any entry; full history list | Data trust |
| 0.5 | App lock (PIN + biometric on Android) and optional encryption (Web Crypto AES-GCM) | Privacy |
| 0.6 | Onboarding profile: age, sex, diabetes yes/no/prediabetes, hypertension yes/no, on medication, units, personal target ranges (editable, "ask your doctor" hint) | Personalisation |
| 0.7 | Disclaimer, emergency screen, consent screen | Compliance |

### Tier 1 — Habit and value (makes it sticky)
1. **Smart reminders**: morning BP, fasting glucose, medication times, evening log nudge, weekly review. Local notifications; quiet hours.
2. **Today screen with "Daily Score" ring** (0–100 built from sleep, activity, food, readings logged, meds taken) and 3 concrete "next best actions".
3. **Medication tracker**: name, dose, schedule, one-tap "Taken / Skipped / Snooze", adherence %, refill reminder.
4. **Streaks and gentle gamification**: logging streak, weekly goal rings, badges (7-day streak, 30 days in range). No shame mechanics.
5. **Doctor Report (PDF)**: 30/90-day summary: averages, min/max, % in range, trend charts, meds adherence, symptoms, notable flags. One-tap share to WhatsApp / email.
6. **Insights engine** (rule-based first, no AI dependency): sleep↔BP, exercise↔glucose, late dinner↔fasting glucose, weekday vs weekend, salt/sweets days. Show only when ≥14 data points; always say "pattern, not proof".
7. **Weekly summary card** each Sunday: what improved, what to watch, one goal for next week.
8. **Daily Planner with polite notifications, voice alarms and medicine timer** (full spec in Part 4B).
9. **Lab reports with general lifestyle awareness cards, IDRS risk check, and the extra features ranked in Part 4C.**

### Tier 2 — Differentiators (the "wow")
1. **Indian food quick-log**: preset library (roti, dal, rice, paneer, idli, dosa, poha, samosa, chai with sugar, etc.) with portion chips (small / medium / large) → estimated carbs, sugar, sodium *bands* (Low / Medium / High). Clearly labelled estimates.
2. **Photo meal logging** (opt-in): photo → Claude vision API estimate → user confirms/edits. Requires a key; must be opt-in with explicit warning that the image leaves the device. (Phase 3, never default.)
3. **Voice logging** ("BP 132 over 86 morning, sugar 98 fasting") via Web Speech API with a confirm screen.
4. **Weight, waist, BMI** and **lab records** (HbA1c, lipid profile, creatinine) with trend and "next test due" reminder (HbA1c every 3 months for diabetics).
5. **Symptoms log** with quick chips (dizzy, thirsty, blurred vision, headache, frequent urination, fatigue). Symptoms combined with a bad reading escalate the flag wording.
6. **Hydration and salt/sugar awareness counters**.
7. **Family / caregiver mode**: read-only share link or exported summary for a parent's or spouse's tracking (many users are adult children tracking for parents). Local-first means share = exported file or QR, not cloud.
8. **Health Connect integration (Android)**: auto-import steps, sleep, heart rate, weight. Removes manual entry. (Capacitor phase.)
9. **Home-screen widgets** and **quick-log notification actions** (log glucose without opening the app).
10. **Hindi + English** (i18n from day one), large-text mode, voice read-aloud of flags.
11. **Emergency card** (lock-screen-friendly): conditions, medications, allergies, emergency contact. High value, low effort.
12. **Export / import / delete-all** (JSON + CSV) for DPDP and trust.

### What to deliberately NOT build
- No diagnosis, no medication dose advice, no "you have diabetes" language.
- No ads, no third-party analytics SDKs, no selling data. This is a trust product.
- No cloud accounts in v2.

---

## PART 3 — Dual-UI design: "Simple" and "Realistic" with a toggle

**One codebase, one component tree, two themes.** The toggle lives in Settings and as a small sun/orb icon in the header. The choice persists and applies instantly with a short cross-fade (no reload). All logic, data, and layout structure are identical; only design tokens, surface treatments, and a few "hero" components change.

### 3.1 Shared foundations
- Mobile-first, 360–430px primary width; desktop becomes a centred 480px phone frame plus a side rail.
- **Bottom navigation, 5 tabs:** Today · Log (+, centre FAB) · Trends · Insights · Me.
- Type scale respects system font size. Min tap target 48px. WCAG AA contrast in both themes. Never rely on colour alone for risk: always icon + word.
- Severity colours (semantic tokens, themed): Normal (green), Elevated (amber), High (orange), Very High (red), Urgent (deep red with pulsing border).
- Light / dark mode in both themes (so 4 combinations); follows system by default.
- Reduced-motion setting disables all animation in both themes.

### 3.2 Theme A — "Simple"
Philosophy: clarity for every age. Fast, calm, low-battery, works on cheap phones.
- Flat surfaces, 1px borders, almost no shadows, 12px radius.
- One typeface (Inter or system sans), big numerals, generous whitespace.
- Single accent colour (teal `#0F766E`), neutral greys.
- Charts: thin lines, simple dots, target band as a pale rectangle.
- Hero Today component: a **flat progress ring** and 3 large action cards.
- Log screen: big segmented chips and numeric steppers, no decorative imagery.
- Icons: outline, 2px stroke (Lucide).
- Animations: 150ms fades only.

### 3.3 Theme B — "Realistic" (ultra-realistic, premium)
Philosophy: feels like a premium medical instrument. Depth, material, light.
- **Glassmorphism and soft neumorphism**: frosted translucent cards (backdrop-blur), layered soft shadows, subtle inner highlights, 20–28px radii, light-direction consistency (light from top-left).
- Background: deep gradient mesh (navy → teal → violet in dark; warm porcelain in light) with slow-moving ambient blobs.
- Typeface pair: a refined display face for numerals (e.g., "Sora" or "Manrope") and Inter for text; tabular numerals for readings.
- **Hero components rebuilt as realistic instruments:**
  - *BP gauge*: analog dial with metallic bezel, glass reflection, animated needle that eases to the latest systolic, tick marks colour-banded by risk zone.
  - *Glucose card*: glucometer-style LCD panel with soft glow digits and a droplet animation on save.
  - *Daily Score*: layered 3D ring with gradient stroke and glowing end cap; rings for sleep / activity / food / meds (Apple-style concentric).
  - *Sleep*: moon-and-stars arc with gradient night sky showing bedtime→wake span.
  - *Heart pulse* line animating subtly on the BP card.
- Charts: smooth spline curves, gradient area fill, glowing last point, animated draw-in, hover/touch tooltip with glass panel, risk bands as soft coloured glows.
- Micro-interactions: haptic feedback on save (Capacitor Haptics), springy button press, confetti-lite on streak milestones, number count-up.
- Iconography: duotone, slightly 3D filled icons.
- Performance guard: a "Reduce effects" switch, and **auto-fallback to Simple-like surfaces** (no blur, no ambient animation) on low-end devices (`navigator.deviceMemory <= 2` or `prefers-reduced-motion`).

### 3.4 Screen inventory (both themes)
1. **Splash / Consent / Disclaimer**
2. **Onboarding** (4 steps): profile → conditions → targets → reminders → choose look (Simple / Realistic preview cards with the live toggle)
3. **Today**: greeting, Daily Score, latest BP and glucose cards, today's meds checklist, 3 next-best-actions, risk banner (if any)
4. **Quick Log sheet** (bottom sheet from + FAB): tabs for BP · Glucose · Food · Sleep · Activity · Weight · Symptoms · Meds
5. **Trends**: range switch (7d / 30d / 90d), metric selector, chart, stats strip (avg, min, max, % in range), history list below
6. **Insights**: risk flags with explanations, correlation cards, weekly summary
7. **Medications**: schedule, adherence calendar, refill tracker
8. **Reports**: generate Doctor PDF, export CSV/JSON
9. **Me / Settings**: profile, targets, reminders, theme toggle, language, units, app lock, data export / delete, emergency card, about and disclaimer
10. **Emergency screen**: reachable from any risk banner and from Me

### 3.5 Wireframe (Today, shared structure)
```
┌──────────────────────────────┐
│ Good morning, Asha      ◐ ⚙ │  ← theme toggle
│ ┌──────────────────────────┐ │
│ │  Daily Score   ◯ 78      │ │  ring (flat | 3D concentric)
│ └──────────────────────────┘ │
│ ┌────────────┐ ┌───────────┐ │
│ │ BP 128/82  │ │ Sugar 96  │ │  dial | LCD
│ │ ● Normal   │ │ ● Fasting │ │
│ └────────────┘ └───────────┘ │
│ Meds today        2 of 3 ✓   │
│ Next best actions             │
│  • Take a 10-min walk         │
│  • Log evening BP (7:30 pm)   │
│ ───────────────────────────── │
│ Today  Trends [+] Insights Me │
└──────────────────────────────┘
```

---

## PART 4 — Risk engine v2 specification (config-driven, unit-tested)

Keep all numbers in `src/risk/thresholds.ts` with a header comment: *"General awareness thresholds. Not diagnostic. Must be reviewed by a qualified clinician before public release. Guidelines differ; user's own doctor targets override."* Allow the user to enter personal targets that override defaults.

**Blood pressure (mmHg), evaluated on the average of the last 7 days of readings, plus per-reading urgent check**
| Tier | Rule (default) |
|---|---|
| Normal | <120 and <80 |
| Elevated | 120–129 and <80 |
| High (Stage 1 range) | 130–139 or 80–89 |
| Very High (Stage 2 range) | ≥140 or ≥90 |
| Urgent | any single reading ≥180 and/or ≥120 → "This reading is outside the usual range. Please seek medical help now or call 112. For awareness only, not a diagnosis." |

**Glucose (mg/dL), by context**
| Context | Normal | Watch | High | Urgent |
|---|---|---|---|---|
| Fasting | 70–99 | 100–125 | ≥126 | <70 or ≥300 |
| 2h post-meal | <140 | 140–199 | ≥200 | <70 or ≥300 |
| Random | <140 | 140–199 | ≥200 | <70 or ≥300 |
| Bedtime | 100–140 | <100 or 141–180 | >180 | <70 |
- For users who declared diabetes: use the *doctor-set targets* if entered; default fasting target 80–130 and post-meal <180.
- **<70 mg/dL** always shows: "This reading is lower than the usual range. Please follow the plan your doctor has given you, or seek medical help now or call 112 if you feel unwell. For awareness only, not a diagnosis." No self-treatment instructions anywhere in the app.
- Never infer diabetes from readings; wording is "readings in the range your doctor may want to review".

**Sleep:** <6h on ≥3 of last 7 nights → Elevated; >9.5h repeatedly → Watch. Sleep session belongs to the wake date.
**Activity:** <150 min/week → Low; ≥150 → Good (WHO baseline).
**Combination rules (escalate wording, never diagnose):** high BP + symptoms; high fasting glucose on 3+ days + thirst/urination; missed meds ≥3 days + rising trend.
**Staleness:** if no reading for 7+ days, flag "not enough recent data" instead of showing a stale score.

Every flag object: `{ id, tier, title, plainLanguage, why (data used), nextStep, seeDoctor: boolean, emergency: boolean }`. `nextStep` is only ever "consult your doctor" (or "seek medical help now / call 112" for Urgent), never a treatment or lifestyle prescription.
Language rules: awareness-only. No "you have", no diagnoses, no drug names or doses, no treatment steps. Always include "for awareness only, not a diagnosis" and end with "please consult your doctor". Reading level ~Class 8, Hindi translations for each string.

---

## PART 4B — Daily Planner, polite notifications, voice alarms and medicine timer

### What the user gets
- **Plan my day:** the user plans meals, walks, water, medicine times, BP/glucose checks, a bedtime wind-down, or any custom item. Each item has a time, optional duration and repeat days. Start from templates (Morning routine, 3 meals, evening walk), edit, save as a routine, copy yesterday's plan, weekly view.
- **A notification always fires** at the planned time, in a polite tone.
- **Voice is optional:** one global Voice toggle plus a per-item override. With voice OFF the notification still appears with normal sound and vibration. Only the speaking is switched off, never the reminder.
- **Alarm style (optional per item):** rings on the alarm volume so it is heard even when the phone is on silent, with a full-screen reminder card.
- **Medicine timer:** "Next medicine in 1h 20m" countdown on Today. Done / Remind me in 10 min / Skip buttons right on the notification. User-set interval timers, and refill reminders.
- **Planner feeds the rest of the app:** completed items count towards Daily Score and the weekly summary ("5 of 6 planned items done"). Positive framing only, never shaming.

### Tone rules (polite, calm, never nagging)
- Examples (English): "Gentle reminder: it is time for your planned medicine. Please follow your doctor's instructions." / "Your planned walk is starting. Whenever you're ready." / "Your planned lunch time is here. Enjoy your meal." / "No worries, you can mark it done later."
- Hindi versions of every message. Optional use of the user's first name. Max 2 repeats, then it stops quietly and marks the item "missed" without any guilt message.
- Reminders only repeat the user's own plan. They never recommend doses, change medicine timing, or suggest what to eat (awareness-only rule).

### Privacy defaults (important)
- Lock-screen privacy ON by default: the notification says "Planned reminder", not the medicine name. A separate "Speak medicine names aloud" toggle is OFF by default.
- Everything runs on the device. No server, no cloud text-to-speech. Voice uses the phone's own speech engine.

### Honest platform limits (why the Android app matters)
- **A browser/PWA cannot reliably fire a notification at an exact time when the app is closed.** There is no dependable web API for scheduled notifications. So on the web, reminders work while the app is open, and the Android app is where reliable alarms and voice live.
- Android needs: notification permission (Android 13+), the exact-alarm permission (user must allow it on newer Android), and battery or auto-start settings on many Indian phone brands (Xiaomi, Oppo, Vivo, Realme, OnePlus, Samsung). The app will include a "Make reminders reliable" guide screen for this.
- Voice works offline using the device's speech engine. Hindi may need the user to download the Hindi voice once; the app will prompt for it.
- Plan: build the planner and a scheduler abstraction in the web phases, then ship the reliable native alarms in the Android phase.

---

## PART 4C — Lab reports, general lifestyle awareness, and more features (research-backed)

### 4C.1 The idea
The user adds a lab report (HbA1c, fasting sugar, lipids and so on). The app shows where the value sits against published reference ranges, then gives a short set of **general lifestyle awareness cards** (move, drinks, plate, salt, sleep, re-test timing) and always ends with "this is for awareness only, please consult your doctor." One tap on any card adds the habit to the Planner (Part 4B).

Example, HbA1c 6.0%:
> "Your HbA1c of 6.0% is in the range guidelines call the **prediabetes range** (5.7 to 6.4%). This is for awareness only and is not a diagnosis. Please consult your doctor."
> Cards: **Keep moving** (many guidelines suggest about 150 minutes of moderate activity a week, for example a brisk 30 to 45 minute walk on most days; start smaller if you are new to exercise) · **Skip sugary drinks** (soft drinks, packaged juice, sweetened tea or coffee, sharbat) · **Half your plate vegetables** · **Go easy on salt** · **Plan your next check** (ask your doctor how often; ADA suggests at least yearly for this range).

**Change from the earlier awareness-only rule (needs your OK and a legal check):** for **lab values** (HbA1c, fasting glucose, post-meal glucose) the app may state the published reference-range label, such as "prediabetes range". It must never say "you have prediabetes", and home meter readings keep the older wording ("readings your doctor may want to review").

### 4C.2 Reference ranges to use
| Test | Normal range | "Prediabetes range" | "Diabetes range" |
|---|---|---|---|
| HbA1c | <5.7% | 5.7 to 6.4% (ADA 2026 confirmed) | ≥6.5% |
| Fasting glucose | <100 mg/dL | 100 to 125 | ≥126 |
| 2-hour post-meal / OGTT | <140 mg/dL | 140 to 199 | ≥200 |
- For HbA1c 5.7 to 6.4 the ADA 2026 chapter was checked directly. The other cut-offs are the standard ADA criteria from general knowledge and should be re-verified by a clinician reviewer.
- **Best design choice: use the reference range printed on the user's own lab report** (every lab prints one, with H/L flags) for all other tests, and fall back to built-in bands only for HbA1c, glucose and BP. Labs differ in methods and ranges, so this is more accurate and safer for you.
- Users who already declared diabetes: do not label ranges; show trend, "share with your doctor", and the same general habit cards.
- Asian Indian reference ranges for body measures: **waist ≥90 cm (men) / ≥80 cm (women)**, **BMI ≥23 overweight, ≥25 obesity** (2009 consensus; a 2024 revision adds stages and functional criteria). Show as "reference ranges for Asian Indians", not as a verdict.

### 4C.3 General guidance library (what the cards say, and where it comes from)
| Card | General message | Source |
|---|---|---|
| Keep moving | About 150 minutes a week of moderate activity (WHO: 150 to 300). Brisk walking counts. Muscle-strengthening on 2 or more days. Move a little instead of sitting for long stretches. | WHO 2020; ADA 2026 (≥150 min/week) |
| After-meal steps | Short, light walks of a few minutes after meals are linked to smoother after-meal sugar curves. Evidence is from small studies, mostly in people without diabetes. | Meta-analysis of 7 studies (Sports Medicine), via news summary |
| Sugary drinks | Soft drinks, packaged juices, sweetened tea or coffee, sharbat add sugar quickly. Water or unsweetened drinks are a gentler choice. Added sugar guidance: under 5% of energy, about 25 to 30 g a day. | ICMR-NIN 2024 |
| Plate | Vegetables, fruit and green leafy vegetables should be roughly half the plate. Prefer whole grains and pulses. Fewer packaged, fried and ultra-processed snacks. Read food labels. | ICMR-NIN 2024 |
| Salt | Under 5 g salt (about one teaspoon, under 2 g sodium) a day. Watch pickles, papad, namkeen, packaged snacks. | WHO salt fact sheet |
| Sleep | Regular sleep of about 7 to 9 hours. | General public-health guidance (not re-verified here) |
| Tobacco and alcohol | Avoid tobacco. Keep alcohol low. | General public-health guidance |
| Re-check | Ask your doctor how often to re-test. ADA suggests at least yearly monitoring for people in the prediabetes range. | ADA 2026 rec. 3.1 |
| Weight and waist (opt-in only) | Some studies of lifestyle programmes used 5 to 7% weight loss; ask your doctor what is right for you. | ADA 2026 rec. 3.3 |

Tone: "many guidelines suggest", "is linked to", "may help". Never "will cure", "reverse", "you must". Max 4 cards at once. Every card shows its source label and a "Why am I seeing this?" link.

### 4C.4 Safety gates (when cards are hidden or softened)
- **Never** include: medicines, doses, supplements, herbal or home remedies, detox, fasting or crash diets, calorie targets, or "stop/start/change medicine". An automated test scans all guidance text for banned terms.
- **Doctor-only tests** (no lifestyle cards, only "outside the printed range, please discuss with your doctor"): creatinine / eGFR, urine albumin, TSH, haemoglobin, vitamin D, B12, ALT/AST, electrolytes. Kidney, thyroid and anaemia diets are individual and can do harm if generic.
- Active **Urgent** flag: show only the "seek medical help" message.
- Kidney disease, pregnancy or breastfeeding declared: hide diet, weight and exercise cards; show "please ask your doctor".
- Age 65+, heart condition, joint problems, recent surgery, or chest pain symptoms logged: exercise card adds "check with your doctor before starting; start slowly".
- User declares they take diabetes medicines: exercise card adds "ask your doctor about activity and low-sugar risk". Any low-sugar reading logged in the last 30 days: hide intensity suggestions.
- BMI under 18.5: no weight-loss content. Weight and waist cards are opt-in and never appear by default; no calorie counting anywhere (protects users prone to unhealthy dieting).
- Report older than 6 months: show "this report is old; a newer result may look different".
- 18+ only (age gate at onboarding).
- **Content governance:** each card is a data record `{id, text_en, text_hi, source, reviewedOn, reviewer, gates[]}` in one reviewable file. CI fails if a card has no source, no Hindi text, a review date older than 12 months, or banned words.

### 4C.5 Getting reports into the app
1. **Manual entry** (first): pick test, value, unit, date, lab name, printed reference range.
2. **Scan or PDF import** (later, behind a flag): on-device text recognition (Android ML Kit / browser OCR) and PDF text extraction. The user **always confirms** each extracted value on a review screen; never auto-save OCR output. Files stay encrypted on the device and are never uploaded.
3. **Unit handling:** HbA1c % and mmol/mol, glucose mg/dL and mmol/L, with tested conversions.
4. **ABHA / ABDM import is not for v2.** As of May 2026 ABHA has 880+ million accounts, but cross-facility record retrieval is still very low, labs have little incentive to integrate, and DPDP compliance for private health apps is still unclear (per a July 2026 policy review). Revisit once the app has users.
5. Trend charts per test and a user-set "next test due" reminder.

### 4C.6 More features worth adding (ranked)
| # | Feature | Why it matters | Effort |
|---|---|---|---|
| 1 | **Indian Diabetes Risk Score check** (age, waist, activity, family history; <30 low, 30 to 50 moderate, ≥60 high) | Validated in India, takes 1 minute, gives a reason to get tested. Awareness-only wording: "worth discussing testing with your doctor." Points: age 35 to 49 = 20, ≥50 = 30; waist 10 or 20; activity 0 to 30; family history 10 or 20. | Low |
| 2 | **Turn advice into plan** (every card has "Add to my plan") | Converts knowledge into action; ties Reports to Planner | Low |
| 3 | **Habit ladder** (10 min, then 20, 30, 45 over 6 weeks) with "after lunch I will walk 10 minutes" prompts | Small steps raise follow-through | Low |
| 4 | **Doctor-visit prep** (auto question list, visit notes, "next appointment" reminder, one-page report) | The clinic visit is the app's real moment of value | Medium |
| 5 | **Home BP coach** (guided 7-day morning and evening measuring routine, correct posture checklist, averages) | Home averages are more reliable than single readings; routine is commonly recommended, verify wording with a clinician | Medium |
| 6 | **Family / parent mode** (large text, simple screens, weekly summary shared by WhatsApp) | Adult children often manage parents' health in India | Medium |
| 7 | **Learn hub** (60-second explainers in Hindi and English with read-aloud: HbA1c, measuring BP, plate, salt) | Trust and understanding, low risk | Low |
| 8 | **Packaged-food label reader** (on-device OCR of sodium and added sugar per serving against the WHO and ICMR-NIN daily limits; user confirms values) | ICMR-NIN tells people to read labels; few apps make it easy | Medium to high |
| 9 | **Festival and fasting templates** (Diwali, Navratri, Ramadan, wedding season) as awareness plans with "if you take medicines, ask your doctor before fasting" | Real-life India use; gentle, no medical advice | Low |
| 10 | **Encrypted backup and restore** (file, or the user's own Google Drive app-data folder, opt-in) | Local-only data is lost with the phone; needed before wide release | Medium |
| 11 | **Privacy shade** (one tap hides all numbers on screen) | Public-place privacy, matches your privacy priority | Low |
| 12 | **Health Connect import** (steps, sleep, heart rate, weight) | Removes manual entry | Medium |
| 13 | **Smart reminder timing** (learns when you usually log and nudges at that time) | Retention | Medium |

### 4C.7 Sources checked while preparing this part
- WHO 2020 physical activity and sedentary behaviour guidelines: https://pmc.ncbi.nlm.nih.gov/articles/PMC7719906 (150 to 300 min moderate, 75 to 150 vigorous, muscle strengthening 2+ days, limit sedentary time)
- ADA Standards of Care 2026, prevention of diabetes: https://pmc.ncbi.nlm.nih.gov/articles/PMC12690170 (prediabetes A1C 5.7 to 6.4%, ≥150 min/week, 5 to 7% weight loss target, yearly monitoring)
- ICMR-NIN Dietary Guidelines for Indians 2024: https://www.nin.res.in/dietaryguidelines/pdfjs/locale/DGI24thJune2024fin.pdf (17 guidelines; added sugar under 5% of energy, 25 to 30 g/day; half the plate vegetables and fruit; avoid HFSS and ultra-processed foods)
- WHO salt reduction fact sheet: https://www.who.int/news-room/fact-sheets/detail/salt-reduction (under 5 g salt, under 2,000 mg sodium)
- Indian Diabetes Risk Score scoring: https://www.ijcmph.com/index.php/ijcmph/article/download/2396/1805/9392 and validation https://pmc.ncbi.nlm.nih.gov/articles/PMC5729677
- Asian Indian BMI and waist cut-offs: https://www.cmcendovellore.org/pub/2025/revised-definition-of-obesity-in-asian-indians-living-in-india.pdf and https://cadiresearch.org/?p=636
- Post-meal walking meta-analysis (secondary news summary only; read the original paper before quoting): https://www.healthline.com/health-news/how-a-2-minute-walk-after-meals-can-help-lower-type-2-diabetes-risk
- Android exact alarms: https://developer.android.com/about/versions/14/changes/schedule-exact-alarms
- ABDM status review (July 2026): https://hmpi.org/2026/07/09/from-infrastructure-to-impact-operationalizing-indias-ayushman-bharat-digital-mission/?pdf=6336
- **Not verified in this session:** glucose and OGTT cut-offs beyond ADA's 5.7 to 6.4 HbA1c range, the 7 to 9 hour sleep figure, the home-BP routine, and India's medical-device software regulation (CDSCO). Have a clinician and a lawyer check these before release.

---

## PART 5 — Data model (v2)

```
Profile      { id, name, birthYear, sex, hasDiabetes: none|pre|t1|t2, hasHypertension, units, targets{...}, language, theme }
BPReading    { id, ts, systolic, diastolic, pulse?, context: morning|evening|other, arm?, notes? }
GlucoseReading { id, ts, value_mgdl, context: fasting|preMeal|postMeal2h|bedtime|random }
Meal         { id, ts, type, items[{foodId, portion}], quality?, carbBand, sugarBand, sodiumBand, photoRef? }
SleepSession { id, wakeDate, bedtime, wakeTime, quality? }
Activity     { id, ts, type, minutes, intensity? }
Weight       { id, ts, kg, waistCm? }
Medication   { id, name, doseLabel, schedule[], active, refillDate? }
MedLog       { id, medId, scheduledTs, status: taken|skipped|missed, loggedTs }
LabResult    { id, date, testKey: HbA1c|FBS|PPBS|LDL|HDL|TG|creatinine|eGFR|uric|TSH|Hb|vitD|B12|ALT|AST|..., value, unit, labName?, printedRefLow?, printedRefHigh?, printedFlag?, source: manual|ocr|pdf, userConfirmed: boolean, fileRef? }
RiskCheck    { id, date, kind: IDRS, inputs{ageBand, waistCm, activity, familyHistory}, score, band }
GuidanceCard { id, text_en, text_hi, source, reviewedOn, reviewer, gates[], planTemplate? }   // static content, not user data
Symptom      { id, ts, kind, severity }
Reminder     { id, kind, time, days[], enabled }   // legacy simple reminders; superseded by PlanItem
PlanItem     { id, kind: meal|walk|water|medicine|check|sleep|custom, title, timeOfDay, durationMin?, days[], startDate, endDate?, medId?, voice: inherit|on|off, alarmStyle: boolean, snoozeMin, maxRepeats, privateMode: boolean, enabled }
PlanOccurrence { id, planItemId, date, scheduledTs, status: pending|done|snoozed|skipped|missed, actedTs?, source: app|notificationAction }
ReminderSettings { voiceGlobal, voiceLanguage: en|hi, speechRate, speakMedicineNames: false, lockScreenPrivacy: true, quietHours{start,end}, medicineBypassQuietHours: true, firstNameInMessages }
```
All timestamps stored as ISO strings with timezone offset; display in local time (IST default).

---

## PART 6 — MASTER PROMPT FOR CLAUDE CODE

Copy everything between the lines into Claude Code, in the existing VitalCheck project folder.

---START OF PROMPT---

# ROLE
You are a senior full-stack engineer and product designer. You are upgrading **VitalCheck**, an existing lifestyle risk-tracker web app (React 18 + TypeScript + Recharts + date-fns, Express + SQLite backend) into a **phone-ready, offline-first, privacy-first health-habit app** for people managing or at risk of diabetes and high blood pressure. Work autonomously end to end. Only stop to ask me something if it is truly unavoidable (for example a login or key only I can provide). Make sensible decisions, record them in `DECISIONS.md`, and keep going.

# NON-NEGOTIABLE PRINCIPLES
1. **Awareness-only app, not a medical device.** VitalCheck only helps users notice patterns in their own data and then directs them to a doctor. It must NEVER: diagnose, say "you have X", interpret results as a condition (the one allowed exception is stating a published reference-range label for lab values, such as "in the range guidelines call the prediabetes range", under the Phase 4C rules), give treatment, medication, dose, diet-prescription or insulin advice, or tell the user to change or stop any medicine. Every flag follows one fixed pattern: (a) what we noticed in your numbers, (b) "this is for awareness only, not a diagnosis", (c) "please consult your doctor". For Urgent flags the pattern is: "This reading is outside the usual range. Please seek medical help now or call 112." Never offer self-treatment steps (no "take 15 g sugar", no "re-check and wait"). Show the disclaimer at first run, on the Insights screen, on every flag, and in every exported PDF footer ("For awareness only. Not medical advice. Consult your doctor."). Require the user to accept a Terms and Disclaimer screen before first use, and record the acceptance locally with a timestamp. Put all flag wording in one file (`src/risk/messages.ts` plus Hindi) so a clinician or lawyer can review it in one place.
2. **Privacy first.** All health data stays on the device. No cloud, no third-party analytics, no tracking SDKs, no network calls with health data (except the optional, opt-in AI food-photo feature, off by default, clearly labelled). Never log health values or personal data to the console or files. Never commit any database, export file, `.env`, or sample health data to git; add them to `.gitignore`. Seed data must be clearly fake and generated by a script that runs only in dev.
3. **Safety-critical logic is tested.** The risk engine must have thorough unit tests, including boundary values.
4. **Accessible.** WCAG AA contrast, 48px touch targets, screen-reader labels, respects system font scaling and reduced-motion.
5. **Commit after each phase** with clear messages. Run lint, typecheck and tests before every commit. Fix what fails.

# PHASE 0 — AUDIT AND ARCHITECTURE
- Read the whole codebase. Write `AUDIT.md` listing what exists, bugs found, and what you will change.
- Remove committed test data from the repo (and from git history only if simple; otherwise just stop tracking and gitignore it) and add `.gitignore` entries.
- **Change storage to offline-first:** introduce a repository layer (`src/data/repositories/*`) with interfaces (`BPRepo`, `GlucoseRepo`, …) and implement them with **Dexie (IndexedDB)**. The UI must only talk to repositories, so storage can later be swapped to SQLite (Capacitor) or a synced backend. Migrate the existing SQLite data model concepts into the new schema (section "Data model" below). Keep the Express server in an `optional-server/` folder, unused by default.
- Set up: Vite + React + TS, React Router, Zustand (or Context) for state, Vitest + React Testing Library, Playwright (basic smoke), ESLint, Prettier, `vite-plugin-pwa`.
- Make it an installable **PWA**: manifest, icons, service worker with offline caching, and local notification scheduling where the browser supports it (with an in-app fallback banner).
- Set up **i18n** (react-i18next) with `en` and `hi` locales. All user-facing strings go through i18n from the start.

# PHASE 1 — DATA MODEL AND RISK ENGINE v2
Implement these entities with TypeScript types and Dexie tables: Profile, BPReading, GlucoseReading, Meal, SleepSession, Activity, Weight, Medication, MedLog, LabResult, Symptom, Reminder, PlanItem, PlanOccurrence, ReminderSettings (see "Data model" and Phase 4B for fields). Timestamps are ISO strings with offset; default timezone Asia/Kolkata; **a sleep session belongs to its wake date**. Support multiple readings per day.

Build `src/risk/` as pure, framework-free functions:
- `thresholds.ts`: all numbers in one documented config, overridable by user targets. Header comment: general awareness thresholds, not diagnostic, to be reviewed by a qualified clinician before public release.
- `evaluate.ts`: returns `Flag[]` where `Flag = { id, tier: 'normal'|'elevated'|'high'|'veryHigh'|'urgent', title, plainLanguage, why, nextStep, seeDoctor, emergency }`. All wording lives in `src/risk/messages.ts` (en + hi) and follows the awareness-only pattern: what we noticed, "for awareness only, not a diagnosis", "please consult your doctor".

Rules to implement:
- **BP** (mmHg), on the 7-day average: Normal <120/<80; Elevated 120–129 and <80; High 130–139 or 80–89; Very High ≥140 or ≥90. **Urgent** for any single reading ≥180 systolic and/or ≥120 diastolic: show "This reading is outside the usual range. Please seek medical help now or call 112." No self-management steps.
- **Glucose** (mg/dL) by context. Fasting: Normal 70–99, Watch 100–125, High ≥126. Post-meal 2h and random: Normal <140, Watch 140–199, High ≥200. Bedtime: Normal 100–140. **Urgent** for <70 and ≥300: show "outside the usual range, follow your doctor's plan or seek medical help / call 112". No self-treatment advice. If the user declared diabetes and entered doctor targets, use those instead.
- **Sleep:** <6h on ≥3 of last 7 nights = Elevated; repeated >9.5h = Watch.
- **Activity:** weekly minutes vs 150 target.
- **Combination escalation:** high BP + symptoms; high fasting glucose on ≥3 days + thirst/urination symptoms; missed meds ≥3 days + rising trend. Escalate wording only.
- **Staleness:** if no readings in 7+ days, return a "not enough recent data" flag rather than a score.
- Language: no "you have", no drug names or doses, reading level ~Class 8, always include "talk to your doctor". Provide Hindi translations for every flag string.
- Write **Vitest unit tests** covering every boundary (e.g., 119/79, 120/79, 129/79, 130/79, 139/89, 140/89, 179/119, 180/90, glucose 69/70/99/100/125/126, etc.), the sleep-after-midnight date rule, and the staleness rule. All must pass.

# PHASE 2 — THEMING SYSTEM (SIMPLE and REALISTIC, with toggle)
Build a design-token system (CSS variables + a `ThemeProvider`) supporting **two looks × light/dark**:
- `simple` and `realistic`, switchable at runtime via a toggle in the header and in Settings, persisted in the Profile (not localStorage-only), with a 250ms cross-fade, no reload.
- Light/dark follows the system by default with manual override.
- A **"Reduce effects"** setting. Auto-enable it when `prefers-reduced-motion` is set or `navigator.deviceMemory <= 2`; it removes blur, ambient animation and heavy shadows.

Every shared component must accept the theme through tokens, and **hero components have two implementations** chosen by theme:
| Component | Simple | Realistic |
|---|---|---|
| Surface/Card | flat, 1px border, 12px radius | frosted glass (backdrop-blur), layered soft shadows, inner highlight, 24px radius |
| Background | solid neutral | animated gradient mesh (navy→teal→violet dark; porcelain light) |
| BP display | big numerals + risk chip | analog dial gauge with metallic bezel, glass reflection, needle easing to latest systolic, risk-banded ticks |
| Glucose display | big numerals + context chip | glucometer-style LCD panel with soft glowing digits and a droplet animation on save |
| Daily Score | flat progress ring | concentric 3D rings (sleep, activity, food, meds) with gradient strokes and glowing end caps |
| Sleep | simple bar of bedtime→wake | moon-and-stars night-sky arc |
| Charts | thin lines, dots, pale target band | smooth splines, gradient fills, glowing last point, animated draw-in, glass tooltip |
| Icons | outline (Lucide) | duotone/filled |
| Feedback | 150ms fades | haptic-style spring press, count-up numbers, light confetti on streak milestones |
Fonts: Simple = Inter/system. Realistic = Sora or Manrope for numerals + Inter for text, tabular numerals. Accent: Simple teal `#0F766E`. Severity colours are semantic tokens in both themes; always show icon + word, never colour alone.
Build a `/design` route (dev only) showing every component in all four combinations for visual QA.

# PHASE 3 — SCREENS
Mobile-first (360–430px); on desktop, centre a 480px phone-width layout. Bottom nav: **Today · Trends · [＋ Log FAB] · Insights · Me**. Build:
1. **Consent + Disclaimer + Emergency numbers** (first run).
2. **Onboarding** (profile, conditions, targets with "ask your doctor" hints, units, reminders, choose Simple/Realistic with live preview).
3. **Today:** greeting, Daily Score (0–100 from sleep, activity, food, readings logged, meds taken), latest BP and glucose cards, today's medication checklist, 3 "next best actions", risk banner when relevant.
4. **Quick Log bottom sheet:** BP (with context, optional pulse), Glucose (with context chips), Food, Sleep, Activity, Weight/waist, Symptoms (chips), Meds. Validate ranges with friendly errors; "Are you sure?" on extreme values; support edit and delete with undo toast.
5. **Trends:** 7d/30d/90d switch, metric selector, chart, stats strip (avg, min, max, % in range), scrollable history list with edit.
6. **Insights:** flags with expandable "why" and "what to do", correlation cards, weekly summary.
7. **Medications:** schedule, taken/skipped/snooze, adherence calendar, refill reminder.
8. **Reports:** generate a **Doctor PDF** (30/90 days: averages, min/max, % in range, charts, meds adherence, symptoms, flags) using a client-side PDF library; share via Web Share API; also CSV and JSON export.
9. **Me/Settings:** profile, targets, reminders, theme toggle, reduce effects, language (en/hi), units, app lock, **Export my data**, **Delete everything** (typed confirmation), emergency card, about.
10. **Emergency screen**, reachable from any Urgent banner.
11. **Planner** ("My Plan" card on Today plus a full screen from Today and Me): see Phase 4B.
12. **Labs and Guidance** (add report, results with reference-range label, general lifestyle cards with "Add to my plan", IDRS risk check, Learn hub, doctor-visit prep): see Phase 4C.

# PHASE 4 — HABIT AND VALUE FEATURES
- **Reminders** (BP morning/evening, fasting glucose, meds, evening log nudge, weekly review) with quiet hours.
- **Streaks and goals** (logging streak, weekly goal rings, badges). Positive, never shaming.
- **Insights engine** (rule-based, no AI): sleep↔BP, exercise↔glucose, late dinner↔fasting glucose, weekday vs weekend. Show only with ≥14 data points and word as "a pattern we noticed, not proof".
- **Indian food quick-log:** a local JSON library of ~80 common foods (roti, rice, dal, sabzi, paneer, idli, dosa, poha, upma, paratha, samosa, chai with sugar, fruits, etc.) with portion chips (S/M/L) mapping to Low/Medium/High **bands** for carbs, sugar, sodium. Label clearly as estimates.
- **Weekly summary card** (Sunday): improved / watch / one goal.
- **Lab records** (HbA1c etc.) with trend and "next test due" reminder.
- **Voice logging** via Web Speech API with a mandatory confirm screen (e.g., "BP 132 over 86 morning").
- **Emergency card** (conditions, meds, allergies, emergency contact).

# PHASE 4B — DAILY PLANNER, POLITE NOTIFICATIONS, VOICE ALARMS, MEDICINE TIMER
Goal: the user plans their day (meals, walks, water, medicine, BP/glucose checks, bedtime wind-down, custom items) and the app reminds them politely at the planned times, with optional spoken voice and an optional alarm style. **The notification ALWAYS fires. The voice toggle only controls speaking.**

**Planner UX**
- "Plan my day" wizard with templates (Morning routine, 3 meals, evening walk, medicine schedule, bedtime wind-down). Each item: kind, title, time, optional duration, repeat days (daily / weekdays / custom), optional end date, and per-item overrides (voice inherit/on/off, alarm style, snooze minutes, max repeats).
- Today shows a timeline of planned items with status (pending / done / snoozed / skipped / missed). Tap to mark done, snooze, skip or reschedule. "Copy yesterday's plan", "Use for tomorrow", and a weekly view.
- Completing a Walk, Water, Meal or Check item should offer one tap to log the matching entry (e.g. mark the walk done, then pre-fill an Activity log). Medicine items write to MedLog.
- Plan completion feeds the Daily Score and weekly summary ("5 of 6 planned items done"). Positive framing only.
- **Medicine timer:** a countdown card on Today ("Next medicine in 1h 20m"), user-set interval timers with a label and duration, and refill reminders. Add a duplicate-tap guard: if the same medicine is marked taken twice within a short window, show "You marked this as taken at 8:02. If you are unsure, please check with your doctor or pharmacist." (awareness only, no dose advice).

**Settings (ReminderSettings)**
- Global Voice toggle plus per-item override. Voice language (en/hi) and speech rate. Alarm style per item (uses the alarm audio stream and a full-screen reminder card).
- Quiet hours, with a setting "Medicine reminders bypass quiet hours" (default ON, user can turn off).
- Lock-screen privacy ON by default (notification text is generic, e.g. "Planned reminder"). "Speak medicine names aloud" OFF by default. Optional first name in messages.
- A "Make reminders reliable" guide screen (see Android section below).

**Behaviour rules**
- Notification always fires at the planned time regardless of voice setting. Voice OFF = normal sound and vibration only. Never skip a notification because voice is off.
- Polite tone, en + hi, all strings in `src/reminders/messages.ts`. Examples: "Gentle reminder: it is time for your planned medicine. Please follow your doctor's instructions." / "Your planned walk is starting. Whenever you're ready." / "Your planned lunch time is here. Enjoy your meal." Never recommend doses, medicine timing changes, foods or exercise intensity: reminders only echo the user's own plan (awareness-only rule applies).
- Notification actions: **Done / Remind me in N min / Skip**, handled without opening the app. Default snooze 10 min, max 2 repeats for medicine and 1 for other items, then stop quietly and mark "missed" with no guilt message. Spoken text is at most 2 short sentences, never looped.

**Architecture**
- Define a `ReminderScheduler` interface (`schedule`, `cancel`, `rescheduleAll`, `checkPermissions`, `speak`). Implement `WebScheduler` and `CapacitorScheduler`. The planner UI and logic only use the interface. Pure planning logic (expanding PlanItems into the next N days of occurrences, snooze and miss rules, quiet hours, channel/voice choice) lives in framework-free functions with Vitest tests.
- **Web (PWA) implementation:** reminders while the app is open (in-app banner plus Web Speech `speechSynthesis` for voice), plus best-effort service-worker notifications. Show an honest in-app notice: "For reliable alarms when the app is closed, use the Android app." Do not pretend the web version is reliable.
- **Android implementation (Phase 6 native work):** exact alarms (AlarmManager) with a boot receiver, timezone-change receiver and app-update receiver that reschedule everything. Use a small custom Capacitor plugin (`VoiceAlarm`) in which the alarm receiver posts the notification and, when voice is on, speaks the message with Android's on-device `TextToSpeech` (via a short foreground service so it works with the app closed). Fallback if TTS is unavailable: separate notification channels per category with a short bundled voice clip. Because Android 8+ fixes sound per channel, create paired channels (`<kind>_voice` and `<kind>_quiet`), pick the channel when scheduling, and reschedule everything when the user flips a voice toggle. Notification actions write to a native outbox that the app imports into Dexie on next open, so actions work when the app is closed.
- Permissions flow: Android 13+ notification permission, exact-alarm permission with a clear explanation screen (and fallback to inexact alarms with a visible warning if the user declines), battery-optimisation and OEM auto-start guidance for Xiaomi, Oppo, Vivo, Realme, OnePlus and Samsung. Per Android's documentation, `USE_EXACT_ALARM` is reserved for calendar and alarm-clock apps under Play policy, so use `SCHEDULE_EXACT_ALARM` (denied by default on new installs targeting Android 13+): call `canScheduleExactAlarms()`, send the user to `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` when it returns false, listen for the permission-state-changed broadcast, and fall back to inexact alarms with a visible warning. Also check Play policy on full-screen intents and document the chosen approach in `DECISIONS.md`.
- Voice runs fully on-device. No cloud TTS, no network. If the Hindi voice is missing, prompt the user to install it once.

**Acceptance tests**
- Unit tests: occurrence expansion (daily/weekdays/custom/end date), snooze and max-repeat/missed logic, quiet hours with medicine bypass, DST/timezone safety (IST default), voice-off still yields a scheduled notification with `voice=false`, channel choice follows the voice flag, duplicate-tap guard.
- Integration: plan an item, advance a fake clock, verify the notification payload, voice flag, and the status changes for done/snooze/skip/missed.
- Manual Android checklist in `ANDROID_BUILD.md`: reminder fires with app closed, after reboot, after timezone change, with voice on and off, in silent mode with alarm style on, and with lock-screen privacy on.

# PHASE 4C — LAB REPORTS, GENERAL LIFESTYLE AWARENESS CARDS, EXTRA FEATURES
Goal: the user adds a lab report; the app shows where each value sits against reference ranges and a short set of **general lifestyle awareness cards**, always with "for awareness only, not a diagnosis, please consult your doctor". Every card can be added to the Planner in one tap. Everything stays on the device.

**Data:** extend `LabResult` (testKey, value, unit, date, labName, printedRefLow/High, printedFlag, source manual|ocr|pdf, userConfirmed, fileRef), add `RiskCheck` (IDRS) and a static `GuidanceCard` content file `src/guidance/cards.ts`.

**Engine (pure, framework-free, `src/guidance/`)**
- `classifyLab(test, value, unit, profile)` returns a reference-range label. For **HbA1c**: normal <5.7%, "prediabetes range" 5.7 to 6.4%, "diabetes range" ≥6.5%. **Fasting glucose:** <100, 100 to 125, ≥126 mg/dL. **Post-meal / OGTT:** <140, 140 to 199, ≥200 mg/dL. For all other tests use the **reference range printed on the user's report**; if none was entered, show the value and "ask your doctor", with no label. Unit conversions (HbA1c % and mmol/mol, glucose mg/dL and mmol/L) with tests.
- Wording rule: "is in the range guidelines call the prediabetes range" and never "you have prediabetes". Lab values only; home meter readings keep the Part 4 wording. For users who declared diabetes, do not show range labels; show the trend and "share with your doctor".
- `selectCards(labs, profile, flags)` returns at most 4 cards, then applies the **safety gates**:
  - Never include medicines, doses, supplements, herbal or home remedies, detox, fasting or crash diets, calorie targets, or any "start/stop/change medicine" text. A Vitest lint scans every card string (en and hi) for banned terms.
  - Doctor-only tests (creatinine, eGFR, urine albumin, TSH, haemoglobin, vitamin D, B12, ALT, AST, electrolytes): no lifestyle cards, only "outside the printed range, please discuss with your doctor".
  - Active Urgent flag: show only the "seek medical help" message.
  - Kidney disease, pregnancy or breastfeeding declared: hide diet, weight and exercise cards.
  - Age 65+, heart condition, joint problems, recent surgery, or chest-pain symptoms logged: exercise card adds "check with your doctor before starting; start slowly".
  - User takes diabetes medicines: exercise card adds "ask your doctor about activity and low-sugar risk". Low-sugar reading in the last 30 days: hide intensity suggestions.
  - BMI under 18.5: no weight content. Weight and waist cards are opt-in, never default. No calorie counting anywhere.
  - Report older than 6 months: show "this report is old".
  - App is 18+ only (age gate at onboarding).
- **Card content (en + hi), each with source label and "Why am I seeing this?":**
  1. Keep moving: "Many guidelines suggest about 150 minutes of moderate activity a week, for example a brisk 30 to 45 minute walk on most days. Start smaller if you are new to exercise." (WHO 2020; ADA 2026). Plan template: Walk 30 min, 5 days.
  2. After-meal steps: "A few minutes of light walking after meals is linked to smoother after-meal sugar curves. Evidence comes from small studies." Plan template: 10-minute walk after lunch.
  3. Skip sugary drinks: "Soft drinks, packaged juices, sweetened tea or coffee and sharbat add sugar quickly. Water or unsweetened drinks are a gentler choice." (ICMR-NIN 2024: added sugar under 5% of energy, about 25 to 30 g a day).
  4. Fill half your plate with vegetables: "Vegetables, fruit and leafy greens roughly half the plate; prefer whole grains and pulses; fewer packaged and fried snacks; read food labels." (ICMR-NIN 2024).
  5. Go easy on salt: "Under 5 g of salt a day (about one teaspoon). Watch pickles, papad, namkeen and packaged snacks." (WHO).
  6. Sleep and sitting: "Regular sleep of about 7 to 9 hours, and a little movement instead of long sitting." (WHO 2020 for sitting).
  7. Plan your next check: "Ask your doctor how often to re-test. ADA suggests at least yearly for this range." Plan template: reminder at the user's chosen interval.
  8. Weight and waist (opt-in only): "Some lifestyle studies used 5 to 7% weight loss. Ask your doctor what is right for you." Show Asian Indian reference ranges (waist ≥90 cm men, ≥80 cm women; BMI ≥23 overweight, ≥25 obesity) as reference only.
- Content governance: each card is `{id, text_en, text_hi, source, reviewedOn, reviewer, gates[], planTemplate?}`. CI fails if a card lacks a source, Hindi text, a review date within 12 months, or contains banned terms.

**Which tests trigger which cards:** HbA1c / fasting / post-meal glucose in the prediabetes or diabetes range → cards 1 to 7 (diabetes range leads with "please consult your doctor soon"). Normal range → one positive "keep it up" card plus the re-check card. BP elevated or high → cards 1, 5, 6. LDL or triglycerides outside the printed range → cards 1, 4. HDL low → card 1.

**Screens and flows**
- **Labs and Guidance** screen: add report (manual form first), results list with range label chips (icon + word, never colour alone), trend chart per test, guidance cards with "Add to my plan", "Next test due" reminder.
- **Scan / PDF import (behind a feature flag):** on-device OCR (Android ML Kit text recognition in the Capacitor phase, browser OCR on web) and PDF text extraction. The user must confirm every extracted value on a review screen. Never auto-save OCR output. Files stay encrypted on the device and are never uploaded.
- **IDRS risk check:** age (<35 = 0, 35 to 49 = 20, ≥50 = 30), waist (women <80 / men <90 = 0; women 80 to 89 / men 90 to 99 = 10; women ≥90 / men ≥100 = 20), activity (vigorous or strenuous work = 0, moderate = 10, mild = 20, none/sedentary = 30), family history (none = 0, one parent = 10, both = 20). Total <30 low, 30 to 50 moderate, ≥60 high. Wording: "Worth discussing testing with your doctor", never "you have diabetes risk of X%".
- **Doctor-visit prep:** auto-built question list from the data, visit notes, next-appointment reminder, one-page PDF.
- **Learn hub:** 60-second explainers in Hindi and English with read-aloud (HbA1c, measuring BP, plate, salt), each with its source.
- **Habit ladder** (10, 20, 30, 45 minutes over 6 weeks) and "after lunch I will walk 10 minutes" prompts feeding the Planner.
- **Encrypted backup and restore** (file export, and optional the user's own Google Drive app-data folder). **Privacy shade** (one tap hides all numbers). **Family / parent mode** (large text, weekly summary via the share sheet).
- Later, behind flags: packaged-food label reader (on-device OCR of sodium and added sugar, user confirms), home BP coach (guided 7-day morning and evening routine), Health Connect import. Do not build ABHA/ABDM import in v2.

**Acceptance tests:** boundary tests for HbA1c 5.6 / 5.7 / 6.4 / 6.5, fasting glucose 99 / 100 / 125 / 126, post-meal 139 / 140 / 199 / 200, unit conversions, every safety gate (CKD, pregnancy, BMI under 18.5, diabetes-medicine user, low-sugar history, urgent flag active, old report, doctor-only tests), banned-term lint passes, every card has en and hi text and a source, "Add to my plan" creates the right PlanItem, OCR values are never saved without confirmation.

# PHASE 5 — SECURITY AND PRIVACY
- **App lock:** 4–6 digit PIN (hashed with PBKDF2/Argon2-style KDF via Web Crypto; never store the raw PIN), auto-lock after inactivity, lock when app is backgrounded. Prepare a biometric hook for the Capacitor phase.
- **Optional encryption at rest:** AES-GCM via Web Crypto, key derived from the PIN; encrypt sensitive tables. Document the recovery limitation clearly ("forgot PIN = data unrecoverable unless exported").
- Strict CSP, no inline scripts, no external CDN dependencies at runtime (bundle fonts locally), dependency audit (`npm audit`) with findings in `AUDIT.md`.
- Hide sensitive values in the app-switcher preview where the platform allows.
- A `PRIVACY.md` and in-app privacy screen in plain language, plus a Play Store Data Safety draft (`PLAY_STORE_NOTES.md`): data collected = none leaves device; health-app declaration notes; DPDP-friendly consent, export and delete.

# PHASE 6 — ANDROID WRAP (Capacitor)
- Add Capacitor, generate the Android project, configure app id `com.vitalcheck.app` (editable), icons, splash, status-bar styling.
- Replace web notifications with `@capacitor/local-notifications`, add Haptics, and biometric unlock (`@capacitor-community/biometric-auth` or equivalent). Implement the native reminder and voice-alarm layer exactly as described in Phase 4B (exact alarms, boot/timezone receivers, on-device TextToSpeech, paired voice/quiet channels, notification-action outbox).
- Prepare Health Connect import (steps, sleep, heart rate, weight) behind a feature flag, with clear permission rationale text.
- Provide a `ANDROID_BUILD.md` with exact steps to produce a debug APK and a signed release AAB, and a Play Store checklist. Do not publish anything or enter any account credentials; tell me precisely what only I can do.

# QUALITY BAR AND DEFINITION OF DONE
- `npm run lint`, `npm run typecheck`, `npm test`, and the Playwright smoke test all pass.
- Lighthouse PWA and Accessibility scores ≥ 90 in both themes.
- Works fully offline after first load; data survives refresh and reinstall of the PWA cache (IndexedDB).
- Every screen verified at 360px, 412px, and desktop, in Simple/Realistic × light/dark. Take screenshots of each into `/docs/screens/` and review them yourself; fix layout, contrast and overflow issues you find.
- Test with a **generated fake dataset** (script `npm run seed:demo`, dev-only, clearly fake, never committed) covering: a normal user, a borderline-BP user, a user with an urgent reading, a user with hypoglycaemia, and a new user with no data. Verify the correct flags for each.
- Update `README.md` with run, test, build and theme instructions, and a short "How the risk engine works" section.

# HOW TO WORK
1. Start by creating a task list for all phases, then execute in order. Commit after each phase.
2. At the end of each phase, print a 5-line summary: what's done, tests status, anything that needs me.
3. If something is ambiguous, choose the safest, most private, simplest option, log it in `DECISIONS.md`, and continue.
4. When all phases are complete, give me a final report: what was built, how to run it, screenshots list, known limitations, and the short list of manual steps that only I can do (e.g. Play Console account, signing key, clinician review of thresholds).

---END OF PROMPT---

## PART 7 — Suggested order of work for you
1. Paste the prompt into Claude Code and let it run through Phases 0–2 first; check the `/design` route to compare Simple vs Realistic.
2. Have a doctor (or a trusted clinician) review `thresholds.ts` and the flag wording before sharing the app with anyone else.
3. Use it yourself or with family for two weeks before Android packaging; real usage will show which of the Tier 2 features matter most.
4. Only then do Phase 6 and the Play Store steps.

---

Market research, regulation notes, decisions and sources are in [research-and-decisions.md](research-and-decisions.md). The v3 build prompt is [claude-code-build-prompt-v3.md](claude-code-build-prompt-v3.md).
