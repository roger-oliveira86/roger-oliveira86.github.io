# ADR 005: Rollback-First Change Management

**Date:** 2024 (retroactively documented)
**Status:** Accepted
**Deciders:** Roger Oliveira (Platform Team Lead), Network Team, DevOps Team
**Affects:** Change management process, incident response, platform reliability

---

## Context

### Problem Statement

Once ADR-001 gave us shared architecture and formal dependency tracking, a second failure mode became visible — one that governance alone didn't fix:

- **Rollback existed on paper, not in practice** — most changes had a documented rollback section, but it had never actually been executed
- **"Fix forward" was the default instinct** — when a change broke something in production, the reflex was to patch the problem in place rather than undo it, because nobody trusted the undo path
- **Incidents lasted longer than they needed to** — teams kept broken changes live for hours or days while attempting a forward fix, instead of reverting in minutes
- **Rollback confidence was inversely related to how critical the change was** — the more sensitive the system, the less likely anyone had tested reverting it

### Why This Matters

Rollback is the cheapest insurance a platform team has. A tested rollback turns "did we break production?" from a crisis into a five-minute decision. An untested one turns it into a multi-day incident, because the first time you find out rollback doesn't work cleanly is exactly the moment you have the least room to discover that.

### Organizational Context

- Traffic-critical, regulated infrastructure means every extra hour of incident time has direct business and compliance cost
- ADR-001 fixed *coordination*; this ADR addresses *reversibility*, a separate and previously invisible risk
- Post-incident reviews kept surfacing the same root cause: rollback was assumed to work, not verified to work

---

## Decision

We will require that **no change is approved for production without a rollback procedure that has been executed and verified in a non-production environment first.**

### What This Means in Practice

1. **Rollback is a deliverable, not a footnote** — every change ticket includes a rollback procedure as a required field, not optional documentation
2. **Rollback is tested before the change ships**, not written and hoped to work — it is executed once in staging/pre-prod as part of the validation gate from ADR-001
3. **Rollback time is measured and tracked** — we record how long a rollback takes, the same way we record deployment time
4. **Rollback is the default response to ambiguity** — if a change's effect in production is unclear or contested, the standing instruction is to revert first and investigate after, not investigate while the change stays live
5. **Every incident retro asks one extra question**: "Could this have ended sooner if we'd rolled back instead of fixing forward?"

---

## Rationale

### Why Test Rollback Before Deployment, Not After?

Testing rollback after something breaks means testing it under the worst possible conditions: time pressure, incomplete information, and an audience. Testing it before deployment, in a calm environment, is the only way to know it actually works before you need it to.

### Why Make Rollback the Default, Not Fixing Forward?

Fixing forward under pressure adds a second unverified change on top of the first one that already broke something. Rollback returns the system to a known-good state, which is a strictly smaller risk than improvising a fix live. The exception is when rollback itself would cause data loss or worse damage than the original issue — those cases are documented explicitly per change, not assumed by default.

### Why Measure Rollback Time?

What gets measured gets trusted. A team that has a number for "we can revert this in 4 minutes" behaves differently under pressure than a team that has a hope.

---

## Consequences

### Positive Outcomes

✅ **Shorter incidents** — reverting a known-bad change is faster than debugging it live
✅ **Less fear of shipping** — engineers deploy more confidently when the undo path is proven, not assumed
✅ **Rollback becomes routine, not a last resort** — reducing the stigma of "we had to roll back" as if it were a failure
✅ **Retros get sharper** — separating "the change was wrong" from "we didn't recover from it well" clarifies what actually needs fixing

### Tradeoffs / Costs

⚠️ **Added time before deployment** — testing rollback adds a step to every change
Mitigated by: for low-risk changes, this step is fast; the cost scales with the risk the change carries, which is exactly where it should scale
⚠️ **Not all changes are cleanly reversible** (e.g., destructive data migrations)
Mitigated by: those changes require an explicit, reviewed exception with a compensating recovery plan instead of a rollback

---

## Alternatives Considered

### Alternative A: "Document Rollback, Trust It Works"
Keep rollback as written documentation without mandatory execution.

**Why we didn't choose this:**
- This was the status quo, and it was the exact gap that caused longer incidents
- A rollback plan nobody has run is a hypothesis, not a procedure

### Alternative B: "Fix Forward as Standard Practice"
Treat rollback as a rare last resort; default to patching issues in place.

**Why we didn't choose this:**
- Adds risk on top of risk during exactly the moment you can least afford it
- Extends incident duration in nearly every retro we reviewed

### Alternative C: "Rollback Only for High-Risk Changes"
Require tested rollback only for changes flagged as high-risk.

**Why we didn't choose this:**
- Risk classification is frequently wrong in hindsight — several of our worst incidents came from changes rated "low risk"
- The cost of testing rollback for a low-risk change is small; the cost of not having it for a misjudged one is not

---

## Implementation

### Timeline
- **Week 1:** Add mandatory rollback field to the change template in Azure DevOps
- **Week 2–3:** Pilot on ingress-layer changes (building on ADR-001's validation gate)
- **Week 4:** Extend to all traffic-critical infrastructure changes
- **Ongoing:** Rollback time tracked per change; reviewed quarterly alongside lead time metrics

### Success Metrics
- **Rollback execution rate** — target: 100% of changes have rollback tested pre-deployment (was: rollback documented but rarely tested)
- **Mean incident duration** — target: reduced by making revert-first the default response
- **Ingress-layer incidents** — zero over 6 months (carried forward from ADR-001, reinforced by this decision)

---

## Related Decisions

- **ADR-001:** Restructure LB/WebCache/Kubernetes Architecture (established the validation gate this ADR extends)
- **ADR-002:** Risk-based vulnerability management
- **Process Document:** Ingress Change Management Runbook

---

## Revision History

| Date | Author | Status | Change |
|------|--------|--------|--------|
| 2024 | Roger Oliveira | Accepted | Initial documentation (retroactively) |

---

## Questions to Consider

1. **What if a change genuinely can't be rolled back cleanly?** → Requires an explicit, reviewed exception with a documented compensating recovery plan.
2. **Who decides to trigger a rollback during an incident?** → Whoever is designated as decision-owner for that change, agreed in advance, not negotiated live.
3. **How do we keep rollback tests from becoming stale?** → Re-verify rollback whenever the underlying change is modified, not just at first approval.
