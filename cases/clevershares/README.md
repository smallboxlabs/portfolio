# Case Study — CleverShares

A teens-first, family-supported platform that turns everyday products into teachable investing stories. This case focuses on the architecture choices and what they enabled.

## Context
We needed two faces on the same truth: an admin side for content and data ops, and a public side for speed and SEO. We also needed story generation to feel “alive” without coupling it to the UI.

## Goals
- Keep data flow predictable and testable.
- Let “stories” evolve independently of the app shells.
- Make ingestion cheap to iterate and safe to rerun.
- Produce explainable scores, not black boxes.

## Architecture in short
Domain and Services at the core. Thin DTO façades to cross boundaries. Separate ingestion job(s) feed a normalized store. StoryEngine reads only from contracts (interfaces), not storage implementation. Admin is React; Public is Next.js. Everything shares types at the edges, not internals.

## Data flow
Vendors → Ingestion Jobs → Normalized DB → Read Models → StoryEngine → Story outputs (scores, badges, copy) → API → UI.

Re-ingesting is idempotent. Hashing on interfaces prevents noisy rewrites. Indicators (SMA/EMA/Bollinger/etc.) are computed once and cached with provenance.

## StoryEngine shape
SpecFactory → StoryType → StoryRegistry.  
A story receives only what it declares. Required props are explicit. Each story returns a score plus rationale. Bands and example ranges live with the story, which keeps tests honest.

## Key decisions (and why)
- **Interfaces over concretes** to let data sources swap without touching the stories.
- **Indicators set at the price-history layer** so stories stay declarative.
- **Certainty scoring** to separate “signal exists” from “we’re sure.”
- **Ranges-as-contract** so tests can pin expected behavior rather than magic numbers drifting.

## Testing approach
Golden examples validate stories against declared bands. An aggregate test asserts all stories stay within their ranges. When examples fail, you either fix the example or refine the algorithm—on purpose, not by accident.

## What worked
- Fast iteration on stories without breaking the app shells.
- Reproducible ingestion thanks to hashing and idempotency.
- Clear seams: UI can ship while StoryEngine evolves.

## What I’d change next
- Promote ranges and certainty into a shared “scoring” lib so every story inherits the same ergonomics.
- Add a small “explain” payload (inputs, checks, final reasoning) for better debugging and UI tooltips.
- Split heavy indicators (ATR/Bollinger) behind a compute queue for large universes.

## Lessons
Small, explicit contracts beat clever plumbing. Keep the story authoring surface friendly, and you get more useful stories. Make the data path boring, and everything else can be interesting.

---

**See also:**  
- Data pipeline notes → `../data-pipeline.md` (placeholder)  
- StoryEngine overview → `../storyengine-overview.md` (placeholder)
