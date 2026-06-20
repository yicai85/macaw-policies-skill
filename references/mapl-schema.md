# MAPL Schema Reference

Complete syntax reference for MACAW Agentic Policy Language (MAPL). Use this for exact
field names, wildcard behavior, constraint types, inheritance rules, and attestation
syntax.

## Contents
- [Policy fields](#policy-fields)
- [policy_id and scope](#policy_id-and-scope)
- [Resources and wildcards](#resources-and-wildcards)
- [Constraints](#constraints)
- [Parameter constraint types](#parameter-constraint-types)
- [denied_parameters (injection defense)](#denied_parameters-injection-defense)
- [Inheritance and merge rules](#inheritance-and-merge-rules)
- [Attestations](#attestations)
- [Time-based and service policies](#time-based-and-service-policies)

## Policy fields

```json
{
  "policy_id": "scope:name",        // REQUIRED. Unique ID, scope-prefixed.
  "name": "Human-readable name",    // optional. Shown in UIs/logs.
  "version": "1.0",                 // optional.
  "description": "What this does",  // optional but recommended.
  "scope": "user",                  // optional: global|company|bu|team|user|app
  "extends": "parent:policy_id",    // optional. Inherit from a parent (narrows only).
  "resources": ["..."],             // allowed operation patterns. [] = defer to parent.
  "denied_resources": ["..."],      // blocks that override allows. Denials always win.
  "attestations": ["..."],          // required proofs before an op proceeds (array).
  "constraints": {
    "rate_limit": 100,              // requests per minute
    "parameters": { },              // per-operation parameter validation
    "denied_parameters": { },       // blocked parameter value patterns
    "attestations": { }             // attestation metadata (approval_criteria, ttl, …)
  }
}
```

Only `policy_id` is strictly required. Everything else defaults to "no additional
restriction at this level."

Note the two places `attestations` appears: the **top-level array** lists *which*
attestations are required (and their conditions); the **`constraints.attestations`
object** holds the *metadata* for each (who can approve, timeouts, TTL, reuse).

## policy_id and scope

`policy_id` convention is `{scope}:{name}`. The prefix encodes the hierarchy level:

| Prefix      | Level                        | Example                  |
|-------------|------------------------------|--------------------------|
| `global:`   | Root, applies to everything  | `global:default`         |
| `company:`  | Organization-wide defaults   | `company:acme`           |
| `bu:`       | Business unit / department   | `bu:finance`             |
| `team:`     | Team                         | `team:analysts`          |
| `user:`     | Individual human user        | `user:alice`             |
| `app:`      | Service / agent (the caller) | `app:report-service`     |

The bottom of the hierarchy is the **caller** — either a human (`user:`) or an automated
service/agent (`app:`). Both inherit from the same team → BU → company chain. (`group:`
also appears for time-bounded access grants; see [time-based policies](#time-based-and-service-policies).)

Resolution order: `global → company → bu → team → user/app`. Each level restricts the
previous. **Deny always overrides allow.**

## Resources and wildcards

In MAPL, **resources are operations** — the path includes the operation, so there's no
separate action field. This avoids M×N rule explosion and makes wildcards natural.

Resource types:

| Type       | Meaning            | Examples                                   |
|------------|--------------------|--------------------------------------------|
| `llm:`     | LLM operations     | `llm:openai/chat.completions`, `llm:anthropic/*` |
| `tool:`    | Tool invocations   | `tool:search_*`, `tool:database/query`     |
| `data:`    | Data access        | `data:reports/**`, `data:sales/**`         |
| `mcp:`     | MCP server resources | `mcp:server/*`                           |
| `service:` | External services  | `service:database`, `service:api/*`        |

Wildcards:

| Pattern   | Matches                              | Example                                      |
|-----------|--------------------------------------|----------------------------------------------|
| (exact)   | Only that exact operation            | `llm:openai/chat.completions`                |
| `*`       | Any chars **except** `/` (one segment) | `tool:*` → `tool:search`; `llm:openai/*` → `chat.completions`, `embeddings` |
| `**`      | Any chars **including** `/` (recursive) | `llm:**` → `llm:openai/gpt-4`; `llm:openai/**` → all OpenAI ops at any depth |
| `*text*`  | Contains substring                   | `*sales*` matches `quarterly_sales.txt`      |

More specific patterns take precedence. `**` alone matches **everything** — use with
caution. Denials (`denied_resources`) always win over allows.

### resources: [] vs ["**"] vs absent

This distinction causes frequent confusion:

| Value                  | Meaning                                                  |
|------------------------|----------------------------------------------------------|
| `"resources": []`      | **Defer to parent** — parent's resources apply unchanged. NOT "deny all". |
| (field absent)         | Same as `[]` — defer to parent.                          |
| `"resources": ["**"]`  | Explicit pass-through — documents "no restriction here". |
| `"resources": ["..."]` | Restrict *within* the parent's scope.                    |

To actually block access, use `denied_resources`, not an empty `resources`.

## Constraints

`constraints` holds operational limits:

```json
"constraints": {
  "rate_limit": 100,
  "parameters": {
    "llm:openai/chat.completions": {
      "model": ["gpt-3.5-turbo", "gpt-4"],
      "max_tokens": {"max": 2000},
      "temperature": {"type": "number", "range": [0.0, 1.0]}
    },
    "tool:database/query": {
      "limit": {"type": "integer", "max": 1000}
    }
  },
  "denied_parameters": {
    "llm:**": {"prompt": ["*DROP TABLE*", "*rm -rf*"]}
  }
}
```

- `rate_limit` — requests per minute. On merge, the **minimum** wins.
- `parameters` — per-operation validation, keyed by resource pattern (wildcards allowed).
- `denied_parameters` — block specific parameter *values* (see below).

## Parameter constraint types

Each parameter maps to a constraint spec. Types:

| Form                                | Meaning            | Notes                                   |
|-------------------------------------|--------------------|-----------------------------------------|
| `["a", "b", "c"]`                   | Enumeration        | Value must be in the list (shorthand for `allowed_values`). String patterns allowed. |
| `{"allowed_values": ["a","b"]}`     | Enumeration (full) | Same as above, explicit form.           |
| `{"max": 4000}`                     | Maximum            | Numeric value ≤ max. Shorthand for `{"type":"integer","max":4000}`. |
| `{"min": 0, "max": 1}`              | Range (min/max)    | Numeric value within [min, max], inclusive. |
| `{"range": [0.0, 2.0]}`             | Range (shorthand)  | Same as min/max. Use min/max OR range, not both. |
| `{"pattern": "^[a-z]+$"}`           | Regex              | String must match the **entire** pattern. |
| `{"max_length": 1000}`              | Max string length  | Prevents oversized/injection input.     |
| `{"min_length": 3}`                 | Min string length  |                                          |
| `{"min_items": 1, "max_items": 100}`| Array size         | Prevents resource exhaustion.           |

Type validation values for `"type"`: `integer`, `number`, `string`, `boolean`, `array`,
`object`. You can combine `type` with the constraints above, e.g.
`{"type": "integer", "min": 1, "max": 4000}`.

Shorthand equivalences:
- `"model": ["gpt-4"]` ≡ `"model": {"allowed_values": ["gpt-4"]}`
- `"max_tokens": {"max": 500}` ≡ `"max_tokens": {"type": "integer", "max": 500}`

## denied_parameters (injection defense)

Blocks dangerous parameter *values*. **Critical gotcha: these use wildcard patterns, NOT
regex, and they are case-sensitive.**

```json
"denied_parameters": {
  "llm:**": {
    "prompt": ["*DROP TABLE*", "*rm -rf*", "*eval(*", "*exec(*"]
  },
  "tool:shell/*": {
    "command": ["*sudo*", "*rm -*", "*dd if=*"]
  },
  "tool:*": {
    "include_credentials": [true],
    "output_path": ["*/etc/*", "*.key"]
  }
}
```

- `*` matches any characters (including spaces, `/`, `:`, special chars).
- Case-sensitive: `*DROP TABLE*` does **not** match `drop table`. Add both casings if
  needed.
- Substring match: `*DROP TABLE*` matches `"foo DROP TABLE bar"`.

## Inheritance and merge rules

When a child `extends` a parent, fields merge via **monotonic restriction** — the result
is always ≤ either input:

| Field              | Merge strategy                                            |
|--------------------|----------------------------------------------------------|
| `resources`        | **Intersection** — both parent and child must allow. `[]` defers to parent. |
| `denied_resources` | **Union** — either parent's or child's denial applies.   |
| `rate_limit`       | **Minimum** — the most restrictive rate wins.            |
| `parameters`       | **Most restrictive wins** — e.g. `min(max_tokens)`, intersection of allowed model lists. |
| `attestations`     | **AND** — all required attestations from every level must be present. |

Resource intersection is domain-aware: `[llm:*] ∩ [llm:openai/*] = [llm:openai/*]`.

Worked example — `company:FinTech → bu:Analytics → user:alice`:
- Company: `resources: [llm:openai/*]`, `max_tokens: 4000`, `rate_limit: 100`
- BU adds: `max_tokens: 2000`, `temperature: 0.3`, `rate_limit: 50`
- User adds: `resources: [llm:openai/chat.completions]`, `model: [gpt-3.5-turbo]`,
  `max_tokens: 500`, `rate_limit: 10`, `denied_resources: [data:executive/*]`
- **Effective:** `resources: [llm:openai/chat.completions]`,
  `denied: [*.secret, *.password, data:executive/*]`, `model: [gpt-3.5-turbo]`,
  `max_tokens: 500`, `temperature: 0.3`, `rate_limit: 10`.

In the Console, **Show Effective Policy** computes this merged result for you.

## Attestations

Attestations are cryptographic proofs that a condition was met. Policies define what
*can* happen; attestations prove something *did* happen (a DLP scan ran, a manager
approved). Required attestations are listed in the **top-level array**, with metadata in
**`constraints.attestations`**.

```json
{
  "attestations": [
    "identity_verified",                          // always required
    "manager_approval::{params.amount > 10000}"   // conditional
  ],
  "constraints": {
    "attestations": {
      "manager_approval": {
        "approval_criteria": "role:manager",      // who can approve (external only)
        "timeout": 300,                           // seconds to block-and-poll for approval
        "time_to_live": 3600,                     // seconds the approval stays valid
        "one_time": true                          // consumed after one use
      }
    }
  }
}
```

**Internal vs external:**

| | Internal | External |
|---|---|---|
| Set by | Tools during workflow execution | Humans / third-party systems |
| `approval_criteria` | not used | **required** |
| `max_uses` | supported | not used |
| `one_time` | supported | supported |
| `time_to_live` | supported | supported |
| Lifecycle | Set → Active → Consumed/Expired | Pending → Approved/Denied → Consumed/Expired |

External attestations enter a **Pending** state; an approver acts on them in
**Console → Activity** (Attestations panel). If `timeout` is set, the request blocks and
polls for approval with exponential backoff until approved/denied/timed out.

**Conditional syntax** `::{condition}` — the attestation is only required when the
condition is true:

| Reference                       | Meaning                  | Example                              |
|---------------------------------|--------------------------|--------------------------------------|
| `params.X`                      | Invocation parameter     | `params.amount > 5000`               |
| `principal.X`                   | Authenticated user attr  | `principal.user_id == 'admin'`       |
| `principal.has_role('X')`       | Role check               | `principal.has_role('manager')`      |
| `principal.has_group('X')`      | Group check              | `principal.has_group('trading')`     |
| `context.has_attestation('X')`  | Existing attestation     | `context.has_attestation('mfa')`     |

Conditions support `AND` / `OR` and comparisons, e.g.
`{params.priority == 'urgent' AND params.amount > 10000}`.

**Tiered approval** pattern — different approvers by threshold:
```json
"attestations": [
  "team_lead_approval::{params.amount > 1000 AND params.amount <= 10000}",
  "manager_approval::{params.amount > 10000 AND params.amount <= 50000}",
  "director_approval::{params.amount > 50000}"
]
```

**Reusable grants** — set `one_time: false` with a `time_to_live` (and optional
`max_uses`) so one approval covers many operations within a window (batch jobs, trading
windows, delegated authority):
```json
"trading_window": {
  "approval_criteria": "role:manager",
  "one_time": false,
  "time_to_live": 3600,
  "max_uses": 100
}
```

## Time-based and service policies

**Time-bounded access** — prefer a `validity` window over permanent overrides for
emergency/temporary access; it auto-revokes when it expires:
```json
{
  "policy_id": "group:emergency-access",
  "validity": {
    "not_before": "2025-01-17T09:00:00Z",
    "not_after": "2025-01-17T17:00:00Z"
  },
  "resources": ["admin:**"],
  "constraints": {"audit_level": "maximum", "require_approval": true}
}
```

**Service / agent policies (`app:`)** define what a *service itself* offers, independent
of the caller:
```json
{
  "policy_id": "app:trading-service",
  "scope": "service",
  "resources": ["trade:execute", "trade:cancel", "quote:get"],
  "constraints": {
    "parameters": {
      "trade:execute": {
        "amount": {"max": 100000},
        "currency": {"allowed_values": ["USD", "EUR", "GBP"]}
      }
    }
  }
}
```

**Dual-perspective enforcement:** the effective access is the **intersection of the
caller's policy and the service's policy** — *both* must allow the operation. Even if the
caller is permitted, the service policy can still block it (defense in depth). When
debugging a denial on a service call, check *both* sides.
