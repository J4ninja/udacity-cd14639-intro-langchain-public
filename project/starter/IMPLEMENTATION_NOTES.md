# Implementation Notes

This document explains the main implementation decisions behind the document assistant starter project, how state and memory work in the workflow, how structured outputs are enforced, and how the assistant behaves in practice.

## 1. Implementation decisions

### Structured outputs
The project uses Pydantic schemas to ensure the model returns predictable data shapes. Key examples include:
- `AnswerResponse` for answers with question, answer, sources, confidence, and timestamp.
- `SummarizationResponse` for summaries, key points, document IDs, and timestamp.
- `CalculationResponse` for expressions, results, explanation, units, and timestamp.
- `UserIntent` for intent classification with `intent_type`, `confidence`, and `reasoning`.
- `UpdateMemoryResponse` for conversation summaries and document IDs.

These schemas are passed to the LLM through structured-output methods, so the model is guided to return data in a consistent format.

### Tool integration
The assistant uses tools for document search and calculation. Tools are registered in the workflow config and made available to the agent so it can retrieve relevant information before answering.

## 2. How state and memory work

### State object
The workflow uses an `AgentState` typed dictionary that carries:
- `user_input`: the current user message.
- `messages`: the message history used by LangGraph.
- `intent`: the classified user intent.
- `next_step`: the next node in the workflow.
- `conversation_summary`: a summarized view of prior turns.
- `active_documents`: document IDs relevant to the current session.
- `current_response`: the latest structured response payload.
- `tools_used`: tool names used during the current turn.
- `session_id` and `user_id`: session metadata.
- `actions_taken`: a list of agent nodes executed in the current turn.

### Memory and persistence
The workflow is compiled with an `InMemorySaver` checkpointer, which lets the graph persist state between invocations. This means the assistant can retain conversation information and continue from prior turns within the same thread.

In `DocumentAssistant.process_message`, the workflow is invoked with a configurable thread ID tied to the current session. That allows state to be reused across multiple turns without losing prior messages, summaries, or document context.

## 3. How structured outputs are enforced

Structured outputs are enforced in two main ways:

1. Pydantic schemas define the expected response format.
2. The LLM is configured to return that schema using structured-output support.

For example, the intent classifier uses `UserIntent`, while the Q&A and calculation nodes use `AnswerResponse`, `SummarizationResponse`, and `CalculationResponse`.

This helps the app avoid brittle string parsing and makes downstream code simpler because the response is already typed and predictable.

### Confidence validation
The confidence fields in the schemas are constrained to the inclusive range $0 \leq \text{confidence} \leq 1$ using Pydantic validation. That prevents invalid values from being accepted by the system.

## 4. Example conversations

### Example 1: Q&A over documents
User:
> What is the total amount on invoice INV-104?

Assistant behavior:
- Classifies the request as `qa`.
- Searches relevant documents.
- Returns an answer with a source ID and a confidence score.

Example structured response:
- `question`: "What is the total amount on invoice INV-104?"
- `answer`: "The total amount is $12,500."
- `sources`: ["INV-104"]
- `confidence`: 0.93

### Example 2: Summarization
User:
> Summarize the latest contract for me.

Assistant behavior:
- Classifies the request as `summarization`.
- Retrieves the relevant document.
- Produces a summary with key points and document IDs.

Example structured response:
- `summary`: "The contract covers a 12-month engagement with milestone-based payments."
- `key_points`: ["12-month term", "milestone payments", "renewal optional"]
- `document_ids`: ["CONTRACT-001"]

### Example 3: Calculation
User:
> What is 125 + 37 * 2?

Assistant behavior:
- Classifies the request as `calculation`.
- Uses the calculator tool.
- Returns a numeric result with an explanation.

Example structured response:
- `expression`: "125 + 37 * 2"
- `result`: 199.0
- `explanation`: "Multiply 37 by 2 first, then add 125."

### Example 4: Follow-up memory behavior
User:
> What was the previous document we discussed?

Assistant behavior:
- Uses the persisted conversation summary and active documents.
- Responds with context from prior turns.

This shows how memory and state allow the assistant to behave like a continuing conversation rather than a one-shot prompt.

