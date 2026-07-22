# PRD: Hospital Appointment Desk — AI Email Triage & Draft Agent

**Status:** Draft for build
**Owner:** [Your name]
**Last updated:** 2026-07-08

---

## 1. Problem Statement

The hospital's Gmail inbox receives a mix of genuine patient queries (appointments, documentation requests, insurance questions, test prep, complaints, clinical concerns) alongside newsletters, receipts, and internal/vendor mail. Staff currently read and triage everything manually. The goal is to have an agent silently do the first pass — identify real queries, draft grounded replies from the hospital's own knowledge documents, and leave everything else alone — so staff only need to review and send, never write from scratch or dig through noise.

## 2. Goals

- Reduce staff time spent triaging and drafting routine replies.
- Never send an email automatically — every reply is a Gmail draft awaiting human approval.
- Never fabricate an answer. If the knowledge documents don't cover it, the agent does nothing except flag it for a human.
- Keep the system inside Gmail's existing UI — no new dashboard, app, or database for staff to check.
- Run unattended on a schedule, at low cost, for low volume (10–20 emails/day).

## 3. Non-Goals

- Not answering clinical questions, giving medical advice, or interpreting reports — these are always escalated (see Section 6).
- Not resolving complaints, billing disputes, or legal-toned messages — always escalated.
- Not building a UI, dashboard, or admin panel of any kind.
- Not building a database or vector store — Gmail labels are the entire state layer.
- Not handling non-English emails in this version.
- Not reading or acting on attachment contents.

## 4. High-Level Architecture

```
GitHub Actions (cron, hourly)
        │
        ▼
 [1] Auth: OAuth refresh token → Gmail API + Drive API
        │
        ▼
 [2] Fetch candidate emails
     - Gmail search: inbox, not already labeled Agent-Processed
     - Message-level check (see 4.1)
        │
        ▼
 [3] Cheap Triage Pass (Groq — small/fast model)
     - Input: email body only
     - Output: is this a genuine patient query? (yes/no + rough category)
        │
        ├── No  → apply Agent-Processed label, stop (no draft, no flag)
        │
        ▼ Yes
 [4] Fetch current Knowledge Base from Drive folder
     - All .md / .docx files in the designated folder, fetched fresh each run
        │
        ▼
 [5] Drafting Pass (Groq — larger model)
     - Input: email body + full KB content + category rules
     - Output: either a complete draft reply, OR an explicit "not covered" signal
        │
        ├── Not covered → apply Agent-Processed + Needs-Human labels, no draft created
        │
        ▼ Covered
 [6] Create Gmail draft in the thread (via Gmail API drafts.create, threaded reply)
        │
        ▼
 [7] Apply Agent-Processed label
        │
        ▼
 [8] On any failure at any step → send failure email to owner, stop run cleanly
```

### 4.1 Message-level tracking (not just thread-level)

Gmail labels apply to individual messages, not whole threads. The agent checks each **message** in a thread, not just the thread as a whole:

- A message is skipped if it already carries `Agent-Processed`.
- A message is skipped if the thread already contains an unsent Gmail draft (checked via `drafts.list` filtered to the thread).
- A message is skipped if the thread already contains a `SENT` message that came after it (i.e., staff already replied manually — the agent should not draft a redundant or conflicting reply).
- If a new patient message arrives on a thread that was previously fully processed, only the **new** message is evaluated — the agent does not re-triage the whole thread history, though it may include prior thread context in the drafting prompt for continuity.

### 4.2 Concurrency guard

GitHub Actions is configured with a `concurrency` group on the workflow so a new hourly run cannot start while a previous run is still in progress (protects against double-processing if a run runs long due to Groq throttling).

## 5. Components

### 5.1 Authentication
- **Gmail:** OAuth 2.0, consumer `@gmail.com` account. Refresh token generated once locally, stored as a GitHub Actions encrypted secret. Scopes: `gmail.modify` (read, label, create drafts) — does not need `gmail.send`, since the agent never sends.
- **Drive:** OAuth 2.0, same Google account, `drive.readonly` scoped ideally to just the designated KB folder (or a folder-restricted approach if using `drive.file` scope with pre-shared file IDs).
- **Known limitation (accepted for now):** the app's OAuth consent screen will be in "Testing" publishing status, meaning the refresh token expires roughly every 7 days. **Decision: accept manual re-authentication approximately weekly** rather than pursuing Google app verification at this stage. This should be revisited if the project scales beyond a personal/pilot use case, since a lapsed token means the agent silently stops running until someone notices and re-authenticates.

### 5.2 Knowledge Base source
- A single Google Drive folder containing `.md` and `.docx` files (e.g., Hospital Overview, Patient Services Guide, Departments/Doctors, Insurance/TPA list, Test Prep instructions, Documentation Services, Agent Response Rules, Email Draft Templates, Style Guide).
- Fetched fresh on every run and passed in full into the drafting prompt context. No caching, no vector search — appropriate at current document volume; revisit if the KB grows large enough to strain context limits or token cost.
- This folder is the **sole source of truth**. The agent must not use any other knowledge, training data, or assumption to answer a query.

### 5.3 Triage pass (cheap check)
- **Purpose:** cheaply filter out newsletters, receipts, automated notifications, vendor mail, and internal mail before running the more expensive drafting pass.
- **Input:** email body text only (per your stated preference — body content, not just headers/sender).
- **Model:** Groq free tier, a small/fast model (e.g. an 8B-class instant model — confirm exact current model name against Groq's live model list at build time, since offerings change).
- **Output:** structured yes/no ("genuine patient query" vs "not a query") plus a rough category guess, used only as a hint for the drafting pass, not a final classification.

### 5.4 Drafting pass
- **Purpose:** for genuine queries, produce a complete, human-sounding draft reply grounded strictly in the KB, OR explicitly signal "not covered."
- **Model:** Groq free tier, a larger/more capable model (e.g. a 70B-class model — confirm current model name at build time).
- **Input:** full email body (and recent thread context if applicable), full KB content, the six-category response rules, and the tone/style guide.
- **Hard rules enforced in the prompt:**
  - Never answer clinical content (symptoms, medication, report interpretation) — always escalate.
  - Never resolve complaints, billing disputes, or legal-toned messages — always escalate.
  - Never confirm/deny something not explicitly in the KB (e.g., an unlisted insurance provider, an unlisted test) — always escalate rather than infer.
  - Never invent patient-specific data (amounts, dates, results) — those come from a human after manual record lookup, never from the agent.
  - If a request implies a third party (e.g., a family member requesting another patient's records), escalate rather than draft, per the Patient Record Lookup Process rules.
  - Output must explicitly state whether it is "COVERED" (draft follows) or "NOT COVERED" (no draft, flag only) — this explicit signal is what the pipeline branches on, not inferred confidence.

### 5.5 Draft creation
- Created via Gmail API `drafts.create`, threaded to the original message (correct `In-Reply-To` / `References` headers, replying to the sender).
- Draft appears in the normal Gmail thread view exactly where staff already work — no separate location.

### 5.6 Labels (state machine)
| Label | Applied when |
|---|---|
| `Agent-Processed` | Always applied once the agent has made a final decision on a message — whether that decision was "not a query," "drafted a reply," or "escalated as not covered." This is what prevents re-processing on the next hourly run. |
| `Needs-Human` | Applied alongside `Agent-Processed` when the drafting pass returns "NOT COVERED," or when the message matches a hard-escalation category (clinical content, complaints, legal language, third-party record requests). No draft is created in these cases. |

### 5.7 Scheduling
- GitHub Actions workflow, `schedule: cron` hourly.
- `concurrency` group set to prevent overlapping runs.
- On any unhandled error (auth failure, Gmail/Drive/Groq API error), the workflow sends a failure notification email to the owner and exits cleanly without leaving partial state (i.e., a message is only labeled `Agent-Processed` after its full pipeline — triage, drafting, draft creation, labeling — completes successfully).

## 6. Email Categories & Handling

| Category | Agent behavior |
|---|---|
| Scheduling / Logistics (clear KB match) | Draft reply using KB |
| Documentation Requests (clear KB match, no third party involved) | Draft reply using KB, asking for identity verification fields per Patient Record Lookup Process |
| Spam / Duplicate / Out-of-scope | No draft; `Agent-Processed` only (spam/out-of-scope) or draft confirming existing details (true duplicates) |
| Clinical Content | No draft; `Agent-Processed` + `Needs-Human` |
| Complaints (including legal-toned) | No draft; `Agent-Processed` + `Needs-Human` |
| Low-confidence / not in KB (includes third-party record requests) | No draft; `Agent-Processed` + `Needs-Human` |

Not a genuine query (newsletter, receipt, internal, promotional) → `Agent-Processed` only, no `Needs-Human`, no draft.

## 7. Cost & Volume

- ~10–20 emails/day expected.
- Two Groq calls per genuine query (triage + drafting); one Groq call per non-query (triage only, short-circuits before drafting).
- Well within Groq's free-tier daily/request limits at this volume — should be reconfirmed against Groq's current published limits at build time, since free-tier terms change.

## 8. Known Risks & Accepted Limitations

- **OAuth token expiry (~7 days):** Accepted for now; requires manual re-auth. Silent failure risk mitigated by the failure-email alert.
- **Third-party data exposure:** Email content — including any patient-written descriptions of symptoms or conditions — is sent to Groq's API for processing. This is a third-party US-based inference provider; no BAA or India-specific data processing agreement is assumed to be in place. This is accepted as a known risk for this pilot scope rather than solved architecturally. Should be revisited before any wider or production healthcare deployment.
- **Attachments are ignored:** the agent does not open or read attachment contents. If a message has an attachment, this should be noted in the escalation (implementation detail: include an "has attachment" flag in the flagged output) so a human knows to check it manually.
- **No re-verification of Groq/Drive/Gmail API changes over time:** model names, scopes, and rate limits should be reconfirmed periodically since these are third-party services that evolve independently of this PRD.

## 9. Testing Plan

- Use the existing `Sample_Test_Emails.md` (20 test cases) covering all six categories plus edge cases: unlisted test/insurance, third-party documentation request, legal-toned complaint, mixed booking+clinical intent, and urgent-but-non-emergency tone.
- Run these through the pipeline manually before enabling the hourly schedule, and verify:
  - Correct label applied in every case.
  - No draft created for any clinical/complaint/low-confidence case.
  - Draft content matches KB facts exactly, with no invented details.
  - Duplicate detection correctly avoids double-booking on Test 4-style follow-ups.

## 10. Future Considerations (explicitly out of scope for v1)

- Google Workspace / verified OAuth app (removes the 7-day token expiry issue).
- Multi-language support.
- Attachment inspection (e.g., flagging if an attached image looks like an insurance card vs. a report).
- Slack or other alerting beyond email-on-failure.
- Vector search / RAG over a larger knowledge base if document volume grows significantly.
