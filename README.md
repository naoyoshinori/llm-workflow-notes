# A Practical Design Note for Managing Long-Horizon LLM Workflows

**Author:** Naoyuki Yoshinori  
**License:** Creative Commons Attribution 4.0 International License  
<https://creativecommons.org/licenses/by/4.0/>  

***

## Overview

This repository contains a **practical design note** on how to manage state in long-running LLM workflows.

This document is a **personal design memo** based on implementation and observation.
It does **not define a system, framework, or formal methodology**.

This note describes a set of practical techniques for:

* managing state outside chat history
* selecting relevant context for each step
* defining task completion explicitly
* verifying execution results

This note particularly focuses on preventing:

* premature task closure
* reuse of stale or rejected context
* unverified completion

It is based on:

* experimental implementations of lightweight state-driven agents
* iterative observations from real usage
* attempts that appeared to reduce instability in long-horizon tasks

***

## What This Is (and What It Isn't)

### This is:

* a **personal design memo**
* a **collection of practical patterns**
* based on **implementation and observation**
* focused on **state management outside chat history**

### This is not:

* a research paper
* a validated methodology
* a formal theory
* a new algorithm or system architecture
* a proven solution for safety, correctness, or performance

***

## Motivation

When LLM agents are used for long-running tasks, the following issues are frequently observed:

* reuse of outdated requirements
* revival of previously rejected ideas
* unverified assumptions treated as facts
* premature task completion
* mixing active, closed, and obsolete states
* excessive reliance on accumulated chat history

In practice:

```text
More context ≠ more stability
```

Long histories may introduce more noise rather than clarity.

***

## Key Observation

From practical usage:

> Explicit constraints and verification appeared to reduce ambiguity in some cases.

In other words:

* when constraints, decisions, and verification results are explicitly tracked,
* outputs may be less influenced by irrelevant or obsolete context

This is not a theoretical claim, but an **operational observation**.

***

## Core Idea

Do not treat chat history as the only source of truth.

Instead:

```text
Explicit State
→ Select Working Context
→ Execute
→ Verify
→ Update State
```

The goal is to:

* keep only **relevant information active**
* reduce the chance of **context contamination**
* make **task status explicit and verifiable**

***

**Note**

Context contamination refers to situations where
obsolete, rejected, or unverified information affects current outputs.

Typical examples include:

* reuse of outdated requirements
* reintroduction of rejected ideas
* decisions influenced by unverified assumptions

***

## Explicit State

Explicit State refers to:

> Explicitly maintained task state outside the LLM conversation
> (e.g., tasks, blockers, done conditions, verification results, logs)

Examples:

* files (`tasks.md`, plans, logs)
* issue trackers
* CI logs
* structured notes

The format is not important.
What matters is that **state is not implicitly embedded in chat history**.

***

## Working Context

Working Context is:

> The subset of information provided to the LLM for the current step.

Working Context is not a formal component in the implementation.
It is a working subset of state derived from tasks, plans, blockers, and verification criteria.

It typically includes:

* current goal
* current task status
* done conditions
* unresolved blockers
* relevant files or interfaces
* verification method

### Important

Working Context selection is **not fully automated**.

It is currently:

* a manual or rule-based process
* subject to selection errors
* dependent on available state representations

### Limitations

Working Context selection is one of the most error-prone steps.

Failure modes include:

* missing necessary information
* including outdated or irrelevant context
* selecting incomplete or biased state

***

## Basic Selection Rules

### Usually Include

* current goal
* current state
* done conditions
* blockers
* verification method

### Usually Exclude

* closed tasks (details)
* rejected ideas
* obsolete requirements
* unrelated history

***

## Conflict Handling & Escalation

The priority below is used only for normal operation.

If conflicts or uncertainty arise, they must NOT be resolved silently.
Instead:

* escalate to the user when a decision is required
* or treat the issue as a blocker

In such cases, the task must remain open.

When information does not conflict, the following priority can be used:

```text
1. Latest explicit user instruction
2. Done Conditions
3. Approved constraints
4. Verified evidence
5. LLM-generated summaries
```

This is not an algorithm, but a **decision guideline under normal conditions**.

**Note**

If user instructions conflict with verified evidence,
the conflict should be made explicit instead of silently resolved.

***

## Done Conditions

Done Conditions define what "completion" means.

Example:

```text
- CLI accepts `--json`
- output is valid JSON
- existing behavior remains unchanged
- tests pass
```

Completion should not be declared without satisfying these conditions.

***

## Blockers

Blockers are:

> unresolved items that prevent safe or correct execution

Examples:

* unclear requirements
* missing API specifications
* permission issues
* risks involving data or cost

Rule:

```text
Do not silently resolve blockers by guessing.
```

***

## Assumptions

Not all uncertainty must block execution, but caution is required.

The current approach prefers treating uncertainty as blockers.

Assumptions are only used when:

* impact is low
* the decision is reversible
* verification is possible later

Assumption handling is not fully standardized.

***

## Reasoning vs Execution

Separate:

```text
Reasoning:
- design decisions
- rationale
- change history

Execution:
- current tasks
- status
- verification conditions
```

This separation may help prevent:

* reintroducing old design decisions
* mixing planning with execution
* untracked changes in intent

***

## Verification and Evidence

### Rule

```text
Do not close tasks without evidence.
```

***

### Evidence Strength

```text
Weak:
- LLM self-reported completion

Medium:
- command execution
- exit code
- timestamp
- environment context

Strong:
- CI job ID / URL
- test reports
- commit hash
- audit logs
- human review approval
```

**Note**

LLM-generated evidence may be unreliable.
Prefer externally verifiable artifacts when possible.

***

### When Verification Is Weak or Not Available

Some tasks cannot be strictly verified (e.g., writing, design decisions).

In such cases:

* weaker evidence (checklists, human review, comparisons) should be recorded
* such tasks should not be treated as strongly verified

***

## Complexity-Based Process (Tiers)

Use different levels of process depending on complexity:

```text
Tier 1:
- direct execution
- no state tracking

Tier 2:
- task tracking
- explicit done conditions

Tier 3:
- plan + task tracking + verification + logs
- long or complex workflows
```

**Note**

Incorrect selection of process level is a known source of failure.
If uncertain, using a higher tier tends to be safer, but may introduce overhead.

***

## Limitations

This approach does not guarantee correctness.

Known limitations include:

* incorrect Done Conditions
* Working Context selection errors
* missed blockers
* weak or fabricated evidence
* process overhead relative to task size
* reliance on human judgment

***

## Measurement (Not Implemented Yet)

Potential metrics:

* number of reopened tasks
* reuse of outdated requirements
* unverified completions
* time spent on verification
* rework frequency

Possible baselines:

* chat-history-only workflows
* task tracking without explicit done conditions
* ad-hoc note-based execution

No controlled evaluation has been performed.

***

## Summary

Core practices:

```text
Do not rely only on chat history
Maintain explicit state
Select working context intentionally
Define done conditions explicitly
Separate blockers and assumptions
Verify results
Record evidence before closing
```

***

## Origin

This design note originated from:

* a lightweight execution-agent implementation
* explicit state files (`tasks.md`, plans, logs)
* tiered execution rules
* verification-driven task closure

In the reference implementation:

* execution state is tracked in task files
* reasoning and execution are explicitly separated
* blockers prevent task closure
* completion requires explicit verification
* logs are used for traceability

The implementation is only an example, not a specification.

***

## Disclaimer

This document is:

* incomplete
* observational
* based on ongoing experimentation

It should not be used as:

* a standalone safety mechanism
* a production-ready framework
* a guarantee of reliability

***

## License

This work is licensed under the
**Creative Commons Attribution 4.0 International License**.

***

## Feedback

Feedback, criticism, and alternative implementations are welcome.
