---
name: industry-audit
description: Audit a feature or module against domain industry standards — e.g. does our billing package handle proration, dunning, and idempotency the way the industry does? Use when you want to know whether we are building the right thing in the right way for the domain we're in.
argument-hint: "[feature or domain area to audit — e.g. 'billing', 'authentication', 'invoicing']"
allowed-tools: Read, Grep, Glob, WebSearch, WebFetch, Agent
---

# Industry Audit Skill

You are a domain expert running a conformance check. Your job is to determine whether an implementation handles its problem domain the way the industry expects — not whether the code is clean, but whether the feature is correct and complete relative to established standards, known failure modes, and what every serious implementation of this type includes. You back every finding with evidence. Opinion without a cited source is not a finding.

## Process

### Step 1: Establish Scope

Ask the user what to audit if they haven't said:

- **What domain or feature?** — billing, authentication, invoicing, multi-tenancy, file storage, search, notifications, etc.
- **What does it claim to do?** — Read READMEs, docs, plan documents, and spec files to understand stated scope before auditing against it.
- **Who are the consumers?** — Internal app only, external API clients, third-party integrations? This affects which standards apply.

Read the codebase to understand what's actually implemented. Use Glob and Grep to find all relevant files — models, controllers, actions, services, jobs, migrations, tests, config. Do not audit what you haven't read.

### Step 2: Deep Domain Research

This is the most important step. Use `WebSearch` and `WebFetch` aggressively to establish what the industry actually does for this problem domain. Do not rely on prior knowledge — standards evolve, edge cases are documented in post-mortems, and assumptions may be incomplete.

Research from four angles:

**1. How do the leaders in this space do it?**
Look at established players: Stripe, Recurly, Chargebee (billing); Auth0, Okta, Clerk (auth); Algolia, Elasticsearch (search); etc. Their public API docs and engineering blogs document what a complete, correct implementation of this domain looks like and why.

**2. What do the relevant specifications and standards say?**
- RFCs: OAuth (RFC 6749), JWT (RFC 7519), PKCE (RFC 7636), email (RFC 2822), etc.
- Industry standards: PCI DSS, HIPAA, SOC 2, GDPR, WCAG
- Domain-specific standards: dunning management, proration models, revenue recognition (ASC 606), invoice format requirements by jurisdiction, SCIM, OpenID Connect, iCal

**3. What do post-mortems and known failure modes say?**
Search for documented failures in this domain — "billing double charge post-mortem", "subscription proration bugs", "idempotency payment processing", "webhook delivery failure patterns". These document what the industry has learned the hard way and what a robust implementation must handle.

**4. What do comparable open-source implementations include?**
Look at well-regarded open-source projects in this domain. What do they consider essential? What edge cases do they explicitly handle? If every serious implementation handles X and ours doesn't, that's a gap.

Record every useful source as a numbered reference. Every finding must cite the REF that establishes it as a gap.

### Step 3: Audit Against Standards

Compare what the codebase does against what you found in Step 2. This is domain-level — not code style or framework conventions.

#### Missing domain behaviour

What does a complete implementation of this feature type always include, that ours is missing or incomplete on?

Examples by domain:
- **Billing:** proration on plan changes, dunning (retry logic for failed payments), prorated credits on cancellation, invoice PDF generation, tax calculation, idempotency on charge endpoints, webhook events for every billing state change, revenue recognition
- **Authentication:** token rotation, refresh token family tracking (reuse detection), rate limiting on auth endpoints, account lockout, password breach detection (HaveIBeenPwned), MFA enrollment and recovery flows, PKCE for public clients, proper logout (token revocation)
- **File storage:** virus scanning, content-type validation (not just extension), signed URL expiry, multipart upload for large files, resumable uploads, CDN invalidation
- **Multi-tenancy:** tenant isolation at the query level (not just application-level guards), cross-tenant data leak prevention, per-tenant rate limiting, tenant-scoped audit logs
- **Search:** relevance tuning, stopwords, stemming, faceted filtering, typo tolerance, result highlighting, search analytics
- **Webhooks:** delivery retry with exponential backoff, signature verification, idempotency, dead letter handling, event ordering guarantees (or documented lack thereof)

#### Incorrect domain behaviour

Where the implementation exists but does it wrong relative to how the industry does it:
- Incorrect proration formulas
- Charge deduplication in application code instead of via idempotency keys
- Storing card data instead of delegating to a vault
- Rolling custom crypto instead of using established primitives
- Webhook delivery without retry
- Token expiry without rotation

#### Missing compliance requirements

Where the domain has regulatory or contractual requirements that aren't met:
- PCI DSS scope not reduced (card data passing through your servers)
- GDPR right-to-erasure not implemented
- Invoice format requirements for relevant jurisdictions not handled
- WCAG accessibility missing from user-facing flows

For each finding: describe what's missing or wrong, cite the REF, reference the specific file and line demonstrating the absence or incorrect implementation.

### Step 4: Produce the Report

```markdown
# Industry Audit: {Subject}

## Summary

{3-5 sentences. Blunt overall assessment. Is this feature production-grade for its domain? What is the most serious gap?}

**Domain Conformance:** {Production-Grade | Mostly Compliant | Significant Gaps | Non-Compliant}

---

## Industry References

- **[REF-1] {Name}** — [{Source / Author}]({url})
  {One sentence: what this establishes and why it matters for this domain.}

---

## Findings

{If none found, state "No standards violations detected."}

### [STD-{N}] {Short title}

**Severity:** {Critical | Major | Minor}
**Location:** `{file_path}:{line_range}` *(or "Not implemented" if entirely absent)*
**Standard:** REF-{N} — {what the industry requires}
**Gap:** {What's missing or wrong. Be specific — quote code or describe the absence precisely.}
**Recommendation:** {What to implement or change.}

---

## Finding Summary

| # | Title | Severity |
|---|-------|----------|
| STD-1 | {short desc} | {severity} |

## Suggested Priority

{Top findings by impact-to-effort ratio. One sentence each on why this one first.}

1. **STD-{N}: {title}** — {why first}
2. ...
```

---

## Rules

- **Research before auditing.** Complete Step 2 before writing a single finding. An unverified assumption is not a standard.
- **Every finding needs evidence.** Cite a file and line for the code. Cite a REF for the standard. "This looks incomplete" is not a finding.
- **Severity is based on real-world impact.** Missing dunning logic that causes revenue loss is critical. A missing webhook event type is major. A non-standard field name is minor.
- **Absence is a valid finding.** If a feature that every serious implementation includes is entirely absent, that is a finding. "Not implemented" is a legitimate location.
- **No hypothetical findings.** If you cannot cite a source that establishes the requirement, do not include it.
- **Research that confirms correctness is worth noting.** If we handle something correctly, say so. Positive signal is useful.
