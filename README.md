# skillz

A collection of Claude skills — installable prompt modules that give Claude deep, book-grounded expertise in specific domains. Each skill lives in its own folder and is loaded by Claude on demand.

## Skills

### `candor` — Difficult Conversation Preparation
Helps you prepare for conversations you've been dreading or avoiding: delivering critical feedback, addressing a conflict, setting a boundary, or delivering unwelcome news. Grounded in the *Radical Candor* framework (Kim Scott), with a focus on being honest *because* you care, not in spite of it. Includes a role-play rehearsal option.

**Triggers on:** "I need to have a hard talk," "I've been avoiding this conversation," "I need to give tough feedback," "I don't know how to say this"

---

### `checklist` — Professional Checklist Builder
Creates short, precise, genuinely usable checklists following the principles from *The Checklist Manifesto* (Atul Gawande). Produces DO-CONFIRM or READ-DO checklists of 5–9 killer items — the steps most likely to be skipped under pressure — formatted to be pulled out mid-task and actually used.

**Triggers on:** "make me a checklist for X," "what should I check before doing Y," "build an SOP," "pre-flight check"

---

### `code_review` — Comprehensive Code Review
A multi-lens code review covering domain model integrity, reliability and operability, application security (OWASP-based), supply chain integrity, identity and access control, and code quality. Goes well beyond style checks to catch architectural problems, security flaws, and reliability gaps that linters miss. Grounded in *Looks Good to Me* (Braganza), the OWASP Code Review Guide, and several other references.

**Triggers on:** "review this code," "check my PR," "is this secure," "does this follow best practices"

Reference materials in `code_review/references/`:
- `comment-patterns.md` — Comment-writing framework (Triple-R, 5P process, MMG Exchange)
- `security-checklist.md` — Full OWASP security checklist with Go and Rust patterns
- `domain-model.md` — DDD checklist: aggregates, value objects, repositories, events
- `reliability.md` — SRE checklist: observability, error handling, resilience, deployment safety

---

### `organize_files` — File Organization (PARA Method)
Turns a cluttered folder into a navigable, maintainable structure using the PARA method (Projects, Areas, Resources, Archives) from *Building a Second Brain* (Tiago Forte). Organizes by where files are *going*, not where they came from. Works on macOS, Linux, and Windows.

**Triggers on:** "organize my files," "my files are a mess," "I can't find anything," "set up a folder structure"

---

### `strategy` — Strategy Development
Builds rigorous strategies using the kernel-based framework from *Good Strategy/Bad Strategy* (Richard Rumelt): a crisp diagnosis, a guiding policy that creates coherence, and coherent actions that follow from it. Also diagnoses and calls out bad strategy (fluff, goals mistaken for strategy, failure to face the challenge) before building anything new.

**Triggers on:** "what's our strategy," "how should we approach this," "we need a plan to win," "we're not growing," "how do we differentiate"

---

## Installation

These skills are designed for use with [Claude Code](https://claude.ai/claude-code) or any Claude agent that supports the skills plugin format.

To install, copy the skill folders into your Claude skills directory (typically `.claude/skills/`) or follow the plugin installation instructions for your Claude setup.

## License

MIT — see [LICENSE](LICENSE). Copyright © 2026 B Shouse.
