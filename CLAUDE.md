# Designs

Published design documents. This repository is public.

## Rules

- Never push to main. Every addition or change is raised as a pull request from a branch.
- Run `git config core.hooksPath .githooks` once per clone. The pre-push hook refuses pushes to main.
- Topics sit at the root. Each document has its own directory under its topic, with README.md as the document and its diagrams beside it.
- Paths carry no version or status. Those belong in the title and the index in the root README.md.
- List every published document in the root README.md.
