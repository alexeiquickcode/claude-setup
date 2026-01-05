Initiate code-reviewer subagent to review **uncommitted changes only**.

Scope: Run `git diff` to identify modified files and review only those files plus their direct dependencies. This is for pre-commit validation.
