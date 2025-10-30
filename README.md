# Project Plan

## Questions:
1. How is a developer’s chain of thought reasoning captured? Does the developer have a separate notes tab to log their reasoning, or do we infer it from any issues they may be encountering?

2. What is the target scope and complexity of the bugs in the codebase? Does the agent need to be trained for looking for small, contained bugs limited to a singular file or multi-file, complex fixes?

3. How complex does each action in the data need to be? Will it include every keystroke completed by the user or general, higher-level data?

4. What will signify a high-quality trace? Are we prioritizing speed and efficiency to get to the solution, or do you care more about the methodology, even if it means sacrificing speed and exploring dead ends?

## Assumptions:
1. Chain-of-Thought is explicitly captured through the custom IDE using a logging tool. The schema will include this explicitly since inferring what the developer is thinking is far more complex than the scope of this project.

2. Higher-level events are sufficient instead of every individual keystroke, as it may cause the training data to be overly ambiguous.

3. Smaller-scale and contained bugs are targeted, as larger-scale fixes may require a long time to fix, allowing the MVP to be focused on single-session traces.

4. The AI judge values the methodology greater than speed so that the entire debugging process can be properly captured, providing more human-like data to train the agent.

### JSON Schema:
```
{
  "trace_id": "uuid-v4-string",
  "developer_id": "user-string-or-uuid",
  "task_id": "bug-ticket-id-string",
  "task_description": "The full text of the bug report.",
  "repo_url": "https://github.com/example/repo.git",
  "repo_commit_hash": "sha-string-at-start",
  "start_timestamp": "2025-10-29 T16:00:00Z",
  "end_timestamp": "2025-10-29 T18:00:00Z",
  “Final_code_diff”: “...”
  "events": [
    {
      "event_id": "uuid-v4-string",
      "timestamp": "2025-10-29 T16:05:10Z",
      "event_type": "thought_log",
      "payload": {
        "thought_text": "..."
      }
    },
    {
      "event_id": "uuid-v4-string",
      "timestamp_iso": "2025-10-29 T16:06:25Z",
      "event_type": "file_edit",
      "payload": {
        "file_path": "example/path",
        "content_diff": “...”
    },
    {
      "event_id": "uuid-v4-string",
      "timestamp_iso": "2025-10-29 T16:07:00Z",
      "event_type": "command_run",
      "payload": {
        "command_text": "example command",
        "working_directory": "example/dir",
        "exit_code": 1,
        "stdout": "...",
        "stderr": "...'"
      }
    },
    {
      "event_id": "uuid-v4-string",
      "timestamp_iso": "2025-10-29 T16:07:15Z",
      "event_type": "thought_log",
      "payload": {
        "thought_text": "..."
      }
    }
  ],
  "qa_results": null
}
```
### Rationale: 

Root Object: Includes metadata and information about the entire session that helps differentiate between sessions, such as timestamp start and end, task description, GitHub link, etc. It also includes a qa results section that can be filled when the AI judge gives it a ranking

Events Array: Since this is time-series data, the data needs to be sorted in chronological order, which is best displayed in an array. The array includes each specific event, whether that is a preliminary reasoning thought, a command executed, etc.

Inside Event:
- Timestamp: to display when a certain action has taken place, allowing for chronological order
- Event Type: dictates the type of event that is occurring at that moment
    - This is important since each event will have a different set of schema needing to be filled
- Payload: depending on the type, each payload will include the specific data that we need to capture
  - Ex. stdin, stdout, stderr for command runs

## High-Level Technical Plan:

### Architecture:

Ingestion API: One POST endpoint will be exposed, in which the client will send the entire JSON to it

Validation: Validate the entire JSON with the schema defined previously

Queuing:
   - API saves the valid trace to the PostgreSQL DB and change qa_results.status to pending
   - The trace id will get pushed to a Redis message queue
   - The API then returns a 202 status, confirming that the trace was received and needs to be processed

QA Judge:
- A separate Python service is continuously listening to the messaging queue
- It runs the AI quality judge by taking in all the developers’ thoughts and sending it through to an LLM API
- GPT-5-nano can be used for this since we do not need a large amount of reasoning for this task, basic summarization, and understanding of the text is enough and will allow for cost savings

## Tech Stack:
Language: Python – Standard for AI and data processing, supports many useful libraries
API: Fast API – A high-performance framework that is easy to use and has good data validation
Database: PostgreSQL – JSONB support allows us to store half-structured traces while also having lots of SQL queries at our disposal
Queue: Redis – Redis is very lightweight and fast for delivering messages. We can utilize the RQ library from Python for this

## Scope & Trade-Offs:

### MVP:
- Upload the entire JSON trace at the end
- Trace will be saved to a PostgreSQL JSONB column
- The worker will call Gemini 2.5 Flash with a simple prompt based on our "methodology over speed" assumption and store a numeric score
- A docker-compose.yml that boots the API, worker, Postgres, and Redis with a single docker-compose up command.
- The README.md will contain this plan and clear run instructions, including one end-to-end example to demonstrate the entire flow

### Nice-to-Haves:
- Incremental Ingestion: This adds significant complexity to the MVP, as we would need to handle event ordering and session management
- Advanced AI Rubric: Instead of a simple score, a more advanced system could include a rubric with greater detail (Ex. Clarity, Debugging Skills, etc.)
- Authentication: The API will be open for anyone to use in this prototype, but ideally, an authentication mechanism would be implemented so that a random person cannot access the API keys and data pipeline
- Git Actions Testing: Cloning the repo and dockerizing a test suite would be ideal; however, with the time frame given, it is not necessary, as the most important part of the data is its quality of the thought process


