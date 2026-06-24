---
title: "I spend more time writing instructions for my AI than writing code. It changed how I work."
date: 2026-03-25
summary: "How I turned Claude Code from an eager intern into a real dev partner — layer by layer, failure by failure."
tags: ["ai-tools", "claude-code", "workflow", "developer-experience"]
draft: false
---

*I spend more time writing instructions for my AI than writing code. It changed how I work.*

---

## The eager intern

I work as a backend developer on an embedded energy management system — multiple repos, a mix of Python and C++, a small team. I've been using Claude Code daily for several months now, across both work and personal projects.

Out of the box, Claude Code is surprisingly capable. It reads your code, understands context, proposes reasonable changes. It's not the chaos some people describe.

But it doesn't know *you*. It doesn't know your naming conventions, your commit message style, that you never put comments on obvious getters, or that your team writes PR descriptions in a specific format. It's verbose where you'd be terse. It adds docstrings you didn't ask for. It makes design decisions on its own — reasonable ones, sure, but not *yours*.

It's like onboarding a solid developer who hasn't read the team's contributing guide yet. The code compiles, the logic is fine, but the review comments pile up: "we don't do it this way here."

I got tired of repeating myself. So I started treating Claude Code not as a tool to use, but as a teammate to onboard.

## Layer by layer

Claude Code's configuration is a stack. Each layer adds precision, and they build on each other. (If you want the full reference, Anthropic documents this in their [Claude Code configuration guide](https://code.claude.com/docs/en/features-overview).)

{{< mermaid >}}
flowchart TB
    A["Skills & Commands\n/review · /commit · /create-pr"]
    B["Rules\ngit-conventions · lint-policy\nauto-introspection"]
    C["CLAUDE.md\nproject overview · pointers to rules"]
    A --> B --> C
    style A fill:#4a6fa5,color:#fff
    style B fill:#6b8cae,color:#fff
    style C fill:#8faab7,color:#fff
{{< /mermaid >}}

### CLAUDE.md — the onboarding doc

A markdown file at the root of your project that Claude reads automatically. Think of it as the contributing guide you'd write for a new hire: project architecture, common commands, code conventions, and pointers to more specific rules and skills. It describes the *what* — what this project is, how it's structured, where to look for guidance.

### Rules — the behavioral contract

Rules live in `.claude/rules/` and define specific behaviors:

- **Coding style** — git conventions, commit format, never disable a linter warning without asking first
- **Guardrails** — never post on Slack or comment on a merge request without my approval, never resolve review comments yourself
- **Investigation discipline** — frame the problem before deep-diving, start with the simplest hypothesis, never publish an unverified claim externally

That kind of knowledge usually lives in one person's head — now it's in the config, available to every conversation.

My favorite is the auto-introspection rule — a self-improvement loop built into the system:

{{< mermaid >}}
flowchart LR
    A["I correct Claude"] --> B["Claude analyzes\nwhat went wrong"]
    B --> C["Proposes a\nconfig update"]
    C --> D["Next conversation\nstarts better"]
    D -.-> A
{{< /mermaid >}}

### Skills — repeatable workflows

Skills are triggered by slash commands. When I type `/review`, Claude doesn't just read the diff — it follows a structured process: check CI status, analyze coverage gaps, look for patterns across the codebase, format feedback in a specific way. `/commit` stages files, runs linters, writes a conventional commit message. `/create-pr` generates a description with summary and test plan.

But skills aren't just about automation. My `/implement` skill is a good example. Before writing any code, it reads the linked issue and challenges the technical choices — questioning assumptions, spotting edge cases, flagging inconsistencies. It's caught design gaps that were missed during the initial review. The implementation only starts once the challenge step is done.

### Memory — context that persists

Each conversation with Claude starts fresh — unless you give it memory. Claude Code has a built-in memory system that stores context across conversations: user preferences, project decisions, ongoing work, external references. When I make a decision in one conversation ("we're going with approach X for this module"), it's available in the next one without me repeating it.

It's the difference between a colleague who remembers last week's architecture meeting and one who asks you to explain everything from scratch every morning.

### The compounding effect

The real power isn't any single layer. It's how they build on each other. Each rule I add, each skill I refine, each memory saved, makes every future interaction slightly better. Claude doesn't forget. It doesn't have a bad day. The configuration *is* the institutional knowledge.

## Scaling across repos: claude-nexus

This worked great — until I had multiple projects. Each with its own `.claude/` directory. Same rules copy-pasted across repos. Fix a rule in one, forget to update the others. Classic config drift.

So I did what any developer would do: I built a tool. [claude-nexus](https://github.com/corentin-core/claude-nexus) is a repo that holds all my shared Claude Code configuration and deploys it via symlinks:

{{< mermaid >}}
flowchart TB
    CN["claude-nexus"] --> SH["shared/\nrules · skills · commands\ndeployed to ALL projects"]
    CN --> PR["projects/\nbudget-forecaster · work-project\nproject-specific overrides"]
    CN --> DP["deploy.sh\none script, all repos in sync"]
    SH --> P1["Project A\n.claude/ (symlinks)"]
    SH --> P2["Project B\n.claude/ (symlinks)"]
    SH --> P3["Project C\n.claude/ (symlinks)"]
    style CN fill:#e07a5f,color:#fff
    style SH fill:#3d405b,color:#fff
    style PR fill:#3d405b,color:#fff
    style DP fill:#3d405b,color:#fff
{{< /mermaid >}}

One `deploy.sh`, all repos in sync. Project-specific config overrides shared defaults when needed.

It's not glamorous. But it solved a real problem: maintaining a coherent AI assistant across an entire project ecosystem.

There's another reason this approach works well: in a team, everyone has their own habits and preferences with AI tools. We haven't agreed on a shared Claude Code configuration yet — so committing `.claude/` into each repo isn't an option. Managing the config on the side, in a personal repo, sidesteps that problem entirely. My setup doesn't interfere with anyone else's workflow.

## Where I actually get value

Claude generates code — that's the obvious use case, and it works. But if I had to pick the moments where it brings the most value, they wouldn't be about code generation. They'd be about everything around it.

### Exploring and thinking

Developers have long known about [rubber duck debugging](https://en.wikipedia.org/wiki/Rubber_duck_debugging) — explaining your problem out loud to a rubber duck to find the answer yourself. Claude is my rubber duck. Except this one talks back.

When I'm facing an architectural decision — how to structure a new module, whether to go event-driven or direct calls — I don't ask Claude to decide for me. I think out loud with it:

- I describe the problem, the constraints, the tradeoffs I see
- Claude pushes back, asks questions I hadn't considered, suggests alternatives
- The architecture still comes from me — but sharper and faster

**One caveat:** this only works if you know your codebase well. Claude can be wrong. It can skim the surface and give you a confident-sounding answer that misses the real complexity underneath. If you don't have the depth to push back, you'll get led astray without realizing it. The rubber duck is useful precisely because *you* are the one doing the thinking.

This might be the most underrated use case. Not "write me a function" but "help me think through this design."

### Automating the boring stuff

Opening a PR with a proper description. Creating GitHub issues with the right labels. Formatting a Slack message to summarize a technical decision. These tasks aren't hard, but they're friction. They break your flow.

I've turned most of these into skills and commands:

- **`/create-pr`** — generates a description with summary and test plan
- **`/commit`** — stages files, runs linters, writes a conventional commit message
- **`/handle-review-comments`** — processes merge request feedback one comment at a time, then runs an introspection step to detect patterns and propose rule updates
- **`/trace-change`** — git archaeology: given a commit, traces the full path from issue to merge request to first release

The time saved per task is small — maybe five minutes. But multiplied by dozens of times a week, it adds up. And more importantly, I stay in the terminal, in the flow, focused on the work that matters.

### Giving Claude better tools

Not everything should be an LLM call. Some tasks are better handled by deterministic scripts — faster, lighter on your token quota, and more reliable. The trick is to build those scripts and let Claude orchestrate them.

At work, investigating production incidents used to mean feeding Claude 100-200k tokens of raw logs. Slow, noisy, and often inconclusive. So I built a set of deterministic Python checks — things like detecting recurring timeout patterns in service logs, flagging disk space or memory issues, or spotting configuration inconsistencies. Each check parses specific log files, matches known patterns, and outputs structured findings. No LLM involved.

{{< mermaid >}}
flowchart LR
    A["Raw logs"] --> B["Layer 1\nDeterministic scripts\n(instant, reliable)"]
    B --> C["Layer 2\nLLM synthesis\n(small context)"]
    C --> D["Layer 3\nFull investigation\n(only if needed)"]
    style B fill:#2d6a4f,color:#fff
    style C fill:#40916c,color:#fff
    style D fill:#95d5b2,color:#000
{{< /mermaid >}}

Most incidents get resolved at layer 1 or 2. The LLM only kicks in on a small, pre-digested context — not on raw noise.

I keep adding new checks as I encounter new incident patterns. The deterministic layer grows over time — and each new check benefits every future investigation.

The broader lesson: an AI assistant becomes much more powerful when you give it specialized tools to call. Instead of asking a general-purpose model to do everything, you build the right script for the job and let the model decide when and how to use it.

### Orchestrating parallel work

When I have multiple independent tasks — a refactor, a bug fix, tests to write — I want them to run in parallel. I built an `/orchestrate` skill for this:

{{< mermaid >}}
flowchart TB
    O["Orchestrator"] --> W1["Worker 1\nworktree A\nrefactor"]
    O --> W2["Worker 2\nworktree B\nbug fix"]
    O --> W3["Worker 3\nworktree C\ntests"]
    W1 --> SC["Shared context\n(messages, staging)"]
    W2 --> SC
    W3 --> SC
    SC --> V["Validation\n& merge"]
{{< /mermaid >}}

Each worker is a standalone `claude -p` process launched in its own git worktree, with explicit permission boundaries. Why not use Claude's built-in agent spawning? Two reasons:

- **Permissions**: native sub-agents inherit the parent's working directory and permissions. Every `cd` to a worktree triggers a permission prompt. It defeats the purpose of autonomous work.
- **Context isolation**: each worker gets a clean, focused context — just the code it needs and its specific task. No pollution from other ongoing work. The orchestrator keeps the global view while workers stay focused on their slice.

Building around this limitation gave me finer control — each worker runs independently, with no interactive approvals needed.

A concrete example: I used this to migrate a module out of a monorepo into multiple separate repos. Manually, that would have been a full day of navigating between Claude instances, copying context, switching branches. With `/orchestrate`, I had 6 merge requests ready to review in under two hours.

## The limits

If the previous sections sound like a success story, let me balance it out. Working with an AI assistant at this level comes with real friction.

**Review stays mine.** The review is the last gate before code enters the codebase. That's where I take responsibility — and that's not something I can delegate to an AI. I sometimes ask Claude for a second opinion on a diff, but the final call is always mine.

**Big tasks need slicing.** You can't just throw a task at Claude, say "figure it out," and expect magic. Give it a large, vague problem and the result will be mediocre — shortcuts, surface-level fixes, missed root causes.

The real skill is breaking work into small, well-defined pieces — and knowing when to step in to make a design choice that Claude can't make for you. Effective AI-assisted development is essentially project management: decompose, scope, sequence, review.

**Context switching is exhausting.** Parallel agents sound great on paper. In practice, I'm the bottleneck. I'm constantly switching between contexts, reviewing output, course-correcting. It's mentally draining in a way that writing code sequentially never was.

**The ownership trap.** This is the big one. When Claude writes code, it compiles, it passes tests, it looks clean. It's tempting to merge and move on. But if you can't explain *why* the code works — if you can't trace the logic, spot the assumptions, predict where it'll break — you don't own it. You're just a reviewer who approved something you didn't fully understand. And that debt compounds silently until something breaks in production and you have no idea where to start.

## The shift

A few months into this setup, I noticed something had changed in my daily work. I was spending less time writing code and more time writing instructions, reviewing output, and making decisions. Less typing, more thinking.

At first it felt wrong — like I wasn't doing my job anymore. Then I realized: the job had changed. I wasn't a developer who uses an AI tool. I was designing how an AI develops:

- Defining conventions
- Building guardrails
- Reviewing work
- Making architectural calls

The code was still getting written — just not by me, most of the time.

So what *is* the job now? For me: you become the architect, the reviewer, and the quality gate. You define what "good" looks like, and you make sure it gets enforced — consistently, across projects, over time.

That's why I spend more time writing instructions than writing code. It changed how I work — and I'm not going back.

---

*This is one way to work with AI. I'm exploring others — including flipping the model entirely and using it to learn rather than produce. More on that when I've lived it enough to write about it honestly.*

*Note on authorship: drafted with Claude (Anthropic) and rewritten across multiple passes. The decisions and the journey itself are mine.*
