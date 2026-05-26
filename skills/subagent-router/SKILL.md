---
name: subagent-router
description: Route tasks to the right sub-agent with well-crafted prompts that follow best practices. Use before spawning any sub-agent for non-trivial work.
version: 1.0.0
author: User
---

# Subagent Router Skill

## Purpose

Route tasks to the most appropriate sub-agent and craft prompts that produce high-quality, correct-on-first-attempt results. This skill prevents the three most common sub-agent failure modes:

1. **Wrong agent** — sending a task to an agent type that lacks the right tools or expertise.
2. **Under-specified prompt** — the agent lacks context, guesses, and produces wrong output.
3. **Delegating understanding** — asking the agent to synthesize or decide instead of telling it what to do.

---

# Agent Selection

## Step 1: Read the Agent Catalog

Before selecting an agent, read the relevant agent file from `agents/` to understand its domain, tools, and best-use patterns. Each agent file follows this structure:

- **Purpose** — what it's built for
- **When to Use** — concrete triggers
- **When NOT to Use** — boundaries (if applicable)
- **Prompt Tips** — domain-specific advice
- **Example Prompt** — template to adapt

## Step 2: Match Task to Agent

Use this decision flow to narrow down which agent file to consult:

```
Task involves writing code?
├─ Yes → Is it a single language/framework?
│   ├─ Yes → Use the language-specific agent
│   └─ No → Use a multi-language or fullstack agent
└─ No → Is it analysis/design/review?
    ├─ Design/architecture → Plan or design-focused agent
    ├─ Security audit → security-review
    ├─ Code correctness → code-review
    ├─ Search/explore codebase → Explore
    └─ Documentation → documentation-engineer
```

If no specialist matches, use `general-purpose` or `claude` (catch-all with all tools).

Key selection principles:
- **Most specialized wins** — pick the agent with the narrowest domain match.
- **One agent per concern** — don't send a React question to a backend agent.
- **Explore is read-only** — for search/research, not for code changes.

## Step 3: Parallelize When Independent

When sub-tasks have no dependencies, spawn them in a **single message with multiple Agent tool calls**. Examples:

- Researching two unrelated parts of the codebase → two agents in parallel
- Code review + security review on the same diff → parallel
- Backend + frontend for the same feature → sequence (frontend depends on API contract)

---

# Prompt Crafting Rules

Every sub-agent prompt must satisfy these five rules.

## Rule 1: Self-Contained

The agent starts with **zero conversation context**. The prompt must stand alone.

```
BAD:  "Fix the bug I mentioned earlier."
GOOD: "Fix nil pointer dereference in user_service.go:142. When
       GetUser is called with an empty userID, the code dereferences
       user.Profile without checking if user is nil."
```

## Rule 2: Goal + Why

State what you're trying to accomplish and why. This lets the agent make judgment calls aligned with your intent.

```
GOOD: "We need to add rate limiting to the login endpoint because
       we saw 10K failed auth attempts last week. The goal is to
       block brute-force attacks without impacting legitimate users."
```

## Rule 3: What You Already Know

Describe findings, ruled-out approaches, and constraints. Prevents the agent from re-investigating dead ends.

```
GOOD: "I've already checked: the DB connection pool is fine (50 conns,
       80% idle), the index on user_id exists, and the query plan shows
       an index scan. The slowness is in the application layer after
       the query returns."
```

## Rule 4: Specific, Not Vague

Hand over file paths, line numbers, function names, and exact instructions. Never say "based on your findings, fix the bug" — that delegates understanding.

```
BAD:  "Look at the auth code and fix any issues you find."
GOOD: "In auth/middleware.go:85-120, the token validation loop uses
       string concatenation instead of strings.Builder. Refactor it
       to use strings.Builder and add a benchmark. The test file is
       auth/middleware_test.go."
```

## Rule 5: Cap the Response

When you only need a specific deliverable, say so.

```
GOOD: "Report in under 200 words: is this migration safe, and if not,
       what specifically breaks?"
```

---

# Workflow

## For Every Sub-Agent Task

1. **Analyze** — domain, type (code/research/review/plan), scope, dependencies.
2. **Select** — read the matching agent file from `agents/`, confirm it fits.
3. **Prompt** — craft using all five rules. Verify with checklist:
   - [ ] Can a stranger understand this without our conversation?
   - [ ] Did I explain why this matters?
   - [ ] Did I share what's already known/tried?
   - [ ] Did I give specific files, lines, and instructions?
   - [ ] Did I set a response cap if the task is narrow?
4. **Mode** — foreground if next step depends on result, background if fire-and-forget.
5. **Verify** — when agent returns, read actual changed files before reporting to user. Agent summaries can be wrong.

---

# Common Patterns

## Research Before Implementation

1. Explore agent → find relevant files and patterns
2. Review findings yourself → understand the landscape
3. Craft a specific prompt for the implementation agent
4. Implementation agent → make the changes

## Independent Reviews

Send code review and security review in parallel — no dependencies.

## Multi-Step with Dependencies

1. Plan agent → design the approach
2. Review the plan yourself → refine if needed
3. Implementation agent → execute with the plan as context

---

# Anti-Patterns

| Anti-Pattern | Why It Fails | Fix |
|-------------|--------------|-----|
| "Based on your findings, fix the bug" | Delegates synthesis to the agent | Read findings yourself, write specific instructions |
| Overly broad prompt | Agent guesses scope, misses details | Narrow scope to what's needed |
| Wrong agent for domain | Lacks tools or vocabulary | Check the agent file in `agents/` |
| No response cap on narrow questions | Wastes context on verbose output | Add "under 200 words" or similar |
| Agent for trivial tasks | Overhead > benefit | Just use Read/Grep/Edit directly |
| Skipping verification | Agent summaries can be wrong | Read actual changed files |
