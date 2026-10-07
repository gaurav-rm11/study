# ⚠️ Criticism and Cautions: When Patterns Go Wrong

This chapter exists because a study folder that only says "patterns are great" would be lying to you.

Design patterns are genuinely useful. They are also, in 2026, one of the most reliably over-applied
ideas in software engineering. Both of those statements are true at once, and the skill worth having
is not "know 23 patterns" — it is **knowing which of the 23 your problem actually has, and whether
your language already solved it for you.**

Read this chapter *before* you read the pattern chapters, or right after your first two. It will
change how you read the rest.

---

## 🗺️ What's in here

1. [The four classic criticisms](#the-four-classic-criticisms)
2. [Kludges for a weak language](#-criticism-1-kludges-for-a-weak-programming-language)
3. [Inefficient solutions and by-the-letter implementation](#-criticism-2-inefficient-solutions--implementing-to-the-letter)
4. [Unjustified use and the pattern-happy phase](#-criticism-3-unjustified-use-and-the-pattern-happy-phase)
5. [Cargo-culting](#-criticism-4-cargo-culting)
6. [The replacement table: pattern → modern language feature](#-the-replacement-table)
7. [The Rule of Three](#-the-rule-of-three)
8. [Genuine need vs speculative generality](#-genuine-need-vs-speculative-generality)
9. [How to push back in code review, kindly](#-how-to-push-back-in-a-code-review-kindly)
10. [What survives all of this](#-what-survives-all-of-this)

---

## The four classic criticisms

The standard critique of design patterns has four heads. In plain English:

```
  +---------------------------------------------------------------+
  |  1. KLUDGES FOR A WEAK LANGUAGE                                |
  |     "You only need Strategy because your language              |
  |      can't pass a function around."                            |
  +---------------------------------------------------------------+
  |  2. INEFFICIENT SOLUTIONS                                      |
  |     "You copied the book diagram instead of solving            |
  |      your actual problem."                                     |
  +---------------------------------------------------------------+
  |  3. UNJUSTIFIED USE                                            |
  |     "If all you have is a hammer, everything looks             |
  |      like a nail."                                             |
  +---------------------------------------------------------------+
  |  4. CARGO-CULTING                                              |
  |     "You did the ritual. You didn't get the plane."            |
  +---------------------------------------------------------------+
```

Let's take each one seriously.

---

## 🔧 Criticism 1: Kludges for a weak programming language

This is the oldest and sharpest criticism. The argument, made most famously by Paul Graham in his
essay *Revenge of the Nerds*, is roughly:

> When you see repeated boilerplate in your programs, that's a sign your language is missing an
> abstraction. Patterns are the shape of the hole.

The claim is that many of the Gang of Four patterns are not eternal truths about software design.
They are **workarounds for things C++ and Java in the early 1990s could not express directly** —
and that a language with first-class functions, closures, generators, or metaprogramming simply
doesn't need them.

This criticism is largely correct. Let's prove it with your own languages.

### Strategy is a function

Here is Strategy, done the textbook way, in TypeScript. Imagine pricing a used-car listing: different
pricing rules for dealer listings, private sellers, and certified pre-owned cars.

```typescript
// ---------- The "classic" Strategy pattern ----------

interface PricingStrategy {
  calculate(basePrice: number, car: Car): number;
}

class DealerPricing implements PricingStrategy {
  calculate(basePrice: number, car: Car): number {
    return basePrice * 1.08 + 4999; // dealer margin + documentation fee
  }
}

class PrivateSellerPricing implements PricingStrategy {
  calculate(basePrice: number, car: Car): number {
    return basePrice;
  }
}

class CertifiedPreOwnedPricing implements PricingStrategy {
  calculate(basePrice: number, car: Car): number {
    return basePrice * 1.12 + 15000; // warranty + inspection
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  priceFor(basePrice: number, car: Car): number {
    return this.strategy.calculate(basePrice, car);
  }
}

// usage
const calc = new PriceCalculator(new DealerPricing());
const price = calc.priceFor(450000, car);
```

That's four classes, one interface, and an indirection layer. Here is the same behaviour:

```typescript
// ---------- The same thing, using the language ----------

type PricingStrategy = (basePrice: number, car: Car) => number;

const dealerPricing: PricingStrategy = (base) => base * 1.08 + 4999;
const privateSellerPricing: PricingStrategy = (base) => base;
const certifiedPreOwnedPricing: PricingStrategy = (base) => base * 1.12 + 15000;

function priceFor(basePrice: number, car: Car, strategy: PricingStrategy): number {
  return strategy(basePrice, car);
}

// usage
const price = priceFor(450000, car, dealerPricing);
```

Six lines. Same substitutability, same open/closed property (add a new function, change nothing else),
same testability — arguably *better* testability, because there is nothing to mock. You just pass
`() => 100`.

The C# version is equally short:

```csharp
// The whole Strategy pattern, in C#
public delegate decimal PricingStrategy(decimal basePrice, Car car);

public static class Pricing
{
    public static readonly PricingStrategy Dealer =
        (basePrice, _) => basePrice * 1.08m + 4999m;

    public static readonly PricingStrategy PrivateSeller =
        (basePrice, _) => basePrice;

    public static readonly PricingStrategy CertifiedPreOwned =
        (basePrice, _) => basePrice * 1.12m + 15000m;
}

public static decimal PriceFor(decimal basePrice, Car car, PricingStrategy strategy)
    => strategy(basePrice, car);
```

Or with no custom delegate at all: `Func<decimal, Car, decimal>`.

**When the class-based Strategy still earns its keep:**

- The strategy has **state** that persists between calls (a connection, a cache, a counter).
- The strategy needs **more than one method** — `Calculate` *and* `Explain` *and* `Validate`. A
  function can't carry three operations; an interface can.
- The strategy is **resolved by a DI container** and needs constructor-injected dependencies
  (a repository, an `ILogger`, an `IHttpClientFactory`).
- The strategy set is **discovered at runtime** — plugins, assembly scanning, config-driven registration.
- You need **metadata**: a `Name`, a `Priority`, an `AppliesTo(Car)` predicate for selection.

That last list is not small. In a C# backend with dependency injection, the interface version is often
genuinely the right call. The point is that you should be able to say *why*, and "because Strategy is a
pattern" is not a why.

→ See [Strategy](../03-behavioral/08-strategy.md) for the full treatment.

### Iterator is built into the language

The Iterator pattern — an object that walks a collection without exposing its internals — was a real
design problem in 1994. Today it's a keyword.

```csharp
// You are not implementing IEnumerator by hand. You are writing this:
public IEnumerable<Listing> ActiveListings(int dealerId)
{
    foreach (var listing in _allListings)
    {
        if (listing.DealerId == dealerId && listing.Status == ListingStatus.Active)
            yield return listing;
    }
}
```

`yield return` *is* the Iterator pattern. The compiler generates a state machine class implementing
`IEnumerator<T>` for you — literally the pattern, written by a tool instead of a human. And for
streaming from RabbitMQ or a paged SQL query, `IAsyncEnumerable<T>` gives you the same thing
asynchronously:

```csharp
public async IAsyncEnumerable<Listing> StreamListingsAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var page = 0;
    while (true)
    {
        var batch = await _repo.GetPageAsync(page++, size: 500, ct);
        if (batch.Count == 0) yield break;

        foreach (var listing in batch)
            yield return listing;
    }
}

// consumer
await foreach (var listing in StreamListingsAsync(ct))
{
    await _publisher.PublishAsync(listing, ct);
}
```

TypeScript has the same thing:

```typescript
function* activeListings(all: Listing[], dealerId: number): Generator<Listing> {
  for (const listing of all) {
    if (listing.dealerId === dealerId && listing.status === "active") {
      yield listing;
    }
  }
}

async function* streamListings(repo: ListingRepo): AsyncGenerator<Listing> {
  let page = 0;
  while (true) {
    const batch = await repo.getPage(page++, 500);
    if (batch.length === 0) return;
    yield* batch;
  }
}

for await (const listing of streamListings(repo)) {
  await publisher.publish(listing);
}
```

If you ever find yourself hand-writing a class with `HasNext()` and `Next()` in C# or TypeScript,
stop. You are re-implementing a language feature.

→ [Iterator](../03-behavioral/03-iterator.md) is still worth reading, because understanding *what*
`yield return` compiles into makes you better at using it. Just don't hand-roll it.

### Command is a lambda (until it isn't)

```typescript
// Classic Command
interface Command {
  execute(): void;
}

class PublishListingCommand implements Command {
  constructor(private readonly listingId: number, private readonly svc: ListingService) {}
  execute(): void {
    this.svc.publish(this.listingId);
  }
}

const queue: Command[] = [new PublishListingCommand(42, svc)];
queue.forEach((c) => c.execute());
```

```typescript
// The same thing
const queue: Array<() => void> = [() => svc.publish(42)];
queue.forEach((run) => run());
```

The closure captures `42` and `svc` exactly as the constructor did. Same deferred execution, same
queuing, same composition.

**But** — and this is where the criticism stops being a slam dunk — Command frequently needs more than
`execute()`:

- **Undo.** A lambda can't tell you how to reverse itself. `interface Command { execute(): void; undo(): void; }`
  is a real interface that no function type replaces.
- **Serialization.** If the command has to survive a trip through RabbitMQ, it must be data, not a
  closure. You cannot serialize a captured scope. You serialize `{ type: "PublishListing", listingId: 42 }`
  and dispatch on it — which is Command-as-a-record, and genuinely useful.
- **Inspection.** Logging "which command is running", deduplicating, prioritizing a queue — all need
  the command to be an inspectable value.

C# lets you have both at once with records:

```csharp
// Command as data — serializable, loggable, queueable
public abstract record Command;

public sealed record PublishListing(int ListingId, string ActorEmail) : Command;
public sealed record UnpublishListing(int ListingId, string Reason) : Command;
public sealed record RepriceListing(int ListingId, decimal NewPrice) : Command;

// Dispatch with pattern matching — no visitor, no double dispatch
public async Task HandleAsync(Command command, CancellationToken ct) => command switch
{
    PublishListing (var id, var actor)   => await _listings.PublishAsync(id, actor, ct),
    UnpublishListing (var id, var reason) => await _listings.UnpublishAsync(id, reason, ct),
    RepriceListing (var id, var price)   => await _listings.RepriceAsync(id, price, ct),
    _ => throw new ArgumentOutOfRangeException(nameof(command), command, "Unknown command")
};
```

That is the Command pattern, and it is also just records plus a `switch`. The pattern didn't disappear —
the *ceremony* did.

→ [Command](../03-behavioral/02-command.md).

### The honest conclusion for criticism 1

The criticism is right about the mechanism and wrong about the value.

**Right:** a large fraction of GoF pattern *code* is language boilerplate. If your language has
first-class functions, you should not be writing one-method interfaces to simulate them.

**Wrong:** the *vocabulary* survives. Saying "the pricing rule is a Strategy" tells a reviewer the
shape of the thing in three words, whether you implemented it as a class or a `Func<>`. The pattern is
the idea; the class hierarchy was only ever one encoding of it.

---

## 🐌 Criticism 2: Inefficient solutions / implementing to the letter

Patterns systematise approaches people were already using. The danger is that a systematised thing
starts to look like a **specification**, and people implement it to the letter instead of adapting it
to their project.

Symptoms:

**Copying the UML diagram instead of solving the problem.** The book shows `AbstractFactory` with two
product families and two concrete factories. Your app has one product family and will always have one.
You build the two-factory structure anyway, with a second factory that is a copy-paste of the first with
a different name.

**Ignoring cost.** Some patterns have real runtime cost, and people apply them without measuring:

```csharp
// Decorator chain built without thinking about the cost
IPriceSource source =
    new LoggingPriceSource(
        new CachingPriceSource(
            new RetryingPriceSource(
                new MetricsPriceSource(
                    new TimingPriceSource(
                        new HttpPriceSource(httpClient))))));
```

Six virtual calls and six allocations per price lookup. On a listing page rendering 40 cars, that is
240 indirections before you do any work. Usually irrelevant. Occasionally, in a hot loop with millions
of iterations, it is the whole problem. The mistake isn't the Decorator — it's applying it without ever
asking what it costs.

The mirror-image mistake is worse: **Flyweight applied where there is no memory pressure.** Flyweight
exists to share immutable state across a very large number of objects. If you have 300 objects, a
Flyweight adds a lookup table, an indirection, and a mutable/immutable split to your model, and saves
you roughly nothing. → [Flyweight](../02-structural/06-flyweight.md) says so itself.

**Singleton as a reflex.** Singleton is the pattern most likely to be implemented "to the letter" and
most likely to be wrong for it. In a C# service with a DI container, you do not write a `Singleton`
class with a static `Instance` property and a lock. You write:

```csharp
// This is your singleton. The container owns the lifetime.
services.AddSingleton<IPriceCache, PriceCache>();
```

One registered lifetime, still injectable, still mockable in tests, no global static, no
double-checked-locking ceremony. The classic implementation buys you a global variable and a testing
problem. → [Singleton](../01-creational/05-singleton.md) covers this in detail; read its cautions
section carefully.

**The fix for criticism 2** is a habit, not a rule: implement the *smallest version of the pattern that
solves today's problem*, and let it grow. A Strategy with two implementations doesn't need a registry,
a factory, and a config-driven resolver. It needs two functions and an `if`.

---

## 🔨 Criticism 3: Unjustified use and the pattern-happy phase

> If all you have is a hammer, everything looks like a nail.

Almost every engineer who learns patterns goes through a phase. It looks like this:

**Week 1.** You read about Factory Method. It clicks. It is genuinely elegant.

**Week 2.** Every `new` in your codebase becomes a factory. `UserFactory`, `LoggerFactory`,
`StringFactory`. You write `ConnectionStringFactoryProvider`.

**Week 3.** A code review comment asks why `getUser()` now goes through four files. You explain
decoupling. The reviewer, who has been here, sighs kindly.

**Week 6.** You delete most of it. You keep two of them, and those two are good.

That progression is *normal and healthy*. You cannot learn where patterns belong without over-applying
them first. The goal is to shorten week 2 to week 3, not to skip it.

Here's the concrete version. This is real code shaped like code I've seen a dozen times:

```typescript
// ------------------------------------------------------------------
// The over-patterned version: formatting a car's display name
// ------------------------------------------------------------------

interface INameFormattingStrategy {
  format(car: Car): string;
}

class StandardNameFormattingStrategy implements INameFormattingStrategy {
  format(car: Car): string {
    return `${car.year} ${car.make} ${car.model}`;
  }
}

abstract class NameFormatterFactory {
  abstract createFormatter(): INameFormattingStrategy;
}

class StandardNameFormatterFactory extends NameFormatterFactory {
  createFormatter(): INameFormattingStrategy {
    return new StandardNameFormattingStrategy();
  }
}

class CarNameService {
  private readonly formatter: INameFormattingStrategy;
  constructor(factory: NameFormatterFactory) {
    this.formatter = factory.createFormatter();
  }
  getDisplayName(car: Car): string {
    return this.formatter.format(car);
  }
}

// 5 types, 2 files, 1 DI registration, to produce:
const name = new CarNameService(new StandardNameFormatterFactory()).getDisplayName(car);
```

```typescript
// ------------------------------------------------------------------
// The version that should have been written
// ------------------------------------------------------------------

export function displayName(car: Car): string {
  return `${car.year} ${car.make} ${car.model}`;
}
```

The second version is not "less engineered." It is **more** engineered, because engineering includes
knowing what not to build. If a second format shows up next quarter, adding a parameter takes thirty
seconds:

```typescript
export function displayName(car: Car, style: "full" | "short" = "full"): string {
  return style === "short"
    ? `${car.make} ${car.model}`
    : `${car.year} ${car.make} ${car.model}`;
}
```

And if a *fifth* format shows up, and they start varying by locale and channel, *then* you extract a
Strategy — and you will extract the right one, because by then you'll have five real examples telling
you what varies.

That's the Rule of Three, which gets its own section below.

---

## 🛩️ Criticism 4: Cargo-culting

Cargo-culting is doing the ritual without understanding the mechanism. The term comes from observing
people build the *shape* of a thing (a runway, a control tower made of bamboo) in the hope that the
thing itself will follow.

In pattern terms, cargo-culting is:

- Naming a class `SomethingFactory` because factories are good, when it has one method that returns
  `new Something()` and nothing is ever substituted.
- Adding an interface for every class ("`IFoo` for `Foo`") because "interfaces are good for testing",
  when nothing else will ever implement it and your mocking library can mock concrete classes anyway.
- Using a Repository interface over an ORM that is already a repository, adding a second abstraction
  layer over the first abstraction layer.
- Building an Observer/event-bus system for two subscribers that are both in the same file.
- Introducing a Mediator because "direct dependencies are coupling", ending up with a mediator that
  knows about all 14 components — which is *more* coupling, concentrated in one class.

The tell for cargo-culting is that the person can name the pattern but cannot name **what would change**
if the pattern were removed. That's the diagnostic question, and it's the one to ask yourself first:

> If I deleted this abstraction and inlined it, what specifically would get worse?

If the answer is a concrete scenario ("we'd have to redeploy the pricing service to change the dealer
margin"), keep it. If the answer is a principle ("it would be less SOLID"), you're cargo-culting.

→ [SOLID and other principles](../04-extras/01-solid-and-principles.md) — principles are a lens, not
a checklist, and the same warning applies there.

---

## 📋 The Replacement Table

This is the practical payoff of criticism 1: **which patterns has your language already absorbed?**

### C#

| Pattern | Modern C# feature that often replaces it | Notes |
|---|---|---|
| [Strategy](../03-behavioral/08-strategy.md) | `Func<T, TResult>`, custom `delegate` | Keep the interface when the strategy needs DI, state, or multiple methods. |
| [Command](../03-behavioral/02-command.md) | `Action`/`Func` for in-process; `record` + `switch` for queued/serialized | Records win when the command crosses a RabbitMQ boundary or needs logging. |
| [Iterator](../03-behavioral/03-iterator.md) | `IEnumerable<T>` + `yield return` | Fully absorbed. Never hand-roll `IEnumerator`. |
| Iterator (async / streaming) | `IAsyncEnumerable<T>` + `await foreach` | The right tool for paged SQL and message streams. |
| [Observer](../03-behavioral/06-observer.md) | `event` / `EventHandler<T>`; `IObservable<T>` for streams | `event` is Observer with compiler support. Use a message bus for cross-process. |
| [Singleton](../01-creational/05-singleton.md) | `services.AddSingleton<T>()` | The container manages lifetime. The classic static version is almost always wrong here. |
| [Abstract Factory](../01-creational/02-abstract-factory.md) / [Factory Method](../01-creational/01-factory-method.md) | DI container registration, keyed services, `Func<TKey, TService>` factory delegates | The container *is* an abstract factory. |
| [Builder](../01-creational/03-builder.md) | Object initializers, optional/named parameters, `record` + `with` | `car with { Price = 450000 }` is a builder for immutable types. Keep Builder for genuinely complex multi-step construction. |
| [Prototype](../01-creational/04-prototype.md) | `record` copy semantics: `with` expressions | Non-destructive mutation is prototype-style cloning, built in. |
| [Visitor](../03-behavioral/10-visitor.md) | Pattern matching on a closed hierarchy, `switch` expressions with type patterns | Visitor exists to fake double dispatch. Pattern matching does it directly and readably. |
| [State](../03-behavioral/07-state.md) | `enum` + `switch` expression for simple machines; classes only when states carry behaviour and data | Don't build 8 state classes for a 4-value status column. |
| [Template Method](../03-behavioral/09-template-method.md) | A method that takes a delegate for each varying step | Inheritance-free, composable, easier to test. |
| [Decorator](../02-structural/04-decorator.md) | Still genuinely useful. Middleware pipelines (`IPipelineBehavior`, ASP.NET middleware) are Decorator. | Keep it — but watch the chain depth. |
| [Adapter](../02-structural/01-adapter.md) | Extension methods, for thin shape-shifting adapters | Full Adapter class when the mismatch is real; extension method when it's cosmetic. |
| [Proxy](../02-structural/07-proxy.md) | `DispatchProxy`, source generators, interceptors | Source generators produce the proxy at compile time — no reflection cost. |
| [Flyweight](../02-structural/06-flyweight.md) | String interning, `ReadOnlyMemory<T>`, `ArrayPool<T>` | The runtime already does the common cases. |
| [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | ASP.NET Core middleware pipeline; `Func<RequestDelegate, RequestDelegate>` | You already use this every request. |
| [Memento](../03-behavioral/05-memento.md) | `record` snapshots; event sourcing | Immutable records *are* mementos. |

A note on **source generators**, since they change the calculus: patterns whose entire purpose was to
generate boilerplate at runtime (Proxy via reflection, some Factory registration) can now be generated
at compile time with no runtime cost and full IntelliSense. If you find yourself building reflection
machinery to avoid writing repetitive code, check whether a generator already exists.

### TypeScript

| Pattern | Modern TS feature that often replaces it | Notes |
|---|---|---|
| [Strategy](../03-behavioral/08-strategy.md) | A function type: `type Price = (c: Car) => number` | The single most-replaced pattern in TS. |
| [Command](../03-behavioral/02-command.md) | Closures `() => void`; discriminated unions for serializable commands | `type Cmd = { kind: "publish"; id: number } \| { kind: "reprice"; id: number; price: number }` |
| [Iterator](../03-behavioral/03-iterator.md) | `function*` generators, `Symbol.iterator`, `for...of` | Fully absorbed. |
| Iterator (async) | `async function*`, `for await...of` | Perfect for paginated HTTP APIs. |
| [Observer](../03-behavioral/06-observer.md) | `EventTarget`, `addEventListener`, or a typed emitter; callbacks | The DOM has been Observer since 1996. |
| [Template Method](../03-behavioral/09-template-method.md) | Higher-order functions taking step callbacks | `withRetry(fn)`, `withTransaction(fn)` — composition beats inheritance here. |
| [Decorator](../02-structural/04-decorator.md) | Function wrapping (`const traced = withTiming(fetchPrice)`); TS `@decorator` syntax for class/method decoration | Function wrapping is simpler and needs no config. |
| [Proxy](../02-structural/07-proxy.md) | The built-in `Proxy` object with traps (`get`, `set`, `has`, `apply`) | A language-level implementation of the exact pattern. |
| [Adapter](../02-structural/01-adapter.md) | Structural typing — often no adapter needed at all | If the shape matches, TS accepts it. No `implements` required. |
| [State](../03-behavioral/07-state.md) | Discriminated unions + exhaustive `switch` | The compiler checks you handled every state. |
| [Visitor](../03-behavioral/10-visitor.md) | Discriminated unions + exhaustive `switch` with a `never` check | Same reason as C# pattern matching. |
| [Abstract Factory](../01-creational/02-abstract-factory.md) | A plain object of factory functions | `const sqlite = { makeRepo: () => ..., makeTx: () => ... }` |
| [Builder](../01-creational/03-builder.md) | Object literals + `Partial<T>` + spread, or `satisfies` | `{ ...defaults, ...overrides }` covers most cases. |
| [Singleton](../01-creational/05-singleton.md) | Module scope — a module is evaluated once | `export const cache = new Map()` is a singleton. |
| [Facade](../02-structural/05-facade.md) | A module with a small public surface; barrel `index.ts` | Still a good idea, just not a class. |
| [Flyweight](../02-structural/06-flyweight.md) | Rarely needed; V8 interns strings and hidden-classes objects | Measure before reaching for this. |

### Worked example: Visitor → pattern matching

This is the replacement that saves the most code, so it's worth seeing in full. Suppose you compute a
listing fee that depends on the kind of seller.

```csharp
// ---------- Visitor, the classic way ----------
public interface ISellerVisitor<TResult>
{
    TResult VisitDealer(Dealer dealer);
    TResult VisitPrivateSeller(PrivateSeller seller);
    TResult VisitAuctionHouse(AuctionHouse house);
}

public abstract class Seller
{
    public abstract TResult Accept<TResult>(ISellerVisitor<TResult> visitor);
}

public sealed class Dealer : Seller
{
    public int ActiveListings { get; init; }
    public override TResult Accept<TResult>(ISellerVisitor<TResult> v) => v.VisitDealer(this);
}
// ... two more subclasses, each with its own Accept ...

public sealed class ListingFeeVisitor : ISellerVisitor<decimal>
{
    public decimal VisitDealer(Dealer d) => d.ActiveListings > 50 ? 0m : 1999m;
    public decimal VisitPrivateSeller(PrivateSeller s) => 499m;
    public decimal VisitAuctionHouse(AuctionHouse h) => h.CommissionRate * 10000m;
}

var fee = seller.Accept(new ListingFeeVisitor());
```

```csharp
// ---------- The same logic with a closed hierarchy + pattern matching ----------
public abstract record Seller;
public sealed record Dealer(int ActiveListings) : Seller;
public sealed record PrivateSeller(string Email) : Seller;
public sealed record AuctionHouse(decimal CommissionRate) : Seller;

public static decimal ListingFee(Seller seller) => seller switch
{
    Dealer { ActiveListings: > 50 } => 0m,
    Dealer                          => 1999m,
    PrivateSeller                   => 499m,
    AuctionHouse a                  => a.CommissionRate * 10000m,
    _ => throw new ArgumentOutOfRangeException(nameof(seller))
};
```

Three types instead of seven. No `Accept` methods polluting the domain model. The property pattern
`Dealer { ActiveListings: > 50 }` expresses the branch condition inline. And adding a new operation
(say, `ListingDurationDays`) means writing one new function — you don't touch the `Seller` hierarchy at
all, which was supposed to be Visitor's whole selling point.

What you lose: the compiler won't force you to handle a newly added `Seller` subtype. If that matters,
`sealed` hierarchies plus a good analyzer, or a discriminated union in TypeScript with a `never` check,
gets it back:

```typescript
type Seller =
  | { kind: "dealer"; activeListings: number }
  | { kind: "private"; email: string }
  | { kind: "auction"; commissionRate: number };

function listingFee(seller: Seller): number {
  switch (seller.kind) {
    case "dealer":
      return seller.activeListings > 50 ? 0 : 1999;
    case "private":
      return 499;
    case "auction":
      return seller.commissionRate * 10000;
    default: {
      const unreachable: never = seller;   // compile error if a case is missed
      throw new Error(`Unhandled seller: ${JSON.stringify(unreachable)}`);
    }
  }
}
```

Add a fourth `kind` and this stops compiling until you handle it. That is exhaustiveness checking, and
it is strictly better than what Visitor gave you.

→ [Visitor](../03-behavioral/10-visitor.md) is still worth understanding — the pattern's *shape*
(separating an operation from the structure it walks) is exactly what this code does. It just does it
without the ceremony.

---

## 3️⃣ The Rule of Three

The single most useful heuristic for "should I abstract this?"

> **Write it once. Write it a second time and wince. Abstract on the third.**

The reasoning is not laziness. It's information:

```
  1 occurrence  ->  You have ZERO evidence about what varies.
                    Any abstraction is a guess.

  2 occurrences ->  You have ONE axis of variation. It's ambiguous:
                    two examples fit infinitely many patterns.
                    (1, 2 -> is it +1? x2? Fibonacci?)

  3 occurrences ->  The shape of the variation is visible.
                    Now the abstraction is derived, not invented.
```

At two occurrences the temptation is strongest and the information is weakest. Duplication is cheap to
fix; a wrong abstraction is expensive, because every future feature has to be bent around it, and
un-abstracting is harder than abstracting (you have to find and re-inline every caller).

### The corollary: duplication is cheaper than the wrong abstraction

This is worth stating plainly because it runs against instinct. If you copy-paste a 12-line function
and change three lines, the cost is: 12 lines of code, and a risk that a bug fix in one doesn't reach
the other. That's a real cost, but it's a *local* cost, and it's easy to fix later — you can always
merge two things that turned out to be the same.

If you build the wrong abstraction, the cost is: every future requirement arrives as a request to add
a boolean flag to the abstraction. Then another one. Six months later there's a function with five
optional parameters where four combinations are illegal and nobody remembers why.

The second cost is much larger and much harder to reverse.

### The exceptions to the Rule of Three

Don't wait for three when:

- **The thing is already known to vary.** You have a written requirement for three payment gateways.
  You have two now and the third is in the sprint. Build the abstraction — you have the information,
  it just arrived from the product owner instead of from the code.
- **It's a hard boundary.** Anything crossing a process, network, or ownership line (a RabbitMQ
  contract, an HTTP API, a database schema) is expensive to change after the fact. Design those
  deliberately from the start.
- **The duplication is dangerous.** Duplicated security checks, duplicated money arithmetic, duplicated
  validation that must stay consistent for correctness. Here the cost of divergence is a bug, not just
  untidiness.

And **wait longer than three** when the abstraction would cross a domain boundary — when the two things
look alike but belong to different parts of the business. Two classes both called `Vehicle` in the
pricing service and the logistics service may share every field today and diverge completely next year,
because they answer to different people. Shared structure is not shared meaning.

---

## 🔍 Genuine need vs speculative generality

"Speculative generality" is the code smell of building for a future that never comes. Here's how to
tell the two apart.

### The test questions

Ask these, in order, before adding an abstraction:

**1. Can I name a second implementation that exists today?**
Not "could exist." Exists, or is scheduled. `IPriceCalculator` with exactly one implementation,
`PriceCalculator`, is not an abstraction — it's a rename.

**2. What breaks if I inline this?**
Give a concrete failure, not a principle. "Our integration tests couldn't run without hitting the real
payment gateway" is concrete. "It would be tightly coupled" is not.

**3. Who asked?**
Is this variation in a requirement, a ticket, a roadmap item, a conversation with the business — or did
it come from your imagination during implementation?

**4. How expensive is it to add later?**
If extracting the abstraction later is a 20-minute mechanical refactor that your IDE can mostly do
(extract interface, extract method), then later is strictly better than now, because later you'll know
what shape it should be. If it's a database migration on a 40-million-row table, do it now.

**5. Does it make today's code easier to read?**
Some abstractions pay for themselves immediately by naming a concept. `applyDealerMargin(price)` is
better than an inline `price * 1.08 + 4999` even if it's called once, because the name carries meaning.
That's not speculative generality — that's just a good function.

### Smells that mean "speculative"

```
  SMELL                                    WHAT IT USUALLY MEANS
  -------------------------------------    -----------------------------------------
  Interface with one implementation,        A rename with extra steps
  ever, and no test double needed

  Abstract base class with one subclass     You designed for a hierarchy that
                                            never arrived

  Config option nobody has ever changed     A decision you were afraid to make

  A "Manager"/"Helper"/"Processor" class    A class with no single responsibility;
                                            the name is a placeholder for thinking

  Generic type parameter used with          Generality with no generality
  exactly one type argument

  Hook / extension point with zero          A door to a room that doesn't exist
  registered handlers

  Plugin architecture with no plugins       The most expensive form of this smell

  A "for future use" parameter,             Dead code that you have to keep
  always passed as null/default             compiling and reading
```

### A concrete before/after

Speculative:

```csharp
// "We might support other messaging systems one day"
public interface IMessageBusAbstractionFactory
{
    IMessageBusConnectionProvider CreateConnectionProvider(IMessageBusSettings settings);
}

public interface IMessageBusConnectionProvider
{
    IMessageBusChannel OpenChannel(string name);
}

public interface IMessageBusChannel : IDisposable
{
    void Publish(string topic, ReadOnlySpan<byte> payload, IMessageProperties properties);
}

// ...four more interfaces, one implementation of each, all RabbitMQ.
```

Genuine — same goal, honest scope:

```csharp
// One interface, at the boundary that actually matters: "we publish events."
public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : notnull;
}

public sealed class RabbitMqEventPublisher : IEventPublisher
{
    private readonly IChannelPool _channels;
    private readonly ILogger<RabbitMqEventPublisher> _logger;

    public RabbitMqEventPublisher(IChannelPool channels, ILogger<RabbitMqEventPublisher> logger)
        => (_channels, _logger) = (channels, logger);

    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : notnull
    {
        var exchange = EventNaming.ExchangeFor(typeof(TEvent));
        var body = JsonSerializer.SerializeToUtf8Bytes(@event);

        await using var channel = await _channels.RentAsync(ct);
        await channel.PublishAsync(exchange, routingKey: string.Empty, body, ct);

        _logger.LogDebug("Published {EventType} to {Exchange}", typeof(TEvent).Name, exchange);
    }
}

// And an in-memory one, which EXISTS, because the tests need it:
public sealed class InMemoryEventPublisher : IEventPublisher
{
    public List<object> Published { get; } = new();

    public Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default)
        where TEvent : notnull
    {
        Published.Add(@event);
        return Task.CompletedTask;
    }
}
```

One interface, two implementations that both exist, both used. The abstraction is justified by a
present fact (tests need a fake) rather than a future hope (we might switch to Kafka). And if you *do*
switch to Kafka, this interface is exactly the seam you need — you didn't have to predict it, you just
had to not over-build.

---

## 🤝 How to push back in a code review, kindly

You will, sooner or later, review a PR where someone has built four classes to do one thing. How you
handle that determines whether they learn something or just feel bad. Some things that work:

### Ask, don't assert

The single highest-leverage move. Compare:

> ❌ "This is over-engineered. We don't need a factory here."

> ✅ "Curious about the factory here — is there a second implementation coming, or a case where we'd
> want to swap it? If it's just the one, I'd lean toward calling the constructor directly and adding
> the factory when we need the second one. Happy to be wrong if there's context I'm missing."

The second version does three things: it asks for information you genuinely might not have, it states
your preference and your reasoning, and it leaves the door open. It also costs you nothing if you turn
out to be wrong.

### Name the cost, not the principle

"This violates YAGNI" is an appeal to authority and invites an argument about YAGNI. Concrete costs
don't:

> "Tracing a price change through `PriceService → IPriceStrategyFactory → PriceStrategyRegistry →
> DealerPriceStrategy` takes four file-opens. If it's one strategy today, could we start with the
> function and split it out when a second shows up?"

Nobody argues with "four file-opens." Everybody argues with "YAGNI."

### Make it about the future reader, not the author

> "When someone's on call at 2am chasing a wrong price, how many hops do they make to find the number?"

This reframes it from "your code is bad" to "we're both trying to help a third person," which is true
and takes the sting out.

### Give the smaller version, don't just ask for one

If you can suggest the five-line replacement in the comment, do. It turns "do more work" into "do less
work," which is a much easier ask:

> "I think this whole file could be:
> ```typescript
> export const displayName = (car: Car) => `${car.year} ${car.make} ${car.model}`;
> ```
> Am I missing a case?"

### Distinguish "I'd do it differently" from "this is a problem"

Most over-patterning is not a bug. Be explicit about severity so people can calibrate:

- **"Blocking:"** this will cause a production issue.
- **"Non-blocking, strong opinion:"** I'd change this; I'll approve either way after we talk.
- **"Nit / take it or leave it:"** preference only.

An unlabelled comment defaults to "blocking" in the author's head. Labelling it costs one word and
prevents a lot of friction.

### Acknowledge what's good, and mean it

Over-patterning usually comes from *caring*. The person was thinking about extensibility, testability,
future maintainers. That instinct is right; only the calibration is off. Say so:

> "I like that you thought about where this would need to vary — that instinct is right. I think the
> variation point is a bit further out than this, though. What if we..."

### Know when to let it go

If the abstraction is:

- small,
- local to one module,
- not on a hot path,
- and easy to delete later,

...and the author feels strongly, **approve it**. You will spend your review capital better elsewhere.
Being right about a three-class Factory is not worth being the reviewer people dread. A codebase with
some unnecessary indirection and a team that talks to each other is in far better shape than the
reverse.

### And when you're the author

Take the note. "Fair — I was thinking we'd need a second one for the auction flow, but that's not
scheduled. Simplified it, we can split it out when it lands" is a great response, and nobody thinks
less of you for it. The engineers who look strongest in review are the ones who change their mind
cheaply.

---

## ✅ What survives all of this

After all four criticisms, here is what's left — and it's more than you might expect.

**The vocabulary survives, completely.** "This is a Decorator" / "let's put an Adapter at the boundary"
/ "that's just a Strategy" communicates a design in seconds across a team, across companies, across
languages. That value is independent of whether you write a class or a lambda. This is, arguably, the
patterns' single biggest contribution: a shared language for design discussions that previously had to
be conducted by drawing on whiteboards.

**The problem catalogue survives.** Even where the solution has been absorbed by the language, the
*problem* each pattern names is real and recurring: "how do I add behaviour to an object without
subclassing it" (Decorator), "how do I let a subsystem notify interested parties without knowing who
they are" (Observer), "how do I walk a structure without exposing its shape" (Iterator). Learning the
catalogue teaches you to *recognise* those problems, which is most of the battle.

**The reading skill survives.** You will read code written by people who did use the classic forms —
in older C# codebases, in Java, in C++, in every framework you depend on. ASP.NET Core's middleware is
Chain of Responsibility. Entity Framework's `DbContext` is a Unit of Work over Repositories. `IObservable<T>`
is Observer. Recognising them makes unfamiliar code legible.

**The structural patterns survive more or less intact.** Adapter, Facade, Decorator, Proxy, Composite
are about *relationships between components*, and no language feature has absorbed them. They're as
relevant as they were in 1994.

**What has genuinely faded:** the one-method-interface patterns (Strategy, Command, Template Method in
its inheritance form), Iterator, and the more ceremonial creational patterns in DI-container codebases.
Not useless — just usually expressible in a tenth of the code.

### The honest summary

```
  +--------------------------------------------------------------+
  |                                                              |
  |   Patterns are a VOCABULARY and a CATALOGUE OF PROBLEMS.     |
  |                                                              |
  |   They are not a checklist, a quality metric, or a           |
  |   target to hit.                                             |
  |                                                              |
  |   Learn all 23. Recognise them everywhere.                   |
  |   Implement maybe six of them by hand, ever.                 |
  |                                                              |
  |   Before you build one, ask:                                 |
  |     - Does my language already have this?                    |
  |     - Do I have three real examples, or one guess?           |
  |     - What concretely breaks if I inline it?                 |
  |                                                              |
  |   A junior writes code. A mid-level engineer adds            |
  |   patterns. A senior engineer removes the ones that          |
  |   weren't needed.                                            |
  |                                                              |
  +--------------------------------------------------------------+
```

---

## 📚 Where to go next

- [SOLID and other principles](../04-extras/01-solid-and-principles.md) — the principles behind the
  patterns, with the same caution applied: they're lenses, not laws.
- Start with the patterns that survived best: [Adapter](../02-structural/01-adapter.md),
  [Decorator](../02-structural/04-decorator.md), [Facade](../02-structural/05-facade.md),
  [Observer](../03-behavioral/06-observer.md), [Builder](../01-creational/03-builder.md).
- Read the ones your language absorbed anyway — [Strategy](../03-behavioral/08-strategy.md),
  [Iterator](../03-behavioral/03-iterator.md), [Command](../03-behavioral/02-command.md),
  [Visitor](../03-behavioral/10-visitor.md) — to understand what `yield return`, `Func<>`, and
  `switch` expressions are actually doing under the hood.
- Read [Singleton](../01-creational/05-singleton.md) last, and read it as a cautionary tale.

---

*Keep this page bookmarked. Come back to it the week after you learn a pattern you love.*
