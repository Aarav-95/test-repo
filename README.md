# 🛠️ Developer Trace Ingestion and Quality Judge MVP

[cite_start]This project aims to capture, ingest, and evaluate developer debugging sessions (traces) to generate high-quality data for training AI agents[cite: 1].

---

## 🚀 Minimum Viable Product (MVP)

[cite_start]The MVP is focused on an **end-to-end flow** for ingesting a complete debugging trace and processing it through an automated quality judge[cite: 97].

### Core Features

* [cite_start]**Trace Ingestion:** The client will **upload the entire JSON trace at the end** of the session[cite: 98].
* [cite_start]**Data Storage:** The trace will be saved to a **PostgreSQL JSONB column**[cite: 99].
* **Automated Quality Judge:** A background worker service will process the trace to:
    * [cite_start]**Validate Code Fix:** Successfully **clone the repo**, **apply the `final_code_diff`**, and **run the repo's test suite** in a Docker container, updating `tests_passed`[cite: 100, 101].
    * [cite_start]**AI Methodology Score:** Call **Gemini 2.5 Flash** with a simple prompt based on the **"methodology over speed"** assumption and store a numeric score[cite: 102, 103, 15, 16].

### Deployment & Setup

* [cite_start]A **`docker-compose.yml`** will boot the API, worker, Postgres, and Redis with a single `docker-compose up` command[cite: 104].
* [cite_start]The `README.md` will contain this plan and **clear run instructions**, including one end-to-end example to demonstrate the entire flow[cite: 105].

---

## 📝 Project Assumptions & Scope Decisions

These decisions define the scope and quality standards for the project:

| Question | Answer / Assumption | Rationale / Detail |
| :--- | :--- | :--- |
| [cite_start]**How is CoT reasoning captured?** [cite: 3] | [cite_start]**Explicitly captured** through the custom IDE using a logging tool[cite: 11]. | [cite_start]The schema will explicitly include the chain-of-thought, as inferring the developer's thought process is too complex for this project's scope[cite: 11, 12]. |
| [cite_start]**Target scope of bugs?** [cite: 5] | [cite_start]**Smaller-scale and contained bugs**[cite: 14]. | [cite_start]Larger-scale fixes may take too long, allowing the MVP to focus on single-session traces[cite: 14]. |
| [cite_start]**Complexity of action data?** [cite: 7] | [cite_start]**Higher-level events** are sufficient[cite: 13]. | [cite_start]Including every keystroke may cause the training data to be overly ambiguous[cite: 13]. |
| [cite_start]**What signifies a high-quality trace?** [cite: 9] | [cite_start]**Methodology is valued greater than speed**[cite: 15]. | [cite_start]This ensures the entire debugging process is properly captured, providing more human-like data to train the agent[cite: 16]. |

---

## 📐 High-Level Technical Plan

### [cite_start]Architecture [cite: 78]

1.  [cite_start]**Ingestion API:** A single **POST endpoint** is exposed to receive the entire JSON trace from the client[cite: 79].
2.  [cite_start]**Validation:** The API validates the JSON against the defined schema[cite: 80].
3.  **Queuing:**
    * [cite_start]The API saves the valid trace to the **PostgreSQL DB** and changes `qa_results.status` to `pending`[cite: 81].
    * [cite_start]The `trace_id` is pushed to a **Redis message queue**[cite: 82].
    * [cite_start]The API returns a `202 status`, confirming receipt and pending processing[cite: 83, 84].
4.  **QA Judge Worker:**
    * [cite_start]A separate **Python service** continuously listens to the messaging queue[cite: 87].
    * [cite_start]It runs **PR validation** by cloning the repo, applying the `final_code_diff`, and running tests[cite: 88].
    * [cite_start]It runs the **AI quality judge** by sending the developers' thoughts through an LLM API[cite: 89].

### [cite_start]Technology Stack [cite: 86]

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Language** | [cite_start]**Python** [cite: 91] | [cite_start]Standard for AI and data processing, supports many useful libraries[cite: 91]. |
| **API** | [cite_start]**Fast API** [cite: 92] | [cite_start]A high-performance framework that is easy to use and has good data validation[cite: 92]. |
| **Database** | [cite_start]**PostgreSQL** [cite: 93] | [cite_start]JSONB support allows us to store half-structured traces while having lots of SQL queries available[cite: 93]. |
| **Queue** | [cite_start]**Redis** [cite: 94] | Very lightweight and fast for delivering messages. [cite_start]We can utilize the **RQ library** from Python[cite: 94, 95]. |
| **AI LLM** | [cite_start]**Gemini 2.5 Flash** [cite: 90] | [cite_start]Does not need a large amount of reasoning for this task; basic summarization and understanding are enough, allowing for cost savings[cite: 90]. |

---

## 💾 JSON Schema Overview

The schema is structured to capture all session metadata and a time-series of developer actions and thoughts.

### Root Object (Metadata)

[cite_start]Includes metadata and information about the entire session that helps differentiate between sessions[cite: 67]:
* [cite_start]`trace_id`: "uuid-v4-string" [cite: 19]
* [cite_start]`developer_id`: "user-string-or-uuid" [cite: 20]
* [cite_start]`task_id`: "bug-ticket-id-string" [cite: 21]
* [cite_start]`task_description`: "The full text of the bug report." [cite: 22]
* [cite_start]`repo_url`: "https://github.com/example/repo.git" [cite: 23]
* [cite_start]`repo_commit_hash`: "sha-string-at-start" [cite: 24]
* [cite_start]`start_timestamp`: "2025-10-29 T16:00:00Z" [cite: 25]
* [cite_start]`end_timestamp`: "2025-10-29 T18:00:00Z" [cite: 26]
* [cite_start]`Final_code_diff`: (Required to test the result of the entire process) [cite: 27, 67]
* [cite_start]`qa_results`: null (Can be filled when the AI judge gives it a ranking) [cite: 66, 68]
* [cite_start]`events`: \[ ... \] (Array containing all chronological events) [cite: 28, 69]

### Events Array Structure

[cite_start]The array includes each specific event (reasoning thought, command executed, etc.) and must be sorted in chronological order[cite: 69, 70]. Each event includes:

* [cite_start]`event_id`: "uuid-v4-string" [cite: 30]
* [cite_start]`timestamp`: (To display when the action took place) [cite: 31, 72]
* [cite_start]`event_type`: (Dictates the type of event, e.g., `thought_log`, `file_edit`, `command_run`) [cite: 32, 73]
* [cite_start]`payload`: (Specific data needed for the event type) [cite: 33, 75]

| Event Type | Key Payload Fields |
| :--- | :--- |
| `thought_log` | [cite_start]`thought_text`: "..." [cite: 32, 34] |
| `file_edit` | [cite_start]`file_path`: "example/path", `content_diff`: "..." [cite: 41, 44, 45] |
| `command_run` | [cite_start]`command_text`: "example command", `working_directory`: "example/dir", `exit_code`: 1, `stdout`: "...", `stderr`: "..." [cite: 48, 51, 52, 53, 54, 55] |

---

## ✨ Nice-to-Haves (Future Work)

These items add significant complexity and are explicitly out of scope for the MVP:

* [cite_start]**Incremental Ingestion:** Handling event ordering and session management for traces ingested event-by-event, rather than as a final JSON upload[cite: 107].
* [cite_start]**Advanced AI Rubric:** Implementing a more advanced system with greater detail (e.g., Clarity, Debugging Skills, etc.) instead of a simple numeric score[cite: 108].
* **Authentication:** Implementing an authentication mechanism to protect the API keys and data pipeline.
