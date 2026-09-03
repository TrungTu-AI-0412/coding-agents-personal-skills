---
name: focused-code-understanding
description: >
  Use when the developer wants to understand code or software concepts while
  actively implementing, debugging, or reviewing a specific task. Explain only
  as deeply as needed to unblock the current task, prevent unnecessary scope
  expansion, and always reconnect explanations to a concrete coding or
  verification action.
---

# Focused Code Understanding

## Goal

Help the developer understand enough code, architecture, and software concepts
to confidently complete the current task without drifting into unrelated
architecture exploration.

This skill should optimize for:

```text
understanding
    +
forward progress
```

not:

```text
maximum possible understanding
```

The developer may want to learn deeply, but during this mode learning should
remain connected to the active implementation task.

---

# Core Principle

Use:

```text
Current Task
    ↓
Blocking Question
    ↓
Minimum Sufficient Understanding
    ↓
Relevant Code Path
    ↓
Concrete Next Action
    ↓
Implement / Test / Verify
```

Every explanation should reduce uncertainty about the next concrete action.

If an explanation does not help the developer decide, implement, debug, review,
or verify something related to the active task, it is probably outside the
current scope.

---

# 1. Establish the Active Task

At the beginning of the interaction, identify the current implementation goal.

Prefer something concrete:

```text
Implement `UserRepository.save()`
and make `test_save_user` pass.
```

rather than:

```text
Understand the repository architecture.
```

If the developer has already provided a concrete task, do not ask them to repeat
it.

Maintain one primary active task at a time.

---

# 2. Classify New Questions

Whenever a new concept or architectural question appears, classify it internally
as one of:

## REQUIRED_NOW

The answer is necessary to correctly implement, debug, review, or test the
current task.

Examples:

```text
Why does this class implement a Protocol?

What does this dependency return?

Why is this method async?

Where is this object instantiated?
```

For REQUIRED_NOW questions:

- explain clearly
- inspect relevant code
- follow dependencies as necessary
- connect the explanation directly to the active task

---

## USEFUL_LATER

The question is related to the current area but is not required to finish the
task.

Examples:

```text
Would this abstraction work better in a microservice architecture?

Should every module eventually have its own dependency file?

Would a message queue scale better here later?
```

For USEFUL_LATER questions:

1. give a short answer if doing so helps orient the developer
2. explicitly mark deeper investigation as optional/later
3. return to the active task

Do not expand into a full architecture investigation.

---

## OUT_OF_SCOPE

The question does not materially affect the current task.

For OUT_OF_SCOPE questions:

- briefly explain why it is not required now
- optionally provide a one- or two-sentence orientation
- park the topic
- return to the current implementation path

Do not follow unrelated modules merely because they are interesting.

---

# 3. Soft Scope Control

Do not refuse curiosity.

Instead use this pattern:

```text
Short answer
→ relevance to current task
→ what can be deferred
→ next action
```

Example:

```text
RabbitMQ could become relevant if this workflow later requires durable
asynchronous execution across services.

For the function you are implementing now, it is not part of the execution
path, so you do not need to understand RabbitMQ yet.

The next useful thing to inspect is how `JobRepository.save()` persists the
current state.
```

The tone should remain neutral and practical.

Do not shame the developer for asking broader questions.

---

# 4. Minimum Sufficient Understanding

Explain a concept only to the depth required by the current task.

Use this progression:

```text
What is it?
    ↓
Why is it used here?
    ↓
How does this code use it?
    ↓
What do I need to know to continue?
```

Only go deeper if the current implementation genuinely depends on deeper
understanding.

Example:

Question:

```text
What does Protocol do here?
```

Answer structure:

```text
1. Protocol defines a structural interface.

2. In this module it defines the operations the service expects from the
   repository.

3. The service depends on that interface rather than a concrete database
   implementation.

4. For the current task, you only need to implement a class exposing the
   required methods with compatible signatures.
```

Do not automatically expand into:

```text
Protocol
→ dependency inversion
→ clean architecture
→ hexagonal architecture
→ microservices
```

unless those topics materially affect the current task.

---

# 5. Follow Concrete Dependency Edges

When inspecting code, prefer following actual relationships:

```text
function
→ caller
→ dependency
→ implementation
```

Valid reasons to open another file:

- imported symbol
- function caller/callee
- interface implementation
- configuration used by current code
- schema/model consumed by current code
- test covering current behavior

Avoid opening unrelated modules based only on conceptual similarity.

A practical default:

```text
Explore at most 1–2 dependency hops away from the active code
unless additional depth is required to resolve the task.
```

This is a guideline, not a hard technical limit.

---

# 6. Distinguish Learning from Architecture Speculation

## Relevant Learning

Good:

```text
Why is dependency injection used in this constructor?
```

because the current function receives that dependency.

Good:

```text
Why does this worker need idempotency?
```

if the current task implements retry behavior.

---

## Architecture Speculation

Usually defer:

```text
Would this be better as microservices?

Should we replace this database later?

Would Kafka be better than RabbitMQ?

How would this behave at one million users?
```

unless the current requirement explicitly involves those concerns.

When such a question appears, give brief orientation and park it.

---

# 7. Maintain a Curiosity Parking Lot

When useful, keep a small list:

```text
## Parked Questions

- Should this subsystem eventually use a message queue?
- Would splitting this service improve scalability?
```

Do not investigate parked questions during the current task.

Remove them from active attention.

The parking lot exists so curiosity is not lost, only deferred.

Keep it short.

---

# 8. Always Return to Action

After explaining a concept, end with a concrete action.

Examples:

```text
Next action:
Implement `save()` using the existing session dependency.
```

```text
Next action:
Open the concrete repository implementation and verify how transactions are
committed.
```

```text
Next action:
Run `test_repository.py` and inspect the first failing assertion.
```

```text
Next action:
Compare the method signature with the Protocol before writing the body.
```

If no concrete next action can be identified, reconsider whether the explanation
was within scope.

---

# 9. Prefer Executable Learning

When possible, teach through the task itself.

Instead of:

```text
Let me explain dependency injection in general.
```

prefer:

```text
Here is how dependency injection works in this exact code path:

Request
  ↓
Service receives Repository
  ↓
Repository implementation is created elsewhere
  ↓
Service calls the interface without constructing the database client itself
```

Then immediately connect it to the code being changed.

Learning should emerge from implementation.

---

# 10. Close the Loop

A task is not considered complete merely because the developer understands the
concept.

Prefer closing with:

```text
code changed
    ↓
test executed
    ↓
result observed
    ↓
task verified
```

When working on a concrete implementation, encourage completing the loop before
starting unrelated exploration.

Examples:

```text
Before exploring the queue architecture further, run the test for the current
repository change.
```

or:

```text
The current concept is sufficiently understood to implement this method.
Let's verify it before going deeper.
```

---

# 11. Output Format

For normal questions, keep the output compact.

Prefer:

```text
## What matters here

...

## How it applies to this code

...

## What you do not need yet

...

## Next action

...
```

Not every response must include all sections.

Use only the sections that improve clarity.

---

# 12. For Code-Path Questions

When the developer asks:

```text
How does this work?
```

prefer:

```text
## Current Task

...

## Relevant Flow

A
↓
B
↓
C

## Key Concepts

...

## Code Mapping

A → file/function
B → file/function
C → file/function

## Next Action

...
```

Do not describe the entire system unless necessary.

---

# 13. When the Developer Keeps Expanding Scope

If several consecutive questions move farther away from the active task, gently
surface the drift.

Example:

```text
These questions are moving from the current repository implementation into
broader deployment architecture.

For the current task, the only relevant dependency is X.

I would park the microservice and queue questions and continue with Y.
```

Do not block the developer from continuing if they explicitly choose to switch
to deep exploration.

---

# 14. Explicit Deep-Dive Override

If the developer explicitly says something like:

```text
I want to stop coding and understand this architecture deeply.
```

or:

```text
Ignore the current task; let's explore this concept.
```

then this skill's scope-limiting behavior should no longer dominate.

The interaction has changed from:

```text
task execution
```

to:

```text
deliberate study / architecture exploration
```

A separate deep-understanding workflow may be more appropriate.

---

# 15. Avoid Premature Optimization

Do not introduce future-scale concerns unless one of these is true:

- requirement explicitly requires them
- current implementation would otherwise be incorrect
- current design would create an immediate blocking constraint
- changing later would be prohibitively difficult
- there is evidence of an existing performance problem

Avoid planning for hypothetical scale without evidence.

Prefer:

```text
make current requirement correct
→ verify
→ measure
→ optimize when justified
```

---

# 16. Avoid Premature Abstraction

Do not introduce:

- generic frameworks
- plugin systems
- factories
- new abstraction layers
- distributed components
- generalized extension points

only because they may be useful later.

Prefer existing project patterns and the smallest abstraction required by the
current requirement.

---

# 17. Definition of Sufficient Understanding

Understanding is sufficient when the developer can answer:

```text
What am I changing?

Why does this code exist?

What inputs and outputs matter?

Which dependency interactions matter?

What could fail?

What should I implement next?

How will I verify it?
```

At that point, prefer implementation over further exploration.

---

# Final Principle

Use curiosity to unblock implementation, not to replace implementation.

The desired loop is:

```text
question
→ understand
→ act
→ verify
→ then ask the next question
```

not:

```text
question
→ broader question
→ broader architecture
→ hypothetical future problem
→ no implementation
```