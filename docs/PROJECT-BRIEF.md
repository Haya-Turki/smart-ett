# ETT Monitoring System — نظام مراقبة ETT

Project brief, decision history, and clinical logic.
Written to be handed to **Claude Design** (or any designer/developer) to continue the work.

---

## 1. What this is

A mobile-first (now also web) app for **monitoring endotracheal tubes (ETT) in hospital critical-care units**.
The nurse opens the app, picks a unit, sees which intubated patients need attention, opens a patient, and gets:

- current tube data (size, depth, days intubated)
- **cuff pressure** with the accepted range
- **secretion assessment** (amount, colour, texture, suction history)
- a **predicted probability of infection**, derived from those inputs
- a **ranked list of likely pathogens** (before culture results arrive)
- a **checklist of required actions** the nurse ticks off and records

Built for a competition demo (مسابقة). All patient data is fictional.

**Positioning that must not change:** the prediction is *decision support*, not diagnosis. Culture and sensitivity remain the reference. Every screen that shows a prediction carries that line.

## 2. Users and context

- **Primary user:** ICU nurse on shift, on a phone, hands busy, needs the answer in one glance.
- **Secondary:** charge nurse / on-call physician receiving escalations.
- **Language:** Arabic, RTL. Clinical terms stay in English inside Arabic text (`cuff`, `ETT`, `Adult ICU`, organism names in Latin).
- **Accounts** are created by system administration — there is no public sign-up.

## 3. Approved screens

| # | Screen | Purpose |
|---|--------|---------|
| 1 | تسجيل الدخول | Facility select + **hospital ID** + password, or facility SSO. No social login, no sign-up link. |
| 2 | الأقسام | All units with patient counts and critical/warning badges. |
| 3 | قائمة مرضى القسم | Patients of the chosen unit, critical first, with search and status filters. |
| 4 | تفاصيل المريض | Tube data, cuff, secretions, infection probability, likely pathogens, required actions, action log. |

Flow: login → units → patients of that unit → patient detail → back at any point.

### Patient card / row shows
المقاس · العمق · مدة التنبيب · آخر تقييم, plus **status chips** that appear only when something is out of range
(`cuff مرتفع`, `cuff منخفض`, `إفرازات غزيرة`, `اشتباه عدوى`, `تشبّع منخفض`, `تقييم متأخر`, `زراعة مرسلة`).
Deliberate decision: the card is **not** tied to one measurement — chips are generated from whatever rules exist, so new parameters can be added without redesigning the card.

## 4. Clinical logic implemented (all thresholds are placeholders — confirm against hospital protocol)

**Cuff pressure** (cmH₂O), accepted range 20–30:
- ≥ 32 → critical `cuff مرتفع`; 31 → warning
- ≤ 17 → critical `cuff منخفض`; 18–19 → warning
- Neonatal (NICU) tubes are **uncuffed** → the cuff panel shows "لا ينطبق" and status is driven by secretions and saturation.

**Infection probability** (0–92%), sum of:
- base 5
- amount: قليلة 0 · متوسطة 12 · غزيرة 25
- colour: صافية 0 · بيضاء 6 · أصفر 12 · أصفر مخضر 20
- temperature: ≥ 38 → 15 · ≥ 37.5 → 7
- days intubated: > 10 → 10 · > 5 → 5
- rising secretion trend → 3

≥ 70 critical · 45–69 warning · below that low.

**Likely pathogens** — fixed target list, weighted per unit (an antibiogram stands in for the model):
`Pseudomonas aeruginosa`, `Klebsiella pneumoniae`, `Acinetobacter baumannii`, `Staphylococcus aureus / MRSA`, `Enterobacter spp.`, `Escherichia coli`, `Proteus spp.`
Adult ICU weights favour Pseudomonas; NICU/PICU weights favour Staph and Klebsiella.

**Required actions** are generated from the patient's own state, not stored:
شفط البلغم فوراً · معايرة أو نفخ الكفة · أخذ عينة قشع للزراعة · مراجعة إعدادات التهوية · إبلاغ الطبيب المناوب · تسجيل التقييم المتأخر · مناقشة الفطام أو فغر الرغامى — falling back to "متابعة روتينية كل 4 ساعات" when nothing is out of range.

**Executing actions changes the data**, which is the core demo moment:
- suction → secretion amount drops, colour improves, infection score falls
- cuff calibration → cuff resets to 24
- culture sent → pathogen panel switches to "الزراعة قيد الانتظار"
- notify → escalation recorded and the button locks
- any recorded action resets "آخر تقييم" to now and writes a line in the action log
- the patient's status, the list row, and the unit's badge counters all recompute

Demo path that lands well: عبدالله الشمري in Adult ICU → tick suction + calibration → infection 78% → 42%, cuff 32 → 24, status حرج → تنبيه, and the unit's critical counter drops by one.

## 5. Data model (per patient)

```
name, mrn, bed, size, depth, days,
cuff (null = uncuffed), amount, color, texture,
temp, spo2, mins (minutes since last assessment),
suctions, trend, cultureSent, escalated, log[], done{}
```

Everything else — status, chips, infection score, pathogen ranking, task list — is **computed**, never stored. Keep it that way: it is what makes the prototype respond to edits.

Units included: Adult ICU (12), PICU (7), NICU (9), CCU (4), ER (3), OR/PACU (2) — 37 fictional patients.

## 6. Design direction

- Deliberately clinical and calm, not consumer-app colourful. Deep slate ink `#12202b`, paper `#f4f7f8`, white surfaces, hairline borders `#dce4e8`.
- Status colours only ever mean status: critical `#a92a21`, warning `#8f5b06`, stable `#12664f`, accent `#1c6a86`.
- Type: IBM Plex Sans Arabic, weights 400/500/600. Sentence case, no all-caps.
- Numbers are large and unlabelled-heavy; supporting text small and muted.
- Mobile: single column, card per patient, bottom tab bar.
- Web: left-of-content sidebar (RTL: right), patients as a table, detail as two columns.
- Navigation: **one back button at the top of every inner screen**, plus "المريض التالي" at the bottom of the patient page. No breadcrumb, no extra button row — this was tried and rejected as noisy.

## 7. Decision history (what changed and why)

1. Started as a generic login screen → changed to **hospital ID** instead of email.
2. Product renamed from a medical asset system to **نظام مراقبة ETT**.
3. Removed Google/Apple sign-in and the sign-up link — accounts come from administration; added facility SSO.
4. Added the **units screen** between login and the patient list, mirroring how hospital systems are organised.
5. "الكفة" renamed to **cuff** throughout.
6. Card metrics realigned into a fixed grid after Arabic/number bidi broke the alignment; durations written with digits for consistency.
7. Cuff removed from the card as a fixed column — replaced by the **generic chip system**, because cuff is not the only tracked parameter.
8. "احمرار الشفة" and "تنبيب مطوّل" dropped; **اشتباه عدوى** added. "cuff منخفض" added for سارة.
9. Length-of-stay added, then removed; intubation durations lengthened to 6–17 days.
10. Secretion **odour** field removed.
11. Infection prediction expanded from a single score to a **ranked pathogen list** over a fixed 7-organism target list.
12. Recommendations turned into a **required-action checklist**, starting with suctioning.
13. Ported from phone mockups to a **working web app** with real data for every unit and patient.
14. Navigation simplified to a single top back button after the bottom button row was rejected; browser history API removed because it fails when the file is opened locally.

## 8. Open questions to resolve before the next build

- **Where do cuff and secretion values come from?** Currently a nurse enters them (manometer reading + observation at suction). If the project includes a sensor on the tube, the cuff panel should become a live reading with a trend chart — a much stronger story for the judges.
- What is the hospital's accepted cuff range and reassessment interval? Both are hard-coded placeholders.
- What data would train the pathogen model — unit antibiogram, historical cultures, or literature priors? Judges usually ask this first.
- Should a rule flag intubation beyond 14 days for a tracheostomy discussion? Currently only shown as a chip.
- Roles and permissions: nurse vs charge nurse vs physician.

## 9. Backlog / next screens

1. **تسجيل تقييم جديد** as a full screen on mobile (exists as a modal on web) — shows how fast data entry is.
2. Cuff trend chart over the last 24 hours.
3. Post-culture state: compare what was predicted with what actually grew.
4. Unit dashboard: VAP rate, average cuff compliance, overdue assessments.
5. Notifications and shift handover summary.
6. English (LTR) version of the interface.

## 10. Prompt to continue in Claude Design

> This is an Arabic (RTL) ICU app for monitoring endotracheal tubes, called نظام مراقبة ETT. Read PROJECT-BRIEF.md for the intent, clinical rules, and design direction, and open app/index.html to see the working prototype. Keep the calm clinical palette, IBM Plex Sans Arabic, the single top back button, and the rule that status chips are generated from the data rather than hard-coded. I want to improve the visual design of the four approved screens and add [the screen you want] without changing the clinical logic.

---

*All patients, MRNs, and values in this project are fictional. Thresholds and the prediction model are illustrative and must be validated against hospital protocol before any real use.*
