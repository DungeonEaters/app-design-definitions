# App Design Definitions — Agent Instructions

## Purpose

Central repository for design and architecture documentation across applications and modules. This is the **source of truth** for design decisions, architecture, and non-functional requirements.

---

## Structure

```
applications/
├── eras-defense/
│   ├── architecture.md          # Main design doc
│   ├── implementation-plan.md   # Execution roadmap
│   ├── backlog.md               # Feature backlog & decisions
│   └── decisions.md (optional)  # ADRs or trade-offs
│
└── <other-app>/
    ├── architecture.md
    └── ...

template-architecture.md  # Template for new applications
```

---

## Key Convention: Link, Don't Duplicate

- **Document design decisions, architecture, and scope** in this repo
- **Link to implementation details** in respective app repos (e.g., code structure, build commands, CI/CD)
- **Avoid copying** code examples, CLI commands, or runtime details that belong in the implementation repo

Example: This repo explains *why* we use VContainer and event channels; [merge-tower-unity/AGENTS.md](../merge-tower-unity/AGENTS.md) explains *how* to use them in code.

---

## Files & Their Purpose

| File | Purpose | Audience |
|------|---------|----------|
| `architecture.md` | Comprehensive design: context, scope, requirements, architecture, components | PMs, Designers, Tech Leads, Senior Engineers |
| `implementation-plan.md` | Step-by-step execution roadmap with phases and milestones | Engineers, Project Leads |
| `backlog.md` | Feature backlog, prioritization, risks, decisions log | PMs, Engineers, Tech Leads |
| `decisions.md` (opt.) | Architecture Decision Records (ADRs), trade-offs, open questions | Tech Leads, Senior Engineers |

---

## Common Tasks for AI Agents

### Task: Update Architecture After Implementation Discovery
1. Read current `architecture.md` 
2. Check implementation repo (e.g., [merge-tower-unity/AGENTS.md](../merge-tower-unity/AGENTS.md))
3. Update sections: **4) Arquitectura Propuesta** if reality diverged from design
4. Ensure design doc reflects actual patterns + rationale

### Task: Add a New Application
1. Create `applications/<app-name>/` folder
2. Copy [template-architecture.md](template-architecture.md) → `applications/<app-name>/architecture.md`
3. Fill in all sections (Executive Summary, Context, Requirements, Architecture, Implementation)
4. Create repo-specific [AGENTS.md](../merge-tower-unity/AGENTS.md) in implementation repo
5. Link to implementation repo in this file's architecture.md

### Task: Track Implementation Progress
1. Update `implementation-plan.md` with phase status
2. Add blockers/risks to `backlog.md`
3. Log resolved decisions in `decisions.md`
4. Link to implementation repo's CI/CD or test status

---

## How Agents Should Use This Repo

1. **Reference architecture** when reviewing code against design intent
2. **Check requirements** before adding new features (is it in scope?)
3. **Use as source of truth** for non-functional requirements (performance, scalability)
4. **Link, don't copy** — point to implementation details elsewhere
5. **Update after major decisions** during development (reflect reality)

---

## Linking to Implementation Repos

Use markdown links to point to live code and build systems:

```markdown
For implementation details, see [merge-tower-unity AGENTS](../merge-tower-unity/AGENTS.md).
Build & test: See [merge-tower-unity CI/CD](./.github/workflows/build.yml).
```

---

## References

- [Eras Defense Architecture](applications/eras-defense/architecture.md)
- [Merge Tower Implementation](../merge-tower-unity/AGENTS.md)
