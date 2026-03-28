---
name: java-development-best-practices
description: "Apply SOLID principles, Clean Code (KISS, DRY, YAGNI), GoF design patterns, and microservices best practices to Java code. Use this skill whenever the user asks to review Java code quality, refactor a class or method, apply a design pattern, check for SOLID/KISS/DRY/YAGNI violations, simplify over-engineered code, remove duplication, improve code structure, or asks about Java best practices. Also use when the user is writing new Java code in a hexagonal architecture project and wants guidance on where things should go or how to structure them properly."
---

# Java Development Best Practices

Apply proven software engineering principles when reviewing, refactoring, or generating Java code — especially in the context of hexagonal architecture projects.

## How to Use This Skill

This skill supports five actions. Determine which one the user needs from their request:

| Action | Trigger phrases | What to do |
|--------|----------------|------------|
| **review** | "review", "check", "is this good", "code quality" | Analyze code against the review checklist below |
| **refactor** | "refactor", "improve", "clean up", "simplify" | Identify issues, propose changes, apply them |
| **suggest** | "how should I", "what pattern", "best way to" | Recommend patterns/approaches with rationale |
| **explain** | "explain", "why", "what does this pattern do" | Teach the principle using their actual code as context |
| **generate** | "create", "write", "implement", "add" | Write new code following all principles below |

If the action isn't clear, default to **review** for existing code or **generate** for new code.

---

## Action: Review

When reviewing code, work through this checklist systematically. Report only actual issues found — don't pad the review with things that are fine.

### Step 1: Architecture Compliance

For hexagonal architecture projects, verify dependency direction is correct:

```
domain/model       → depends on NOTHING (pure Java + Reactor)
application/use-cases → depends on domain/model ONLY
driven-adapters/*   → depends on domain/model (implements ports)
entry-points/app    → depends on use-cases + adapters (wires DI)
```

Check for violations:
- Domain classes importing Spring/persistence annotations (should be pure)
- Use cases importing adapter implementations (should use port interfaces)
- Controllers accessing domain ports directly (should go through use cases)

### Step 2: SOLID Violations

Scan for the most common violations in priority order:

1. **Single Responsibility**: Does the class do more than one thing? A sign: you need "and" to describe it. Services that both validate AND persist AND notify are doing too much.
2. **Dependency Inversion**: Is the code depending on concrete classes instead of interfaces? Especially in use cases — they should inject ports, not adapters.
3. **Interface Segregation**: Are there large interfaces forcing implementors to provide methods they don't need? A repository interface with 15 methods when most callers use 2 is a smell.
4. **Open/Closed**: Would adding a new variant (payment type, notification channel) require modifying existing code? If yes, suggest Strategy or similar.
5. **Liskov Substitution**: Do subclasses override methods with surprising behavior or throw unexpected exceptions?

### Step 3: Clean Code Principles (KISS, DRY, YAGNI)

These three principles catch the most common sources of unnecessary complexity:

**KISS (Keep It Simple, Stupid)** — The simplest solution that works is usually the best. Flag when:
- A simple `if/else` was replaced with a full Strategy/Factory pattern for only 2 cases that aren't likely to grow
- An abstraction layer exists but has only one implementation and no foreseeable second one
- A reactive chain uses complex operators (`flatMapMany`, `zipWith`, `switchIfEmpty` chaining) when a simpler sequence would work
- Utility classes or helpers were created for logic used only once
- Inheritance hierarchies exist where composition (or just a plain method) would suffice

**DRY (Don't Repeat Yourself)** — Every piece of knowledge should have a single source of truth. Flag when:
- The same validation logic is duplicated in controller AND use case (pick one — validate at the boundary)
- Identical mapping code appears in multiple adapters (extract a shared mapper)
- Configuration values are hardcoded in multiple places instead of using `@ConfigurationProperties`
- The same query logic is copy-pasted across repository methods

**YAGNI (You Aren't Gonna Need It)** — Don't build for hypothetical future requirements. Flag when:
- Interfaces exist with only one implementation and no clear reason for the abstraction (exception: hexagonal ports, which are intentional abstractions)
- Methods or parameters are added "just in case" but nothing calls them
- Generic/parameterized solutions were built when a concrete one covers the actual need
- Feature flags, configuration switches, or extension points exist for features nobody has asked for

### Step 4: Code Readability

Focus on things that actually hurt readability (not cosmetic preferences):
- Methods over 30 lines that should be extracted
- Unclear naming (single-letter variables, generic names like `data`, `result`, `temp`)
- Boolean parameters that make call sites unreadable (`process(true, false, true)`)
- Deep nesting (3+ levels) that could be flattened with early returns
- Swallowed exceptions (empty catch blocks)
- `.block()` calls in reactive code (breaks non-blocking contract)

### Step 5: Output Format

Present findings grouped by severity:

```markdown
## Code Review: <FileName>

### Issues Found

**Architecture** (if any violations)
- [file:line] Description of the violation and why it matters

**SOLID** (if any violations)
- [file:line] Which principle is violated and a concrete fix

**KISS/DRY/YAGNI** (if any violations)
- [file:line] Which principle is violated, what's the simpler/cleaner alternative

**Code Readability** (if any issues)
- [file:line] What's wrong and how to improve it

### Strengths
- What the code does well (always include at least one)
```

---

## Action: Refactor

### Step 1: Diagnose First

Read the code the user wants refactored. Identify what's actually wrong — don't refactor for the sake of it. Common refactoring drivers:
- A class grew too large (split by responsibility — **SRP**)
- Duplicated logic across classes (extract shared behavior — **DRY**)
- Hard to test (usually a DI problem — **Dependency Inversion**)
- Hard to extend (missing abstraction — **Open/Closed**)
- Mixing concerns (business logic in controller, persistence logic in domain)
- Over-engineered solution for a simple problem (simplify — **KISS**)
- Code built for requirements that don't exist yet (remove — **YAGNI**)

### Step 2: Propose Before Changing

Briefly explain what you'd change and why. For example:
> "The `OrderService` is handling validation, persistence, and notification — three responsibilities. I'd extract a `OrderValidator` and use an event/port for notifications, keeping the service focused on orchestration."

### Step 3: Apply Changes

When making changes, follow these rules:
- Preserve existing behavior (refactoring should not change what the code does)
- Move code to the correct hexagonal layer if it's in the wrong place
- Introduce interfaces (ports) when breaking dependencies between layers
- Use `@RequiredArgsConstructor` + `final` fields for constructor injection (Lombok is available)
- Keep reactive chains intact — don't introduce blocking calls

---

## Action: Suggest

When the user asks for advice on how to approach something:

1. **Understand the context**: What are they building? Which layer of the architecture does it belong in?
2. **Recommend a pattern**: Pick from GoF or microservice patterns (read `references/patterns-reference.md` for the full catalog if needed)
3. **Show it in their code**: Don't just name the pattern — sketch how it would look in their specific codebase with their actual class names

Prioritize simplicity. A straightforward if/else is better than a Strategy pattern when there are only two cases that won't grow. Patterns should solve a problem the user actually has, not a theoretical future one.

---

## Action: Explain

When explaining a principle or pattern:

1. Start with the problem it solves (not the definition)
2. Show a "before" example using their actual code if available
3. Show the "after" with the pattern applied
4. Explain when NOT to use it — every pattern has a cost

---

## Action: Generate

When writing new code, apply these principles automatically:

### Architecture Placement

| What you're creating | Where it goes | Package |
|---------------------|---------------|---------|
| Entity, Value Object | `domain/model` | `com.<safename>.model` |
| Port interface (outbound) | `domain/model` | `com.<safename>.model` |
| Use case interface | `application/use-cases` | `com.<safename>.usecases` |
| Use case implementation | `application/use-cases` | `com.<safename>.usecases` |
| Repository adapter | `driven-adapters/<db>` | `com.<safename>.<db>` |
| REST controller | `entry-points/app` | `com.<safename>.app` |
| Config class | `entry-points/app` | `com.<safename>.app` |
| DTO (request/response) | `entry-points/app` | `com.<safename>.app` |

### Clean Code Principles

Apply KISS, DRY, and YAGNI as a filter on every decision when generating code:
- **KISS**: choose the simplest approach that solves the actual problem. Don't introduce a pattern unless it earns its complexity (e.g., Strategy for 2 cases is overkill — use `if/else`).
- **DRY**: if you're about to write the same logic in two places, extract it. Shared mappers, common validation, reusable query methods.
- **YAGNI**: only build what's been asked for. No extra endpoints "just in case", no generic frameworks for one use case, no unused parameters.

### Code Style

- **Naming**: classes `PascalCase`, methods `camelCase` (verb+noun), constants `UPPER_SNAKE_CASE`, booleans prefixed with `is`/`has`/`can`/`should`
- **Injection**: constructor injection via `@RequiredArgsConstructor` + `private final` fields
- **Reactive**: `Mono` for 0-1, `Flux` for 0-N, never `.block()` outside tests
- **DTOs**: use separate request/response DTOs at entry points — never expose domain entities directly to REST
- **Exceptions**: create domain-specific exceptions (`UserNotFoundException extends RuntimeException`), handle globally with `@RestControllerAdvice`
- **Config**: prefer `@ConfigurationProperties` records over scattered `@Value` annotations
- **Tests**: name as `should<Expected>When<Condition>()`, follow Arrange-Act-Assert, mock only outbound ports

### POM Dependencies

When generating code that uses classes from another module, add the dependency to the module's `pom.xml`:
```xml
<dependency>
    <groupId>com.<safename>.<module></groupId>
    <artifactId><artifact-id></artifactId>
    <version>${project.version}</version>
</dependency>
```

---

## Reference Material

For detailed pattern catalogs (GoF creational/structural/behavioral, microservice patterns, exception handling structures), read `references/patterns-reference.md`. Only load this when you need to look up a specific pattern or provide an in-depth explanation — the guidelines above cover the common cases.
