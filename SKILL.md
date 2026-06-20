---
name: macaw-policies
description: >-
  Write, revise, review, and troubleshoot MACAW MAPL policies (MACAW Agentic
  Policy Language) — the JSON authorization rules that control what AI agents,
  LLMs, tools, and MCP servers can do in MACAW Security. Use this whenever the
  user mentions a MAPL policy, a MACAW policy, policy_id / extends / resources /
  denied_resources / constraints / attestations, a `PermissionDenied` or DENIED
  agent call, wants to grant or restrict an agent's access to a model or tool,
  set token/rate/model limits, require human approval for an operation, design a
  company → BU → team → user policy hierarchy, debug why a request was blocked,
  or paste/generate policy JSON for the MACAW Console. Trigger even if the user
  doesn't say "MAPL" but is clearly describing agent access control in MACAW.
---

# MACAW MAPL Policies

MAPL (MACAW Agentic Policy Language) is a JSON authorization language that decides
what an AI agent, LLM, tool, or MCP server is allowed to do inside MACAW Security.
Policies inherit hierarchically (organization → business unit → team → user/app) and
combine with **intersection semantics**: at every level permissions can only get
narrower, never wider. This is the single most important mental model — internalize
it before writing or debugging anything.

Use this skill to author new policies, revise existing ones, review a policy for
mistakes, or troubleshoot a denied request. The user works in the **MACAW Console**
(`console.macawsecurity.ai`), so produce policy JSON they can paste into
**Console → Policies**, and frame debugging around the Console tabs (Logs, Traces,
Policies).

## The five rules that govern everything

Almost every MAPL bug or design mistake traces back to one of these. Keep them in mind:

1. **Resources ARE operations.** There is no separate "action" field. The resource
   path *is* the thing being done: `llm:openai/chat.completions`, `tool:database/query`.
   This is why wildcards work naturally and why you never write `read`/`write` separately.
2. **Denials always win.** Anything matched by `denied_resources` or `denied_parameters`
   is blocked, even if `resources` also allows it. Denials are the safety net.
3. **Children can only restrict, never expand.** A child policy that `extends` a parent
   cannot grant something the parent didn't allow. If you need to *grant* more, fix the
   parent — don't fight the hierarchy.
4. **`resources: []` means "defer to parent", NOT "deny all".** Empty (or absent)
   resources = "I add no new restriction at this level; the parent's scope applies."
   This trips people up constantly. To actually block everything, use `denied_resources`.
5. **Most restrictive wins on merge.** When policies inherit, resources intersect,
   denials union, numeric limits take the minimum, and required attestations AND together.

## Authoring workflow

When the user asks for a new or revised policy:

1. **Identify the level.** Is this company-wide, a business unit, a team, an individual
   user, or a service/agent (`app:`)? The `policy_id` prefix encodes it:
   `global:`, `company:`, `bu:`, `team:`, `user:`, `app:`. Pick the *highest* level that
   fits — broad rules belong at the top so they're written once and inherited.
2. **Start from what the parent already allows.** A child only narrows. List only the
   restrictions this level adds; leave `resources: []` (or omit it) to inherit the
   parent's scope unchanged.
3. **Allow the resources, then deny the dangerous ones.** Put the operations they need in
   `resources`, then add safety denials (`*.secret`, `*.password`, `admin:**`, etc.) in
   `denied_resources`.
4. **Add constraints for cost and safety.** Cap models, `max_tokens`, `rate_limit`,
   temperature; block injection strings via `denied_parameters`. See
   [references/mapl-schema.md](references/mapl-schema.md) for every constraint type.
5. **Add attestations if a human or external check must gate the action.** Use the
   conditional `::{...}` syntax so approval is only required when it matters (e.g. large
   amounts). See [references/mapl-schema.md](references/mapl-schema.md#attestations).
6. **Sanity-check against the five rules**, then hand over JSON ready to paste into
   **Console → Policies → Create Policy** (the Console validates syntax on save).

Always output valid, complete JSON the user can paste directly. Briefly explain what
each non-obvious field does and call out anything the parent hierarchy might override.

## Minimal template

```json
{
  "policy_id": "team:analysts",
  "name": "Analytics Team Policy",
  "description": "What the analytics team's agents may do",
  "scope": "team",
  "extends": "bu:data",
  "resources": [
    "llm:openai/chat.completions",
    "tool:database/query"
  ],
  "denied_resources": [
    "*.secret",
    "data:executive/*"
  ],
  "constraints": {
    "rate_limit": 100,
    "parameters": {
      "llm:openai/chat.completions": {
        "model": ["gpt-3.5-turbo", "gpt-4"],
        "max_tokens": {"max": 2000}
      }
    }
  }
}
```

Only `policy_id` is strictly required, but real policies almost always set `resources`,
`extends`, and `constraints`. For the full field list, wildcard rules, constraint types,
inheritance merge table, and attestation syntax, read
[references/mapl-schema.md](references/mapl-schema.md).

## Troubleshooting a denied request

When a request was blocked (`DENIED` in Logs, or a `PermissionDenied` exception), the
cause is almost always (a) a denial pattern matching, (b) the resource not being in the
*effective* (merged) policy, (c) a constraint violation on a parameter, or (d) a missing
attestation. Work the Console top-down: **Logs → Traces → Policies**, then use
**Show Effective Policy** to see the merged result — the denial is usually obvious once
you look at the effective policy rather than any single level.

Don't guess at the fix from one policy file. Read
[references/troubleshooting.md](references/troubleshooting.md) for the full Console debug
workflow and a checklist mapping each denial symptom to its cause and fix.

## References

- [references/mapl-schema.md](references/mapl-schema.md) — Complete MAPL syntax: every
  field, resource type, wildcard, constraint type, the inheritance merge table, and
  attestation/conditional syntax. Read this whenever you need exact syntax.
- [references/troubleshooting.md](references/troubleshooting.md) — Console debug workflow
  (Logs/Traces/Policies), a symptom→cause→fix checklist, and the most common mistakes.
  Read this whenever a request is being denied/allowed unexpectedly or a policy "isn't
  working."
