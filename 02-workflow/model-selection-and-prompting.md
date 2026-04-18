# Model Selection and Prompting

## Why This Matters

Choosing the wrong model for a task wastes tokens, produces worse results, and slows you down. A complex architecture decision sent to Haiku gives shallow output. A simple file rename sent to Opus burns tokens you did not need to spend. This guide gives you a decision framework and a set of prompting habits that compound over time.

---

## 1. Model Selection Strategy

Three models. Three distinct use cases. Know which one to reach for before you type.

### The Models

**Sonnet 4.6 — The Everyday Model**

Faster output, lower cost than Opus, more capable than Haiku. The right choice for most daily tasks: writing code, editing files, creating documents, iterating quickly on ideas. When in doubt, start here.

**Haiku 4.5 — The Token Saver**

Lightweight and cheap. Best for tasks that require no file uploads, no complex reasoning, no multi-step coordination. Chat questions, quick lookups, web searches, short summaries. Use it when budget matters more than depth.

**Opus 4.7 — The Reasoning Engine**

Highest capability. Designed for complex tasks: multi-file architecture, hard debugging, production code where correctness is non-negotiable. Pair with Extended Thinking for tasks that benefit from deliberate reasoning. Do not default to Opus for everything — reserve it for work that actually demands it.

### Decision Table

| Task Type | Recommended Model | Why |
|---|---|---|
| Chat question, no files | Haiku 4.5 | Fast and cheap — no depth needed |
| Web search, quick lookup | Haiku 4.5 | Lightweight retrieval task |
| Writing or editing a document | Sonnet 4.6 | Speed + quality balance |
| Creating a file (.xlsx, .docx, .pptx) | Sonnet 4.6 | File creation does not need deep reasoning |
| Single-file code change | Sonnet 4.6 | Sufficient capability for bounded tasks |
| Multi-file feature implementation | Opus 4.7 | Complex coordination across files |
| Architecture decision or system design | Opus 4.7 | Requires deep reasoning |
| Debugging complex cross-module bugs | Opus 4.7 | Root cause analysis benefits from reasoning |
| Rapid iteration on a prototype | Sonnet 4.6 | Speed matters more than depth here |
| Production code with correctness constraints | Opus 4.7 | When wrong means broken |

### Decision Flow

```
Is this task complex? (multi-file, architectural, production-critical)
├── YES → Does it need Extended Thinking or Cowork?
│         ├── YES → Opus 4.7 + Extended Thinking
│         └── NO  → Opus 4.7
└── NO  → Does it require uploading files?
          ├── YES → Sonnet 4.6
          └── NO  → Does it need more than a quick answer?
                    ├── YES → Sonnet 4.6
                    └── NO  → Haiku 4.5
```

---

## 2. Token Optimization Habits

Token efficiency is not about being cheap — it is about getting better results per conversation. Long, unfocused sessions degrade output quality as context fills up with noise.

### Before You Send

**Edit, do not follow up.** If you wrote a message and realized it is incomplete, edit and resend it — do not send a second correction message. Each additional message carries the weight of everything that came before it.

**Batch related tasks.** One message with three related requests is cheaper and more coherent than three messages sent sequentially. The model sees the full context once, not three times.

**Specify the output format upfront.** "No commentary. Just the output." saves a paragraph of preamble on every response. Be explicit: "Return only the code block, no explanation."

**Specify length constraints.** "Keep it under 200 words" prevents verbose answers you will not read anyway.

### File Uploads

**Convert PDFs to Markdown before uploading.** A PDF parsed by OCR is noisy and large. A clean `.md` file is smaller, parseable, and yields better results. Use any PDF-to-Markdown converter before uploading.

**Keep files under 2,000 words per upload.** Split large documents into focused sections. Upload only the section relevant to the current task.

**Upload source, not compiled output.** Upload your `.ts` file, not the compiled `.js`. Upload your `.md`, not the rendered PDF.

### Session Management

**New topic = new chat.** When you switch to a fundamentally different task, start a fresh session. The model spends tokens re-reading an entire history that is now irrelevant. Fresh context is cheaper and more accurate.

**Turn off Search when you do not need it.** Web search adds latency and token cost for every response. Disable it for tasks that are self-contained in the files you have already uploaded.

**Turn off Extended Thinking when the task is straightforward.** Extended Thinking is valuable for hard problems. For routine tasks it adds cost without adding quality. Switch it on deliberately, not by default.

**Plan in Chat, build in Cowork.** Use a regular Chat session to discuss the approach, clarify requirements, and finalize the plan. Then open a Cowork session to execute. This separation keeps the execution context clean and prevents planning noise from polluting implementation.

---

## 3. Prompt Templates by Task Type

These templates encode the habits from section 2 into reusable patterns. Adapt the brackets to your task.

### Quick Task

When you need a fast output with no back-and-forth:

```
You are a [role]. [Task] this [input].
Keep it under [length].
Tone: [casual / formal / technical].
No preamble. Just the output.
```

Example:

```
You are a copywriter. Rewrite this product description to be more direct and action-oriented.
Keep it under 80 words.
Tone: professional.
No preamble. Just the output.
```

### File Creation

When generating a structured output file (.xlsx, .docx, .pptx):

```
Create a professional [file type] for: [purpose].
Context: [who will use it and how].
It should cover: [list of sections or data].
Rules: [specific constraints — formatting, length, tone, required fields].
```

Example:

```
Create a professional project status report (.docx) for: weekly engineering update.
Context: sent by the engineering lead to non-technical stakeholders.
It should cover: completed this week, in progress, blockers, next week priorities.
Rules: no jargon, each section max 3 bullet points, executive summary at the top.
```

### Cowork / Complex Task

When starting a multi-step implementation session — do not let the model start immediately:

```
I want to [task] so that [what success looks like].
Read the uploaded files completely.
DO NOT start yet.
Ask me clarifying questions using AskUserQuestion so we can refine the approach step by step.
```

Example:

```
I want to refactor the authentication module to use refresh tokens so that sessions
persist across browser restarts without re-login.
Read the uploaded files completely.
DO NOT start yet.
Ask me clarifying questions using AskUserQuestion so we can refine the approach step by step.
```

### Desired Results

When you have an outcome in mind but need the model to help define the path:

```
I want to [desired outcome] with [constraints].
Ask me questions using AskUserQuestion before you start.
```

Example:

```
I want to reduce the p95 latency on our API by 40% with no schema changes.
Ask me questions using AskUserQuestion before you start.
```

---

## 4. Cowork Workflow

Cowork sessions are for execution, not exploration. Follow this sequence:

**Before opening Cowork:**

1. Plan in Chat first. Discuss the approach, clarify ambiguities, confirm the file map.
2. Finalize your task list. Each task should have a clear outcome and a verification step.
3. Prepare your uploads. Convert PDFs to `.md`. Split large files.

**Inside Cowork:**

- Turn on Extended Thinking for all complex tasks. It pays for itself in fewer correction cycles.
- Upload `.md` files rather than PDFs — parsing is cleaner and results are better.
- Use the Cowork/Complex Task prompt template (section 3) — ask the model to read all files and ask clarifying questions before it starts.

**After the Cowork session:**

1. Download all generated files immediately — sessions do not persist indefinitely.
2. Save a `session-notes.md` capturing decisions made, files changed, and open questions.
3. Start a fresh session for the next task — do not continue the same Cowork session into a new topic.

---

## 5. Connectors

Claude connects to external tools via MCP connectors: Slack, Google Drive, Notion, Figma, GitHub, Linear, and 50+ more. When your workflow involves data that lives in these systems — project context in Notion, design specs in Figma, tickets in Linear — connecting the source directly is faster and more accurate than copy-pasting.

Connectors extend the workflow beyond local files. A well-configured connector setup means you can ask Claude to pull the latest spec from Notion, check open tickets in Linear, and reference the Figma design — without leaving the conversation.

---

## 6. Anti-Patterns

| Anti-Pattern | Why It Hurts | What to Do Instead |
|---|---|---|
| Sending multiple short follow-up messages | Inflates context, degrades coherence | Edit the original message and resend |
| Uploading large PDFs | Noisy OCR, high token cost | Convert to `.md` first, split if needed |
| Not specifying output format | Gets verbose preamble + explanation you did not need | Add "No preamble. Just the output." |
| Not specifying length | Gets a wall of text | Add "Keep it under X words" |
| Continuing a stale session into a new topic | New task drowns in old context | Start a fresh session |
| Using Opus for simple tasks | Wastes tokens with no quality gain | Use Sonnet 4.6 or Haiku 4.5 |
| Using Haiku for complex reasoning | Shallow output that requires correction | Use Opus 4.7 with Extended Thinking |
| Turning on Extended Thinking by default | Unnecessary latency and cost on simple tasks | Enable it deliberately for hard problems |
| Starting Cowork before planning in Chat | Misaligned execution that needs undoing | Plan first in Chat, then switch to Cowork |

---

*Model taxonomy inspired by the Claude Model Tree by Ruben Hassid (how-to-ai.guide).*
