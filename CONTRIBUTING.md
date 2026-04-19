# Contributing

## Philosophy

This project is a **starting point, not a destination**.

The core idea — your skills need a guardian that improves them through
use — is simple enough to adapt to any workflow. The AI landscape
evolves too fast for a centralized project to stay canonical. Every
developer's stack, conventions, and LLM preferences are different.

This repository is designed to be **forked and adapted**. Take the
guardian concept, reshape it to fit your workflow, and make it yours.
The forge teaches you to forge — then you forge your own way.

## How to use this project

### Fork, don't PR

Rather than contributing back to this repository:

1. **Fork it** — make it your own
2. **Adapt guardian-nurturer** — change the guardian template, add new
   review modes, modify the self-improvement mechanism, integrate
   a different trace analysis tool
3. **Bring your own skills** — from any source. The guardian works
   with any skills, not just skills-forge output
4. **Evolve your guardians** — they'll diverge from the template
   through use, and that's the point
5. **Share your fork** — if your adaptation is useful, publish it
   as its own project with its own identity

### What to adapt

| What | Where | How you might change it |
|------|-------|------------------------|
| Guardian template | guardian-nurturer.md | Different review modes, different checklist structure, different self-update rules |
| Self-improvement rules | guardian-nurturer.md, Key Principles | Stricter/looser evidence requirements, auto-approve mode, different append policies |
| Trace analysis | guardian-nurturer.md, VCC Integration | Replace VCC with a different session analysis tool |
| Skill building (optional) | skills-forge.md | Add phases, change source ingestion, different skill format |
| Packaging (optional) | agent-packager.md | Different distribution targets, different validation |
| Feedback loop | skills-forge.md, Phase 8 | Different memory structure, different lesson extraction |

### The ecosystem we hope for

The ideal outcome is not one canonical guardian framework, but a
flourishing ecosystem of forks — each adapted to a different workflow,
stack, or team. Some possible directions:

- A fork with guardians for Copilot/Cursor skills instead of Claude Code
- A fork that replaces VCC with a different trace analysis tool
- A fork with fully automated guardian loops (no user approval needed)
- A fork focused on a specific domain (game dev, web, data science)
- A fork that adds automated eval pipelines for guardian effectiveness
- A fork with multi-guardian orchestration across skill domains
- A fork that integrates guardian findings into CI/CD gates

Each fork is its own *faber* — shaped by the practice of its maker.

## Reporting issues

If you find a bug in the **base framework** (broken YAML frontmatter,
incorrect guardian template structure, VCC integration error), you are
welcome to open an issue. We'll fix foundational problems that affect
all forks.

But feature requests, workflow preferences, and "it should work
differently" suggestions are better addressed in your own fork. The
framework is intentionally minimal so it's easy to reshape.

## License

MIT.
