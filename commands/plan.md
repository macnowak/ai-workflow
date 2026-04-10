---
description: Create implementation plan from user story
argument-hint: [PROJECT]-[NUM]
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Create Implementation Plan

Create phased implementation plan for $ARGUMENTS.

## What This Command Creates

- File: `docs/plans/PLAN-{PROJECT}-{NUM}-{feature}.md`
- 4 phases with agent assignments
- Checkbox-based progress tracking
- Test scenarios per phase
- Completion criteria

## Process

1. Get current date
2. Read user story from `docs/stories/{PROJECT}-{NUM}-*.md`
3. Generate 4-phase plan:
   - Phase 1: Persistence (@persistence-developer)
   - Phase 2: Domain Logic (@domain-developer)
   - Phase 3: REST API (@rest-api-developer)
   - Phase 4: Integration & Testing
4. Identify which phases can be skipped

## Plan Template

```markdown
# Implementation Plan: {Feature}

**Story**: {PROJECT}-{NUM}
**Created**: {Date}

## BDD Scenarios

{Copy from story}

---

## Phase 1: Persistence Layer

**Agent**: `@.claude/agents/persistence-developer.md`
**Skip if**: Story doesn't introduce new entities

### Tasks
- [ ] Create {Entity} entity
- [ ] Create {Entity}Repository
- [ ] Create Liquibase changelog
- [ ] Write repository tests

### Completion Criteria
- [ ] All entities created
- [ ] Liquibase changeset added to master
- [ ] Tests pass: `./gradlew test`

---

## Phase 2: Domain Logic

**Agent**: `@.claude/agents/domain-developer.md`
**Skip if**: Pure CRUD with no business logic

### Tasks
- [ ] Create {Feature}Facade
- [ ] Create {Feature}Configuration
- [ ] Create {Feature}Metrics
- [ ] Create exception classes
- [ ] Write facade tests

### Completion Criteria
- [ ] Facade implements business logic
- [ ] Beans configured in Configuration class
- [ ] Tests pass: `./gradlew test`

---

## Phase 3: REST API

**Agent**: `@.claude/agents/rest-api-developer.md`
**Skip if**: Backend-only feature

### Tasks
- [ ] Create {Feature}Controller
- [ ] Create request/response DTOs
- [ ] Create .http files
- [ ] Write controller tests

### Completion Criteria
- [ ] All endpoints have .http files
- [ ] OpenAPI documentation complete
- [ ] Tests pass: `./gradlew test`

---

## Phase 4: Integration & Testing

### Tasks
- [ ] Run `./gradlew test`
- [ ] Run `./gradlew check`
- [ ] Manual testing with .http files
- [ ] Fix integration issues

### Completion Criteria
- [ ] All tests pass
- [ ] ArchUnit rules pass
- [ ] Manual verification complete
```

## Usage

```
/plan UCRM-001
```
