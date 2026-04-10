---
description: Perform automated code review of changes using the Code Review Agent to verify architecture compliance and code quality.
allowed-tools: Read, Bash, Grep, Glob, Task, Edit
---

# Review Command

Perform automated code review of changes made by agents or developers, verifying compliance with:
- **Architecture patterns**: Facade encapsulation, module boundaries, hexagonal architecture
- **Core principles**: KISS, YAGNI, SRP, DRY
- **Code quality**: Domain patterns, persistence rules, REST API standards, testing quality

## Instructions

This command delegates to the specialized Code Review Agent defined in `.claude/agents/code-reviewer.md`.

### Step 1: Determine Review Mode

Based on user invocation, determine the review mode:

**Quick Mode** (5-10 minutes):
- Invocation: `/review quick` or `/review fast`
- Focus: Critical issues only (facade encapsulation, public classes, Spring annotations)
- Use case: Pre-commit checks, after individual phases

**Thorough Mode** (15-30 minutes) - DEFAULT:
- Invocation: `/review` or `/review thorough`
- Focus: Full checklist (all patterns, principles, quality checks)
- Use case: Before PR, comprehensive quality check

**Layer-Specific Mode**:
- Invocation: `/review persistence`, `/review domain`, `/review api`, `/review client`
- Focus: Rules relevant to that layer only
- Use case: After completing a specific implementation phase

### Step 2: Invoke Code Review Agent

Use the Task tool to invoke the Code Review Agent:

```
Task tool with:
- subagent_type: "general-purpose"
- description: "Code review of recent changes"
- prompt: "You are the Code Review Agent from .claude/agents/code-reviewer.md.

Review mode: [quick/thorough/layer-specific]

Follow the review process defined in the agent documentation:
1. Identify changes (git diff, filter files)
2. Load rules (smart loading based on file types)
3. Execute review checklist
4. Generate report with severity levels
5. Offer to fix critical issues

CRITICAL: Focus on facade encapsulation pattern first - this is the most important check."
```

### Step 3: Handle Agent Response

The agent will return a code review report. Present it to the user with:
- Summary (files reviewed, issues found)
- Critical issues (must fix) 🔴
- Moderate issues (should fix) ⚠️
- Minor issues (consider) 💡
- Positive observations ✅
- Recommendations

### Step 4: Offer Next Steps

Based on the review results:

**If no issues found**:
```
✅ Code review PASSED - no issues found!

Next steps:
- Run ./gradlew check to verify Checkstyle and ArchUnit
- Create commit with /commit
```

**If only minor issues found**:
```
⚠️ Code review passed with minor suggestions.

Would you like to:
1. Proceed to commit (minor issues are optional)
2. Fix minor issues first
```

**If moderate or critical issues found**:
```
❌ Code review found [X] issues that need attention.

I can:
1. Fix all critical issues automatically
2. Fix specific issues you choose
3. Provide guidance on how to fix them manually

Would you like me to fix the issues?
```

## Review Modes Details

### Quick Mode (/review quick)
**Checks**:
- ✅ Facade encapsulation violations (orchestrators/controllers using repositories)
- ✅ Public visibility on internal classes (helpers, validators, configurations)
- ✅ Spring annotations on domain classes (@Component, @Service)
- ✅ Missing @Transactional on facade methods
- ✅ God classes (>500 lines)

**Skips**:
- Minor issues, style checks, documentation

**Time**: 5-10 minutes

---

### Thorough Mode (/review or /review thorough)
**Checks**: Everything from Quick Mode plus:
- Core principles (KISS, YAGNI, SRP, DRY)
- Domain patterns (rich entities, Tell Don't Ask)
- Persistence rules (JPA annotations, Liquibase migrations)
- REST API compliance (controller design, DTOs, .http files)
- Testing quality (BDD structure, coverage, meaningful tests)
- Code quality (naming, anti-patterns, Law of Demeter)

**Time**: 15-30 minutes

---

### Layer-Specific Modes

**Persistence Review** (/review persistence):
- JPA entity design
- Repository patterns
- Liquibase migrations
- Column type mapping

**Domain Review** (/review domain):
- Facade encapsulation
- Rich domain models
- Business logic placement
- Transaction boundaries

**API Review** (/review api):
- Controller design (thin controllers)
- DTOs and validation
- HTTP status codes
- .http test files

**Client Review** (/review client):
- REST client patterns
- Resilience patterns (CircuitBreaker, Retry)
- Request/Response DTOs
- Client tests

---

## What Gets Checked

### ✅ Code Review Agent Checks
- Architecture patterns (facade encapsulation, module boundaries)
- Design principles (KISS, YAGNI, SRP, DRY)
- Logic placement (controller vs facade vs entity)
- Business logic quality
- Test meaningfulness

### ⚠️ Build Tools Check (complementary)
- **Checkstyle**: Unused imports, naming, formatting
- **ArchUnit**: .http files, field injection, deprecated APIs
- **Gradle test**: Compilation, test execution

**Important**: Always run `./gradlew check` after code review to verify build tool checks.

---

## Common Violations Caught

### 🔴 Critical Issues

**1. Facade Encapsulation Violation**:
```java
// ❌ CRITICAL
public class ReserveOrchestrator {
    private final LimitRepository limitRepository;  // Should be LimitFacade!
}
```

**2. Public Internal Classes**:
```java
// ❌ CRITICAL
public class LimitChecker {  // Should be package-private!
    // Module boundary violation
}
```

**3. Spring Annotations on Domain**:
```java
// ❌ CRITICAL
@Service  // Don't use @Service on domain classes!
public class LimitFacade { }
```

### ⚠️ Moderate Issues

**1. Anemic Domain Model**:
```java
// ❌ MODERATE - Logic outside entity
public class LimitService {
    public boolean check(Limit limit, BigDecimal value) {
        return value.compareTo(limit.getThreshold()) <= 0;  // Should be in entity!
    }
}
```

**2. Method Too Long**:
```java
// ❌ MODERATE - 80+ lines
public void reserve(...) {
    // Too much logic in one method
}
```

### 💡 Minor Issues

**1. Missing Javadoc**:
```java
// 💡 MINOR
public void reserveLimit(...) {  // Add Javadoc
```

**2. Magic Numbers**:
```java
// 💡 MINOR
if (retryCount > 3) {  // Use named constant
```

---

## Usage Examples

```bash
# Quick pre-commit check (fast)
/review quick

# Comprehensive review before PR (default)
/review

# Thorough mode (explicit)
/review thorough

# Review only domain layer changes
/review domain

# Review only REST API changes
/review api

# Review only persistence layer changes
/review persistence
```

---

## Important Notes

- **Focus on facade encapsulation first** - this is the most critical architectural pattern
- **Don't duplicate build tool checks** - agent focuses on semantic understanding
- **Use smart rule loading** - only loads relevant documentation based on changed files
- **Be explicit about tools** - use Bash, Read, Grep, Glob tools appropriately
- **Offer to fix issues** - don't just report, help resolve

---

## Integration with Workflow

Code review fits into the AI-assisted workflow as **Phase 6**:

1. Specification → 2. Story → 3. Planning → 4. Implementation (Phases 1-4) → 5. Integration & Testing → **6. Code Review** → 7. Verification & Delivery

**When to use**:
- After completing all implementation phases
- Before creating commits
- Before creating pull requests
- After each individual phase (quick mode)

---

## Success Criteria

A code review is successful when:
- ✅ All critical issues resolved
- ✅ Code follows established patterns
- ✅ Tests cover new functionality
- ✅ No architectural violations
- ✅ Build tools pass (`./gradlew check`)

---

## Tips

1. **Review early and often**: Use quick mode after each phase
2. **Focus on critical first**: Fix blocking issues before proceeding
3. **Learn from reviews**: Good patterns → apply elsewhere, violations → avoid in future
4. **Integrate into workflow**: Make it part of your standard process
5. **Use for refactoring validation**: "Review my refactoring changes"

---

## Reference

- **Agent Documentation**: `.claude/agents/code-reviewer.md`
- **Core Principles**: `CLAUDE.md`
- **Architecture Patterns**: `docs/ARCHITECTURE.md`
- **Domain Rules**: `.claude/agents/domain-developer.md`
- **Persistence Rules**: `.claude/agents/persistence-developer.md`
- **REST API Rules**: `.claude/agents/rest-api-developer.md`
