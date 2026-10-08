# System Design Study Guide

> **Goal:** Become interview-ready in system design — both the *big picture* (HLD) and the *code-level details* (LLD).
> **Approach:** Spaced repetition + interleaving + Feynman technique + active recall.

---

## What is System Design?

System design is the practice of deciding **how** software is built. It has two layers:

| Layer | What it answers | Sample question |
|-------|----------------|----------------|
| **High-Level Design (HLD)** | How do *services* fit together? | "Design Instagram." |
| **Low-Level Design (LLD)** | How do *classes* inside one service fit together? | "Design the Parking Lot class." |

Most modern interviews test **both**, sometimes in the same round. This guide treats them as separate tracks so you can focus on one at a time.

---

## Folder Structure

```
system-design/
├── README.md                       ← you are here
│
├── high-level-design/              ← Architecture, scalability, distributed systems
│   ├── README.md                   ← HLD entry point + master glossary
│   ├── 00-90-day-blueprint.md      ← Day-by-day study plan
│   ├── 01-fundamentals.md          ← Scalability, latency, CAP, availability
│   ├── 02-networking-and-protocols.md
│   ├── 03-databases.md
│   ├── 04-caching.md
│   ├── 05-message-queues-and-streaming.md
│   ├── 06-load-balancing-and-proxies.md
│   ├── 07-storage-and-cdn.md
│   ├── 08-microservices-and-architecture-patterns.md
│   ├── 09-consistency-and-consensus.md
│   ├── 10-real-world-system-designs.md   ← URL shortener, Twitter, Uber, etc.
│   ├── 11-active-recall-cards.md
│   ├── 12-feynman-explanations.md
│   └── 13-resources.md
│
└── low-level-design/               ← OOP, SOLID, patterns, classic LLD problems
    ├── README.md                   ← LLD entry point + master glossary
    ├── 00-study-plan.md            ← 30-day LLD plan
    ├── 01-oop-fundamentals.md      ← Classes, objects, 4 pillars
    ├── 02-solid-principles.md
    ├── 03-uml-and-class-diagrams.md
    ├── 04-design-patterns-overview.md
    ├── 05-api-design-and-rest.md
    ├── 06-schema-and-data-modeling.md
    ├── 07-concurrency-and-threading.md
    ├── 08-classic-lld-problems.md  ← Parking Lot, Elevator, ATM, Chess, etc.
    ├── 09-active-recall-cards.md
    └── 10-feynman-explanations.md
```

> Cross-reference: detailed code for each design pattern lives in `../design-patterns/common/`.

---

## Where to Start

### If you have **0 weeks** of system design experience
1. Read `high-level-design/README.md` (master glossary).
2. Read `low-level-design/README.md` (master glossary).
3. Pick ONE track based on your upcoming interview. Most companies test both eventually.
4. Open the 90-day blueprint (HLD) or 30-day study plan (LLD) and start day 1.

### If you have **a specific interview soon**
- "Design a service that scales to millions" → HLD track.
- "Design Parking Lot / Elevator / ATM" → LLD track.
- "Design Uber" (broad question) → both tracks.

### If you're refreshing
- Use `*-active-recall-cards.md` files for self-quiz.
- Practice real problems in `high-level-design/10-real-world-system-designs.md` and `low-level-design/08-classic-lld-problems.md`.

---

## The 4 Study-Mode Pillars (Applied Throughout)

1. **Spaced Repetition** — Re-read each topic on days 1, 3, 7, 14, 30.
2. **Interleaving** — Mix two topics per session. Never one topic for 90+ minutes.
3. **Feynman Technique** — Write a 12-year-old explanation of each concept. If you can't simplify it, you don't understand it.
4. **Active Recall** — Close the book. Write what you remember. Compare. Repeat.

---

## How This Guide Differs from Most Resources

- **Beginner-first.** Every file opens with a "Before You Begin" plain-English glossary. No assumed vocabulary.
- **Self-contained.** You don't need other books or videos to follow the explanations (those are nice-to-haves in `high-level-design/13-resources.md`).
- **Study-mode, not reference-mode.** Each file has active recall questions at the end. Read it like a textbook, not Wikipedia.
- **Hand-drawn ASCII diagrams.** No image files to load. Everything reads in any text editor.
- **Both layers covered.** Most online guides cover only HLD. LLD has its own complete track here.

---

## Quick Cheat: HLD vs LLD

| Concern | HLD | LLD |
|---------|-----|-----|
| Scope | Whole system | Inside ONE service |
| Output | Architecture diagram | Class diagram + sequence diagram |
| Vocabulary | Servers, DB, cache, queue | Classes, methods, interfaces |
| Tools | Boxes & arrows | UML |
| Scaling | Sharding, replication | Concurrency, locks |
| Trade-offs | Consistency vs availability | Flexibility vs simplicity |
| Failure mode | Server dies | Exception thrown |
| Patterns | CQRS, Saga, Pub/Sub | Factory, Strategy, Observer |
| Question style | "Design Uber" | "Design the Ride Matching class" |

---

## Progress Tracker

### HLD Track (90 days)
- [ ] Weeks 1–4: Foundations (fundamentals, networking, DB, caching, queues, LBs, storage, microservices, consensus)
- [ ] Weeks 5–8: Real-world systems (Twitter, YouTube, Uber, etc.)
- [ ] Weeks 9–13: Mock interviews + final polish

### LLD Track (30 days)
- [ ] Week 1: OOP + SOLID + UML
- [ ] Week 2: Design patterns (Creational, Structural, Behavioral)
- [ ] Week 3: APIs, Schemas, Concurrency
- [ ] Week 4: Classic problems + mocks

---

Good luck. The fastest way to be good at system design is to design lots of systems — start now, even if your first design is messy.
