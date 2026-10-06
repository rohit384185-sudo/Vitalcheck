# VitalCheck v3 — Claude Code Master Prompt

**How to use:** open Claude Code in an empty folder (or the old VitalCheck repo, which becomes reference only), and paste everything between `START OF PROMPT` and `END OF PROMPT`. If you have `vitalcheck-v2-plan.md`, put it in `docs/` first. The prompt does not need it, and it wins on any conflict.

---START OF PROMPT---

# 0. ROLE AND WORKING AGREEMENT

You are a principal mobile engineer, product designer and QA lead building **VitalCheck v3**: a premium, privacy-first, offline-first Android health-habit app for adults in India who manage or want to prevent type 2 diabetes and high blood pressure, and for the family members who help them. iOS is a later port, so keep the code iOS-ready.

Work autonomously, end to end, in the order of the phases below. Ask me a question only when something is truly unavoidable (an account, a physical device, a signing key, a legal decision). For everything else choose the safest, simplest, most private option, record it in `DECISIONS.md` with a one-line reason, and continue. Create a task list for the phases and keep it updated. Commit after every phase with a clear message, after running analyze, tests and the size report. Never commit secrets, health data, databases, exports or keystores. The old React/Express code, if present, is reference only: port its test vectors, not its architecture.

At the end of every phase print five lines: what is done, test status, current install size, anything that needs me, and the next phase.

# 1. PRODUCT DEFINITION

- **Promise:** "Know your numbers. Build your habits. Take them to your doctor."
- **Users:** adults 18+, India first (Hindi and English at launch), including 50+ users and caregivers who track for a parent or spouse.
- **Why it can win** (market research, Oct 2026): India has about 101 million people with diabetes, 136 million with prediabetes and 315 million with hypertension (ICMR-INDIAB, Lancet Diabetes & Endocrinology 2023). Five apps were checked on their store pages: BeatO (1M+ installs, 4.6 stars; pairs with its own glucometer, offers doctor consults and a chatbot), Omron Connect (1M+; syncs Omron monitors, with premium insights; reviews mention daily logouts, lost device connections and crashes), mySugr (500K+, 4.6; glucose logbook and meter sync; reviewers ask for dark mode), sugar.fit (100K+; paid CGM coaching programme; lists sharing data with third parties) and Fitterfly (100K+; coaching, CGM, AI coach, voice meal logging; its iOS app is 183 MB). They are strong on hardware, coaching and CGM. The gap is not basic tracking (BeatO and Fitterfly also track vitals and food); it is a free, hardware-free, login-free, on-device-private app that also plans the day with reliable voice reminders in Indian languages, prepares the doctor visit and supports caregivers. One industry summary reports only about 3 to 4% of fitness-app users still active at day 30 on average, with manual-entry fatigue a main reason, so passive data capture, onboarding that shows value in under a minute, and well-timed reminders are the core retention design.
- **Our edge:** one app for BP, glucose, labs, food, walking, medicines and a daily plan; free core with no hardware needed; no login; data stays on the phone; polite Hindi and English voice reminders that work with the app closed; a premium, realistic interface with a Simple alternative; a one-page doctor report; family mode.
- **Draft intended-use statement** (put in `docs/INTENDED_USE.md`, the store listing and About): "VitalCheck helps you record your health readings and daily habits, see your own trends, and prepare for conversations with your doctor. It is for general awareness and wellness tracking only. It is not a medical device and does not diagnose, treat, cure or prevent any medical condition. Always consult a doctor for medical advice. In an emergency call 112."

# 2. NON-NEGOTIABLE RULES

## 2.1 Awareness-only and regulatory safety
India's CDSCO software guidance (21 Jul 2026) says the wellness exemption depends on the software having no medical purpose, and that a "wellness" label cannot cancel diagnostic, monitoring, prediction or treatment claims. Google Play and Apple apply similar scrutiny. A disclaimer alone does not decide this. Therefore:
1. **ComplianceProfile.** Implement a build-time setting with three levels and make every interpretation feature obey it from one place (`lib/compliance/compliance_profile.dart`):
   - `full`: everything below (for my personal and beta use).
   - `standard` (**default for the `store` flavor**): shows reference-range labels, trend charts, plans and general habit cards; hides the IDRS score and any numeric "risk" score; keeps the urgent "seek medical help / call 112" banner.
   - `trackingOnly`: records, charts and reminders only, no labels; urgent banner remains but is configurable.
   Counsel may later change the default, so no feature may bypass this.
2. **Words.** In user-facing text never use: diagnose, detect, predict, treat, cure, reverse, prevent, "you have", "risk of", "abnormal", "healthy/unhealthy" about a person's number. Use: "reference range", "usual range", "above/below the usual range", "pattern we noticed", "worth discussing with your doctor". Name the notes "Heads-up" in the UI. Internal code names are free.
3. **Never build:** a symptom checker, a diagnosis or treatment chatbot, medicine dose or timing advice, drug-interaction checks, camera-only measurement of blood pressure, glucose or SpO2 (Apple 1.4.1 forbids these and Play requires regulatory clearance), calorie targets, crash diets, fasting plans, supplements or home remedies, generative-AI text that states anything about a person's health, ads, third-party analytics SDKs, health-data cloud sync, or selling or sharing data.
4. Every Heads-up, lab result and exported report carries: "For awareness only, not a diagnosis. Please consult your doctor." Urgent: "This reading is outside the usual range. Please seek medical help now or call 112." The first-run Terms screen must be accepted (timestamp stored locally). Add the exact "not a medical device" wording Google Play requires for non-regulated health apps after checking the current policy text, and an 18+ age gate.
5. Write `docs/COUNSEL_QUESTIONS.md`: the questions a lawyer must answer before public launch (intended-use class under MDR 2017, whether reference-range labels, lab labels and the IDRS count as monitoring or prediction, wording of the Terms, DPDP position for an on-device-only app, liability limits).

## 2.2 Privacy and security (my standing top priority)
- All health data lives only on the device, in a **SQLCipher-encrypted** database whose key is held in the Android Keystore. No accounts. No Firebase, no analytics, no ad or tracking SDKs, no network permission unless a flagged opt-in feature needs it (list every network call in `docs/NETWORK_CALLS.md`; default is none).
- Set `allowBackup="false"` and exclude the database from device transfer and cloud backup (Android auto-backup would copy health data to Google Drive; Apple forbids personal health information in iCloud). Provide my own **encrypted export/import** (file protected with a user passphrase) instead.
- App lock (PIN with a slow KDF, plus biometrics via the OS), auto-lock on background, `FLAG_SECURE` on sensitive screens (hides them in the app switcher and screenshots; user can switch off), a one-tap **Privacy shade** that hides all numbers, lock-screen-safe notifications by default.
- No health values, names or medicine names in logs, crash reports, clipboard or notification text unless the user turned that on. Redact in release builds.
- Consent and rights (India's DPDP Rules 2025 notice, consent, breach and security duties apply from 13 Nov 2026): plain-language privacy notice, itemised permissions each with a reason, consent log, withdraw consent as easily as giving it, **Export my data** and **Delete everything** (typed confirmation), `PRIVACY.md`, a Play Data Safety draft in `PLAY_STORE_NOTES.md`, and a short breach-response note. Counsel confirms the legal position.
- Only the permissions each shipped feature needs, requested at the moment of use with a rationale screen.
- Fonts, models, animations and the food database are bundled; nothing downloads at runtime except optional on-demand asset packs from Google Play itself.

## 2.3 Quality
- Accessible: TalkBack labels on every control and gauge ("Systolic 128 millimetres of mercury, within the usual range"), text scale to 200% without clipping, contrast at least 4.5:1 (also over glass), 48 dp touch targets, reduced-motion respected, never rely on colour alone.
- Performance: 60 fps on a low-cost phone (3 GB RAM, mid-range 2022 chip). Cold start under 2.5 s. No jank in the Realistic look or it auto-degrades (section 6).
- Safety-critical logic (sections 4 and 5) is pure Dart with boundary tests. Coverage of `domain/` at least 90%.

# 3. STACK AND ARCHITECTURE (decision made; do not relitigate)

The v2 plan used a web view wrapper. v3 needs 60 fps realistic gauges and animation, Bluetooth, Health Connect, exact alarms with voice, and widgets on cheap phones, so use **Flutter (current stable, Dart 3) with Kotlin platform code for alarms and services**. Check `flutter --version`; if Flutter or the Android SDK is missing install them per the official docs, and only if that is impossible tell me exactly what to install.

- Structure: feature-first folders, `domain/` (pure Dart, no Flutter imports), `data/` (Drift + SQLCipher repositories behind interfaces), `presentation/`. State: Riverpod. Navigation: go_router. Immutable models. `flutter analyze` with strict lints must be clean.
- Android: minSdk 26, targetSdk = current Play requirement. Flavors `personal` (ComplianceProfile.full, demo-seed command available) and `store` (ComplianceProfile.standard, no seed code compiled in).
- Native Kotlin modules (method channel plus event channel) in `android/`: `ReminderScheduler` (AlarmManager exact alarms, boot, timezone-change and app-update receivers that reschedule everything), `VoiceAlarmService` (on-device `TextToSpeech`, short-lived), `NotificationActionReceiver` (Done / Remind me in 10 min / Skip handled while the app is closed, writing to a native outbox that Dart imports on next open), `StepCounterService` for Live Walk.
- Plugins: choose maintained packages, pin versions, check licences, and record the choice in `DECISIONS.md`: `drift` + `sqlcipher_flutter_libs`, `flutter_secure_storage`, `local_auth`, `health` (or a small custom Health Connect module), `flutter_reactive_ble` or `flutter_blue_plus`, `google_mlkit_text_recognition` (bundled model), `speech_to_text` (prefer on-device recognition and disclose when the OS may use the network), `pdf`, `share_plus`, `home_widget` or native widgets, `rive`/`lottie`, `device_info_plus`, `workmanager`, `intl` with ARB localisation. Avoid anything unmaintained.
- Permissions to use: POST_NOTIFICATIONS, SCHEDULE_EXACT_ALARM (**not** USE_EXACT_ALARM, which Play restricts to calendar and alarm-clock apps; check `canScheduleExactAlarms()`, open the system exact-alarm settings screen when false, fall back to inexact alarms with a visible warning), RECEIVE_BOOT_COMPLETED, VIBRATE, Health Connect read permissions only for data types I use, Bluetooth scan/connect (declare `neverForLocation`), ACTIVITY_RECOGNITION only for Live Walk. Full-screen intents are restricted on recent Android; default to a high-priority heads-up notification and add the full-screen alarm card only if current Play policy allows it for this app (document the finding). Verify whether starting the voice service from the alarm receiver is permitted on current Android and use the foreground-service type that Play policy allows; if not, fall back to a notification channel with a bundled short sound.
- Tooling: unit, widget, **golden** (screenshot) and integration tests; `tools/size_report.sh`; GitHub Actions (or local script) running analyze, tests and size gate. No secrets needed anywhere in v3.

# 4. DOMAIN ENGINES (pure Dart, ported behaviour; boundary-tested)

All numbers live in one documented config file (`thresholds.dart`) with the header: "General awareness reference ranges. Not diagnostic. To be reviewed by a qualified clinician before public release. The user's own doctor-set targets override." Users can enter personal targets. Users who declared diabetes see trends and "share with your doctor" instead of range labels for lab values.

## 4.1 Blood pressure (mmHg)
Use the 7-day average of readings (needs 3 or more) for the standing tier, and every single reading for the urgent check. Tiers: Normal below 120 and below 80; Elevated 120 to 129 and below 80; High 130 to 139 or 80 to 89; Very high 140 or more, or 90 or more. **Urgent** if any single reading is 180 or more systolic, or 120 or more diastolic. Entry validation: systolic must exceed diastolic; unusual values (systolic under 80 or over 200, diastolic over 130) ask "Are you sure?" once.

## 4.2 Glucose (mg/dL), by context
| Context | Usual | Above/below | High | Urgent |
|---|---|---|---|---|
| Fasting | 70 to 99 | 100 to 125 | 126 and up | under 70, or 300 and up |
| 2 h after meal / random | under 140 | 140 to 199 | 200 and up | under 70, or 300 and up |
| Bedtime | 100 to 140 | under 100, or 141 to 180 | over 180 | under 70 |
Support mmol/L display with tested conversion. Low-reading text never gives treatment steps: "This reading is lower than the usual range. Please follow the plan your doctor has given you, or seek medical help now or call 112 if you feel unwell."

## 4.3 Lab results
HbA1c: below 5.7% usual; 5.7 to 6.4% "in the range guidelines call the prediabetes range" (ADA Standards of Care 2026); 6.5% or more "in the range guidelines call the diabetes range, only a doctor can confirm what this means, please consult your doctor soon". Fasting glucose 100 to 125 and 126 or more, 2-hour 140 to 199 and 200 or more (standard ADA criteria; have a clinician confirm). All other tests: compare with the **reference range printed on the user's report** (user enters low and high), otherwise show the value only with "ask your doctor". Doctor-only tests (creatinine, eGFR, urine albumin, TSH, haemoglobin, vitamin D, B12, ALT, AST, electrolytes) never produce lifestyle cards. Reports older than 6 months show "this report is old". HbA1c shown to one decimal; % and mmol/mol conversion tested.

## 4.4 Habit cards (general lifestyle awareness; max 4 shown, "show all" optional)
Each card is data: `{id, title_en, title_hi, text_en, text_hi, source, reviewedOn, reviewer, gates, planTemplate}`. Cards: **Keep moving** ("Many guidelines suggest about 150 minutes of moderate activity a week, for example a brisk 30 to 45 minute walk on most days. Start smaller if you are new to exercise." WHO 2020 says 150 to 300 minutes; ADA 2026 says at least 150); **Short walk after meals** (light walking of a few minutes after meals is linked to smoother after-meal sugar curves; evidence is from small studies, mostly in people without diabetes); **Skip sugary drinks** (ICMR-NIN 2024: added sugar under 5% of daily energy, about 25 to 30 g); **Half your plate vegetables** (ICMR-NIN 2024: vegetables, fruit and leafy greens about half the plate, whole grains and pulses, fewer packaged and fried foods, read labels); **Go easy on salt** (WHO: under 5 g salt, under 2 g sodium a day); **Sleep and sitting** (about 7 to 9 hours; move instead of long sitting); **Plan your next check** (ask your doctor how often; ADA suggests at least yearly for the prediabetes range); **Weight and waist** (opt-in only; Asian Indian reference: waist 90 cm or more men, 80 cm or more women; BMI 23 or more overweight, 25 or more obesity; no weight-loss targets or calories). Every card shows its source and "Why am I seeing this?" and has **Add to my plan**.
**Gates:** a banned-term lint (medicine names, dose, supplement, herbal, remedy, detox, calorie, cure, reverse, "stop taking", fasting diet) fails CI; no cards during an active Urgent flag; hide diet, weight and exercise cards if kidney disease, pregnancy or breastfeeding is declared (show "please ask your doctor"); add "check with your doctor before starting, start slowly" for age 65+, heart condition, joint problems or recent surgery; add "ask your doctor about activity and low-sugar risk" if the user takes diabetes medicines, and hide intensity advice if a low reading was logged in the last 30 days; BMI under 18.5 hides weight content; every card must have a source, Hindi text and a review date within 12 months or CI fails.

## 4.5 Other rules
Sleep: under 6 h on 3 or more of the last 7 nights gives a Heads-up; a sleep session belongs to its wake date (tested across midnight, IST default, timestamps stored with offset). Activity: weekly minutes against 150. Staleness: no readings for 7 days shows "not enough recent data", never a stale conclusion. Combination wording escalations (high BP plus symptoms; repeated high fasting readings plus thirst or urination; several missed medicines plus rising trend) only strengthen "please talk to your doctor". **IDRS** (hidden in `standard`/`trackingOnly`): age under 35 = 0, 35 to 49 = 20, 50 or more = 30; waist women under 80 / men under 90 = 0, women 80 to 89 / men 90 to 99 = 10, women 90 or more / men 100 or more = 20; activity vigorous = 0, moderate = 10, mild = 20, none = 30; parents with diabetes none = 0, one = 10, both = 20; total under 30 low, 30 to 50 moderate, 60 or more high; wording "worth discussing testing with your doctor".

# 5. INSIGHTS ENGINE AND NOTIFICATIONS

## 5.1 Insights (rule-based, on-device, explainable)
Compute only when there are at least 14 days with both variables. For pairs such as short sleep vs next-morning systolic, a walk after dinner vs next fasting glucose, weekday vs weekend readings, and high-salt-food days vs evening BP: split days into with/without the factor, require at least 5 days in each group and a difference of at least 5 mmHg (BP) or 8 mg/dL (glucose) (product defaults, flag for clinician review). Wording: "On days after under 6 hours of sleep, your morning top number averaged 134, compared with 127 on other days (9 and 13 days). This is a pattern we noticed, not proof. You may want to mention it to your doctor." Show the numbers and a small chart behind each insight; allow "hide this insight". No generative AI writes health statements. If I enable the optional on-device Gemini Nano feature (ML Kit GenAI, offline, only on supported phones), limit it to rewriting or summarising text that already passed the rules above, behind a feature flag, off by default, with a static fallback.

## 5.2 Notifications (polite, calm, useful)
- **Plan reminders** (meal, walk, water, medicine, BP or sugar check, bedtime, custom). Always fire at the planned time. A global Voice toggle plus per-item override controls only the speaking: voice off still shows a notification with normal sound and vibration. Optional alarm style (alarm audio stream, loud on silent). Notification actions: Done, Remind me in 10 minutes, Skip, working with the app closed. Snooze default 10 min; at most 2 repeats for medicine and 1 for other items, then mark "missed" quietly with no guilt message.
- Tone examples, English and Hindi: "Gentle reminder: it is time for your planned medicine. Please follow your doctor's instructions." / "Your planned walk is starting. Whenever you're ready." / "Your planned lunch time is here. Enjoy your meal." / "No worries, you can mark it done later." Spoken text is at most two short sentences. Reminders only repeat the user's own plan; they never suggest doses, timing changes or foods.
- Privacy defaults: the lock screen shows "Planned reminder", not the medicine name; medicine names are not spoken unless the user turns it on; quiet hours 22:00 to 07:00 with "medicine reminders bypass quiet hours" ON (changeable).
- **Medicine timer:** a "next medicine in 1 h 20 min" countdown on Today, user interval timers, and refill reminders. Duplicate-tap guard: marking the same medicine twice close together shows "You marked this as taken at 8:02. If you are unsure, please check with your doctor or pharmacist."
- **Daily Briefing** (default 07:30, editable): today's plan and the first item. **Evening wrap-up** (default 20:30): "You completed 4 of 6 planned items today. Thank you." **Sunday weekly summary**: what improved, what to watch, one goal. These and other nudges (log reminders, lab re-check, walk suggestions) share a **budget of 3 non-plan notifications a day**, adaptive timing (move within one hour of the user's usual logging time), and back-off: after 3 ignored nudges of one kind, reduce its frequency. Plan reminders never move or back off.
- A **notification inbox** inside the app lists everything sent. Include a "Make reminders reliable" guide screen with exact steps for battery and auto-start settings on Xiaomi/Redmi, Oppo, Vivo, Realme, OnePlus and Samsung, with a test-reminder button and a check that alarms survive a restart.

# 6. DESIGN SYSTEM: SIMPLE AND REALISTIC

One component tree, two looks, switched by a header toggle and in Settings (saved in the profile, 250 ms cross-fade, no restart), each in light and dark (follows the phone). Tokens in one place. All fonts bundled: Sora or Manrope for numerals, Inter for text, Noto Sans Devanagari for Hindi, with room for more scripts.

**Simple:** flat surfaces, 1 px borders, 12 px radius, one teal accent, big numerals, outline icons, 150 ms fades. Fast on any phone.

**Realistic (premium, instrument-like; build with CustomPainter, fragment shaders and Rive, not images of gauges):**
- Light model: key light from the top left, applied consistently: three-layer soft shadows, 1 px inner highlight on the upper-left edge, subtle noise overlay on glass.
- Glass cards: background blur 18 to 24 plus tinted translucent fill and a gradient hairline border; guarantee text contrast with an inner scrim.
- Background: animated mesh-gradient fragment shader (navy to teal to violet in dark, warm porcelain in light), slow drift capped at 30 fps and paused in the background or in battery saver.
- **BP dial:** brushed-metal bezel (sweep gradient), risk-banded arc and ticks, glass reflection, needle driven by a spring simulation, a soft haptic click when it settles, a gentle pulse line behind it.
- **Glucose card:** glucometer LCD panel with glowing seven-segment digits (use an OFL-licensed font and record it in `LICENSES.md`) and a droplet animation on save.
- **Daily Score:** three concentric gradient rings (plan, logging, activity) with glowing end caps and count-up numbers; **Sleep:** moon-and-stars arc for bedtime to wake time.
- **Charts:** smooth splines, gradient fill, glowing last point, animated draw-in, glass tooltip with crosshair, reference ranges as soft bands.
- Micro-interactions: spring button presses, shared-axis page transitions, hero motion from the card to detail, light confetti on streak milestones. Sound off by default.
- **Performance guard:** on first launch and continuously, measure frame times and device class; if raster time averages over 12 ms or the phone is low-end, auto-switch effects to "lite" (no blur, no animated background, simpler shadows) while keeping the layout. A "Reduce effects" switch overrides. Golden tests cover every component in 2 looks x 2 themes; a `/design-lab` debug screen shows them side by side.

**Screens** (bottom navigation Today, Trends, [+ Log], Plan, Me; Insights and Labs reachable from Today and Me): Terms, Age gate and Emergency numbers; onboarding in about 60 seconds with value shown before any long form (3 questions, then a first reading or "try with sample data"); Today (greeting, Daily Score, live BP and glucose cards, next-up and medicine timer, today's plan, Heads-ups, steps and sleep from Health Connect); Quick Log sheet (BP, glucose, food, sleep, activity, weight and waist, symptoms, medicines, voice entry with a confirm screen); Trends (7, 30, 90 days, stats strip with average, min, max and share of readings within the user's target, editable history with undo); Insights; Labs and Guidance; Plan; Medicine cabinet; Doctor Pack; Family; Learn; Notification inbox; Emergency; Settings.

# 7. FEATURES BY PHASE (do in order; each ends with tests, goldens where visual, size report, commit)

## Phase 0 — Foundation
Audit any old code into `AUDIT.md`; scaffold the Flutter app with flavors; encrypted Drift database with migrations and repository interfaces; Riverpod; go_router; i18n (en, hi) with all strings in ARB files; theme tokens; lint, CI script, `tools/size_report.sh` (reports Play-style download size per ABI via an app bundle and `bundletool`, plus `--analyze-size`), `docs/` stubs, `LICENSES.md`, `.gitignore` for databases, exports, keystores and `.env`.

## Phase 1 — Domain engines
Everything in section 4 as pure Dart with tests: every boundary value (BP 119/79, 120/79, 129/79, 130/79, 139/89, 140/89, 179/119, 180/90; glucose 69, 70, 99, 100, 125, 126, 139, 140, 199, 200, 299, 300; HbA1c 5.6, 5.7, 6.4, 6.5), unit conversions, the midnight sleep rule, staleness, every card gate, the banned-term lint over all card strings in both languages, IDRS vectors, ComplianceProfile switching, insight thresholds. Provide Hindi strings for every message, marked "needs native-speaker review" in `docs/TRANSLATION_REVIEW.md`.

## Phase 2 — Design system and components
Both looks, both themes, all components from section 6, performance guard, goldens, design lab.

## Phase 3 — Core app
Terms, age gate, onboarding, Today, Quick Log with validation and undo, Trends, Heads-ups, Labs with manual entry, habit cards and **Add to my plan**, Settings, app lock, Privacy shade, export/import (encrypted) and delete-all. Seeded fake data only in the `personal` flavor through a debug command; the demo set covers a normal user, borderline BP, an urgent reading, a low glucose reading, a prediabetes-range HbA1c, a user with declared diabetes, and a new empty user, and each shows the correct flags.

## Phase 4 — Real-time data
- **Health Connect:** availability check, permission rationale screen, read steps, sleep, heart rate, weight, hydration, exercise, blood pressure and blood glucose; use the changes API plus a periodic background sync (WorkManager) so Today updates live; de-duplicate against manual entries; label every value with its source ("You", "Health Connect", "Bluetooth meter"); write data back only if the user opts in, only their own entries, and never invented data. Handle "not installed", "permission denied" and revoked permission gracefully.
- **Bluetooth meters:** pair with BLE devices that use the standard Bluetooth SIG Blood Pressure (0x1810, measurement 0x2A35) and Glucose (0x1808, measurement 0x2A18) services; parse IEEE-11073 SFLOAT values, units, timestamps and context; unit-test the parser with recorded byte fixtures; import with a confirm step; keep a "tested devices" list in `docs/DEVICES.md` that stays empty of brand claims until I test a real meter (many consumer meters use proprietary protocols).
- **Live Walk:** start a walk from the plan or Today; foreground service of the health type with the step counter (no GPS by default); live steps, time and pace; optional gentle voice cheers every few minutes; ends by offering to log the minutes and mark the plan item done.
- **Live Today:** the dashboard is stream-driven, so a new reading, a synced step count or a completed plan item updates the score, rings and cards without a refresh.
- Camera-based pulse is **out of scope for v3**.

## Phase 5 — Planner, voice alarms and notifications
Everything in section 5.2 with the native Kotlin layer, the reliability guide, templates (morning routine, three meals, evening walk, medicine schedule, bedtime wind-down, festival and travel days), copy yesterday's plan, weekly view, completion feeding the score, Daily Briefing, evening wrap-up, weekly summary, budget, adaptive timing and back-off, the inbox. Tests: occurrence expansion (daily, weekdays, custom, end date), snooze and max-repeat to missed, quiet hours with medicine bypass, timezone and daylight changes, voice-off still scheduling a notification with `voice=false`, duplicate-tap guard, and a manual Android checklist in `docs/MANUAL_TEST_PLAN.md` (fires with the app closed, after reboot, after a timezone change, voice on and off, on silent with alarm style, with lock-screen privacy on).

## Phase 6 — Medical assistance (non-diagnostic)
- **Medicine cabinet:** user-entered names, schedule, stock count and refill reminder; optional strip or label scan only to pre-fill the name text for the user to confirm; no interaction or dose checks.
- **Lab vault:** scan or import a report (on-device OCR with a mandatory review screen, never auto-save), files kept inside the encrypted store, trend per test, "next test due" reminder.
- **Doctor Pack:** a one-page PDF (30 or 90 days: averages, min and max, share of readings within the user's targets, charts, plan and medicine adherence, symptoms, notable Heads-ups, latest labs, a question list, a visit-notes field and next-appointment reminder) shared with the system share sheet. Footer: the disclaimer.
- **Emergency:** one screen with big buttons to open the phone dialer for 112 (and 108), a stored emergency card (conditions, allergies, medicines, emergency contacts, blood group) viewable from the lock screen only if the user opts in, and "find nearby hospital or pharmacy" through a maps intent (no API key, no location stored).
- **Family:** several local profiles on one phone with a large-text simple mode for a parent, and a weekly summary the caregiver can send through the share sheet. No cloud.
- **Learn:** 40 short illustrated cards in Hindi and English with read-aloud (HbA1c, measuring BP correctly, the plate, salt, walking, sleep), each with its source.
- **Appointments:** doctor visits and lab tests with reminders.

## Phase 7 — Engagement
The insights engine (5.1), streaks and weekly goal rings and badges (positive only), a habit ladder (10, 20, 30, 45 minutes over six weeks), challenges between family members that only share counts the user exports, home-screen widgets (today score and next item; quick log) and a quick-settings tile, voice logging ("BP 132 over 86 morning") with confirm, **Indian food quick-log** (offline database of about 400 common foods and drinks with portion chips mapping to Low / Medium / High bands for carbohydrate, sugar and sodium, labelled as estimates, with original vector illustrations; no scraped photos), and an optional opt-in **air-quality-aware walking nudge** that sends only a coarse city name to a public air-quality source (verify a suitable free source and its terms; if none is acceptable, drop this feature and say so).

## Phase 8 — Security, privacy, compliance documents
App lock hardening, `FLAG_SECURE`, backup exclusion tests, redaction tests, consent log, `PRIVACY.md`, `PLAY_STORE_NOTES.md` (Data Safety draft, Health apps declaration answers, Health Connect declaration, exact-alarm and permission justifications, store listing text that never claims diagnosis), `docs/INTENDED_USE.md`, `docs/COUNSEL_QUESTIONS.md`, `docs/NETWORK_CALLS.md`, and `docs/DPDP_CHECKLIST.md`. Dependency audit and licence report.

## Phase 9 — Hardening and release prep
Accessibility audit with TalkBack notes; profile the Realistic look on the slowest available device or emulator profile; battery check for the background sync and alarms; a device test matrix in `docs/MANUAL_TEST_PLAN.md` (Redmi/Xiaomi, Samsung, Realme/Oppo/Vivo, Pixel, Android 8 to latest); release build; `ANDROID_BUILD.md` with exact steps for a debug APK and a signed AAB; the closed-testing plan; and a store-asset brief. If an emulator or device is unavailable here, build, run unit, widget and golden tests, and list precisely what must be tested by hand.

# 8. APP SIZE BUDGET (target landing zone 60 to 70 MB)

Size is an outcome, not a goal. Ship real content and report the honest number; **never add dummy files to inflate the size.** If the honest total is under 60 MB, that is fine: report it and stop. Play shows a warning for downloads over 200 MB on mobile data, so keep the **base install at or below about 70 MB**, hard-fail the CI size gate above 80 MB, and put optional heavy content in on-demand asset packs (Play limits: base module 500 MB; install-time packs plus modules 4 GB; on-demand packs up to 30 GB; all compressed).

Expected contents (all estimates, to be measured and replaced by real numbers in `docs/SIZE_REPORT.md`): Flutter engine and framework plus Kotlin code, roughly 12 to 20 MB; bundled on-device text recognition models for English and Devanagari, roughly 10 to 20 MB; fonts (Latin, Devanagari, seven-segment), 3 to 6 MB; Rive and Lottie animations and shader files, 3 to 6 MB; the Indian food database with original illustrations, 8 to 15 MB; Learn hub illustrations and content, 5 to 10 MB; lab reference data, localisation and icons, 2 to 4 MB. On-demand packs: additional Indian languages and extra illustrated lessons. Every asset must be original, public domain or licensed, listed in `LICENSES.md`; do not download copyrighted photos or fonts.

# 9. DEFINITION OF DONE

- `flutter analyze`, all tests, goldens and the banned-term lint pass; domain coverage 90% or more.
- No network call exists unless listed in `docs/NETWORK_CALLS.md`; release logs contain no health data; backups exclude the database.
- Each flavor builds. `store` runs with ComplianceProfile.standard and contains no demo-seed code.
- Both looks and both themes verified at 360, 412 and tablet widths with screenshots in `docs/screens/`; text at 200% scale does not clip.
- Reminders work with the app closed, after reboot and after a timezone change on a real device or are listed as pending manual tests.
- `docs/SIZE_REPORT.md` shows the measured base download size per ABI.
- Final report: what was built, how to run it, screenshots list, known limitations, measured size, and the short list of things only I can do.

# 10. WHAT ONLY I CAN DO (list these at the end, with exact steps)

Create the Google Play developer account and complete identity verification; run a closed test with at least 12 testers for 14 continuous days before applying for production access (current Play rule for new personal accounts); complete the Health apps declaration, Data Safety form and Health Connect declaration; create and safely store the release signing key; test with a real Bluetooth BP monitor and glucometer; have a clinician review `thresholds.dart` and all card and message wording; have an Indian regulatory or health-law professional answer `docs/COUNSEL_QUESTIONS.md`; have a native Hindi speaker review the translations.

---END OF PROMPT---
