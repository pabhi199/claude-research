# Claude Code: Subagents & Skills — Why They Exist and When to Use Them

> **Reading time:** ~12 minutes  
> **Who is this for?** Developers and teams using Claude Code who keep hearing "subagents" and "skills" but aren't sure when or why to reach for them.  
> **What you'll leave with:** A clear mental model for both features, the exact moment each one becomes useful, and links to go deeper.

---

## Before We Begin: The Root Problem

Every Claude Code power user eventually hits the same wall.

You start a session. It's productive. You explore the codebase, discuss architecture, fix a few bugs. Then somewhere around hour two, something quietly breaks. Responses get vague. Claude starts contradicting decisions it made earlier. Suggestions drift from the patterns you established. The output quality degrades — not because Claude got worse, but because **the context window is full of noise**.

You have two choices: start a fresh session and lose everything you built up, or keep going and accept declining quality.

**Subagents and Skills are the answer to this problem** — but they each solve a different version of it. Understanding which problem each one solves is the key to using them well.

---

## The Mental Model: One Sentence Each

Before the details, plant these two sentences:

> **Skills** = "Instructions Claude should always follow for this type of task."

> **Subagents** = "A separate Claude that handles one job and reports back."

Skills are about *consistent behavior*. Subagents are about *isolated, parallel work*.

---

## Part 1: Skills

### Why Do Skills Exist?

Imagine you're onboarding a new developer. You don't explain the entire codebase every morning. You write a runbook, a style guide, a checklist — and you hand it to them once.

Without Skills, using Claude Code is like hiring that developer, throwing away the runbook, and re-explaining everything from scratch every single session. You'd write in CLAUDE.md: "When writing React, always use TypeScript, use named exports, test with Vitest." Then three weeks later you add: "When reviewing PRs, check for accessibility issues, check for console.logs, follow this checklist." Then another week: "When writing API endpoints, validate inputs with Zod, return consistent error shapes..."

CLAUDE.md becomes a wall of text. Every session loads all of it, even when you're just renaming a variable. The context gets heavy. Claude loses focus on what's actually relevant *right now*.

**Skills fix this by loading instructions only when they're needed.**

### What Is a Skill?

A Skill is a markdown file (`.md`) that lives in a designated folder. It contains a name, a description, and a set of instructions. That's it.

```
~/.claude/skills/
  pr-review/
    SKILL.md          ← instructions + trigger description
    checklist.md      ← optional supporting file

.claude/skills/       ← project-level, shared with team
  api-endpoint/
    SKILL.md
    error-shapes.json
```

The description you write in the SKILL.md is what Claude reads to decide *when to activate it*. When you ask Claude to review a PR, it matches your request against all available Skill descriptions and loads the matching one. The others stay out of the context entirely.

### Two Types of Skills

**Type 1: Capability Uplift**
These give Claude abilities it doesn't have natively — how to create a `.docx` file using your company's template, how to run your internal deploy script, how to generate a weekly report in your specific format.

**Type 2: Encoded Preferences**
These guide Claude to do things it already knows how to do — *your way*. "When writing tests, use this structure." "When writing commit messages, follow Conventional Commits." "When reviewing code, always check these five things."

### When Does a Skill Come Into the Picture?

Reach for a Skill when you notice yourself saying the same thing to Claude across multiple sessions. The clearest signals:

- You've typed the same set of instructions three times in different sessions
- CLAUDE.md is getting long with task-specific instructions that aren't always relevant
- You want your whole team to get the same Claude behavior without everyone configuring it individually
- You've built a multi-step workflow that Claude keeps doing slightly differently each time

If the answer to "should Claude always know this?" is no, but "should Claude know this when doing X?" is yes — that's a Skill.

### What a Skill File Looks Like

```markdown
---
name: pr-reviewer
description: >
  Use this when reviewing a pull request, checking code quality,
  or when the user says 'review my PR' or 'check this diff'.
---

# PR Review Instructions

## Always check for:
1. Missing input validation
2. console.log or debug statements left in
3. Tests that cover the happy path AND error cases
4. Accessibility issues in any UI changes
5. Consistent error response shapes for API routes

## Format your review as:
- **Blockers** (must fix before merge)
- **Suggestions** (worth fixing, not blocking)
- **Nits** (optional polish)

Do not approve if there are any Blockers.
```

Claude reads the `description` to know when to load this. Everything under it is what Claude does when it's loaded.

### Where Skills Live (Scope)

| Location | Who sees it | Use it for |
|----------|-------------|-----------|
| `~/.claude/skills/` | You, across all projects | Personal workflows, your coding style |
| `.claude/skills/` | Your whole team, in this repo | Team standards, project-specific processes |

Skills in the project folder get committed to the repo. Every developer who clones it gets the same Claude behavior automatically.

### The Skill vs. CLAUDE.md Decision

| Put it in CLAUDE.md | Put it in a Skill |
|--------------------|-------------------|
| Rules relevant to *every* session (architecture decisions, what not to touch, naming conventions) | Task-specific instructions (PR review checklist, deploy steps, doc format) |
| Project overview, stack, key file locations | Multi-step workflows with supporting files |
| Things that are always true | Things that are true only when doing X |

> **Simple rule:** If it bloats CLAUDE.md and is only relevant sometimes — it belongs in a Skill.

📖 **Official Reference:** [Claude Code Skills Documentation](https://docs.anthropic.com/en/docs/claude-code/skills)

---

## Part 2: Subagents

### Why Do Subagents Exist?

Here's a scenario. You ask Claude to implement a new feature. To do it well, Claude needs to:

1. Explore 40 files to understand the current codebase patterns
2. Read the existing test suite to understand testing conventions
3. Check the existing API spec to understand endpoint shapes
4. Write the feature itself
5. Write the tests
6. Review the changes for security issues

If all of this happens in one conversation, by step 4 the context is packed with intermediate findings from steps 1-3. By step 6, Claude is reviewing code while simultaneously carrying the memory of every file it read, every pattern it noted, every dead end it explored. The context is noisy, and the review quality suffers.

Now imagine you could spin up a *separate* Claude for step 1 — it explores files, comes back with findings, and then disappears. Another separate Claude for steps 4 and 5. A third separate Claude for step 6, which only sees the final code — not all the exploration baggage.

**That's what Subagents are.**

Each subagent gets its own fresh context window, a specific job, and a specific set of tools. It does its job and reports back. The main conversation stays clean.

### What Is a Subagent?

A Subagent is a pre-configured AI assistant that Claude Code can delegate tasks to. It has:

- Its own **system prompt** (what it knows and how it behaves)
- Its own **context window** (starts fresh every time, no history from the parent)
- Its own **tool permissions** (it can be restricted to read-only tools, for example)
- A **description** that tells Claude when to use it

The subagent's only connection to the parent conversation is:
1. The prompt the parent sends it ("Go explore these 40 files and summarize the patterns")
2. The result it sends back ("Here's what I found")

That's it. No shared memory. No shared context.

### The Two Key Superpowers

**1. Context Isolation**
The subagent's exploration work never pollutes the main conversation. It does the noisy work (reading files, trying things, backtracking) and only returns the clean signal. The main conversation stays focused on the actual goal.

**2. Parallelism**
Multiple subagents can run at the same time. During a code review, a style-checker, a security-scanner, and a test-coverage agent can all run simultaneously — what used to take minutes becomes seconds.

### When Does a Subagent Come Into the Picture?

The moment to reach for a subagent is when you have a **bounded side task** that:

- Needs to explore a lot of context (but you don't want that exploration cluttering the main session)
- Could run in parallel with other work
- Repeats across different projects and should behave consistently
- Needs different tool permissions than the main session (e.g., read-only access)

Practical examples:
- "Go audit all files in `/src/api` for input validation issues and give me a summary"
- "Read the full test suite and summarize the testing patterns being used"
- "Review this diff for security issues" (read-only, isolated from the implementation work)
- "Generate documentation for all the templates in this project" (can run in parallel per template)

### The Four Subagent Locations (Scope)

| Location | Scope | Use it for |
|----------|-------|-----------|
| `~/.claude/agents/` | You, across all projects | Personal specialists you bring everywhere |
| `.claude/agents/` | Your team, this project | Shared team specialists, committed to repo |
| Session-defined | Current session only | One-off agents you create on the fly |
| Plugin agents | Anyone with the plugin | Community agents installed via plugin |

When two agents in different locations have the same name, Claude resolves them in order: **session → project → user → plugin**. The most local definition wins.

### What a Subagent File Looks Like

```markdown
---
name: security-reviewer
description: >
  Expert security reviewer. MUST BE USED when reviewing code changes,
  pull requests, or when the user asks for a security review or audit.
  Specializes in injection attacks, auth issues, and data exposure risks.
tools:
  - Read
  - Grep
  - Glob
---

You are a senior security engineer reviewing code for vulnerabilities.

**Your focus areas:**
- SQL/command/path injection risks
- Authentication and authorization flaws
- Sensitive data exposure (keys, PII in logs)
- Insecure deserialization
- Missing input validation

**Rules:**
- You can ONLY read files. Never edit anything.
- Flag every issue with: Severity (Critical/High/Medium/Low), location, and remediation
- If you find a Critical issue, lead with it immediately
- Return a structured report, not a conversational response
```

Key things to notice:
- The **description** drives when Claude delegates to this agent
- The **tools** list restricts what it can do (read-only in this case — it physically cannot edit files)
- The **system prompt** shapes what it knows and how it reports back

### The Context Window Rule: The Most Important Technical Detail

This is the most common mistake people make with subagents:

> **The subagent starts with a fresh context. It has no idea what the parent conversation discussed.**

The only way to give a subagent context is through the prompt you send it. If the parent conversation decided "use PostgreSQL, not SQLite," the subagent doesn't know that unless you tell it explicitly in the delegation prompt.

This is by design. The fresh context is the whole point — it's what prevents pollution. But it means your delegation prompts need to be self-contained and specific.

**Weak delegation:**
```
"Review the recent changes."
```

**Strong delegation:**
```
"Review the changes in /src/api/payments.ts. We're building a payment 
processing module that handles Stripe webhooks. We decided to validate 
all inputs with Zod schemas. Check specifically for missing validation, 
hardcoded secrets, and anything that could expose card data in logs."
```

### Creating Your First Subagent

The fastest path is through Claude itself:
```
/agents
```
This opens the subagent interface. Choose "Create New Agent," describe what you want, select which tools it should have access to, and let Claude generate the initial prompt. Then customize it.

You can also just create the markdown file directly in `.claude/agents/`.

### Built-in Subagents (No Setup Required)

Claude Code ships with a general-purpose subagent you can use immediately without creating anything. If `Agent` is in your allowed tools (it is by default), you can say:

```
"Use a subagent to explore the /tests directory and summarize 
what testing patterns and conventions are being used."
```

Claude will spin up a fresh subagent for the exploration automatically. You don't need a custom definition for straightforward delegation tasks.

📖 **Official Reference:** [Claude Code Subagents Documentation](https://docs.anthropic.com/en/docs/claude-code/subagents)  
📖 **SDK Subagents Reference:** [Subagents in the Claude Code SDK](https://docs.anthropic.com/en/docs/claude-code/sdk/subagents)

---

## Part 3: How Skills and Subagents Work Together

They're not alternatives to each other — they solve different layers of the same problem.

Here's a realistic workflow that uses both:

**Goal:** Implement a new API endpoint, review it, and generate documentation.

```
Main conversation (Claude Code)
│
├── Delegates to: codebase-explorer subagent
│   └── Explores existing endpoints, returns patterns found
│       (fresh context, read-only tools, stays clean)
│
├── Uses: api-endpoint SKILL
│   └── Follows your team's standard for input validation,
│       error shapes, Zod schemas (loaded from .claude/skills/)
│
├── Implements the endpoint (main context, focused)
│
├── Delegates to: security-reviewer subagent (in parallel with →)
├── Delegates to: test-writer subagent
│   └── Both run at the same time, return results
│
└── Uses: documentation SKILL
    └── Generates docs in your team's standard format
```

The main conversation stays clean and focused on coordination. The dirty work (exploration, review, test generation) happens in isolated contexts. The team-standard behaviors (how endpoints are shaped, how docs are formatted) are encoded once in Skills.

---

## Part 4: The Full Layered System

Skills and Subagents exist within a larger set of Claude Code building blocks. Here's how they relate:

| Layer | What It Is | Problem It Solves |
|-------|-----------|------------------|
| **CLAUDE.md** | Project-wide rules, always loaded | "Claude doesn't know our project setup" |
| **Skills** | Task-specific instructions, loaded on demand | "I keep repeating myself every session" |
| **Subagents** | Isolated agents for specific tasks | "My context is getting polluted with exploration noise" |
| **Hooks** | Scripts that run automatically on tool events | "I want to enforce rules, not just suggest them" |
| **MCP Servers** | External tools and data sources | "Claude needs access to Jira / Slack / my database" |

Think of it as a stack. CLAUDE.md is always there as the foundation. Skills add task-specific behavior on top. Subagents handle the heavy lifting in isolation. Hooks enforce rules deterministically. MCP connects to the outside world.

---

## Common Mistakes to Avoid

**On Skills:**
- Don't put Skills content in CLAUDE.md — it loads every session even when irrelevant, adding noise
- Don't make Skill descriptions too broad ("use when writing code") — they'll fire when they shouldn't
- Don't make Skill descriptions too narrow ("use when writing a React component with TypeScript in /src/components/auth/") — they'll never fire

**On Subagents:**
- Don't assume the subagent knows what the parent discussed — always include needed context in the delegation prompt
- Don't use a subagent for a quick, simple task — the overhead isn't worth it for a two-line answer
- Don't give subagents more tool permissions than they need — a reviewer subagent shouldn't have write access

**On both:**
- Don't try to solve everything with one tool. "Should I use a Skill or a Subagent here?" — if the answer is "it's reusable team instructions," use a Skill. If the answer is "it needs its own context and possibly runs in parallel," use a Subagent.

---

## Quick Decision Guide

```
Is the work something Claude should just "know" when a task type comes up?
    YES → Skill
    NO  ↓

Is the work a bounded side task that needs its own context?
    YES → Subagent
    NO  ↓

Does it involve rules every session should follow?
    YES → CLAUDE.md
    NO  ↓

Does it involve real-time data or external services?
    YES → MCP Server
```

---

## References & Deep Dives

### Official Anthropic Documentation

| Topic | Link | What You'll Find |
|-------|------|-----------------|
| Subagents (Claude Code) | https://docs.anthropic.com/en/docs/claude-code/subagents | Full spec: file format, scope, tool permissions, examples |
| Subagents (SDK) | https://docs.anthropic.com/en/docs/claude-code/sdk/subagents | How SDK applications orchestrate subagents |
| Skills | https://docs.anthropic.com/en/docs/claude-code/skills | Skill file format, trigger descriptions, team sharing |
| CLAUDE.md | https://docs.anthropic.com/en/docs/claude-code/claude-md | Project config file that works alongside Skills |
| Hooks | https://docs.anthropic.com/en/docs/claude-code/hooks | Lifecycle scripts for enforcement and automation |

### Community Resources

| Resource | Link | Best For |
|----------|------|---------|
| 154+ Community Subagents | https://github.com/VoltAgent/awesome-claude-code-subagents | Ready-made agents across 10 categories (DevOps, Security, Testing, etc.) |
| Full Stack Guide (MCP + Skills + Subagents + Hooks) | https://alexop.dev/posts/understanding-claude-code-full-stack/ | Seeing all the layers together with real config examples |
| Skills Deep Dive (Firecrawl) | https://www.firecrawl.dev/blog/best-claude-code-skills | Real-world skills worth using today |
| Subagents Guide (Medium) | https://medium.com/@sathishkraju/claude-code-subagents-the-complete-guide-to-ai-agent-delegation-d0a9aba419d0 | Context window management and delegation patterns |

### Anthropic's Official Course
Anthropic has published dedicated courses for both topics:
- **Skills course:** Building, configuring, and sharing Skills across teams
- **Subagents course:** Context management, delegation, and specialized workflows

Find them at: https://docs.anthropic.com/en/docs/resources/courses

---

> 📌 **Last Updated:** June 2026  
> 📌 **Author:** *(your name)*  
> 📌 **Related Articles:** [Claude Code: MCP Explained](link) · [CLAUDE.md Best Practices](link) · [Claude Code Hooks Guide](link)
