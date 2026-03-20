# Working With Claude — How One Developer Built Complex Systems With AI

This document describes how StockSignal and CleverShares were built through sustained human-AI collaboration. Not a tutorial, not a productivity hack — an honest account of what worked, what was learned, and what this kind of collaboration actually looks like in practice.

---

## The Setup

StockSignal is a production system with 190+ API endpoints, 480+ signal specifications, 104,000+ stocks, and multiple frontends. CleverShares is a gamified financial literacy platform with ~196 domain entities, 14 solution projects, and a simulation loop that exercises the full game through real code paths. Both were built by one non-traditional developer working with Claude (Anthropic) as a primary coding partner, and ChatGPT (OpenAI) as a thinking and alignment partner.

The human is not a traditional software engineer. He doesn't write code line by line. He thinks in systems, structures, and responsibilities. His skill is knowing what something should look like, how pieces should relate, and when something feels wrong — even when he can't always articulate the technical reason.

Claude writes the code. The human builds the scaffolding that makes the code coherent.

---

## What "Scaffolding" Means

The most important thing the human learned — and the thing that makes this collaboration effective — is that **AI doesn't maintain context on its own. You have to build the context for it.**

StockSignal has:
- A 300-line `CLAUDE.md` file that defines architecture, layer responsibilities, coding principles, and decision rules
- 73 documentation files in the backend `docs/` folder
- 22 documentation files inside the SignalEngine alone
- 86 product and strategy documents in a separate `stocksignal-docs` repository
- A `working-with-claude.md` file that defines roles, values, and when to ask vs. act
- A `claude-context.md` that consolidates philosophy, signals, stories, and architecture into one entry point
- Session continuity notes (`WORKING_NOTES.md`) that carry state between conversations

None of this documentation was written because someone told him to. It was written because **without it, every new conversation with Claude starts from zero.** The documentation is not for humans reading the code. It is scaffolding that allows an AI to re-enter a complex system and make good decisions.

This is the core insight: **the quality of AI output is bounded by the quality of the scaffolding you give it.**

---

## How the Collaboration Actually Works

### The human decides what, Claude decides how

The human doesn't dictate implementation details. He describes what a layer should do, what responsibility it owns, what it should not do. Claude figures out the code.

Example: The human said signals should be "pure measurements with no interpretation." Claude built a signal engine with 480+ specs, each with a declarative definition and an executable type, organized by epistemic category. The human didn't design the type hierarchy or the category system — he defined the constraint, and the structure emerged.

### Documents before code

Before building a new subsystem, the human and Claude (or ChatGPT) write a document. Not a spec in the traditional sense — more like a philosophy document. What is this thing? What is it not? What are the boundaries?

Examples:
- `story-engine-philosophy.md` — Written before stories were implemented. Defines what a story is, how it emerges from signals, what it can and cannot claim.
- `ai_narration_context_prompt_template.md` — Written before any AI content generation. Defines tone, rules, canonical prompt structure.
- `company_reality_descriptions_cybernetic_lens.md` — Written before company descriptions. Defines the "cybernetic lens" perspective.
- `architecture-spine-core-principles.md` — Written to stabilize architectural decisions after they were made. Prevents drift.

These documents serve two purposes:
1. **Alignment** — The human and AI agree on what something means before building it
2. **Re-entry** — Future sessions can read the document and understand the intent without re-deriving it

### Correction through conversation, not code review

The human doesn't read every line of code. He can't — and that's fine. Instead, correction happens through conversation.

When something feels wrong, the human says so. Not technically — structurally. "This feels too complicated." "This doesn't belong here." "Why are there three files doing the same thing?"

Claude then investigates, explains, and either justifies the approach or fixes it. The human's role is not to know the answer — it's to detect when the shape is wrong.

### Loops, not sprints

The work doesn't follow a sprint cadence. It follows loops:

1. **Discuss** — What are we building? What does it mean?
2. **Document** — Write down the philosophy and constraints
3. **Build** — Claude implements within the constraints
4. **Check** — Human reviews the shape (not the code)
5. **Correct** — Adjust if the shape drifted
6. **Document again** — Capture what was learned

These loops happen within a single session and across sessions. The documentation accumulates, and each new session starts with more context than the last.

---

## What the Human Learned

### 1. You are not managing a tool — you are managing context

The single most important skill in working with AI is context management. Not prompt engineering. Not clever instructions. Just: **making sure the AI has access to the right information at the right time.**

This means:
- Writing things down when they're decided
- Keeping documents current
- Structuring information so AI can find it
- Creating a `CLAUDE.md` that serves as a landing page for the system

Without this, every session is a fresh start. With it, sessions build on each other.

### 2. Your value is in structure, not syntax

The human doesn't need to know C# syntax to build a C# system. What he needs to know is:
- What layers should exist
- What responsibility each layer owns
- When something is in the wrong place
- What the product should feel like

This is genuinely enough. The AI handles syntax, patterns, framework knowledge, and implementation details. The human handles meaning, intent, and structural coherence.

### 3. Documents are the actual product

The code will change. Features will be added and removed. But the documents — the philosophy, the constraints, the architectural principles — those are what make the system coherent over time.

StockSignal's `CLAUDE.md` is not documentation in the traditional sense. It's a **control surface**. It tells the AI how to think about the system. When the human updates it, the AI's behavior changes. It's the most leveraged file in the entire codebase.

### 4. Verifying "shape" is more important than verifying "correctness"

The human can't verify that a signal calculation is mathematically correct. But he can verify:
- Does this signal have one responsibility?
- Is the spec separate from the type?
- Does this follow the same pattern as other signals?
- Does this feel like it belongs?

Shape verification catches most structural problems. The remaining bugs are implementation details that tests and iteration catch.

### 5. Two AIs serve different roles

ChatGPT and Claude are not interchangeable. In this workflow:

- **ChatGPT** serves as a thinking partner — translating vague feelings into clear concepts, pushing back on fuzzy ideas, helping the human articulate what he means
- **Claude** serves as an executor — reading the codebase, implementing features, maintaining architectural consistency, making judgment calls within constraints

The human sits between them. ChatGPT helps him think. Claude helps him build. The documents connect the two.

---

## What Claude Learned (Patterns That Emerged)

### Respect existing patterns before creating new ones

The `working-with-claude.md` in the codebase says it directly: "Before writing new code: look for an existing pattern, reuse or extend it, reduce duplication, only then add new code."

This rule exists because the biggest risk in AI-assisted development is **pattern proliferation**. Without this constraint, every session creates its own approach, and the codebase fragments.

### Confidence threshold for structural changes

Claude can implement, optimize, clean up, and reduce duplication freely. But Claude must pause and ask before: introducing new architectural layers, renaming core concepts, moving responsibility between domains, or rewriting stable logic.

This rule exists because the human can't detect every structural change in real-time. Some changes look small in code but are large in meaning. The pause gives the human a chance to verify the shape.

### "Finished" has a specific meaning

A task is finished when:
- Unnecessary layers are removed
- Code shape matches mental model
- Ownership of responsibility is clear
- Future debugging cost is reduced

Not when all edge cases are handled. Not when tests cover 100%. The optimization target is **clarity under pressure** — can someone re-enter this code six months later and understand what it does?

---

## What Works Better Than Average

### The documentation density

Across both projects: 180+ documents for StockSignal (73 backend + 22 signal engine + 86 product docs), plus CleverShares' three-layer governance system (CLAUDE.md, CLAUDE_RULES.md, BUSINESS_RULES.md) with canonical domain references, game traces, and generation contracts. Most projects this size have a README and maybe an architecture diagram.

This isn't over-documentation. Every document exists because without it, a future AI session would make worse decisions. The documentation is infrastructure, not bureaucracy.

### The layer discipline

The layered architecture (Controller → Facade → BusinessService → Repository → Domain) has held through 190+ endpoints, 480+ signals, and 35 batch executors. Layers didn't leak. Responsibilities didn't blur. This happened because the rules were documented early and reinforced in every session.

### The philosophical grounding

StockSignal doesn't just have technical architecture — it has philosophical architecture. "Signals describe what IS, not what will happen." "Stories emerge from alignment, not prediction." "Describe, don't prescribe."

These aren't marketing statements. They are **constraint rules** that shape every technical decision. When Claude needs to decide how a new signal should work, the philosophy tells it. When a story feels wrong, the philosophy explains why.

### The honest incompleteness

The documents don't pretend the system is finished or perfect. `WORKING_NOTES.md` lists semantic drift. `CLAUDE.md` says "imperfect is expected." The architecture principles say "not everything fits neatly, and that's okay."

This honesty makes the collaboration more effective. When the AI knows what's uncertain, it makes better judgment calls about when to ask vs. when to act.

---

## CleverShares: Where the Collaboration Model Evolved

CleverShares introduced patterns that didn't exist in StockSignal, demonstrating that the collaboration model adapts to the domain.

### Mock as specification source

StockSignal's business rules emerged from philosophy documents and architectural reasoning. CleverShares' rules emerge from a **frontend mock** — a UI prototype that represents what the game should feel like. The mock is evidence of intent; Claude and the human examine it together to extract the implicit rules, which are then documented in `BUSINESS_RULES.md` as the canonical authority.

This inverts the typical spec-first workflow. The mock comes first. The spec is derived. The simulation verifies.

### Hard boundaries as trust mechanism

CleverShares added `CLAUDE_RULES.md` — a document that explicitly lists what the AI is not allowed to do (no schema changes, no repository modifications, no layer re-architecture). This didn't exist in StockSignal because the codebase was smaller when the collaboration started.

The hard boundaries serve a trust function: the human can delegate more confidently because the most dangerous categories of change are excluded by rule. The AI works freely within its allowed scope and stops when it reaches a boundary. This reduces the human's verification burden without reducing the AI's productivity.

### Simulation as the third collaborator

The student simulation loop acts almost as a third collaborator. When the human and AI make a business rule change, the simulation immediately tells them whether it broke something. This tightens the feedback loop — changes are verified in seconds, not discovered in testing days later.

Game traces (dated snapshots of simulation output) provide a temporal record of how behavior evolved. When something unexpected appears in a trace, it prompts a conversation: was this intended? Is the rule correct? Should the simulation change, or should the code?

---

## Where Improvements Could Be Made

### Session continuity is still fragile

Despite the documentation, each new conversation still requires re-orientation. The `CLAUDE.md` helps enormously, but complex ongoing work (like the signal quality audit or the article production pipeline) still loses momentum between sessions.

Better session handoff — a structured "here's exactly where we left off" format — would reduce the warmup cost of each session.

### Testing lags behind implementation

The collaboration naturally favors building over testing. The human's instinct is to see the shape, and shape verification doesn't require tests. As a result, test coverage is lower than it should be for a system this complex.

A deliberate testing loop — "we built three features, now we write tests for them" — would improve long-term stability.

### Verification of generated content

AI-generated content (company descriptions, coordination classifications) is generated once and kept. But verification of that content is manual and incomplete. A systematic quality review process for generated content would strengthen the product.

### Knowledge dependency on documents

If the documents became stale, the AI's decision quality would degrade rapidly. There's no automated check for document freshness. A "last verified" date or a periodic review cadence would help.

---

## The Core Principle

> The goal is not elegance. The goal is **clarity under pressure**.

This principle, from the project's `working-with-claude.md`, captures the entire collaboration philosophy. The system is not built to be impressive. It is built to be understood — by the human, by the AI, and by anyone who encounters it later.

The scaffolding exists so the AI can work. The AI works so the human can build something real. The documentation survives so the system can grow without losing coherence.

That's the loop. That's what makes it work.
