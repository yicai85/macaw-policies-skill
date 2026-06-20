# macaw-policies

A [Claude Code](https://claude.com/claude-code) skill for writing, revising, reviewing,
and troubleshooting **MACAW MAPL policies** — the JSON authorization rules
([MACAW Agentic Policy Language](https://www.macawsecurity.ai/docs)) that control what AI
agents, LLMs, tools, and MCP servers are allowed to do inside MACAW Security.

## What it does

- **Author** new policies at the right hierarchy level (`company:` → `bu:` → `team:` →
  `user:`/`app:`), with paste-ready JSON for the MACAW Console.
- **Revise & review** existing policies against the rules that actually govern behavior
  (deny-wins, monotonic restriction, intersection-on-merge, `resources: []` = defer).
- **Troubleshoot** denied requests — maps each denial symptom (resource not allowed,
  denial pattern, parameter violation, missing/expired attestation) to its cause and fix,
  framed around the Console `Logs → Traces → Policies` workflow.

## Layout

```
macaw-policies/
├── SKILL.md                       # workflow, the five governing rules, template
└── references/
    ├── mapl-schema.md             # full syntax: fields, wildcards, constraints,
    │                              #   inheritance merge table, attestation syntax
    └── troubleshooting.md         # Console debug workflow + symptom→cause→fix checklist
```

## Install

Clone (or copy) into your Claude Code skills directory:

```bash
git clone https://github.com/yicai85/macaw-policies.git ~/.claude/skills/macaw-policies
```

The skill then loads automatically and triggers when you mention MAPL/MACAW policies,
`policy_id` / `resources` / `attestations`, a `PermissionDenied` or `DENIED` agent call,
or ask to grant/restrict an agent's access.

## Reference

Built from the MACAW Security documentation: <https://www.macawsecurity.ai/docs>
