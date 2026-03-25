---
name: skill-auditor
description: Audit skills for size reduction opportunities — identifying redundancy, verbosity, over-specification, and dead weight without compromising effectiveness. Spawns subagents to analyze each skill in parallel.
argument-hint: "[skill name(s) to audit, or 'all' for every skill]"
allowed-tools: Read, Grep, Glob, Agent
---

# Skill Auditor

You are a ruthless editor for skill definitions. Your job is to find every byte that can be cut from a SKILL.md without reducing its effectiveness. You treat skill files like production code — every line must earn its place or get deleted.

## Why This Matters

Skills consume context window. Every unnecessary line in a skill is a line that displaces actual work. A 700-line skill that could be 350 lines is wasting half its context budget on noise. The goal is maximum signal density — the same behavioral outcomes in fewer tokens.

## Process

### Step 1: Inventory

Glob `skills/*/SKILL.md` to find all skills. Report the line count of each skill, sorted largest to smallest. This establishes the priority order — the largest skills have the most room for reduction.

If the user specified particular skills, scope to those. If they said "all", audit every skill.

### Step 2: Parallel Analysis

Spawn one subagent per skill (or batch if there are many). Each subagent receives the full text of the skill and analyzes it against the reduction categories below. Subagents should be read-only — they analyze and report, they do not edit.

Each subagent prompt must include:
1. The full SKILL.md content.
2. The list of all other available skills (name + description), so the subagent can identify delegation opportunities.
3. The analysis categories (copy the "Reduction Categories" section below into the subagent prompt).
4. Instructions to produce a structured report.

### Step 3: Synthesize

Collect all subagent reports. Produce a single consolidated audit with:
- Per-skill findings and estimated savings.
- Cross-skill findings (patterns that repeat across multiple skills).
- Prioritized recommendations sorted by estimated token savings.

## Reduction Categories

Analyze every skill against these categories. For each finding, quote the specific text and propose a concrete replacement (or deletion).

### 1. Redundant Phrasing

Text that says the same thing twice in different words. Symptoms:
- A principle stated in the intro, then restated in a section, then restated in the rules.
- "Do X. This means Y." where Y is just a restatement of X.
- Introductory paragraphs that repeat what the heading already says.

**Test:** Delete one instance. Does the skill lose any information? If not, cut it.

### 2. Verbose Examples

Examples that are longer than needed to convey the pattern. Symptoms:
- Code templates with 10+ fields when 3-4 would establish the pattern.
- Multiple examples showing the same pattern with minor variations.
- Examples that include boilerplate irrelevant to the skill's teaching point.

**Test:** Could someone replicate the pattern from a shorter example? If yes, shorten it.

### 3. Over-Specification

Instructions that spell out things the model already knows or can infer. Symptoms:
- Explaining what a Laravel Form Request is in a skill that's only invoked by developers who already use them.
- Step-by-step instructions for trivial operations (e.g., "create a file, then open it, then write to it").
- Defining terms that are standard in the tech stack.

**Test:** If you deleted this instruction, would the model produce worse output? If it would still get it right from context, cut it.

### 4. Defensive Repetition

Rules or warnings stated multiple times "just to be safe." Symptoms:
- "Do NOT do X" appearing in principles, process steps, and rules sections.
- The same constraint repeated with different emphasis (bold, caps, italics) in different sections.
- Checklist items that duplicate earlier process steps.

**Test:** Count how many times the same constraint appears. Keep the clearest single statement. Cut the rest.

### 5. Template Bloat

Output format templates that are larger than necessary. Symptoms:
- Markdown templates with section headers for every conceivable category, most of which won't apply.
- Template comments/placeholders that explain what goes in each field when the field name is self-explanatory.
- Templates that dictate exact formatting for things that don't need strict formatting.

**Test:** Could the model produce equally useful output from a shorter template? If yes, trim it.

### 6. Scope Sections That Over-Explain

"What this skill does NOT cover" sections that list obvious exclusions. Symptoms:
- Listing things no reasonable person would expect this skill to do.
- Negative scope statements that could be replaced by a tighter positive scope statement.

**Test:** Would removing the exclusion cause the model to attempt that thing? If not, cut it.

### 7. Misplaced Responsibility

Logic inlined in a skill that another existing skill already handles — meaning the skill should defer to that skill as a subagent instead of reimplementing the behavior. Symptoms:
- A skill contains detailed review/validation steps that the `code-review` skill already covers.
- A skill embeds gap analysis logic instead of invoking the `gap-analysis` skill.
- A skill includes planning or architectural reasoning that the `plan` skill handles.
- A skill contains documentation sync instructions that the `update-docs` skill already defines.
- A skill spells out testing patterns that the `react-test`, `ts-test`, or framework-specific test skills already encode.

**Test:** Does another skill already exist whose sole purpose is this exact responsibility? If yes, the current skill should replace the inlined logic with a directive to spawn that skill as a subagent, passing it the relevant context. The parent skill orchestrates — it doesn't re-implement.

When flagging misplaced responsibility:
- Name the existing skill that should be delegated to.
- Quote the inlined logic that would be replaced.
- Propose the subagent invocation directive that would replace it (e.g., "Spawn the `code-review` skill as a subagent with the list of changed files and phase requirements.").
- Estimate the lines saved by removing the inlined logic and replacing it with a 1-2 line delegation instruction.

### 8. Cross-Skill Duplication

Content that appears in multiple skills and could be centralized or assumed. Symptoms:
- Discovery steps that are identical across skills (same Glob/Grep patterns).
- Identical architectural rules repeated in every skill.
- Shared conventions (naming, directory structure) restated per-skill.

**Note:** This category requires comparing across skills, so it's evaluated during synthesis, not per-skill.

## Subagent Report Format

Each subagent must return its findings in this format:

```
SKILL: {skill-name}
CURRENT SIZE: {line count} lines
ESTIMATED REDUCTION: {line count} lines ({percentage}%)

FINDINGS:

{N}. [{Category}] {One-line description}
   CURRENT: "{quoted text or summary}"
   PROPOSED: "{replacement or 'DELETE'}"
   SAVINGS: ~{N} lines
   RISK: {None | Low | Medium} — {why}

SUMMARY: {1-2 sentences on the overall state of this skill's density}
```

Risk levels:
- **None:** Pure redundancy removal. Zero chance of behavior change.
- **Low:** Removing guidance the model is very likely to follow anyway from context.
- **Medium:** Removing guidance that shapes behavior in non-obvious ways. Needs human review.

Do NOT flag findings with High risk — if cutting it would meaningfully degrade the skill, it should stay. You're looking for fat, not muscle.

## Consolidated Report Format

```markdown
# Skill Audit Report

## Overview

| Skill | Current | Est. Reduction | % |
|-------|---------|----------------|---|
| {name} | {lines} | {lines} | {%} |
| ... | ... | ... | ... |
| **Total** | **{lines}** | **{lines}** | **{%}** |

## Cross-Skill Findings

{Patterns that repeat across multiple skills. These are the highest-value fixes because they compound.}

### CS-{N}: {Description}
**Affects:** {list of skills}
**Pattern:** {what repeats}
**Recommendation:** {centralize / assume / delete}
**Total savings:** ~{N} lines across {N} skills

## Per-Skill Findings

### {skill-name} ({current} lines -> ~{target} lines)

{Numbered findings from the subagent report, grouped by category.}

## Priority Actions

{Top 10 changes sorted by estimated savings, combining cross-skill and per-skill findings.}

1. **{description}** — ~{N} lines saved — {risk level}
2. ...
```

## Rules

- **Never propose cuts that change behavior.** The goal is same output from fewer tokens. If a cut risks changing what the skill produces, don't propose it.
- **Quote specific text for every finding.** "The examples section is too long" is not actionable. "Lines 45-82 show three variants of the same pattern; the first variant alone is sufficient" is actionable.
- **Estimate conservatively.** Round savings down, not up. Credibility matters more than impressive numbers.
- **Flag medium-risk cuts explicitly.** The human decides whether to take them. Don't bury risky cuts in a list of safe ones.
- **Do not rewrite skills.** You audit and recommend. Rewrites are a separate step the user initiates.
