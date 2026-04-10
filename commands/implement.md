---
description: Execute one phase of implementation plan
argument-hint: Phase [1|2|3|4|5]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task
---

# Execute Implementation Phase

Execute $ARGUMENTS of the current implementation plan.

## What This Command Does

1. Reads current plan from `docs/plans/PLAN-*.md`
2. Identifies which agent to use for the requested phase
3. Invokes agent via Task tool with context from plan
4. Updates plan checkboxes on completion

## Phase-to-Agent Mapping

| Phase | Agent | Creates |
|-------|-------|---------|
| Phase 1 | `@.claude/agents/persistence-developer.md` | Entities, repositories, Liquibase |
| Phase 2 | `@.claude/agents/integration-client-developer.md` | REST clients, DTOs, resilience patterns |
| Phase 3 | `@.claude/agents/domain-developer.md` | Facades, services, metrics |
| Phase 4 | `@.claude/agents/rest-api-developer.md` | Controllers, DTOs, .http files |
| Phase 5 | (1) `component-test-infrastructure.md` only if structure missing, then (2) `component-test-developer.md` | Test infrastructure (once per project), then BDD component tests for the story |

## Process

1. Read current plan file
2. Extract tasks for requested phase
3. Invoke appropriate agent(s) with plan context
   - **Phase 5:** (1) If component-test infrastructure is missing (see agent guard), invoke component-test-infrastructure once; (2) Always invoke component-test-developer (creates tests for the story from plan)
4. Agent(s) create code + tests + docs
5. Update plan checkboxes

## Agent Invocation Example

```
When user runs: /implement Phase 1

1. Read plan: PLAN-UCRM-001-create-order.md
2. Extract Phase 1 tasks:
   - Create Order entity
   - Create OrderRepository
   - Create Liquibase changelog
3. Invoke: Task tool with subagent_type="persistence-developer"
4. Provide context from plan + story + spec
5. Agent generates all Phase 1 deliverables
6. Update plan: mark Phase 1 tasks as complete
```

```
When user runs: /implement Phase 2

1. Read plan: PLAN-UCRM-001-create-order.md
2. Extract Phase 2 tasks:
   - Create InventoryServiceClient
   - Create client configuration and properties
   - Create request/response DTOs
3. Invoke: Task tool with subagent_type="integration-client-developer"
4. Provide context from plan + story + spec
5. Agent generates all Phase 2 deliverables
6. Update plan: mark Phase 2 tasks as complete
```

```
When user runs: /implement Phase 5 OR /implement component tests

1. Read plan: PLAN-XXX-001-....md
2. Extract Phase 5 tasks (component tests for story)
3. Check if component-test infrastructure exists (e.g. src/componentTest/.../BaseComponentTest.java, ContainersSetup.java, componentTest in build.gradle.kts).
   - If MISSING: Invoke Task tool with subagent_type="component-test-infrastructure". Agent will create structure. (Called at most once per project.)
   - If EXISTS: Do not invoke component-test-infrastructure; agent would no-op anyway.
4. Invoke Task tool with subagent_type="component-test-developer"
   - Provide context: plan + story from docs/stories (BDD scenarios) + spec
   - Agent creates: {Feature}ComponentTest.java mapping story scenarios to given/when/then tests
5. Update plan: mark Phase 5 tasks as complete
```

## Usage

```
/implement Phase 1  # Persistence layer
/implement Phase 2  # Integration clients
/implement Phase 3  # Domain logic
/implement Phase 4  # REST API
/implement Phase 5  # Component test infrastructure + BDD tests for story
```

## Notes

- Phase 5: infrastructure agent is invoked only when structure is missing (once per project); component-test-developer is always invoked for the story.
- Agents work autonomously based on plan context
- Plan checkboxes updated after agent completes
- Run `./gradlew check` (or `./gradlew componentTest` after Phase 5) after each phase
