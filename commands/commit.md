---
description: Analyze the current working directory changes and create a commit following conventional commits format.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task, Git
---

# Commit Command

Analyze the current working directory changes and create a commit following conventional commits format.

## Instructions

1. Run `git status` to see all untracked and modified files
2. Run `git diff` for staged changes and `git diff HEAD` for all changes
3. Run `git log -5 --oneline` to see recent commit style
4. Analyze the changes and determine:
   - The type: `feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `style`, `perf`, `ci`, `build`
   - The scope: Extract from branch name if it contains a ticket (e.g., `feature/VII-2642-description` → `VII-2642`), or ask user
   - Clear, concise description of what changed and why
5. Create commit with format: `type(TICKET): description`
   - Use present tense ("add feature" not "added feature")
   - Keep description concise but meaningful
   - Focus on "what" and "why", not "how"
6. Create the commit using heredoc format:
   ```bash
   git commit -am "$(cat <<'EOF'
   type(TICKET): description

   EOF
   )"
   ```
7. Run `git status` after commit to verify

## Conventional Commit Types

- **feat**: New feature
- **fix**: Bug fix
- **chore**: Maintenance tasks, dependency updates
- **refactor**: Code restructuring without functionality change
- **docs**: Documentation changes
- **test**: Adding or updating tests
- **style**: Code style/formatting changes
- **perf**: Performance improvements
- **ci**: CI/CD configuration changes
- **build**: Build system changes (Gradle, Maven, dependencies)

## Important Notes

- NEVER push to remote unless explicitly asked
- If scope/ticket cannot be determined from branch name, ask the user
- If there are no changes to commit, inform the user
- Follow the existing commit style shown in git log
