# Prompting Patterns in Claude Code

Reference of prompting patterns found in the Claude Code source code, with file locations and examples.

---

## 1. Good/Bad Example Pairs

**File**: `src/tools/AgentTool/built-in/verificationAgent.ts:84-115`

Shows a **rejected** example alongside an **accepted** example to define the quality bar. Forces the model to understand what's insufficient vs adequate.

```
Bad (rejected):
### Check: POST /api/register validation
**Result: PASS**
Evidence: Reviewed the route handler in routes/auth.py. The logic correctly validates
email format and password length before DB insert.
(No command run. Reading code is not verification.)

Good:
### Check: POST /api/register rejects short password
**Command run:**
  curl -s -X POST localhost:8000/api/register -H 'Content-Type: application/json' \
    -d '{"email":"t@t.co","password":"short"}' | python3 -m json.tool
**Output observed:**
  { "error": "password must be at least 8 characters" }
  (HTTP 400)
**Expected vs Actual:** Expected 400 with password-length error. Got exactly that.
**Result: PASS**
```

**When to use**: When the model consistently produces low-quality outputs that technically match the format but miss the intent.

---

## 2. Structural Output Examples with `<example>` Tags

**File**: `src/services/compact/prompt.ts:79-143`

Uses `<example>` tags to show exact output structure with numbered sections and placeholder content. Also includes **meta-examples** — examples of what user instructions might look like:

```xml
<example>
<analysis>
[Your thought process, ensuring all points are covered thoroughly and accurately]
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]

2. Key Technical Concepts:
   - [Concept 1]
   - [Concept 2]

3. Files and Code Sections:
   - [File Name 1]
      - [Summary of why this file is important]
      - [Important Code Snippet]
...
</summary>
</example>
```

Meta-examples of possible inputs:

```xml
<example>
## Compact Instructions
When summarizing the conversation focus on typescript code changes and also remember the mistakes you made and how you fixed them.
</example>
```

**When to use**: When you need precise structural output with many sections, and the model needs to handle variable additional instructions.

---

## 3. Rationalization Pre-Blocking (Anti-Pattern List)

**File**: `src/tools/AgentTool/built-in/verificationAgent.ts:53-61`

Names the **exact rationalizations** the model will generate, then tells it to do the opposite:

```
You will feel the urge to skip checks. These are the exact excuses you reach for:
- "The code looks correct based on my reading" — reading is not verification. Run it.
- "The implementer's tests already pass" — the implementer is an LLM. Verify independently.
- "This is probably fine" — probably is not verified. Run it.
- "Let me start the server and check the code" — no. Start the server and hit the endpoint.
- "I don't have a browser" — did you actually check for mcp tools? If present, use them.
- "This would take too long" — not your call.
If you catch yourself writing an explanation instead of a command, stop. Run the command.
```

**When to use**: When the model has known failure modes you can predict. Name them explicitly before they happen.

---

## 4. Domain-Specific Strategy Branching

**File**: `src/tools/AgentTool/built-in/verificationAgent.ts:28-40`

Provides **type-specific worked examples** for ~10 change types, each with concrete steps, followed by a generalization rule:

```
**Frontend changes**: Start dev server → check for browser automation tools → curl subresources → run frontend tests
**Backend/API changes**: Start server → curl/fetch endpoints → verify response shapes → test error handling
**CLI/script changes**: Run with representative inputs → verify stdout/stderr/exit codes → test edge inputs
**Infrastructure/config changes**: Validate syntax → dry-run where possible → check env vars
**Bug fixes**: Reproduce the original bug → verify fix → run regression tests
**Database migrations**: Run migration up → verify schema → run migration down → test against existing data
**Refactoring**: Existing test suite MUST pass unchanged → diff the public API surface
...
**Other change types**: The pattern is always the same — (a) figure out how to exercise this change directly, (b) check outputs against expectations, (c) try to break it.
```

**When to use**: When behavior should vary by context. Enumerate the common cases with specific steps, then provide a generalization rule for everything else.

---

## 5. Constraint-First with Consequences

**File**: `src/services/compact/prompt.ts:19-26`

States the constraint, then states **what happens if violated**:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

Also reinforced with a trailer at the end of the prompt:

```
REMINDER: Do NOT call any tools. Respond with plain text only —
an <analysis> block followed by a <summary> block.
Tool calls will be rejected and you will fail the task.
```

**When to use**: When the model is known to violate a constraint. State it at the top AND bottom. Include concrete consequences.

---

## 6. Step-by-Step with Parallelism Hints

**File**: `src/tools/BashTool/prompt.ts:96-125`

Uses numbered steps with explicit notes about which can run in parallel vs sequentially, plus a format example:

```
1. Run the following bash commands in parallel, each using the Bash tool:
  - Run a git status command...
  - Run a git diff command...
  - Run a git log command...
2. Analyze all staged changes and draft a commit message:
  - Summarize the nature of the changes...
  - Draft a concise (1-2 sentences) commit message...
3. Run the following commands in parallel:
   - Add relevant untracked files to the staging area.
   - Create the commit with a message ending with:
     Co-Authored-By: ...
   - Run git status after the commit completes to verify success.
   Note: git status depends on the commit completing, so run it sequentially after the commit.
4. If the commit fails due to pre-commit hook: fix the issue and create a NEW commit
```

With a HEREDOC `<example>` for exact command formatting:

```xml
<example>
git commit -m "$(cat <<'EOF'
   Commit message here.

   Co-Authored-By: ...
   EOF
   )"
</example>
```

**When to use**: Multi-step procedures where some steps can be parallelized. Be explicit about dependencies.

---

## 7. Hidden Scratchpad (Chain-of-Thought with Post-Processing)

**File**: `src/services/compact/prompt.ts:31-44, 311-334`

The model reasons inside `<analysis>` tags, then produces a clean `<summary>`. The scratchpad is stripped by code before the output reaches context:

```
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts.
In your analysis process:

1. Chronologically analyze each message and section of the conversation. For each section identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like file names, code snippets, function signatures, file edits
   - Errors that you ran into and how you fixed them
2. Double-check for technical accuracy and completeness.
```

The `formatCompactSummary()` function strips `<analysis>` and reformats `<summary>`:

```typescript
formattedSummary = formattedSummary.replace(/<analysis>[\s\S]*?<\/analysis>/, '')
```

**When to use**: When you want higher quality output but don't want the reasoning in the final result. Let the model think, then strip the thinking.

---

## 8. Sentinel-Based Output Parsing

**File**: `src/tools/AgentTool/built-in/verificationAgent.ts:117-127`

Requires a **literal sentinel string** on the last line for programmatic consumption:

```
End with exactly this line (parsed by caller):

VERDICT: PASS
or
VERDICT: FAIL
or
VERDICT: PARTIAL

Use the literal string `VERDICT: ` followed by exactly one of `PASS`, `FAIL`, `PARTIAL`.
No markdown bold, no punctuation, no variation.
```

Each verdict has defined semantics:
- **FAIL**: include what failed, exact error output, reproduction steps
- **PARTIAL**: what was verified, what could not be and why, what the implementer should know

**When to use**: When the output will be parsed by code. Be explicit about exact format, prohibit common variations (bold, punctuation).

---

## 9. Modular Prompt Composition (Software Pattern)

**File**: `src/constants/prompts.ts`

The system prompt is built from ~20+ composable functions that are conditionally included:

```typescript
function getSimpleIntroSection(outputStyleConfig) { ... }
function getSimpleSystemSection() { ... }
function getSimpleDoingTasksSection() { ... }
function getHooksSection() { ... }
function getLanguageSection(languagePreference) { ... }
function getMcpInstructionsSection(mcpClients) { ... }
```

Sections are gated by user type, features, and configuration:

```typescript
...(process.env.USER_TYPE === 'ant'
  ? [`Default to writing no comments. Only add one when the WHY is non-obvious...`]
  : []),
```

Uses a `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker to separate cacheable static content from dynamic user-specific content.

**When to use**: When prompts need to vary by user type, feature flags, or runtime configuration. Treat prompts as code — compose them from functions with conditional logic.

---

## 10. Adversarial Probe Suggestions

**File**: `src/tools/AgentTool/built-in/verificationAgent.ts:63-69`

Provides concrete attack vectors as seeds, not as a rigid checklist:

```
Functional tests confirm the happy path. Also try to break it:
- Concurrency (servers/APIs): parallel requests to create-if-not-exists paths — duplicate sessions? lost writes?
- Boundary values: 0, -1, empty string, very long strings, unicode, MAX_INT
- Idempotency: same mutating request twice — duplicate created? error? correct no-op?
- Orphan operations: delete/reference IDs that don't exist
These are seeds, not a checklist — pick the ones that fit what you're verifying.
```

**When to use**: When the model needs creative but grounded exploration. Give concrete examples but explicitly say they're starting points, not an exhaustive list.

---

## Summary Table

| Pattern | File | Purpose |
|---|---|---|
| Good/Bad example pairs | `verificationAgent.ts` | Ground quality expectations |
| `<example>` structural templates | `compact/prompt.ts` | Define exact output shape |
| Rationalization pre-blocking | `verificationAgent.ts` | Prevent known failure modes |
| Domain-specific strategies | `verificationAgent.ts` | Adapt behavior to context |
| Constraint-first with consequences | `compact/prompt.ts` | Prevent tool/format misuse |
| Parallel/sequential step numbering | `BashTool/prompt.ts` | Optimize multi-step execution |
| Hidden scratchpad (CoT) | `compact/prompt.ts` | Improve quality, strip reasoning |
| Sentinel output parsing | `verificationAgent.ts` | Enable programmatic consumption |
| Modular prompt composition | `constants/prompts.ts` | Runtime-configurable prompts |
| Adversarial probe seeds | `verificationAgent.ts` | Grounded creative exploration |
