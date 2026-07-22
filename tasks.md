# Hospital Appointment Desk Agent - Implementation Tasks

This document outlines the atomic tasks required to build the AI Email Triage & Draft Agent, organized into phases based on dependencies.

## Phase 1: Project Setup & Authentication
**Dependencies:** None
- [ ] 1.1 Initialize project repository and package manager (e.g., Node.js or Python).
- [ ] 1.2 Set up environment variable management (e.g., `.env` file) for credentials.
- [ ] 1.3 Create a local OAuth 2.0 authorization script to generate the initial Google refresh token.
- [ ] 1.4 Configure Google Cloud Project with Gmail API and Drive API enabled.
- [ ] 1.5 Configure OAuth consent screen and create credentials (Client ID, Client Secret).
- [ ] 1.6 Implement an authentication module that uses the refresh token to obtain temporary access tokens for Gmail and Drive APIs.
- [ ] 1.7 Add required Google API scopes (`https://www.googleapis.com/auth/gmail.modify`, `https://www.googleapis.com/auth/drive.readonly`).

## Phase 2: Knowledge Base Fetching (Drive API)
**Dependencies:** Phase 1
- [ ] 2.1 Implement a function to authenticate the Drive API client.
- [ ] 2.2 Implement a function to list all `.md` and `.docx` files within a specified Google Drive folder ID.
- [ ] 2.3 Implement a function to download the content of the identified files.
- [ ] 2.4 Implement a parser to extract text from `.docx` files.
- [ ] 2.5 Implement a function to aggregate and concatenate all KB text into a single context string.

## Phase 3: Email Ingestion & Thread Validation (Gmail API)
**Dependencies:** Phase 1
- [ ] 3.1 Implement a function to search the Gmail inbox for candidate messages (query: `in:inbox -label:Agent-Processed`).
- [ ] 3.2 Implement a function to fetch the full details of a specific message (headers, body text, threadId, messageId).
- [ ] 3.3 Implement text extraction to cleanly pull the plain text body from the email payload.
- [ ] 3.4 Implement thread validation: Check if a thread already contains an unsent draft.
- [ ] 3.5 Implement thread validation: Check if a thread already contains a `SENT` message with a timestamp later than the candidate message.
- [ ] 3.6 Create a filtering module that skips messages failing the validation checks in 3.4 and 3.5.

## Phase 4: LLM Integration & Prompts (Groq API)
**Dependencies:** Phase 2, Phase 3
- [ ] 4.1 Initialize the Groq API client with the `GROQ_API_KEY`.
- [ ] 4.2 **Triage Pass:** Design the system prompt for the triage pass (small/fast model).
- [ ] 4.3 **Triage Pass:** Implement the function to call Groq for triage (Input: email body, Output: JSON structured yes/no + category).
- [ ] 4.4 **Drafting Pass:** Design the system prompt for the drafting pass, embedding the hard rules (clinical, complaints, third-party) and KB context.
- [ ] 4.5 **Drafting Pass:** Implement the function to call Groq for drafting (Input: email body + KB string, Output: explicit "COVERED" + draft text OR "NOT COVERED").

## Phase 5: Gmail Actions (Drafting & Labeling)
**Dependencies:** Phase 1, Phase 4
- [ ] 5.1 Implement a function to ensure the `Agent-Processed` and `Needs-Human` labels exist in the Gmail account (create if missing).
- [ ] 5.2 Implement a function to create a new draft reply in Gmail (`drafts.create`), correctly utilizing the `threadId` and standard reply headers (`In-Reply-To`, `References`).
- [ ] 5.3 Implement a function to apply labels to a specific message ID.
- [ ] 5.4 Map the LLM output to the labeling logic (e.g., apply `Needs-Human` if NOT COVERED or if specific categories are detected).

## Phase 6: Orchestration & Error Handling
**Dependencies:** Phase 2, Phase 3, Phase 4, Phase 5
- [ ] 6.1 Create the main execution loop: Fetch KB -> Fetch Emails -> Loop Emails -> Triage -> Draft (if needed) -> Create Draft (if covered) -> Apply Labels.
- [ ] 6.2 Implement comprehensive `try/catch` error boundaries around API calls.
- [ ] 6.3 Implement a failure notification function: Send an email to the system owner (via Gmail API) detailing any unhandled errors.
- [ ] 6.4 Ensure the main loop exits cleanly on critical failures to prevent partial state processing.

## Phase 7: Deployment & CI/CD
**Dependencies:** Phase 6
- [ ] 7.1 Create a `.github/workflows/agent.yml` file for GitHub Actions.
- [ ] 7.2 Configure the workflow to trigger on a schedule (`cron` for hourly execution).
- [ ] 7.3 Configure the workflow `concurrency` group to prevent overlapping runs.
- [ ] 7.4 Set up the workflow to use repository secrets for all environment variables (Google Client ID, Secret, Refresh Token, Groq API Key).
- [ ] 7.5 Document the manual process for refreshing the OAuth token every ~7 days.

## Phase 8: Testing
**Dependencies:** Phase 6
- [ ] 8.1 Execute the script locally against the 20 test cases from `Sample_Test_Emails.md`.
- [ ] 8.2 Verify that `Agent-Processed` is applied correctly.
- [ ] 8.3 Verify that `Needs-Human` is applied appropriately (clinical, complaints, low-confidence).
- [ ] 8.4 Verify that draft content is strictly grounded in the KB facts.
- [ ] 8.5 Verify duplicate/follow-up thread detection logic works as expected.
