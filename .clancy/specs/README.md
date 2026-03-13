# specs/

This directory contains requirements documents for this project.

Each file describes one area of concern. Ralph reads all files here during the
planning phase to produce `IMPLEMENTATION_PLAN.md`.

## Guidelines

- One file per milestone or topic area (e.g. `milestone-1-stacktrace.md`)
- Describe the *what* and *why*, not the *how* — let Ralph figure out the implementation
- Be specific about acceptance criteria so Ralph knows when a task is done
- Keep files focused; split large topics into multiple files

## Example structure

```
specs/
├── README.md              ← this file
├── milestone-1-foo.md     ← first milestone requirements
└── milestone-2-bar.md     ← second milestone (can be a placeholder)
```
