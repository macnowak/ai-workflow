# Claude Agents Overview

This directory contains specialized agents for the Operations Limits Service project. Each agent is responsible for a specific layer or aspect of the application.

## Available Agents

### 🏗️ Implementation Agents

| Agent | Purpose | Key Responsibilities |
|-------|---------|---------------------|
| **[domain-developer](domain-developer.md)** | Domain/Business Logic | Facades, domain entities, business rules, configuration |
| **[persistence-developer](persistence-developer.md)** | Data Persistence | JPA entities, repositories, Liquibase migrations |
| **[rest-api-developer](rest-api-developer.md)** | REST API Layer | Controllers, DTOs, OpenAPI specs, HTTP tests |
| **[integration-client-developer](integration-client-developer.md)** | External Integrations | REST clients, DTOs, resilience patterns |
| **[component-test-infrastructure](component-test-infrastructure.md)** | Test Infrastructure | TestContainers setup, test utilities |
| **[component-test-developer](component-test-developer.md)** | Component Tests | BDD test scenarios, test implementation |

### 🔍 Quality Assurance Agents

| Agent | Purpose | When to Use |
|-------|---------|-------------|
| **[code-reviewer](code-reviewer.md)** | Automated Code Review | After any code changes, before commits, during PR creation |

## Using the Code Review Agent

### Quick Start

```bash
# Review all modified files
"Review my recent changes"

# Review specific files
"Review the ReserveOrchestrator changes"

# Check compliance with rules
"Check if I followed the facade encapsulation pattern"

# Before committing
"Run code review before I commit"
```

### What It Checks

The Code Review Agent validates:
- ✅ **Architecture patterns** (Facade Encapsulation, Module Boundaries)
- ✅ **Core principles** (KISS, YAGNI, SRP, DRY)
- ✅ **Layer-specific rules** (Domain, Persistence, API)
- ✅ **Code quality** (Naming, style, complexity)
- ✅ **Testing** (Coverage, quality, BDD structure)

### Review Severity Levels

- 🔴 **Critical**: Must fix (architecture violations, broken patterns)
- ⚠️ **Moderate**: Should fix (code quality, missing tests)
- 💡 **Minor**: Consider (style, documentation)

### Example Review Output

```
## 🔍 Code Review Results

### Summary
- Files Reviewed: 3 files
- Issues Found: 2 issues (1 critical, 1 moderate)
- Overall Status: ⚠️ NEEDS IMPROVEMENT

### Critical Issues 🔴
1. [Architecture] ReserveOrchestrator.java:23
   Problem: Direct dependency on CounterRepository
   Fix: Use CounterFacade instead
```

## Agent Workflow

### Typical Development Flow

```
1. User requests feature
   ↓
2. Appropriate implementation agent executes
   ↓
3. Code Review Agent automatically reviews changes
   ↓
4. Issues reported and optionally fixed
   ↓
5. User commits clean code
```

### Manual Review Flow

```
1. User makes changes manually
   ↓
2. User requests: "Review my changes"
   ↓
3. Code Review Agent analyzes git diff
   ↓
4. Reports violations and suggestions
   ↓
5. User fixes issues or agent offers to fix
```

## Creating New Agents

When creating a new agent, follow this structure:

```markdown
---
name: Agent Name
description: Brief description of agent's purpose
---

# Agent Name

## Purpose
Clear statement of what this agent does

## When to Use This Agent
Specific scenarios and triggers

## Responsibilities
Detailed list of what agent creates/modifies

## Implementation Guidelines
Step-by-step process

## Quality Checks
What to verify before completion

## Examples
Concrete examples of agent usage
```

## Best Practices

### For Implementation Agents
1. **Read existing patterns** before generating code
2. **Follow layer-specific rules** strictly
3. **Generate tests** alongside implementation
4. **Run code review** after completion
5. **Document decisions** in code comments

### For Code Review Agent
1. **Be specific** - provide line numbers and exact issues
2. **Provide context** - explain why it's a violation
3. **Reference rules** - point to documentation
4. **Suggest fixes** - don't just identify problems
5. **Acknowledge good patterns** - positive feedback matters

## Integration with Development Process

### Pre-Commit Hook (Recommended)
```bash
# In .git/hooks/pre-commit
"Run code review on staged changes"
# Blocks commit if critical issues found
```

### CI/CD Pipeline
```yaml
# In .gitlab-ci.yml
code-review:
  script:
    - "Automated code review of all changes"
  allow_failure: false  # Fail build on critical issues
```

### Pull Request Template
```markdown
## Pre-Merge Checklist
- [ ] Code review agent passed
- [ ] All tests passing
- [ ] Documentation updated
- [ ] No critical issues
```

## Troubleshooting

### "Agent didn't catch my violation"
→ The rule might not be in the checklist yet
→ Submit feedback to improve detection

### "False positive reported"
→ Review the rule interpretation
→ Update agent documentation if needed

### "Too many minor issues"
→ Adjust severity threshold in configuration
→ Focus on critical and moderate first

## Continuous Improvement

The agents in this directory evolve based on:
- **Team feedback** on violations and false positives
- **New patterns** introduced in the codebase
- **Architectural decisions** documented in ADRs
- **Common mistakes** that should be prevented

To improve an agent:
1. Identify the gap or issue
2. Update the agent's checklist/rules
3. Add examples of violations
4. Test with real code
5. Document the change

## Getting Help

- **General guidance**: See `../CLAUDE.md`
- **Architecture patterns**: See `../docs/ARCHITECTURE.md`
- **Agent-specific help**: See individual agent files
- **Code review rules**: See `code-reviewer.md`

---

**Remember**: These agents are tools to enforce consistency and quality. They should help, not hinder. If a rule doesn't make sense in context, discuss it with the team.
