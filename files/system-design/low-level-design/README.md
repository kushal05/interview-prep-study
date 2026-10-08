# Low-Level Design (LLD) Study Guide

> **What is Low-Level Design?**
> LLD (also called "Object-Oriented Design" or "OOD") is about designing the **internals of one service** — its classes, methods, interfaces, and how they collaborate. You decide what objects exist, what each one knows, and how they call each other.
>
> **LLD answers questions like:**
> - What classes should I create?
> - Where does this logic belong?
> - How do I make this code easy to change later?
> - Which design pattern fits this problem?
>
> **LLD is the opposite of High-Level Design (HLD)**, which is about distributed architecture (servers, databases, queues). See `../high-level-design/`.

---

## When Does Each Show Up in Interviews?

| Question type | Layer | Example |
|---------------|-------|---------|
| "Design Instagram" | HLD | Servers, DB, CDN, feed fan-out |
| "Design the Instagram **Like button class**" | LLD | Classes, state, concurrency |
| "Design Parking Lot" | LLD | Vehicle, Slot, Ticket classes; SOLID; patterns |
| "Design Uber" | Both — HLD for scale, LLD for matching algorithm |

---

## How to Use This Guide

Same study-mode pillars as HLD: spaced repetition, interleaving, Feynman technique, active recall.

### Suggested Order (Beginners)

1. `00-study-plan.md` — your 30-day LLD plan
2. `01-oop-fundamentals.md` — classes, objects, the 4 OOP pillars
3. `02-solid-principles.md` — the 5 rules of good OO design
4. `03-uml-and-class-diagrams.md` — how to draw your design
5. `04-design-patterns-overview.md` — quick reference for the 23 GoF patterns
6. `05-api-design-and-rest.md` — designing APIs at the code level
7. `06-schema-and-data-modeling.md` — designing tables and entities
8. `07-concurrency-and-threading.md` — locks, threads, race conditions
9. `08-classic-lld-problems.md` — Parking Lot, Elevator, ATM, etc.
10. `09-active-recall-cards.md` — self-test questions
11. `10-feynman-explanations.md` — explain in simple words

---

## File Map

| # | File | What You Learn |
|---|------|---------------|
| 00 | Study plan | A 30-day LLD plan |
| 01 | OOP Fundamentals | Class, object, encapsulation, inheritance, polymorphism, abstraction |
| 02 | SOLID | Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion |
| 03 | UML | Class diagrams, sequence diagrams, use case diagrams |
| 04 | Design Patterns | Creational / Structural / Behavioral patterns reference |
| 05 | API Design | RESTful endpoints, request/response design, versioning |
| 06 | Schema/Data Modeling | Entities, relationships, normalization at the code level |
| 07 | Concurrency | Threads, locks, race conditions, deadlocks |
| 08 | Classic LLD Problems | Parking Lot, Elevator, ATM, Library, Chess, Vending Machine, etc. |
| 09 | Active Recall | Self-test questions |
| 10 | Feynman | Plain-words explanations |

---

## Beginner's Master Glossary

If a term in any file confuses you, check here first.

| Term | Plain English |
|------|--------------|
| **Class** | A blueprint. Says what data an object has and what it can do. |
| **Object / instance** | One specific thing made from the class. (A class `Dog`, an object `Rex`.) |
| **Field / attribute / property** | A piece of data on an object (`age`, `name`). |
| **Method** | A function attached to an object. Acts on its data. |
| **Constructor** | A special method that runs when you create the object. Sets up the initial state. |
| **Encapsulation** | "Hide the internals." Code outside the class doesn't touch fields directly; it goes through methods. |
| **Inheritance** | One class extends another, gaining its fields and methods. (`class Dog extends Animal`) |
| **Polymorphism** | "One name, many forms." `Animal a = new Dog(); a.speak()` calls Dog's version. |
| **Abstraction** | Hiding details behind a simpler interface. (You press the brake pedal; you don't think about hydraulics.) |
| **Interface** | A contract — "any class that implements me must provide these methods." No actual code. |
| **Abstract class** | A class that is partly abstract (some methods unimplemented). Cannot be instantiated directly. |
| **Composition** | One object holds another as a field. ("A Car HAS-A Engine.") |
| **Aggregation** | A loose form of composition — the parts can exist outside the whole. |
| **Association** | The general "two objects know about each other" relationship. |
| **Coupling** | How much one class depends on another. Loose = good. |
| **Cohesion** | How focused one class is. High cohesion = good. |
| **Static** | Belongs to the class itself, not any instance. Shared by all objects. |
| **Public / private / protected** | Visibility modifiers — who can see this member. |
| **DTO (Data Transfer Object)** | A class with just fields, used to move data between layers. |
| **DAO (Data Access Object)** | A class whose job is to read/write a specific type from the database. |
| **POJO** | Plain Old Java Object — a class with no special framework annotations. |
| **Pattern** | A reusable solution to a common design problem. See file 04. |
| **Thread** | An independent line of execution inside a program. |
| **Race condition** | Bug where 2 threads touch the same data and produce the wrong answer. |
| **Lock / mutex** | A "talking stick" — only one thread holds it at a time, so others wait. |
| **Atomic** | An operation that completes fully or not at all — no in-between. |
| **Concurrency** | Multiple things in progress at once. |
| **Refactor** | Change the code's structure without changing what it does. |

---

## The LLD Interview Framework (Use Every Time)

```
Step 1: CLARIFY REQUIREMENTS (5 min)
├── What does the system DO? List features.
├── What's IN scope vs OUT of scope?
├── Single-user or multi-user? Concurrent access?
└── What are the inputs/outputs?

Step 2: IDENTIFY ENTITIES (5 min)
├── Underline nouns in the requirements — those are candidate classes
├── Decide: which deserve their own class? Which are just fields?
└── Spot relationships: HAS-A (composition), IS-A (inheritance), USES (association)

Step 3: DEFINE CLASSES & INTERFACES (10 min)
├── For each class, list its fields and methods
├── Use interfaces for varying behaviors (Strategy pattern hint)
├── Apply SOLID — especially Single Responsibility
└── Sketch a class diagram (boxes + arrows)

Step 4: APPLY PATTERNS (10 min)
├── Which design patterns naturally fit?
├── Common picks:
│   - Factory: many subtypes to create
│   - Strategy: swappable algorithms
│   - Observer: one-to-many notifications
│   - State: object behavior depends on internal state
│   - Singleton: only ONE instance allowed (use sparingly!)
└── Justify each pattern's use

Step 5: HANDLE EDGE CASES & CONCURRENCY (10 min)
├── What if 2 threads do this at once?
├── What if input is invalid?
├── What if a dependency fails?
└── Where do locks go? Are they fine-grained or coarse-grained?

Step 6: WALK THROUGH A SCENARIO (5 min)
└── "Let's trace: a user parks a car. Here's the call sequence..."
```

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

Both layers are tested separately in interviews. A "design Uber" question often expects HLD; a "design parking lot" question always expects LLD.
