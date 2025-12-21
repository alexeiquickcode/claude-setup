---
description: Generate documentation for a module or codebase following the code-docs skill principles
---

ultrathink: Document the specified code following the principles in `.claude/skills/code-docs.md`.

**Your task:**
1. Read and understand the target code
2. Identify what type it is (application/infra vs algorithmic/library)
3. Write documentation that clarifies what the code cannot express on its own

**Input:** $ARGUMENTS (file path, directory, or module name to document)

If no arguments provided, document the current project's root README.

**Output:** Create or update the appropriate documentation file (README.md, docs/, or inline).

Remember: Be direct. No fluff. Code speaks for itself—docs fill the gaps.
