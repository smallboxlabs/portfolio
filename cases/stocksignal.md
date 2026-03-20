# StockSignal — Case Study

A stock analysis and discovery platform built end-to-end. Backend, two frontends, infrastructure, data pipeline, and content generation — all designed, built, and deployed by one developer.

**Live at [stocksignal.me](https://stocksignal.me)**

---

## What StockSignal Is

StockSignal transforms raw financial data into structural observations about companies. It answers **"What is this company as a system?"** rather than "Will this stock go up?"

It is not a prediction engine, not a trading tool, and not financial advice. It describes structure — calmly, honestly, and with stated limits.

**Target audience:** Thoughtful investors, engineers, people who want to understand companies structurally rather than chase predictions.

---

## Scale

| Metric | Count |
|--------|-------|
| Stocks in data pipeline | 104,000+ across global exchanges |
| REST API endpoints | 190+ (public + admin) |
| Signal specifications | 480+ |
| Story types | 270+ (situational + diagnostic) |
| Indexed pages | 3,400+ |
| Domain entities | 103 |
| DTOs (API contracts) | 187 |
| BusinessServices | 41 |
| Repositories | 28 |
| Batch executors | 35 |

---

## Architecture

The backend follows a strict layered architecture. Each layer has one responsibility, and responsibilities do not leak between layers.

```
┌─────────────────────────────────────────┐
│ Controllers (HTTP entry point)          │
│ Knows HTTP. No business logic.          │
│ Public = read-only. Admin = mutations.  │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│ Facade (switchboard, not brain)         │
│ Sequences service calls.                │
│ Maps entities → DTOs. No decisions.     │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│ BusinessServices (where meaning lives)  │
│ Validation, authorization, calculations.│
│ Returns entities or Models.             │
│ Zero knowledge of HTTP or DTOs.         │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│ Repositories (data access boundary)     │
│ Single ownership per query.             │
│ Identity scoping enforced here.         │
│ Returns materialized entities.          │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│ Domain (pure business entities)         │
│ Structure and relationships only.       │
│ No knowledge of persistence or HTTP.    │
└─────────────────────────────────────────┘
```

**Why this shape:** When something goes wrong, there's exactly one place to look. Data access problems → Repositories. Business logic problems → BusinessServices. Request/response problems → Facade. HTTP problems → Controllers.

---

## The Signal → Story Chain

This is the core intellectual model of the system.

### Signals (measurement)

A signal is a single structural observation about a company. It measures one continuous value (0–100), describes what IS, and never predicts.

Examples:
- **Earnings Quality** — How reliably reported earnings convert to cash
- **Growth Consistency** — Stability of revenue/earnings trajectories
- **Cash Flow Margin** — Operating cash generation relative to revenue

480+ signal specifications, each with a declarative definition (what it means) and an executable type (how to calculate it).

Signals are organized into five epistemic categories:
- **Geometrical** — Where is it in a reference frame? (drawdown, percentile, distance to support)
- **Dynamic** — How is it moving? (rate of change, momentum, acceleration)
- **Structural** — What state is it in? (breakout, regime, structure intact/broken)
- **Temporal** — How long has this been true? (duration, age, time since peak)
- **Fundamental** — What constraints exist? (margins, leverage, cash flow stability)

### Stories (interpretation)

A story emerges when multiple independent signals align. Stories describe system state — they name recognizable conditions.

Examples:
- **Quality Compounder** — Self-reinforcing capital generation (requires Earnings Quality + Growth Consistency + Cash Flow Margin)
- **Dividend Fortress** — Sustainable, well-funded dividend structure
- **Deep Value Position** — Price significantly below structural value indicators

Stories are explicit about limits. Every story states what it cannot tell you.

### The boundary

Signals compress structure. Stories express meaning. If interpretation appears before measurement, something is wrong.

**Composition rule:** Stories combine signals *across* epistemic categories, never within. Combining two geometrical signals amplifies conviction without adding knowledge. Combining geometrical + fundamental creates genuine insight.

---

## Data Pipeline

The system ingests data from TwelveData across 104,000+ stocks globally.

### Data flow

```
Stock creation (batch import from TwelveData)
    ↓
Raw data ingestion (profiles, prices, financials, dividends, earnings)
    ↓
Derived computation (statistics, ratios, indicators)
    ↓
Signal evaluation (480+ structural measurements)
    ↓
Story resolution (pattern recognition from signal alignment)
    ↓
Content generation (AI narration via Claude API)
    ↓
Page rendering (public website, SEO, multi-language)
```

### Batch Coordinator

35 batch executors handle all background processing: data imports, price history at multiple intervals, financial statement ingestion, signal computation, AI content generation, and cache warming.

Key design decisions:
- **Staleness-based prioritization** — Stocks with oldest data update first
- **Credit accounting** — TwelveData API credits tracked per call, throttling when approaching limits
- **Idempotent execution** — Re-running any batch job produces correct results without data loss
- **UpdatesStart/UpdatesEnd tracking** — Every data type records when processing began and completed, enabling failure detection and progress monitoring

### TwelveData integration

The vendor integration is intentionally semantically coupled to TwelveData. Domain entities share shape with the external API models where appropriate, reducing translation layers.

The integration boundary is strict: TwelveData code makes API calls and records metadata (timing, credits, outcome). It does not decide retry behavior or control flow — that belongs to BatchCoordinator.

---

## AI Content Generation

StockSignal uses Claude (Anthropic) to generate three types of content:

1. **Company reality descriptions** — What kind of real-world system is this company?
2. **Coordination classification** — What economic role does this company play?
3. **Industry context** — What defines this industry structurally?

### Narration rules

Every AI-generated text follows strict rules:
- Describe, do not prescribe
- No predictions, no advice, no buy/sell language
- Always state limits
- Calm, factual tone — no excitement
- Present-tense structural observations only

These are not guidelines — they are enforced through the prompt structure. Every AI call follows a canonical template: paradigm header → invariant rules → curated context → task frame.

### Single-pass strategy

AI content is generated once and kept. It is not regenerated on each request. This controls cost and ensures stability. Only primary stocks (top selections per industry) receive AI-generated content.

---

## Authentication & Multi-tenancy

- **JWT-based authentication** with cookie and header dual-mode
- **Role-based authorization**: Super Admin (full access) and Regular Admin (scoped to SubWebsite)
- **SubWebsite filtering** enforced at the Repository level — where data is accessed, not where requests arrive
- **Admin impersonation** for debugging user-facing views

The identity constraint travels downward through the call stack. Controllers don't enforce scoping. BusinessServices don't enforce scoping. Repositories do — consistently, in one place.

---

## Integrations

| Service | Purpose | Boundary |
|---------|---------|----------|
| **TwelveData** | Financial market data (prices, fundamentals, profiles) | Isolated in `/TwelveData/`, credit-tracked |
| **Stripe** | Subscription billing, checkout, webhooks | `/ExternalServices/Stripe/` |
| **Anthropic Claude** | AI content generation (descriptions, classifications) | `/AIContent/Anthropic/` |
| **DeepL** | Translation (English → Danish, Polish) | `/ExternalServices/DeepLTranslation/` |
| **SMTP (Brevo)** | Transactional email (magic links, notifications) | `/ExternalServices/Email/` |

---

## Frontend

### Public website (Next.js)

Server-rendered, SEO-optimized, multi-language (English, Danish, Polish).

Page types:
- **Stock pages** — Company identity, coordination role, stories, signals
- **Industry/Sector/Country pages** — Contextual groupings
- **Screener** — Filter stocks by structural characteristics
- **Glossary** — Term definitions
- **Articles** — Editorial perspective

Custom SVG price charts and Canvas network visualizations — no external charting libraries.

### Admin dashboard (React + Mantine)

Stock management, signal/story editing, batch job monitoring, user management, system health. Drag-and-drop reordering, role-based routing.

---

## Infrastructure & Deployment

**Production:** Hetzner (Ubuntu server)

| Component | Detail |
|-----------|--------|
| Reverse proxy | nginx with SSL |
| Process management | systemd with security sandboxing and watchdog |
| CI/CD | GitHub webhook → deploy script → zero-downtime restart |
| Database | PostgreSQL with automated scheduled backups |
| Monitoring | System health endpoint (public, no auth required) |

No containers. Native deployment chosen for simplicity, debuggability, and direct control.

---

## Key Design Decisions

### Layered architecture with strict boundaries

**Decision:** Each layer has one responsibility. No leaking.

**Why:** When systems grow, the biggest cost is "where does this logic live?" With strict layers, every question has one answer. Data problems → Repositories. Business logic → BusinessServices. HTTP concerns → Controllers.

**Trade-off:** More files, more ceremony on simple operations. Accepted because the alternative — tangled layers — becomes exponentially more expensive over time.

### Semantic coupling to TwelveData

**Decision:** Domain entities share shape with external API models where natural.

**Why:** Adding abstraction layers between the vendor and the domain would obscure the actual data flow. The shapes are genuinely similar — a balance sheet is a balance sheet.

**Trade-off:** Replacing TwelveData would require touching domain entities. Accepted because clarity now is worth more than hypothetical vendor flexibility later.

### Specification + Type split for signals and stories

**Decision:** Every signal has two files — a declarative spec (what it means) and an executable type (how to calculate it).

**Why:** Meaning and behavior change for different reasons. A signal's definition is stable. Its calculation may be refined. Separating them allows evolution without confusion.

**Trade-off:** 960+ files for signals alone. Accepted because each file is small, focused, and independently testable.

### No predictions, no advice

**Decision:** The system describes structure. It never tells users what to do.

**Why:** Predictions create liability, erode trust when wrong, and attract the wrong audience. Structural descriptions remain true regardless of what the market does next.

**Trade-off:** Less engaging for users who want "just tell me what to buy." Accepted because the product is designed for a different kind of user.

### Batch-driven over real-time

**Decision:** All data processing happens in scheduled batches, not real-time streams.

**Why:** Batch processing is predictable, auditable, credit-efficient, and compatible with external API rate limits. The product doesn't need real-time data — structural characteristics don't change by the minute.

**Trade-off:** Data can be hours old. Accepted because the product is about structure, not price action.

### No Entity Framework migrations in development

**Decision:** Drop and recreate the database during development. Migrations only in production.

**Why:** Fast iteration. Adding a field to an entity should take seconds, not require writing migration files. The database schema is derived from the code, not maintained separately.

**Trade-off:** Development database is ephemeral. Accepted because seed data handles initial state.

### Named return types at layer boundaries

**Decision:** No tuples or anonymous types crossing layer boundaries. Always named entities, Models, or DTOs.

**Why:** Anonymous types at boundaries create shapes that belong to neither layer. They resist refactoring and make the codebase harder to navigate.

**Trade-off:** Occasionally creating a small DTO for a simple return. Accepted because the cost is minimal and the clarity is significant.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend | C#, .NET 9, ASP.NET Core |
| ORM | Entity Framework Core 9.0 (code-first) |
| Database | PostgreSQL |
| Public frontend | Next.js 16, React 19, TypeScript, Tailwind CSS |
| Admin frontend | React 18, Vite, Mantine UI, TypeScript |
| AI | Anthropic Claude API |
| Translation | DeepL API |
| Payments | Stripe |
| Email | SMTP via Brevo |
| Hosting | Hetzner (Ubuntu, nginx, systemd) |
| CI/CD | GitHub webhooks → bash deploy scripts |
| Testing | xUnit, Moq, FluentAssertions, EF InMemory |

---

## What This System Demonstrates

**System design at scale** — 104,000+ stocks, 480+ signals, 190+ endpoints, all running in production with clear structure.

**Layered architecture that holds** — The layer boundaries established early have not degraded as the system grew. New features slot into existing patterns.

**Domain modeling with integrity** — Signals and stories are not just features — they are a coherent intellectual model enforced through code structure.

**End-to-end ownership** — From database schema through API design, frontend rendering, AI content generation, deployment automation, and production monitoring. One developer, one system, fully operational.

**Pragmatic decisions over theoretical purity** — Every trade-off is explicit and accepted. The system optimizes for clarity, not cleverness.

---

*Architecture diagram available in [/cv/architecture.png](/cv/architecture.png)*
