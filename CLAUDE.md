# Designs

Published design documents. This repository is public.

## Rules

- Never push to main. Every addition or change is raised as a pull request from a branch.
- Run `git config core.hooksPath .githooks` once per clone. The pre-push hook refuses pushes to main.
- Topics sit at the root. Each document has its own directory under its topic, with README.md as the document.
- Diagrams go in the document's images/ directory: the SVG source and a PNG render of each. Documents embed the SVG.
- Every diagram carries a version stamp in its bottom right corner, with the same versions as the document header.
- Paths carry no version or status. Those belong in the title and the index in the root README.md.
- Every document states the product versions it describes in its header table, and the root README.md repeats them.
- List every published document in the root README.md.
