# Final Assessment — Backend Development

**Weight:** 30% of final grade
**Duration:** 2 weeks
**Mode:** Individual project
**Stack (mandatory):** Node.js, Express.js, MySQL (with parameterized queries only), Redis

---

## Scenario — "MediBook" Clinic Appointment System

A community clinic wants a backend service for booking doctor appointments. Doctors see patients in fixed 30-minute slots during working hours (09:00–17:00), each doctor can handle at most **12 appointments per day**, and the clinic administrator needs reports. You will design the database, build the API, and handle the operational concerns (background jobs, logging, testing).

**Entity limit:** Your database must contain **exactly 3 entities**:

| Entity | Purpose |
|---|---|
| `patients` | People who book appointments |
| `doctors` | Doctors with a specialty, consultation fee, and room |
| `appointments` | A patient's booking with a doctor at a specific date/time (links the other two) |

You may add columns as needed, but no additional tables (lookup/locale JSON files don't count).

---

## Part A — Database Design & Normalization

The clinic currently tracks everything in this single spreadsheet:

| appt_ref | patient_name | patient_email | patient_phone | doctor_name | specialty | consultation_fee | room | appt_datetime | amount_paid | booked_on |
|---|---|---|---|---|---|---|---|---|---|---|
| AP-001 | Ama Mensah | ama@mail.com | 0244-111-222 | Dr. Boateng | Dermatology | 150.00 | R-12 | 2026-08-10 09:00 | 150.00 | 2026-07-01 |
| AP-002 | Kojo Owusu | kojo@mail.com | 0555-333-444 | Dr. Boateng | Dermatology | 150.00 | R-12 | 2026-08-10 09:30 | 120.00 | 2026-07-02 |
| AP-003 | Ama Mensah | ama@mail.com | 0244-111-222 | Dr. Adjei | Paediatrics | 100.00 | R-03 | 2026-08-15 10:00 | 100.00 | 2026-07-03 |

Study the spreadsheet, then produce a **database schema diagram** of your design in fully normalized form (up to 3NF). The diagram must show **everything**:

- all 3 entities with every column and its MySQL data type,
- primary keys and foreign keys, clearly marked,
- the relationships between the entities and their cardinality (1:N, etc.),
- every constraint that enforces a business rule (`NOT NULL`, `UNIQUE`, `CHECK`) — in particular, it must be visible from the diagram that *a doctor cannot have two appointments at the same date/time*.

Use any tool you like (dbdiagram.io, draw.io) and include it in your repository as `schema-diagram` (image or PDF).

## Part B — Database Implementation & Project Setup

1. **Table creation.** Translate your Part A diagram into a MySQL DDL script, `database/schema.sql`, that creates the database and the 3 tables — data types, primary keys, foreign keys, and every constraint shown on the diagram. The script must run cleanly on a fresh MySQL server. Also provide `database/seed.sql` with realistic sample data: at least 3 patients, 3 doctors (including one with no appointments), and 8 appointments (including one doctor-day at or near the 12-slot daily cap).

2. **Project structure.** Set up the Express project in a **models / controllers / routes** structure:

```
├── database/          schema.sql and seed.sql (Part B.1)
└── src/
    ├── models/        all application SQL lives here — nowhere else
    ├── controllers/   business logic; calls model functions, never the DB directly
    └── routes/        maps URLs and HTTP methods to controllers
```

Every SQL statement in the application must use **parameterized queries** (`?` placeholders via `mysql2`). SQL found in a controller or route file loses marks.

## Part C — Controllers & Endpoints

Implement endpoints for the operations below. **Designing the API is part of the assessment**: you choose the URL paths, the HTTP methods, and how the endpoints are grouped into controllers and routes — your choices should follow REST conventions and will be marked. The business rules listed are **requirements** — *how* you satisfy them is your design decision, and you will be asked to defend it at the viva.

1. **Basic CRUD** — standard **C**reate, **R**ead (one and all), **U**pdate, and **D**elete operations for patients and for doctors, with sensible validation. (Appointments get their create/delete behaviour from the two operations below — no separate CRUD needed there.)

2. **Book an appointment**. Rules that must always hold:
   - Appointments can only be booked in the future, within working hours (09:00–17:00), on a 30-minute slot boundary.
   - A doctor never exceeds 12 appointments in a day.
   - Two patients can never hold the same doctor slot — even when both requests arrive at the same instant.

3. **Cancel an appointment**. Rules:
   - The refund is computed by the server from the notice given: more than 48 hours → 100%; 24–48 hours → 50%; less than 24 hours → 0%. The response states the refund amount.
   - The cancellation and its refund record succeed or fail together.
   - A cancelled slot becomes bookable by another patient, while the cancellation record is kept for refund history.
   - A cancellation notifies the waitlist (Part E).

4. **Doctor statistics** — for a given doctor: revenue collected, upcoming appointment count, appointments per day for the next 7 days, and utilization % against the 12-slot daily cap.

Operations 5 and 6 are where your **advanced SQL** is assessed: the results must be produced by the SQL in your models (joins, aggregates, etc) — not fetched raw and computed in JavaScript.

5. **Doctors report** — every doctor's total appointments and revenue collected, including doctors with zero appointments.
6. **Busy days report** — every (doctor, date) where more than 80% of the 12-slot daily capacity is booked.

### General requirements (all endpoints)

Input validation with meaningful `400` messages; correct status codes (`201`, `404`, `409`, `422`); a central error-handling middleware.

## Part D — Node.js `fs` Module

**Audit log:** every successful booking and cancellation must be appended as a JSON line to `logs/audit.log` using the `fs` module (async, non-blocking — no `*Sync` calls inside request handlers). The logger must create the `logs/` directory on first run if it doesn't exist, and a logging failure must **not** fail the user's request.

## Part E — Queues & Redis

1. **Waitlist queue:** when the requested slot (or the doctor's whole day) is fully booked, offer the patient a waitlist spot — push them onto a queue (Redis list, or Bull/BullMQ). When a cancellation frees a slot, a **worker process** (separate from the web server) pops the first waitlisted patient and books them in. Think through: what happens if the promoted booking itself fails? The patient must not simply vanish from both the queue and the appointments table.
2. **Notification jobs:** every confirmed appointment enqueues a "confirmation SMS/email" job; the worker processes it by writing the message to the audit log (no real sending needed).

## Part F — Unit Testing

Using Jest, write **at least 6 meaningful unit tests**:

- The refund calculator from Part C.3 — cover all three tiers **and both boundaries** (exactly 24 hours, exactly 48 hours). Boundary behaviour must match your documented policy.
- One validation function of your choice — e.g., the slot-alignment/working-hours check — including at least one edge case (17:00 exactly? 09:15? a malformed date?).

Tests must not require a live MySQL or Redis connection — that's the point of extracting pure functions. `npm test` must run them.

## Part G — i18n & l10n: Course Feedback Page

Create a **simple HTML page** (served by your Express app at `GET /feedback`) containing a short course-feedback form with **exactly 5 essay questions** (each answered in a `<textarea>`):

1. Which topic in this course did you find most challenging, and why?
2. Describe one concept you understood only after the practical assignments. What made it click?
3. How has your approach to designing databases or APIs changed since the start of the trimester?
4. What would you improve about how this course was taught or assessed?
5. How do you plan to apply what you learned in this course going forward?

**i18n/l10n requirement (keep it simple):** the page title, instructions, and all 5 questions must be loaded from locale files (e.g., `locales/en.json` and `locales/fr.json` — English plus one other language of your choice). Visiting `/feedback?lang=fr` shows the translated page; the default is English, and an unsupported `lang` value must fall back to English rather than crash. Display today's date on the page formatted for the active locale (e.g., using `Intl.DateTimeFormat`).

---

## Submission Requirements

- Git repository (with meaningful commit history — a single "final commit" will be questioned) containing:
  - `schema-diagram` (image or PDF), `schema.sql`, and `seed.sql` (as specified in Part B)
  - Application source, worker process, tests, locale files
  - `README.md` with: setup steps, `.env.example`, and the short written justifications requested in Parts C, D, and E
- The app must start with `npm install && npm start` (plus documented MySQL/Redis setup) and tests with `npm test`.
- A short **viva/demo (10 minutes)** walkthrough.

**Automatic penalties:** SQL injection vector anywhere in the app (−15), secrets committed to the repo (−5), app does not start per README instructions (−10).







# MediBook API Rubric

**Final Assessment — Backend Development · 100 pts · scaled to 30% of final grade**

| Criteria | Exemplary | Proficient | Developing | Needs Improvement | Pts |
|---|---|---|---|---|---|
| **Database Design & Normalization** *(Part A)* | Diagram is fully normalized to 3NF. All 3 entities appear with every column and its MySQL data type; PKs and FKs clearly marked; cardinality correct; business-rule constraints (`NOT NULL`, `UNIQUE`, `CHECK`) visible — including the rule that a doctor cannot hold two appointments at the same date/time. **15 to >11.25 pts** | Normalized design with minor omissions — a missing data type, an unmarked key, or one constraint absent; relationships and cardinality mostly correct. **11.25 to >7.5 pts** | Partial normalization (visible redundancy remains) or the diagram omits several types, keys, or constraints; cardinality unclear. **7.5 to >3.75 pts** | Diagram missing, illegible, or essentially the spreadsheet redrawn; not normalized. **3.75 to >0 pts** | **/15** |
| **Database Implementation & Project Structure** *(Part B)* | `database/schema.sql` runs cleanly on fresh MySQL and matches the diagram exactly; sensible types (`DECIMAL` for money, `DATETIME` for slots); all constraints implemented; `seed.sql` meets the spec (incl. the zero-appointment doctor and the near-cap day). Clean models / controllers / routes structure; every query parameterized; SQL exists only in models. **15 to >11.25 pts** | Scripts run with small gaps — a weak type, one missing constraint, or seed data missing one required case; structure correct with an isolated SQL leak outside models. **11.25 to >7.5 pts** | Script errors, missing FKs, or schema deviating from the diagram; folder structure present but SQL scattered through controllers or routes. **7.5 to >3.75 pts** | `schema.sql` does not run or contradicts the diagram; no meaningful project structure. **3.75 to >0 pts** | **/15** |
| **Core API: CRUD & REST Design** *(Part C.1, general requirements)* | All patient and doctor CRUD endpoints work with sensible validation; URL paths, HTTP methods, and controller/route grouping follow REST conventions; unknown ids return 404; meaningful 400 messages; central error-handling middleware in place. **8 to >6 pts** | Most endpoints work with minor gaps (a missing 404, a small bug, one non-RESTful route choice). **6 to >4 pts** | Several endpoints missing or buggy; validation partial; error handling ad hoc. **4 to >2 pts** | Major endpoints missing or broken; no REST discipline; no central error handling. **2 to >0 pts** | **/8** |
| **Business Rules & Critical Thinking** *(Parts C.2–C.3: booking & cancellation)* | All booking rules enforced — future-only, working hours, 30-minute boundary, 12-per-day cap, and no shared slot **even under simultaneous requests**. Refund tiers computed server-side with both boundaries defined; cancellation and refund record succeed or fail together; a cancelled slot becomes rebookable while its record is kept; waitlist notified; correct 409/422 usage. Design decisions defended convincingly at the viva. **13 to >9.75 pts** | Most rules enforced; concurrency only partially handled (e.g., the daily-cap race remains) or one refund boundary undefined; minor status-code slips. **9.75 to >6.5 pts** | Only 2–3 rules enforced; refund computed incorrectly or trusted from the client; cancelled slots stay blocked forever or the record is lost. **6.5 to >3.25 pts** | 0–1 rules enforced; little evidence of deliberate design decisions. **3.25 to >0 pts** | **/13** |
| **Statistics & Advanced SQL Reports** *(Parts C.4–C.6)* | Doctor statistics (revenue, upcoming count, per-day counts for 7 days, utilization % vs the 12-slot cap) are correct. Both reports are produced by SQL in the models (joins, aggregates); doctors with zero appointments appear in the doctors report; the busy-days threshold is computed correctly against capacity. **9 to >6.75 pts** | Reports mostly correct — one aggregate slip, or zero-appointment doctors missing, or a minor piece computed in JavaScript. **6.75 to >4.5 pts** | Results largely computed in JavaScript instead of SQL, or several figures wrong. **4.5 to >2.25 pts** | Statistics/report operations missing or broken. **2.25 to >0 pts** | **/9** |
| **`fs` Module: Audit Log** *(Part D)* | Every successful booking and cancellation appended as a JSON line to `logs/audit.log` using async `fs` calls; `logs/` auto-created on first run; a logging failure never fails the user's request; no `*Sync` calls inside request handlers. **10 to >7.5 pts** | Log works with minor slips — an occasional missed event type, or sync usage outside the request path. **7.5 to >5 pts** | Logging partial; `*Sync` calls inside handlers; or a log failure breaks the request. **5 to >2.5 pts** | Audit log absent or broken. **2.5 to >0 pts** | **/10** |
| **Queues & Redis** *(Part E)* | Fully-booked slot/day leads to a waitlist offer via a Redis-backed queue; a **separate worker process** books the first waitlisted patient when a cancellation frees a slot; failure of that promoted booking is handled so the patient is never lost from both the queue and the table; confirmation jobs enqueued per appointment and processed to the audit log. **15 to >11.25 pts** | Waitlist and worker function, but the promoted-booking failure case is unhandled, or the worker runs inside the web server process; notifications work. **11.25 to >7.5 pts** | A queue exists but promotion is broken or manual; notification jobs missing. **7.5 to >3.75 pts** | No working Redis/queue usage. **3.75 to >0 pts** | **/15** |
| **Unit Testing** *(Part F)* | At least 6 meaningful tests: refund calculator covers all three tiers **and both exact boundaries** (24h, 48h) using fixed dates; a validation function tested including an edge case; `npm test` runs green with no MySQL or Redis running. **10 to >7.5 pts** | 6 tests present but one boundary missing or an assertion weak. **7.5 to >5 pts** | Fewer than 6 tests, happy-path only, or tests require live services. **5 to >2.5 pts** | Tests absent or failing. **2.5 to >0 pts** | **/10** |
| **i18n & l10n Feedback Page** *(Part G)* | `/feedback` serves the 5 essay questions as textareas with all visible text loaded from locale files; both languages work via the lang switch; unsupported values fall back to English without crashing; today's date is formatted for the active locale. **5 to >3.75 pts** | Both locales work but a string is hard-coded or the date is not localized. **3.75 to >2.5 pts** | Only English works, or translations are hard-coded into the HTML instead of locale files. **2.5 to >1.25 pts** | Page missing or broken. **1.25 to >0 pts** | **/5** |

**Total: /100**

---

## Automatic penalties (applied after scoring)

| Condition | Effect |
|---|---|
| SQL injection vector anywhere in the app | −15 |
| Secrets (real passwords, `.env`) committed to the repo | −5 |
| App does not start following the README instructions | −10 |
| Cannot explain own code at the viva | Academic integrity referral; affected criteria re-scored at Developing or below |
