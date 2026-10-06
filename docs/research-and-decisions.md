# Research, decisions and sources (October 2026)

> Extracted from the working spec. Store pages and rules were read on 4 Oct 2026; install counts are Play Store ranges. See the list at the end for what was not verified.


The full build prompt for v3 is [claude-code-build-prompt-v3.md](claude-code-build-prompt-v3.md). Where it differs from the earlier [v2 spec](product-spec-v2.md), v3 wins.

### 8.1 Competitors checked on their store pages
| App | Play Store | What it centres on | Notes |
|---|---|---|---|
| BeatO | 1M+ installs, 4.6 stars, 45.6K reviews | Own glucometer pairing, doctor consults, AI chatbot, vitals, Health Connect | Data Safety: no data shared |
| Omron Connect | 1M+, 4.5, about 116K reviews | Sync with Omron BP monitors, scales, ECG; trends; premium insights | Reviews: daily logouts, disconnections, crashes |
| mySugr | 500K+, 4.6, about 139K reviews | Glucose logbook, Accu-Chek sync, HbA1c estimate; PRO adds reports | Reviewers ask for dark theme; lists third-party sharing |
| sugar.fit | 100K+, 4.5, 16K reviews | Paid CGM and coaching programme (physical exam needed) | Lists sharing location and personal data |
| Fitterfly | 100K+, 4.2 (Play); 4.6 on App Store, 183.2 MB | CGM and glucometer sync, AI nutrition tracking, AI coach, 100+ videos, voice meal logging | Programmes are paid |
Not checked: HealthifyMe, Dario, Welldoc, Twin Health, Apple Health and Samsung Health features. Review these before claiming a market gap in marketing copy.

### 8.2 What the research means for the product
- **Market:** about 101 million people with diabetes, 136 million with prediabetes, 315 million with hypertension (ICMR-INDIAB, Lancet Diabetes & Endocrinology, 2023).
- **Gap:** not basic tracking, but a free, no-hardware, no-login, on-device-private app with a day planner, reliable voice reminders in Indian languages, a doctor-visit pack and family mode.
- **Retention:** one industry summary puts day-30 retention for fitness apps at about 3 to 4% (top apps about 25%) and says 70% of users stop within 100 days. Main causes: manual-entry fatigue, slow onboarding, generic content, privacy worries. Design response in v3: Health Connect and Bluetooth passive capture, 60-second onboarding, adaptive notification timing, a notification budget, widgets and voice logging.
- **Stack change:** v3 uses Flutter with Kotlin services instead of a web view wrapper, because the realistic gauges, BLE, Health Connect, exact alarms with voice and widgets need native performance on cheap phones. The old React code is reference only. The HTML preview remains a design reference.

### 8.3 App size (you asked for 60 to 70 MB)
Size should come from real content, not padding. A bigger download lowers installs on low-end phones and mobile data (Play warns above 200 MB), but 60 to 70 MB is easy to reach honestly with bundled on-device text recognition models, Indian-language fonts, animations, an offline Indian food database with illustrations and the Learn hub. Google Play limits: base module 500 MB, install-time packs 4 GB, on-demand packs 30 GB, all compressed. v3 sets a CI gate (hard fail above 80 MB) and puts optional heavy content in on-demand packs. For reference, Fitterfly's iOS app is 183 MB.

### 8.4 Regulation and store rules that shape the design
- **CDSCO software guidance (21 Jul 2026, under MDR 2017):** software for general wellness and routine parameter tracking is outside the rules only if it has no medical purpose. A wellness label cannot neutralise diagnostic, monitoring, prediction or treatment claims; the example given is an app interpreting heart-rate patterns to flag atrial fibrillation risk. **This means a disclaimer alone does not remove regulatory exposure.** The v3 prompt adds a ComplianceProfile (`full`, `standard`, `trackingOnly`) so an Indian health-law adviser can reduce interpretation (tier labels, lab-range labels, IDRS score) without rewriting the app. The `store` build defaults to `standard`, which hides the IDRS score. Get the adviser's written view before the Play Store launch.
- **DPDP Rules 2025 (notified 13 Nov 2025):** notice, consent, breach notification and security duties start 13 Nov 2026; data protection officer and grievance duties 13 May 2027; verifiable parental consent for under-18s (the app is 18+ only). On-device-only data may reduce exposure, but counsel must confirm.
- **Google Play:** Health apps declaration, public privacy policy, Health Connect data treated as sensitive, non-regulated health apps must say they are not a medical device and do not diagnose, treat, cure or prevent any condition; device-sensor measurements must disclose supported devices; apps must remind users to consult a professional.
- **Apple App Review 1.4.1 and 5.1.3:** apps that claim to measure blood pressure, glucose, temperature or blood oxygen using only the phone's sensors are not permitted; apps should remind users to check with a doctor; health data may not be used for advertising or stored in iCloud. This is why camera-based BP and glucose are excluded.
- **Android alarms:** `USE_EXACT_ALARM` is for calendar and alarm-clock apps only under Play policy; use `SCHEDULE_EXACT_ALARM`, which is denied by default on new installs targeting Android 13+.
- **Individual Play developer accounts:** a closed test with at least 12 testers opted in for 14 continuous days is required before applying for production access (review usually takes up to 7 days).
- **Gemini Nano via ML Kit GenAI:** runs offline on supported phones (the page does not list which), and the developer is solely responsible for output safety. v3 keeps it off by default and never lets it write health statements.

### 8.5 Open decisions for you
1. Is a legal opinion on intended use (and the ComplianceProfile default) done before launch, or do you launch a limited closed test first?
2. Privacy-first means no analytics, so you cannot see where users drop off; Play Console gives installs, uninstalls and crash data only. Do you want an opt-in, anonymous counter later?
3. Business model (free core with optional paid family profiles, backup or reports, versus fully free) is not decided and is not in the build prompt.
4. Whether to add iOS after Android; Apple needs a paid developer account and has the stricter health rules above.

### 8.6 Sources
- BeatO: https://play.google.com/store/apps/details?id=com.healtharx.beato
- sugar.fit: https://play.google.com/store/apps/details?id=fit.sugar.android
- Fitterfly (Play): https://play.google.com/store/apps/details?id=com.FitterFly.Wellness ; (App Store): https://apps.apple.com/in/app/fitterfly-metabolic-health/id1492561575
- mySugr: https://play.google.com/store/apps/details?id=com.mysugr.android.companion
- Omron Connect: https://play.google.com/store/apps/details?id=com.omronhealthcare.omronconnect
- CDSCO medical device software guidance 2026: https://www.pureglobal.com/news/india-cdsco-medical-device-software-guidance-2026
- DPDP Rules 2025 timeline: https://ca.hoganlovells.com/en/publications/indias-digital-personal-data-protection-act-2023-brought-into-force-
- Google Play app size limits: https://support.google.com/googleplay/android-developer/answer/9302563
- Google Play health content policy: https://support.google.com/googleplay/android-developer/answer/16679511
- Google Play new personal account testing rule: https://support.google.com/googleplay/android-developer/answer/14151465
- Apple App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Health Connect data types: https://developer.android.com/health-and-fitness/health-connect/data-types
- Gemini Nano / ML Kit GenAI: https://developer.android.com/ai/aicore
- Retention figures (secondary source): https://sahha.ai/blog/health-app-churn-retention/
- ICMR-INDIAB prevalence (news report of the Lancet study): https://morungexpress.com/315-mn-indians-suffer-from-hypertension-101-mn-are-diabetic-icmr-study
- Camera pulse study (FibriCheck validation, hospital setting only): https://pmc.ncbi.nlm.nih.gov/articles/PMC5591405
- Not verified this session: Bluetooth SIG service numbers (0x1810, 0x1808) and the rule allowing the voice service to start from an exact-alarm receiver (the prompt tells Claude Code to verify), the Android foreground-service type for Live Walk, exact sizes of ML Kit models and Flutter output (the prompt asks for measured numbers).
