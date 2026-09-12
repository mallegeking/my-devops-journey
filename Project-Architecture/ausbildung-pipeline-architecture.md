# Ausbildung Application Pipeline

Semi-automated sourcing, application and reply-tracking system for one candidate applying to Pflege Ausbildungsplätze in Germany.

Status: planning
Owner: Sticks
Candidate: one (friend, Zimbabwe, B1 held, B2 in progress)
Target Ausbildungsstart: April 2027
Application window: September to November 2026

---

## 1. Goal and non-goals

### Goal

Get the candidate to interview stage at a German Pflege employer, without Sticks doing 100 applications by hand.

### Non-goals

These stay out of scope permanently. They are the candidate's responsibility or a later project.

- Certificate recognition (anabin, ZAB Zeugnisbewertung)
- Visa application, Sperrkonto, Lebensunterhaltsnachweis
- Language learning
- Anything that could be read as commercial Arbeitsvermittlung
- Multi-candidate support

The system is a personal productivity tool, not a product. Design decisions should stay boring.

---

## 2. Operating principle

**Nothing leaves the system without a human clicking approve.**

The AI drafts and classifies. Sticks decides. The candidate's own mailbox sends. This is not a compliance nicety, it is the thing that keeps the output quality high enough to be worth sending at all.

---

## 3. Architecture

```
                      ┌──────────────────────┐
                      │  BA Jobsuche API     │
                      │  (Ausbildungsstellen)│
                      └──────────┬───────────┘
                                 │ daily
                      ┌──────────▼───────────┐
                      │  Ingest worker       │  Python, containerised
                      │  fetch → normalise   │
                      └──────────┬───────────┘
                                 │
                      ┌──────────▼───────────┐
                      │  PocketBase          │  postings, employers,
                      │  datastore + admin   │  applications, messages,
                      │  UI = approval queue │  candidate
                      └────┬────────┬────────┘
                           │        │
              ┌────────────▼──┐  ┌──▼──────────────┐
              │ Scoring job   │  │ Generation job  │
              │ rules-based   │  │ Claude API      │
              └───────────────┘  └──┬──────────────┘
                                    │ draft
                      ┌─────────────▼────────┐
                      │  APPROVAL GATE       │  Sticks reviews,
                      │  (human)             │  edits, approves
                      └─────────────┬────────┘
                                    │
                      ┌─────────────▼────────┐
                      │  Sender              │  SMTP, candidate's
                      │  throttled 5-8/day   │  own mailbox
                      └─────────────┬────────┘
                                    │
                      ┌─────────────▼────────┐
                      │  Reply poller        │  IMAP, every 2h
                      │  + classifier        │  Claude API
                      └─────────────┬────────┘
                                    │
                      ┌─────────────▼────────┐
                      │  Alerting            │  Telegram / ntfy
                      │  interview → notify  │
                      └──────────────────────┘

Orchestration: n8n (schedules, glue, retries)
Runtime: k3s homelab, deployed via Argo CD from a repo
```

### Component choices and why

| Component | Choice | Reason |
|---|---|---|
| Job source | BA Jobsuche REST API | Largest Ausbildung dataset in Germany, no scraping needed |
| Datastore | PocketBase | Already known, and its admin UI is the approval queue for free. Swap to Postgres only if this outgrows it |
| Orchestration | n8n | Scheduling, retries, email nodes. Also the practice he wants |
| Generation | Claude API | Anschreiben drafting and reply classification |
| Sending | SMTP via candidate's existing mailbox | Deliverability. A fresh domain lands in spam |
| Notifications | Telegram bot or ntfy | Approval prompts and interview alerts on phone |
| Deployment | k3s + Argo CD | Consistent with existing homelab GitOps |

---

## 4. Data model

### `candidate`
Single row. The facts every Anschreiben draws from.

```
name, dob, nationality, current_location
language_level, language_cert_date, b2_expected_date
school_qualifications, anabin_status, zab_status
work_experience[], motivation_notes
earliest_start_date, mobility (bundesland preferences)
```

### `employers`
Deduplicated across postings. This is where the real intelligence accumulates.

```
id, name, traeger_type (kette | kirchlich | kommunal | privat_klein)
size_estimate, website, contact_email, contact_person
international_experience (bool | unknown)
contacted_at, response_type, notes
blacklisted (bool), blacklist_reason
```

### `postings`
```
refnr (PK), employer_id, title, beruf_code
location, plz, lat, lon, bundesland
start_date, application_url, contact_email
raw_json, first_seen, last_seen
score, score_breakdown (json), lane (email | portal | skip)
status (new | scored | drafted | applied | dead)
```

### `applications`
```
id, posting_refnr, employer_id
anschreiben_draft, anschreiben_final
status (draft | approved | sent | replied | interview | rejected | ghosted)
drafted_at, approved_at, sent_at
followup_sent_at, thread_message_id
```

### `messages`
```
id, application_id, direction (in | out)
subject, body, received_at
classification (interest | rejection | question | autoreply | unclear)
needs_human (bool)
```

---

## 5. Scoring model

The qualification step matters more than the sending step. Target roughly 60 to 100 well-chosen applications, not 500 sprayed ones.

Signals, weighted:

**Strong positive**
- Listing mentions international applicants, Visum, Anerkennung, or Sprachförderung
- Employer is a chain or large Träger (Caritas, Diakonie, AWO, Korian, Alloheim, Kursana) with actual HR capacity
- Start date in the April 2027 window
- Direct email contact rather than portal-only
- Employer previously responded to anything

**Moderate positive**
- Explicit B1 or B2 requirement stated (means they have thought about non-native applicants)
- Wohnheim or Wohnraum mentioned
- Multiple Ausbildungsplätze in one listing

**Negative**
- Portal-only with mandatory account creation
- Sole-proprietor or single-location small operation
- Listing demands German school-leaving certificate specifically
- Employer already contacted (one application per employer, ever)

Output: score 0 to 100, plus a `score_breakdown` json so the reasoning is inspectable. Anything under a threshold never reaches the drafting stage.

---

## 6. Build stages

Each stage ends in something usable on its own. If the project stalls at stage 3, stages 0 to 2 still delivered value.

### Stage 0 — Manual baseline (week of 18 Aug)
**Before any code.** Do 10 applications entirely by hand.

- Write the Anschreiben template and iterate it 10 times
- Write the German one-page employer PDF explaining the §16a process, ZAV-Vorabzustimmung and the 12-week timeline
- Note which employers replied and what they asked

Output: a template that is known to work, a real corpus for prompting, and 10 applications already out the door during the window.

This stage is non-negotiable. Automating an untested template just produces 100 bad letters faster.

### Stage 1 — Ingest (week of 25 Aug)
- BA API client, `X-API-Key: jobboerse-jobsuche`
- Filter to Ausbildung + Pflege Berufskennziffern, nationwide
- Normalise into `postings`, dedupe employers into `employers`
- Daily cron via n8n
- PocketBase deployed on k3s, in the homelab-apps repo

Output: a browsable, current database of every relevant posting in Germany.

### Stage 2 — Scoring (week of 1 Sep)
- Rules engine over `postings`
- Lane assignment: email / portal / skip
- Manual review of top 30 to sanity-check the scoring against Stage 0 experience

Output: a ranked worklist. **At this point the system is already useful even with zero automation beyond it.**

### Stage 3 — Generation (week of 8 Sep)
- Claude API call per posting: candidate profile + posting text + template + few-shot from Stage 0 letters
- Must reference something specific from the listing, no generic output
- Write to `applications.anschreiben_draft`

Output: drafts queued for review.

### Stage 4 — Approval gate (week of 15 Sep)
- v1 is the PocketBase admin UI plus a Telegram notification
- Review, edit, set status to `approved` or `rejected`
- Do not build a custom UI unless the admin UI genuinely gets in the way

Output: a review loop that takes minutes per day.

### Stage 5 — Sending (week of 22 Sep)
- SMTP from the candidate's mailbox, signed by him
- Hard throttle: 5 to 8 per day, working hours, weekdays only
- Attach CV, certificates, employer one-pager
- Store `thread_message_id` for reply matching
- Kill switch: a single flag that stops all sending

Output: applications going out without manual effort.

### Stage 6 — Reply tracking (week of 29 Sep)
- IMAP poll every 2 hours
- Match to `applications` via thread id or fuzzy sender match
- Claude classifies into interest / rejection / question / autoreply / unclear
- Anything classified as interest or unclear pushes a phone notification immediately

Output: nothing gets missed, and interview requests surface within hours.

### Stage 7 — Follow-up and reporting (week of 6 Oct)
- One nudge at day 12 for `sent` with no reply, then mark `ghosted`
- Weekly digest: applied, replied, response rate by employer type, remaining pool
- Feed response data back into the scoring weights

Output: a self-correcting loop.

---

## 7. What stays manual, permanently

- Portal-only applications. Browser automation against Bewerbungsportale breaks constantly and costs more time than it saves.
- Any reply requiring a real answer.
- Interview scheduling and preparation.
- Every approval decision.

---

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Emails land in spam | Existing mailbox with history, hard throttle, no new domain, no tracking pixels |
| Generic AI letters get ignored | Stage 0 first, mandatory specific reference to each listing, human edit pass |
| Employer contacted twice | Uniqueness constraint on `employers.contacted_at`, one application per employer ever |
| BA API changes or blocks | Cache aggressively, back off on errors, keep raw_json so re-parsing is possible offline |
| Pool exhausted before window closes | Widen to Bundesland-adjacent Berufe (Pflegefachhelfer, Altenpflege) rather than lowering letter quality |
| Project outruns the window | Stages 0 to 2 alone deliver the core value. Ship in order |

---

## 9. Open decisions

- [ ] Confirm Berufskennziffern to filter on (Pflegefachmann/-frau, Altenpflege, Pflegefachhelfer)
- [ ] Whose mailbox sends, and is an app password available
- [ ] Telegram bot vs ntfy for approvals
- [ ] Whether to widen to Handwerk (Bäckerei, Metzgerei) as a second lane, or drop it
- [ ] Where the candidate's documents live and how they get attached
- [ ] Verify the WHO Health Workforce Safeguards List 2026 final publication, since Zimbabwe appears on the drop list from the 2023 version

---

## 10. First action

Stage 0. Ten applications by hand, this coming week. Everything else is downstream of knowing what actually works.
