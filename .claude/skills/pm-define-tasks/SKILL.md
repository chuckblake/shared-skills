---
name: pm-define-tasks
description: Turn a rough work idea into one or more well-specified tasks through targeted questions — never inventing details. Produces structured JSON ready to paste into any backlog system (Notion, Linear, GitHub Issues, etc.). Use when the user wants to define work before implementing: feature ideas, bug reports, problem statements, rough plans. Trigger on "define this task", "write up an issue for", "I want to build X", "break this down", "turn this into tasks", or any time the user describes work they want to spec before starting.
user-invocable: true
argument-hint: "[describe the work idea, or just start talking]"
---

# PM Define Tasks

Turn a rough work idea into one or more well-specified tasks through targeted questions. Output is structured JSON — paste it into any backlog system.

**The discipline that makes this valuable:** ask first, write second. Never invent context the user didn't provide. A well-specified task shapes everything that follows — bad specs multiply into bad work.

## When to use

Use when the user wants to spec work before doing it:

- Feature they want built
- Bug they want fixed
- Problem statement or complaint they want turned into a task
- Refactor or technical debt they want to address
- "Write up an issue for X", "I want to build Y", "break this down"

If the work is genuinely trivial (one-line fix, config change with no acceptance criteria worth writing), say so and suggest doing it directly. Save this skill for work that benefits from being written down.

## One task vs. many

Before asking questions, decide:

- **One task** — a single piece of work, one PR or one unit of effort. Use when the work fits in a few hours and has one clear outcome.
- **Many tasks (an epic)** — multiple independently completable tasks that together accomplish a larger goal. Use when the work decomposes into 2+ tasks with meaningful relationships.

Default to one task if in doubt. Don't manufacture epics from single-task work. If you're unsure, ask: "This looks like one task — do you see more here than I do?"

## Core principles

**Vertical slices, not horizontal layers.** Each task should deliver a working sliver of value end-to-end, not a layer (models, then controllers, then views).

**Independently completable.** Each task can be finished and shipped without requiring another task to be done first. Dependencies are the exception, not the rule.

**Single responsibility.** One task = one PR. If the description says "and also" or "then," that's two tasks.

**Right-sized.** A task is roughly 1–4 hours of focused work. Bigger → split. Tiny → combine.

**Out of scope is load-bearing.** Naming what a task should *not* touch prevents scope creep. Never omit it.

**Never invent details.** If you don't know something, ask. Do not fill in the goal, the trigger, the constraint, or the acceptance criterion if the user didn't provide it.

## The procedure

### Step 1: Restate and confirm

Read the user's idea. Restate it in 2–4 sentences covering:

- What the work accomplishes
- Why it exists (the underlying problem or opportunity)
- Rough scope — what's in, what's adjacent

Ask the user to confirm or correct before proceeding. Do not skip this step.

### Step 2: Call one task vs. many

State your call in one sentence. If the user disagrees, follow their lead.

If **one task**, skip Steps 3–5 and go to Step 6.

### Step 3: Ask 2–4 clarifying questions

Ask only the questions you genuinely need answered. Good questions:

- What's the success criterion — how will we know this is done?
- Are there constraints on the approach?
- What's the priority — ship something fast, or build it right?
- Are there parts already decided, or is everything open?

Don't ask more than 4 questions. If you need more, the idea is probably too big.

### Step 4: Identify slices (many-task only)

List the user-visible (or system-visible) outcomes that together compose the work. One sentence per slice. Note dependencies between slices.

Slices should be vertical where possible.

### Step 5: Identify scaffolding and spikes (many-task only)

Look for:

- **Setup work** — installing a dependency, adding configuration, creating a base abstraction. Often its own task if non-trivial.
- **Spikes** — genuine unknowns that need investigation before implementation can be specified.

### Step 6: Produce the output

Output one JSON object. For one task:

```json
{
  "type": "task",
  "tasks": [
    {
      "title": "...",
      "context": "2–4 sentences: why this exists and what came before it",
      "goal": "One sentence: what this task accomplishes when complete",
      "acceptance_criteria": [
        "Concrete, testable statement",
        "Concrete, testable statement"
      ],
      "out_of_scope": [
        "Things someone might think to do but shouldn't",
        "Future work that isn't part of this"
      ],
      "implementation_notes": "Optional hints about approach, files, patterns. Omit if nothing useful to say.",
      "dependencies": {
        "blocks": [],
        "blocked_by": []
      }
    }
  ]
}
```

For many tasks (an epic):

```json
{
  "type": "epic",
  "epic": {
    "title": "...",
    "context": "2–4 sentences: why this exists and what success looks like",
    "approach": "2–5 sentences: shape of the solution and why this slicing makes sense",
    "out_of_scope": [
      "Things adjacent to this epic that aren't part of it"
    ],
    "dependency_notes": [
      "task-02 and task-03 can run in parallel; both depend on task-01"
    ]
  },
  "tasks": [
    {
      "ref": "task-01",
      "type": "task",
      "title": "...",
      "context": "...",
      "goal": "...",
      "acceptance_criteria": ["..."],
      "out_of_scope": ["..."],
      "implementation_notes": "...",
      "dependencies": {
        "blocks": ["task-02"],
        "blocked_by": []
      }
    },
    {
      "ref": "task-02",
      "type": "spike",
      "title": "Spike: ...",
      "question": "The specific question this spike answers",
      "why_spike": "Why we can't specify the implementation yet",
      "investigation_approach": ["Step or thing to look at"],
      "deliverable": "A written decision or a follow-up task with concrete acceptance criteria",
      "time_box": "2 hours",
      "dependencies": {
        "blocks": [],
        "blocked_by": ["task-01"]
      }
    }
  ]
}
```

Output the JSON in a fenced code block. After the block, add a one-line summary of what was produced: how many tasks, whether it's an epic or standalone, and anything the user should review carefully.

## Anti-patterns to refuse

Redirect the user if their framing pushes toward these:

- **"Refactor X"** with no concrete trigger or done state. Push back: "What specifically should be different when this is done?"
- **"Improve performance"** with no measurable target. Push back: "What's the current number, what's the target?"
- **"Add tests"** as a standalone task. Tests come with their feature.
- **"Polish the UI"** as a single task. Almost always 5+ small tasks hiding inside.
- **"Migrate to X"** as a single task. These are epics — decompose them.
