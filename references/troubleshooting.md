# Troubleshooting MAPL Policies

Use this when a request is denied (or allowed) unexpectedly, or a policy "isn't working."
The user operates in the MACAW Console, so all steps below are Console-based.

## Console debug workflow: Logs → Traces → Policies

Work top-down. Most denials become obvious once you look at the **effective (merged)
policy** rather than any single policy file.

1. **Find the denial — Console → Logs → Denials tab.** Filter by agent or time range to
   locate the specific `DENIED` event. Each event shows which policy rule fired. (Code
   path: a `PermissionDenied` exception carries `reason` and `policy_rule`.)
2. **Trace the request — Console → Traces → Request Flow.** Find the request and view the
   full event sequence to see exactly where it failed (which boundary / which hop).
3. **See how policies combined — Console → Traces → Policy Trail.** Shows which policies
   were evaluated and how they merged to produce the decision. This is where you catch a
   *parent* denial the user forgot about.
4. **Inspect & fix — Console → Policies.** Open the relevant policy, click
   **Show Effective Policy** to see the merged result of the whole inheritance chain, then
   edit (or use the Policy Assistant to generate a new rule). The Console validates JSON
   syntax on save.
5. **Re-test and confirm — Console → Logs.** Re-run the request and verify the event now
   shows `ALLOWED`.

## Symptom → cause → fix checklist

| Symptom | Likely cause | Fix |
|---|---|---|
| Request denied even though the user's policy allows it | A **parent** policy denies it, or its `resources` don't include it. Inheritance intersects — the child can't add what the parent never allowed. | Check **Show Effective Policy**. Add the resource at the parent (or the highest level that should have it), not the child. |
| Denied with a `denied_resources` / `denied_parameters` rule firing | **Denials always win.** A broad denial like `admin:**` or `*.secret` is matching. | Narrow the denial pattern, or confirm the block is intentional. Denials cannot be overridden by an allow at a lower level. |
| Empty `resources: []` seems to block everything | Misconception — `[]` means **defer to parent**, not deny-all. If the effective result is empty, the *parent* chain allows nothing. | Add the needed resources at a level that has scope, or fix the parent. To intentionally block, use `denied_resources`. |
| A `denied_parameters` pattern isn't catching the bad input | These are **wildcard patterns, not regex**, and **case-sensitive**. `*DROP TABLE*` won't match `drop table`. | Use `*` wildcards (not regex syntax). Add each casing you need to block, e.g. both `*DROP*` and `*drop*`. |
| A `parameters` regex `pattern` rejects a valid value | `pattern` must match the **entire** string, and it *is* real regex (unlike `denied_parameters`). | Anchor/loosen the regex. Verify with the exact value from the Logs event. |
| Token/rate limit lower than expected | On merge, numeric limits take the **minimum** across the chain. A stricter ancestor wins. | Find the level setting the low value via Show Effective Policy; raise it there if appropriate. |
| Model rejected even though child allows it | Allowed-value lists **intersect** on merge. The model must be in *every* level's list. | Add the model to the ancestor whose list excludes it. |
| Request blocks/hangs then fails | A required **external attestation** is pending approval and `timeout` elapsed. | Approve it in **Console → Activity → Attestations**, or adjust the attestation's `timeout`/condition. |
| Attestation required when it shouldn't be | The conditional `::{...}` is missing or too broad, so it's always required. | Add/narrow the condition, e.g. `manager_approval::{params.amount > 10000}`. |
| Service call denied though the caller is allowed | **Dual-perspective enforcement** — the `app:` service policy must *also* allow it. | Check the service (`app:`) policy too; effective access = caller ∩ service. |
| Policy edit didn't take effect | Saved to the wrong level, or another level overrides it. | Confirm via Show Effective Policy; remember deny-wins and most-restrictive-wins. |

## The usual suspects (in order)

When you don't know where to start, check these in sequence — they cover the large
majority of "why was this denied" cases:

1. **A denial is matching** (`denied_resources` / `denied_parameters`) — denials beat allows.
2. **The resource isn't in the effective policy** — a parent never allowed it, or
   intersection narrowed it away.
3. **A parameter constraint is violated** — wrong model, `max_tokens` over the limit,
   value outside a range, fails a `pattern`, array too long.
4. **A required attestation is missing or pending.**

## Designing to avoid problems

- Keep top-level (company) policies **broad and minimal** — they're the ceiling everything
  inherits. Over-restricting here breaks everyone downstream.
- Push restrictions **down** to BU/team/user, not up.
- Don't create `user:` policies to *grant* extra access — children can only narrow. Move
  the grant to the team/BU, or use a (time-bounded) group.
- Prefer **time-bounded `group:` access** (with a `validity` window) over permanent
  overrides for temporary/emergency needs — it auto-revokes.
- Always confirm intended behavior with **Show Effective Policy** before declaring a policy
  done; the merged result is what actually runs.

For exact field/constraint/attestation syntax referenced above, see
[mapl-schema.md](mapl-schema.md).
