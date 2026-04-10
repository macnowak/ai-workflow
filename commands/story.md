---
description: Create user story with BDD acceptance criteria from specification
argument-hint: [PROJECT]-[NUM] "{Story Description}"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, AskUserQuestion
---

# Create User Story

Create a user story for $ARGUMENTS with BDD acceptance criteria following agile best practices.

## What This Command Creates

- File: `docs/stories/{PROJECT}-{NUM}-{short-desc}.md`
- Feature statement (As a... I want... So that...)
- BDD acceptance criteria (Given/When/Then)
- Business rules and edge cases
- Link to parent specification

## Story Boundary Rules

A story must represent a **user-facing, demonstrable increment of value**. Ask: "Can a user or calling service actually use this once it ships?" If the answer is no, it is a task, not a story.

**Do NOT create separate stories for technical layers.** The following are tasks, not stories:
- "Add repository method X" — no user can call a repository method
- "Implement the domain facade" — a facade with no endpoint is invisible to users
- "Write component tests" — testing is part of every story, not a story of its own

**Persistence → domain → REST → tests is an implementation sequence.** It belongs in the *plan phases* (via the `/plan` command), not as separate stories.

**Valid reasons to split a spec into multiple stories:**
- A core feature vs. a distinct optional enhancement (e.g. JSON API vs. CSV export)
- Genuinely separable user workflows that each ship value independently
- Scope so large it clearly spans multiple sprints

**Default: one story per specification.** When in doubt, keep it together.

## Process

1. Get current date
2. **Find parent specification**:
   - Search `docs/specifications/` for files matching `SPEC-{PROJECT}-*.md` OR `{PROJECT}-*.md`
   - If multiple found, ask user which one to use
   - If none found, ask user if they want to proceed without a spec
   - Read spec content (works with any markdown format, not just `/spec` generated)
3. **Determine how many stories to create**:
   - If the spec's "Stories to Create" section lists stories, evaluate each one against the story boundary rules above
   - If listed stories are layer-based tasks (e.g. "implement repository", "implement facade"), consolidate them into one story
   - Only keep separate stories where there is a genuine business distinction
   - Ask the user if you are unsure about the right breakdown
4. **Extract context from spec** (if available):
   - Look for sections like: Features, Requirements, Data Models, API Contracts
   - Extract relevant information (flexible parsing, not strict template)
5. Ask clarifying questions if needed
6. Create story file(s) with BDD scenarios

## Story Template

```markdown
# User Story: {Feature Title}

**Story ID**: {PROJECT}-{NUM}
**Specification**: {Link to parent spec if found, or "Standalone"}
**Created**: {Date}

## Feature Statement

As a {user type},
I want {capability}
so that {benefit}.

## Business Rules

- Rule 1: Description
- Rule 2: Description

## Acceptance Criteria

### Scenario: {Happy Path}

\`\`\`gherkin
Given {context}
When {action}
Then {expected outcome}
And {additional outcome}
\`\`\`

### Scenario: {Error Case}

\`\`\`gherkin
Given {context}
When {invalid action}
Then {error handling}
\`\`\`
```

## Usage

```
/story UCRM-001 "Create new order via REST API"
```
