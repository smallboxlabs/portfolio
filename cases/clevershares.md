# CleverShares — Case Study

A gamified financial literacy platform for young people. Designed to teach real-world financial concepts through simulated trading, structured progression, and family agreements — built end-to-end by one developer working with AI.

---

## What CleverShares Is

CleverShares teaches financial literacy through play. Students trade real stocks with virtual cash, complete lessons, earn XP, progress through levels, and negotiate real-world agreements with family members backed by in-game achievement.

It is not a toy trading app. The underlying model encodes genuine financial mechanics — fees, position sizing, portfolio weight — so that the learning transfers. The gamification exists to create engagement structure, not to simplify away complexity.

**Target audience:** Students and young people learning about money, investing, and economic structure. Families who want financial education to have real-world consequences.

---

## Scale

| Metric | Count |
|--------|-------|
| Domain entities | ~196 |
| Solution projects | 14 |
| Game mechanics | XP, levels, quests, achievements, shop, arena, agreements |
| Screen types | 10+ (gated by level progression) |
| Business rules document | Canonical, actively maintained |

---

## Architecture

CleverShares shares the same layered architecture principles as StockSignal, adapted for a game-mechanics domain.

```
┌──────────────────────────────────────────────┐
│ StudentPageController (READ)                 │
│ Returns complete page DTOs for every screen. │
│ Read-only. No mutations.                     │
├──────────────────────────────────────────────┤
│ StudentController (ACT)                      │
│ Handles all student actions: trade, answer,  │
│ allocate XP, buy items, claim rewards.       │
└──────────────┬───────────────────────────────┘
               │
┌──────────────▼───────────────────────────────┐
│ Facade (switchboard)                         │
│ Coordinates BusinessService calls.           │
│ Assembles DTOs. Generates display strings.   │
│ No business decisions.                       │
└──────────────┬───────────────────────────────┘
               │
┌──────────────▼───────────────────────────────┐
│ BusinessServices (where meaning lives)       │
│ XP calculations, level progression, fees,    │
│ agreement state machines, trading rules.     │
│ Returns entities or Models. Never DTOs.      │
└──────────────┬───────────────────────────────┘
               │
┌──────────────▼───────────────────────────────┐
│ Repository (R.cs — single DbContext)         │
│ All data access. EF Core, SQL Server.        │
└──────────────┬───────────────────────────────┘
               │
┌──────────────▼───────────────────────────────┐
│ Domain (~196 entities)                       │
│ Pure business state: students, XP, stocks,   │
│ agreements, lessons, shop, achievements.     │
└──────────────────────────────────────────────┘
```

### Read → Act → Read

This is the core interaction contract. Every user flow follows it:

1. Frontend calls a page endpoint → gets a complete screen DTO
2. Student sees state, decides to act
3. Frontend calls an action endpoint → action executes through Facade → BusinessServices
4. Frontend calls the page endpoint again → sees updated state

The UI only knows what page DTOs tell it. The UI can only do what action endpoints allow. This pattern is not a suggestion — it is enforced by the controller split and verified in generation.

---

## The Domain: Game Mechanics as a Coherent Model

CleverShares is not a collection of features. It is a progression system where every mechanic feeds into a unified model.

### XP System (the ledger)

XP is stored as a transaction ledger. Positive entries are earnings (lessons, trades, quests). Negative entries are allocations (XP committed to family agreements).

```
totalEarnedXp >= 0                        (sum of positive transactions)
allocatedXp >= 0                          (abs of negative transactions)
availableXp = totalEarnedXp - allocatedXp
allocatedXp <= totalEarnedXp
```

These are not guidelines — they are invariants enforced in code and verified during generation. Total Earned XP drives levels, rank, salary, and leaderboard position. Allocating XP to agreements reduces Available XP but never reduces level. This distinction is business-critical: a student who commits XP to a family agreement should not lose their rank.

### Level Progression

Levels are gated by cumulative XP. Each level requires more XP than the last: `xpForLevel(N) = 200 + (N - 1) × 50`. Levels unlock screens:

| Level | Unlocks |
|-------|---------|
| 1 | Invest, Academy |
| 2 | Agreements |
| 3 | Arcade |
| 4 | Office |
| 5 | Shop |
| 6 | Social |
| 7 | Arena |
| 8 | Leaderboard |
| 9 | Entrepreneur |
| 10 | Real Estate |

This gating is structural, not cosmetic. A level-1 student cannot access the shop. The system presents features only when the student has enough context to use them.

### Agreements (the bridge to real life)

Agreements are the most distinctive mechanic. A student allocates XP toward a goal. A family member defines a real-world reward. When the student earns enough XP, the agreement completes and the family member fulfills their commitment.

State machine: `Draft → Pending → Active → Completed` (or `Cancelled` with XP refund).

This creates a feedback loop between game progression and real-world consequences. The XP is not abstract — it represents engagement that the family has agreed to reward.

### Trading Simulation

Students trade real stocks with virtual cash ($10,000 starting balance). Trades include fees, position tracking, and portfolio weight calculations. The mechanics are realistic enough that lessons transfer, but simplified enough that mistakes are inexpensive.

---

## What Makes CleverShares Architecturally Unusual

### The Mock-Driven Domain Model

CleverShares uses **interface contracts as the specification layer** for domain entities. Before an entity exists, an interface defines what it means.

```
IAgreement → "has TargetXp, IconKey"
ISubAgreement → "has Name, Description" (localized)
```

Domain entities implement these interfaces. Generation code validates against them. The interface is the contract — the entity is the implementation.

This pattern serves a specific purpose: **the interface defines business meaning, the entity handles persistence.** When generation creates an agreement, it does not check if the Agreement entity has the right fields. It checks if the entity satisfies the `IAgreement` contract. When a new property is needed, it appears in the interface first — then propagates to the entity and the generation.

This is not traditional mocking (Mockito, Moq). It is closer to design-by-contract: the interface is the specification, and the domain model is derived from it.

### The Student Simulation Loop

CleverShares has an integration test pattern disguised as data generation. The `DatabaseGenerationService` does not just create test data — it **simulates student journeys through the actual system.**

```
Phase 1: Infrastructure      → SubWebsites, users
Phase 2: Content             → Topics, lessons, categories
Phase 3: Stock data          → CSV import
Phase 4: Data ingestion      → Process until SubStocks exist
Phase 5: Educational content → Lessons, quizzes, facts
Phase 6: XP system           → Levels, quests, arcade, achievements
Phase 7: Shop & arena        → Shop items, pitch dialogues
Phase 8: Student journeys    → Full onboarding + activity simulation
Phase 9: Daily loop          → 10 days of simulated activity
```

Phases 8 and 9 are where it gets interesting. The simulation:

1. **Calls real Facade methods** — not test doubles, not mock data. The same code path that serves production requests.
2. **Follows Read → Act → Read** — reads page DTOs, performs actions, reads updated DTOs, verifies changes.
3. **Checks invariants after every action** — XP balance equations, cash balance after trades, quest completion prerequisites.
4. **Throws on violation** — if any invariant is broken, generation fails. No silent data corruption.

This means that **every time the database is rebuilt, the entire game loop is exercised end-to-end.** If a business rule change breaks the XP invariant during a simulated trade, you find out immediately — not when a student hits it in production.

The simulation produces realistic mid-progression data: students at different levels, varied lesson completion, active and completed agreements, trading histories. This data is useful for development, but the verification is the real value.

### Scaffolding for AI Collaboration

CleverShares pushes the scaffolding concept further than StockSignal. The system has three layers of documentation that control how AI works within the codebase:

**Layer 1: CLAUDE.md** — How to think about the code. Layer responsibilities, return types, key patterns, the Read → Act → Read contract. This is the entry point.

**Layer 2: CLAUDE_RULES.md** — Hard boundaries. No repository changes. No schema changes. No layer re-architecture. If a task requires crossing these boundaries, stop and ask. This prevents the AI from making well-intentioned structural changes that the human cannot verify.

**Layer 3: BUSINESS_RULES.md** — The canonical product contract. Every business rule — XP formulas, level thresholds, fee calculations, agreement states, screen unlock gates — documented in one place. Backend enforces it. Frontend aligns to it. Changes update this document first.

The three layers serve different purposes:
- CLAUDE.md answers "how should I write code here?"
- CLAUDE_RULES.md answers "what am I not allowed to do?"
- BUSINESS_RULES.md answers "what should the product actually do?"

This separation means the AI can work confidently within its allowed boundaries while the human retains control over structural and product decisions.

---

## How the Human and AI Work Together on CleverShares

The collaboration model is the same as StockSignal, but the game domain adds a distinctive element: **business rules are discovered, not invented.**

### Rules derived from the mock

The frontend mock (designed by the human) represents product intent — what the game should feel like, what screens exist, what actions are available. But the mock doesn't specify invariants, formulas, or edge cases.

The process:

1. Human designs the mock (UI, flows, screen progression)
2. Human and AI examine the mock together
3. Business rules are extracted from the mock's implicit behavior
4. Rules are documented in BUSINESS_RULES.md
5. Backend enforces the documented rules
6. The simulation verifies them

This is a distinctive workflow. Most projects start with a spec and build toward it. CleverShares starts with a mock and derives the spec from it. The mock is evidence of intent. The documentation is the authority. The simulation is the proof.

### The ambiguity protocol

When a rule cannot be determined with confidence from the mock, it is not guessed. It is documented as an ambiguity in BUSINESS_RULES.md and flagged for a product decision.

This protocol exists because the human is not always available to answer immediately, and the AI should not fill gaps with assumptions. An explicit "we don't know yet" is more valuable than a plausible guess that becomes an invisible commitment.

---

## Key Design Decisions

### Interface contracts before entities

**Decision:** Define domain meaning through interfaces. Entities implement those interfaces.

**Why:** The interface is stable — "an agreement has a target XP" is a product decision. The entity is flexible — how that XP is stored, indexed, or loaded may change. Separating them allows the domain model to evolve without re-deriving meaning.

### Simulation as verification

**Decision:** Data generation exercises the full system through real Facade calls with invariant checking.

**Why:** Traditional unit tests verify individual methods. The simulation verifies that the methods compose correctly into a complete student journey. It catches integration errors — a level-up that doesn't unlock the right screen, an XP allocation that breaks the balance equation — that unit tests would miss.

**Trade-off:** Generation is slower than simple seed data. Accepted because the verification catches real bugs that would otherwise reach students.

### Three-document AI governance

**Decision:** Separate "how to code" (CLAUDE.md), "what's forbidden" (CLAUDE_RULES.md), and "what the product does" (BUSINESS_RULES.md).

**Why:** A single document conflates different concerns. The AI needs different information at different times. When implementing a feature, it needs CLAUDE.md. When considering a structural change, it needs CLAUDE_RULES.md. When calculating XP, it needs BUSINESS_RULES.md. Separation allows the AI to find the right answer quickly.

### Game traces as frozen observations

**Decision:** Periodically record what actually happens during simulation and store as dated, append-only snapshots.

**Why:** Game traces capture emergent behavior that no single test verifies. A trace might reveal that a student accumulates XP faster than expected, or that agreements complete before the intended timeline. These observations inform product decisions without being prescriptive.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend | C#, .NET 9, ASP.NET Core |
| ORM | Entity Framework Core 9.0 (code-first) |
| Database | SQL Server |
| Admin frontend | React, TypeScript |
| Public game | Next.js |
| External data | TwelveData API (stock prices) |
| Translation | DeepL API |
| Email | SMTP |
| Solution structure | 14 projects with strict dependency direction |

---

## What This System Demonstrates

**Domain modeling for game mechanics** — XP ledgers, level progression, agreement state machines, trading simulation — all modeled as a coherent domain with explicit invariants.

**Simulation-driven verification** — The system proves itself correct every time it rebuilds. The student simulation loop exercises real code paths and enforces real invariants.

**Interface-driven design** — Domain meaning is specified through contracts, not implementation. The mock informs the spec. The spec governs the code.

**Structured AI collaboration** — Three layers of documentation (intent, boundaries, rules) give the AI enough context to work autonomously within clear constraints.

**Rules derived from reality** — Business rules are not invented from requirements documents. They are extracted from the product mock, documented canonically, and verified through simulation.

---

*CleverShares shares architectural DNA with [StockSignal](stocksignal.md) but applies the same principles to a fundamentally different domain — demonstrating that the architecture and collaboration model generalize beyond a single product.*
