# 🧩 What Is a Design Pattern?

> **Start here.** This is the theory primer for the whole folder. Nothing below assumes you have
> read the Gang of Four book, or any book. By the end you will know what a pattern *is*, what it
> is *not*, where the idea came from, how the 23 classic patterns are grouped, and how the whole
> thing sits on top of the SOLID principles you have probably already bumped into.

---

## 🎯 The one-paragraph version

A **design pattern** is a named, tested, reusable *shape* for solving a problem that keeps coming
back in software design. It is not code. It is not a library. It is a description of how a small
handful of objects can be arranged and how they should talk to each other, such that a particular
kind of change becomes cheap later.

That is genuinely the whole idea. Everything else in this folder is the catalogue.

---

## 🧱 A pattern is a blueprint, not a recipe

The single most useful mental model:

```
  ALGORITHM                              PATTERN
  ---------                              -------
  a recipe                               a blueprint
  "do these steps, in this order"         "here is the shape of the result"
  deterministic                           adaptable
  same code every time                    different code every time
  answers "how do I compute X?"           answers "how do I structure X so it survives change?"
```

An algorithm — quicksort, Dijkstra, binary search — defines a clear set of actions that achieves a
goal. You can copy quicksort out of a textbook, translate it to TypeScript, and it is *the same
algorithm*. Its correctness is provable. Its steps are fixed.

A pattern is a level higher. It tells you what the finished structure looks like and what
properties it has, but the exact order of implementation, the naming, the number of classes, and
the language features you lean on are all up to you. Two teams applying Strategy to two different
problems will produce code that does not look alike at the token level — and both are correct
applications of the pattern.

This is why you cannot "install" a pattern. There is no `npm i observer`.

### A concrete side-by-side

Here is an algorithm. It is a recipe. There is a right answer.

```typescript
// ALGORITHM: binary search. Copy it, translate it, it stays the same algorithm.
function binarySearch(sorted: number[], target: number): number {
  let lo = 0;
  let hi = sorted.length - 1;

  while (lo <= hi) {
    const mid = (lo + hi) >>> 1;
    const value = sorted[mid];

    if (value === target) return mid;
    if (value < target) lo = mid + 1;
    else hi = mid - 1;
  }

  return -1;
}
```

Here is a pattern. It is a blueprint. There is no single right answer — there is a shape.

```typescript
// PATTERN: Strategy. The *shape* is "swap the algorithm behind a stable interface".
// Domain: pricing a used-car listing on a marketplace.

interface PricingStrategy {
  readonly name: string;
  price(listing: Listing): Money;
}

interface Listing {
  askingPrice: Money;
  year: number;
  kilometres: number;
  sellerType: 'dealer' | 'individual';
  certified: boolean;
}

type Money = { amount: number; currency: 'INR' };

class AsIsPricing implements PricingStrategy {
  readonly name = 'as-is';
  price(listing: Listing): Money {
    return listing.askingPrice;
  }
}

class CertifiedPreOwnedPricing implements PricingStrategy {
  readonly name = 'certified-pre-owned';
  constructor(private readonly warrantyCost: number) {}

  price(listing: Listing): Money {
    const inspection = 3_500;
    return {
      amount: listing.askingPrice.amount + this.warrantyCost + inspection,
      currency: 'INR',
    };
  }
}

class DealerMarkupPricing implements PricingStrategy {
  readonly name = 'dealer-markup';
  constructor(private readonly marginPercent: number) {}

  price(listing: Listing): Money {
    const multiplier = 1 + this.marginPercent / 100;
    return {
      amount: Math.round(listing.askingPrice.amount * multiplier),
      currency: 'INR',
    };
  }
}

class ListingPricer {
  constructor(private strategy: PricingStrategy) {}

  use(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  quote(listing: Listing): Money {
    return this.strategy.price(listing);
  }
}
```

Now notice: nothing above is *the* Strategy pattern. Somebody else implementing Strategy in C#
might use a `Func<Listing, Money>` delegate and no interface at all:

```csharp
// Same pattern. Completely different code. Still Strategy.
public sealed class ListingPricer
{
    private Func<Listing, Money> _strategy;

    public ListingPricer(Func<Listing, Money> strategy) => _strategy = strategy;

    public void Use(Func<Listing, Money> strategy) => _strategy = strategy;

    public Money Quote(Listing listing) => _strategy(listing);
}

// Registration somewhere in composition root:
var pricer = new ListingPricer(listing => listing.AskingPrice);
pricer.Use(listing => listing.AskingPrice with
{
    Amount = (long)Math.Round(listing.AskingPrice.Amount * 1.08m)
});
```

Both are Strategy. The blueprint says: *isolate the varying behaviour behind a stable seam, and let
the caller choose which one is plugged in at runtime.* How you express the seam — interface,
delegate, function type, abstract class, discriminated union — is your business.

Full chapter: [Strategy](../03-behavioral/08-strategy.md).

---

## 🚫 What a design pattern is NOT

This section exists because most of the confusion about patterns is confusion about category.

### ❌ Not an algorithm

Covered above. An algorithm computes; a pattern arranges. You can unit-test an algorithm against a
known-correct output. You cannot unit-test "did I apply Decorator correctly" — you can only ask
whether the code is easier to change now.

### ❌ Not a library or a framework

There is no package to install. `RxJS` is not the Observer pattern, it is a library that is *built
using* Observer ideas. `System.Text.Json`'s converters are not the Strategy pattern, they are an
API that *uses* Strategy. When you `npm install` something, you get an implementation. When you
learn a pattern, you get a way to think.

The confusion runs the other way too: many things you already use are patterns wearing a product
name.

| Thing you already use | Pattern underneath |
|---|---|
| `IEnumerable<T>` / `IEnumerator<T>` in C#, `Symbol.iterator` in JS | [Iterator](../03-behavioral/03-iterator.md) |
| ASP.NET Core middleware pipeline | [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) + [Decorator](../02-structural/04-decorator.md) |
| `IServiceCollection` / DI container registrations | [Abstract Factory](../01-creational/02-abstract-factory.md), [Factory Method](../01-creational/01-factory-method.md) |
| `EventEmitter` in Node, C# `event` keyword | [Observer](../03-behavioral/06-observer.md) |
| `StringBuilder`, fluent query builders, `HttpRequestMessage` builders | [Builder](../01-creational/03-builder.md) |
| Entity Framework lazy-loading proxies | [Proxy](../02-structural/07-proxy.md) |
| A repository that wraps three different upstream APIs into one shape | [Adapter](../02-structural/01-adapter.md) or [Facade](../02-structural/05-facade.md) |
| `undo`/`redo` in any editor | [Command](../03-behavioral/02-command.md) + [Memento](../03-behavioral/05-memento.md) |

You have almost certainly written several of these already without naming them. That is the normal
path: **patterns are discovered, not invented.** Somebody notices the same arrangement appearing in
project after project, names it, writes it down, and now everyone can point at it.

### ❌ Not copy-paste code

A pattern chapter in this folder will show you code. That code is an *illustration*, not a
deliverable. If you paste it in unchanged you will usually get the wrong number of classes, the
wrong names, and indirection you did not need. The value is in the shape and the trade-off, not the
characters.

### ❌ Not a goal

This is the big one. Nobody gets points for using patterns. "We used seven patterns in this module"
is a bug report, not a brag. A pattern is a cost you pay — extra types, extra indirection, extra
reading — in exchange for a specific kind of future flexibility. If you are not buying flexibility
you actually need, you paid for nothing.

The healthy sequence is:

```
  1. Write the simple, direct thing.
  2. Feel the pain when it changes.      <-- this step is not optional
  3. Recognise the shape of the pain.
  4. Apply the pattern that relieves it.
```

Jumping from 1 to 4 is how codebases end up with `AbstractListingProcessorFactoryProvider`.

### ❌ Not object-oriented-only (but the classic 23 mostly are)

The 23 classic patterns were written in 1994 in the vocabulary of class-based OO — C++ and
Smalltalk. Several of them exist to work around things those languages lacked. In a language with
first-class functions (JavaScript, TypeScript, modern C#), Strategy and Command collapse into "pass
a function"; Template Method often collapses into "pass a couple of callbacks". That does not make
them wrong — it makes them cheaper. The *problem* each pattern names is still real.

---

## 📐 What a pattern description conventionally contains

Patterns are documented formally so that people can reproduce them across wildly different
contexts. A catalogue entry — including the ones in this folder — usually carries these sections:

- **Intent** — one or two sentences stating the problem and the solution together. This is the part
  you memorise.
- **Motivation** — a concrete story where the naive design breaks, and how the pattern fixes it.
- **Applicability** — when to reach for it, and (just as importantly) when not to.
- **Structure** — a diagram of the participating classes/objects and their relationships.
- **Participants** — what each role in the structure is responsible for.
- **Collaborations** — how the participants talk to each other at runtime.
- **Consequences** — the trade-offs. What you gain, what you pay. Never skip this section.
- **Implementation notes** — the practical gotchas, per language.
- **Sample code** — an illustration in a real language.
- **Known uses** — places in real systems where this shows up.
- **Related patterns** — what it is often confused with, what it composes well with.

If you only ever read two sections of a pattern write-up, read **Intent** and **Consequences**.
Intent tells you whether the pattern is even about your problem. Consequences tells you whether the
cure is worse than the disease.

Here is what the "Structure" part looks like in ASCII, using Strategy again so you have something
concrete to anchor on:

```
   +------------------+          uses          +--------------------+
   |  ListingPricer   |---------------------->|  PricingStrategy    |   <-- the stable seam
   |  (Context)       |                        |  (interface)        |
   +------------------+                        +--------------------+
   | - strategy       |                        | + price(l): Money   |
   | + quote(l)       |                        +--------------------+
   +------------------+                                  ^
                                                         |  implements
                                    +--------------------+--------------------+
                                    |                    |                    |
                          +------------------+ +--------------------+ +-------------------+
                          |  AsIsPricing     | | CertifiedPricing   | | DealerMarkup      |
                          +------------------+ +--------------------+ +-------------------+
                          | + price(l)       | | + price(l)         | | + price(l)        |
                          +------------------+ +--------------------+ +-------------------+
```

The arrow that matters is the one from `ListingPricer` to the *interface*, not to any concrete
class. Almost every pattern in the catalogue has one arrow like that. Finding it is how you read a
pattern diagram fast.

---

## 📜 Where this came from: a short, honest history

### 1️⃣ Christopher Alexander — a building architect, not a programmer

The idea of a "pattern" as a unit of design documentation comes from **Christopher Alexander**, an
architect who wrote about towns and buildings. His book *A Pattern Language: Towns, Buildings,
Construction* describes a "language" for designing the built environment, where the units of that
language are patterns: recurring, nameable solutions to recurring problems in how people live in
space. How high a window should sit. How many storeys a building should have. How much green space
a neighbourhood needs.

The insight software borrowed was not any particular architectural rule. It was the *format*: a
recurring problem in a context, plus a solution described generally enough to be reapplied, plus a
name so people can talk about it.

### 2️⃣ 1994 — the Gang of Four

Four authors — **Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides** — picked up
Alexander's idea and applied it to object-oriented programming. In 1994 they published *Design
Patterns: Elements of Reusable Object-Oriented Software*, cataloguing **23 patterns** for common
problems in OO design.

The book's title is a mouthful, so people started calling it "the book by the gang of four", which
compressed to **"the GoF book"**. The four authors are collectively **the Gang of Four (GoF)**, and
the 23 patterns are **the GoF patterns**. When someone in a code review says "that's a GoF pattern",
this is what they mean.

The examples in the original book are in C++ and Smalltalk. That is worth knowing, because it
explains why a few of the patterns feel heavier than they need to be in TypeScript or C#: they are
routing around language limitations that no longer exist for you.

### 3️⃣ Since then

Dozens more OO patterns have been described, and the "pattern approach" spread well beyond OO
design — concurrency patterns, enterprise integration patterns, messaging patterns (relevant if you
work with RabbitMQ: publish/subscribe, competing consumers, dead-letter queue, saga), data access
patterns, cloud patterns. The GoF 23 remain the shared core because they are the ones everybody has
read.

One more historical note worth carrying: patterns were **discovered**, not designed. Nobody sat
down and invented Observer. People wrote Observer-shaped code over and over across unrelated
projects, and eventually somebody wrote it down and named it. That is why the catalogue feels
arbitrary in places — it is a field guide to things that were already living in the wild.

---

## 💡 Why bother learning them?

You can be a working programmer for a decade without knowing a single pattern by name. Many people
are. You will still *implement* several of them, by feel, without knowing they have names. So why
spend the time?

### Reason 1: a toolkit of tried and tested solutions

These 23 arrangements have been beaten on by millions of developers across three decades. When you
hit a problem that one of them fits, you are not designing from scratch — you are picking up a
solution whose failure modes are already documented. And even for problems none of them fit, reading
them trains you in the underlying moves: program to an interface, prefer composition over
inheritance, isolate what varies, depend on abstractions.

That second effect is the bigger one, honestly. The patterns are a **worked-examples course in
object-oriented design principles**. The catalogue is the excuse; the design instinct is the prize.

### Reason 2: shared vocabulary

This is the reason that pays off on day one.

Without vocabulary:

> "So what I want is, the thing that writes listings to Elasticsearch shouldn't know who else cares.
> It should just announce that a listing changed, and then whoever registered interest gets called,
> and we can add more of those later without touching the writer, and ideally they can unregister..."

With vocabulary:

> "Make it an Observer."

Four words. Same meaning. Everyone in the room now knows the shape, the participants, and roughly
what the code will look like. Patterns compress design conversations by an order of magnitude — in
code review, in design docs, in onboarding, and in interviews.

The same compression works in the other direction, for reading code. When you open an unfamiliar
C# repo and see `IHandler`, `Handle`, and a `_next` field, you do not have to trace it — you
recognise Chain of Responsibility and you already know where to look for the bug.

### Reason 3: it makes you a better reader of frameworks

Every framework you use is built out of these. Once you can name them, framework source stops being
magic. You will read ASP.NET Core's middleware, or Express's `app.use`, or RxJS operators, and see
familiar furniture.

### ⚠️ And the honest counterweight

Patterns are also the single most over-applied idea in software. The failure mode is real and it
has a name: **pattern-happy code**, where every simple thing is wrapped in three layers of
indirection because indirection felt professional. Symptoms:

- An interface with exactly one implementation, forever, that nobody ever mocks.
- A factory that only ever returns `new Foo()`.
- A Strategy interface with one strategy.
- A Visitor over a type hierarchy that has not changed in four years and never will.

The rule of thumb: **a pattern is justified by a change you can name.** If you cannot finish the
sentence "we are adding this indirection because *X* is going to vary", do not add it. Write the
direct code. You can refactor into a pattern later — that is precisely what patterns are good at,
being refactoring destinations.

---

## 🗂️ The three categories

All 23 GoF patterns are grouped by **intent** — what kind of problem they exist to solve.

```
                   +--------------------------------------+
                   |          GoF design patterns          |
                   +--------------------------------------+
                      |               |                |
              +-------+-----+  +------+------+  +------+--------+
              | CREATIONAL  |  | STRUCTURAL  |  |  BEHAVIORAL   |
              +-------------+  +-------------+  +---------------+
              | how objects |  | how objects |  | how objects   |
              | get made    |  | fit together|  | talk & share  |
              |             |  |             |  | responsibility|
              +-------------+  +-------------+  +---------------+
                  5 patterns      7 patterns        11 patterns
```

A three-word memory hook: **creational = make**, **structural = compose**, **behavioral = talk**.

### 🏭 Creational patterns — object creation mechanisms

**Definition:** creational patterns provide object creation mechanisms that increase flexibility and
reuse of existing code. They decouple *what gets created* and *how* from the code that uses the
result.

The problem they all address: `new ConcreteThing()` welds the caller to a specific class at compile
time. That is fine until you need a second kind of thing, or the construction gets complicated, or
you need exactly one instance, or creation is expensive.

| Pattern | One-line intent |
|---|---|
| [Factory Method](../01-creational/01-factory-method.md) | Define an interface for creating an object, but let subclasses decide which class to instantiate. |
| [Abstract Factory](../01-creational/02-abstract-factory.md) | Create whole *families* of related objects without naming their concrete classes. |
| [Builder](../01-creational/03-builder.md) | Construct a complex object step by step, so the same construction process can produce different representations. |
| [Prototype](../01-creational/04-prototype.md) | Create new objects by copying an existing instance rather than constructing from scratch. |
| [Singleton](../01-creational/05-singleton.md) | Ensure a class has exactly one instance and give it a global access point. (Read the consequences section on this one carefully — it is the most criticised pattern in the book.) |

### 🧱 Structural patterns — assembling objects into bigger structures

**Definition:** structural patterns explain how to assemble objects and classes into larger
structures while keeping those structures flexible and efficient. They are about *composition* —
how one object can wrap, front, or stand in for another.

Almost all of them are variations on "object A holds a reference to object B and presents some
interface". What distinguishes them is *intent*, not shape. Adapter and Decorator can look
byte-for-byte similar; they differ in why you wrote them. Getting this straight is most of the work
of learning the structural group.

| Pattern | One-line intent |
|---|---|
| [Adapter](../02-structural/01-adapter.md) | Convert one interface into another that a client expects, so incompatible things can work together. |
| [Bridge](../02-structural/02-bridge.md) | Split an abstraction from its implementation so the two can vary independently. |
| [Composite](../02-structural/03-composite.md) | Compose objects into tree structures and let clients treat individual objects and compositions uniformly. |
| [Decorator](../02-structural/04-decorator.md) | Attach additional responsibilities to an object dynamically, keeping the same interface. |
| [Facade](../02-structural/05-facade.md) | Provide one simplified interface in front of a complicated subsystem. |
| [Flyweight](../02-structural/06-flyweight.md) | Share fine-grained objects to support very large numbers of them efficiently. |
| [Proxy](../02-structural/07-proxy.md) | Provide a stand-in for another object to control access to it (lazy loading, caching, remoting, permissions). |

A quick discriminator you will want later:

```
  Adapter    : interface differs      -> "make B fit A's socket"
  Decorator  : interface identical    -> "same socket, more behaviour"
  Proxy      : interface identical    -> "same socket, controls access"
  Facade     : interface simplified   -> "one socket instead of nine"
  Bridge     : two axes of variation  -> "two hierarchies, wired at composition time"
```

### 🔁 Behavioral patterns — communication and responsibility

**Definition:** behavioral patterns take care of effective communication and the assignment of
responsibilities between objects. They are about the *runtime conversation* — who calls whom, in
what order, and how tightly the caller is bound to the callee.

This is the largest group, and in day-to-day backend work it is the most useful one.

| Pattern | One-line intent |
|---|---|
| [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | Pass a request along a chain of handlers; each decides to handle it or forward it. |
| [Command](../03-behavioral/02-command.md) | Turn a request into a standalone object, so you can queue it, log it, retry it, or undo it. |
| [Iterator](../03-behavioral/03-iterator.md) | Traverse the elements of a collection without exposing how it is stored. |
| [Mediator](../03-behavioral/04-mediator.md) | Route communication between objects through a single hub so they stop referring to each other directly. |
| [Memento](../03-behavioral/05-memento.md) | Capture and restore an object's internal state without breaking its encapsulation. |
| [Observer](../03-behavioral/06-observer.md) | Let objects subscribe to and be notified of events in another object. |
| [State](../03-behavioral/07-state.md) | Let an object change its behaviour when its internal state changes — as if it changed class. |
| [Strategy](../03-behavioral/08-strategy.md) | Define a family of interchangeable algorithms and make them swappable at runtime. |
| [Template Method](../03-behavioral/09-template-method.md) | Define the skeleton of an algorithm in a base class and let subclasses fill in specific steps. |
| [Visitor](../03-behavioral/10-visitor.md) | Separate an operation from the object structure it operates on, so new operations can be added without changing the structure. |

That is 5 + 7 + 10 = 22 in the tables above; the twenty-third is **Interpreter**, a behavioral
pattern for evaluating sentences in a small grammar. It is far and away the least-used of the 23 in
ordinary application code — if you need it, you generally reach for a parser generator or a proper
expression tree instead — which is why this folder does not give it its own chapter.

---

## 🪜 Levels of abstraction: idioms → design patterns → architectural patterns

Patterns differ by **scale**. The road-construction analogy from the source material is a good one:
you can make a junction safer by putting up traffic lights, or by building a multi-level interchange
with underground pedestrian passages. Both are valid solutions to "this junction is dangerous". They
are not the same size of intervention.

```
  scale
    ^
    |   ARCHITECTURAL PATTERNS      shape of the whole system
    |   Layered, Hexagonal/Ports & Adapters, MVC/MVVM, CQRS,
    |   Event-Driven, Microservices, Pipes & Filters, Saga
    |   ------------------------------------------------------
    |   DESIGN PATTERNS             shape of a handful of classes
    |   the GoF 23: Strategy, Observer, Decorator, Builder, ...
    |   ------------------------------------------------------
    |   IDIOMS                      shape of a few lines, language-specific
    |   RAII (C++), IDisposable + using (C#), IIFE (JS),
    |   try-with-resources (Java), null-coalescing, the
    |   module pattern, `as const`, discriminated unions
    +-------------------------------------------------------->
                                                    generality
```

### Idioms — the low-level end

The most basic, lowest-level patterns are usually called **idioms**. They typically apply to a
single programming language, and they are about how you express something, not how you structure a
system.

```cpp
// C++ idiom: RAII. The destructor releases the resource; there is no `finally`.
class FileHandle {
public:
    explicit FileHandle(const char* path) : file_(std::fopen(path, "rb")) {}
    ~FileHandle() { if (file_) std::fclose(file_); }

    FileHandle(const FileHandle&)            = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    std::FILE* get() const { return file_; }

private:
    std::FILE* file_;
};
```

```csharp
// C# idiom: the same *problem* (deterministic cleanup), different idiom:
// IDisposable plus a using declaration.
public sealed class FileHandle : IDisposable
{
    private readonly FileStream _stream;

    public FileHandle(string path) => _stream = File.OpenRead(path);

    public Stream Stream => _stream;

    public void Dispose() => _stream.Dispose();
}

// Caller:
using var handle = new FileHandle("listings.csv");
// disposed at end of scope
```

```typescript
// TypeScript idiom: exhaustiveness checking with a discriminated union.
// Nothing to do with GoF; entirely about expressing one thing well in one language.
type ListingEvent =
  | { kind: 'created'; listingId: string }
  | { kind: 'priceChanged'; listingId: string; newPrice: number }
  | { kind: 'sold'; listingId: string; soldAt: string };

function describe(event: ListingEvent): string {
  switch (event.kind) {
    case 'created':
      return `Listing ${event.listingId} created`;
    case 'priceChanged':
      return `Listing ${event.listingId} repriced to ${event.newPrice}`;
    case 'sold':
      return `Listing ${event.listingId} sold at ${event.soldAt}`;
    default: {
      const unreachable: never = event;   // compile error if a case is missed
      return unreachable;
    }
  }
}
```

Idioms do not transfer between languages. RAII has no meaning in JavaScript. The `never`
exhaustiveness trick has no meaning in C++.

### Design patterns — the middle

This is the GoF layer, and the subject of this folder. Design patterns describe how a *small group
of classes or objects* — typically two to six roles — should be arranged. They transfer across any
language that has some notion of polymorphism, which is essentially all of them.

Scope test: if the answer fits inside one module or one bounded piece of a service, and involves a
handful of types, it is a design pattern.

### Architectural patterns — the high end

The most universal and highest-level patterns are **architectural patterns**. They can be
implemented in virtually any language, and unlike the other two levels they describe the structure
of an *entire application*: layering, deployment boundaries, where state lives, how components find
each other.

Examples you will meet in backend work:

- **Layered / N-tier** — controllers → services → repositories → database.
- **Hexagonal (Ports and Adapters)** — the domain sits in the middle and knows nothing about HTTP,
  SQL, or RabbitMQ; everything external plugs in through a port. Notice that "Adapter" here is the
  same *idea* as the GoF Adapter, applied at system scale.
- **MVC / MVVM** — separating presentation, state, and coordination.
- **CQRS** — separate models for writes and reads, often with separate stores.
- **Event-driven / publish-subscribe** — services communicate by publishing events to a broker. If
  you run RabbitMQ, you are living in this one. It is Observer, promoted to a deployment topology,
  with durability and delivery guarantees bolted on.
- **Pipes and Filters** — a message flows through independent processing stages.
- **Saga** — coordinating a business transaction across services that have no shared database.

The boundaries between the three levels are genuinely fuzzy. Observer at the object level is
pub/sub at the system level. Chain of Responsibility at the object level is a middleware pipeline at
the framework level and Pipes-and-Filters at the architecture level. That fuzziness is a feature:
the same few ideas keep working at every scale, which is why learning the middle layer pays off at
both ends.

---

## 🧭 How patterns map onto SOLID

This section is not in the classic catalogue, but it is the thing that made patterns click for a lot
of people, so it is worth its own treatment. (Deeper dive: [SOLID and other principles](../04-extras/01-solid-and-principles.md).)

**The short version: SOLID states the goals. Patterns are worked examples of reaching them.**

Every pattern in the catalogue is, in effect, someone answering "how do I actually satisfy
Open/Closed here?" or "how do I actually invert this dependency?" with concrete furniture. If SOLID
tells you *what good looks like*, patterns tell you *what to type*.

Here is the mapping, principle by principle.

### S — Single Responsibility Principle

> A class should have one reason to change.

Patterns that exist to *split* a responsibility out of an overloaded class:

- **[Strategy](../03-behavioral/08-strategy.md)** — "how to compute it" leaves the class that
  decides *when* to compute it.
- **[Command](../03-behavioral/02-command.md)** — "what to do" becomes its own object, separate from
  "when to do it" and "who asked".
- **[Visitor](../03-behavioral/10-visitor.md)** — operations over a structure move out of the
  structure's own classes.
- **[Memento](../03-behavioral/05-memento.md)** — snapshotting moves out of the object being
  snapshotted (it only exposes a sealed capsule) and into a caretaker.
- **[Builder](../01-creational/03-builder.md)** — assembly logic leaves the product class.
- **[Facade](../02-structural/05-facade.md)** — "orchestrating the subsystem" becomes one class's
  single job so every caller stops doing it.

The before/after in C#:

```csharp
// BEFORE: this class changes when tax rules change, when the discount policy
// changes, AND when the notification channel changes. Three reasons.
public sealed class OrderProcessor
{
    public void Process(Order order)
    {
        decimal tax = order.State == "MH" ? order.Total * 0.18m : order.Total * 0.12m;

        decimal discount = order.Customer.IsDealer
            ? order.Total * 0.05m
            : order.Total > 500_000m ? 10_000m : 0m;

        order.Final = order.Total + tax - discount;

        var smtp = new SmtpClient("mail.internal");
        smtp.Send(new MailMessage("noreply@example.com", order.Customer.Email,
            "Order confirmed", $"Total {order.Final}"));
    }
}
```

```csharp
// AFTER: three seams, three reasons to change, three separate classes.
public interface ITaxPolicy      { decimal TaxFor(Order order); }
public interface IDiscountPolicy { decimal DiscountFor(Order order); }
public interface IOrderNotifier  { Task NotifyAsync(Order order, CancellationToken ct); }

public sealed class OrderProcessor
{
    private readonly ITaxPolicy _tax;
    private readonly IDiscountPolicy _discount;
    private readonly IOrderNotifier _notifier;

    public OrderProcessor(ITaxPolicy tax, IDiscountPolicy discount, IOrderNotifier notifier)
    {
        _tax = tax;
        _discount = discount;
        _notifier = notifier;
    }

    public async Task ProcessAsync(Order order, CancellationToken ct)
    {
        order.Final = order.Total + _tax.TaxFor(order) - _discount.DiscountFor(order);
        await _notifier.NotifyAsync(order, ct);
    }
}
```

`ITaxPolicy` and `IDiscountPolicy` are Strategies. `IOrderNotifier` is a port that will be satisfied
by an Adapter over SMTP (and later, maybe, an Adapter over your RabbitMQ publisher). Three
principles served by one refactor.

### O — Open/Closed Principle

> Open for extension, closed for modification. Add behaviour by adding code, not by editing existing
> code.

This is the principle patterns serve most directly. Roughly two-thirds of the catalogue is a
different answer to "how do I add a case without touching a switch statement?"

- **[Strategy](../03-behavioral/08-strategy.md)** — new algorithm = new class, no edit.
- **[Decorator](../02-structural/04-decorator.md)** — new responsibility = new wrapper, no edit to
  the wrapped class.
- **[Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md)** — new handler =
  insert into the chain.
- **[Factory Method](../01-creational/01-factory-method.md)** / **[Abstract Factory](../01-creational/02-abstract-factory.md)** —
  new product = new creator, callers untouched.
- **[Visitor](../03-behavioral/10-visitor.md)** — new operation = new visitor. (Note the trade-off:
  Visitor is open for new *operations* but closed against new *node types*. The only pattern where
  you should read the fine print twice.)
- **[State](../03-behavioral/07-state.md)** — new state = new class, instead of a new branch in
  three switch statements.
- **[Observer](../03-behavioral/06-observer.md)** — new reaction to an event = new subscriber. The
  publisher never learns about it.

The canonical smell that Open/Closed is being violated:

```typescript
// Every new channel means editing this function. Closed for extension, open for
// modification -- exactly backwards.
function notify(channel: string, listingId: string, body: string): void {
  if (channel === 'email') {
    sendEmail(listingId, body);
  } else if (channel === 'sms') {
    sendSms(listingId, body);
  } else if (channel === 'push') {
    sendPush(listingId, body);
  } else {
    throw new Error(`unknown channel ${channel}`);
  }
}
```

```typescript
// Strategy + a registry. New channel = new object + one registration line.
interface NotificationChannel {
  readonly key: string;
  send(listingId: string, body: string): Promise<void>;
}

class EmailChannel implements NotificationChannel {
  readonly key = 'email';
  constructor(private readonly mailer: Mailer) {}
  async send(listingId: string, body: string): Promise<void> {
    await this.mailer.send({ subject: `Listing ${listingId}`, body });
  }
}

class SmsChannel implements NotificationChannel {
  readonly key = 'sms';
  constructor(private readonly gateway: SmsGateway) {}
  async send(listingId: string, body: string): Promise<void> {
    await this.gateway.push({ text: body.slice(0, 160) });
  }
}

class Notifier {
  private readonly channels = new Map<string, NotificationChannel>();

  register(channel: NotificationChannel): this {
    this.channels.set(channel.key, channel);
    return this;
  }

  async notify(key: string, listingId: string, body: string): Promise<void> {
    const channel = this.channels.get(key);
    if (!channel) throw new Error(`unknown channel ${key}`);
    await channel.send(listingId, body);
  }
}
```

Honest caveat: the second version is more code. If you will only ever have email, the first version
is better. Open/Closed is a bet on a specific axis of change — take the bet only when you have
reason to believe the change is coming.

### L — Liskov Substitution Principle

> A subtype must be usable anywhere its base type is expected, without the caller noticing.

Patterns lean on this constantly — every time a client holds an interface reference and a concrete
implementation is swapped in, LSP is what makes it safe. Patterns most directly about honouring it:

- **[Decorator](../02-structural/04-decorator.md)** and **[Proxy](../02-structural/07-proxy.md)** —
  both *require* perfect substitutability. A decorator that changes the contract (throws where the
  wrapped object returned, returns null where it returned a value) is a bug, not a decoration.
- **[Adapter](../02-structural/01-adapter.md)** — the adapter must honour the target's contract, not
  just its method signatures. Mapping an upstream's 404 into an empty list when the target contract
  promises "throws NotFound" is an LSP violation hiding as an adapter.
- **[Template Method](../03-behavioral/09-template-method.md)** — the base class documents what a
  hook is allowed to do; a subclass that violates the invariant breaks every caller of the base.

```csharp
// LSP-safe Decorator: same contract, plus caching. Callers cannot tell.
public interface IListingRepository
{
    Task<Listing?> GetAsync(string id, CancellationToken ct);
}

public sealed class CachingListingRepository : IListingRepository
{
    private readonly IListingRepository _inner;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _ttl;

    public CachingListingRepository(IListingRepository inner, IMemoryCache cache, TimeSpan ttl)
    {
        _inner = inner;
        _cache = cache;
        _ttl = ttl;
    }

    public async Task<Listing?> GetAsync(string id, CancellationToken ct)
    {
        if (_cache.TryGetValue<Listing?>(id, out var cached))
            return cached;

        var listing = await _inner.GetAsync(id, ct);   // same exceptions propagate
        _cache.Set(id, listing, _ttl);
        return listing;
    }
}
```

The decorator adds a cache and nothing else observable: same return type, same null semantics, same
exceptions escaping. That is what "substitutable" means in practice.

### I — Interface Segregation Principle

> Clients should not be forced to depend on methods they do not use. Many small interfaces beat one
> fat one.

- **[Adapter](../02-structural/01-adapter.md)** — the target interface is usually a *narrow* one
  shaped to what the client actually needs, not a mirror of the upstream's full surface.
- **[Facade](../02-structural/05-facade.md)** — arguably the pattern-scale version of ISP: expose
  the three operations callers need, not the subsystem's forty.
- **[Strategy](../03-behavioral/08-strategy.md)** — a strategy interface with one method is ISP in
  its purest form.
- **[Bridge](../02-structural/02-bridge.md)** — the implementor interface is deliberately kept
  primitive and small; the abstraction side builds richer operations out of it.

```typescript
// Fat interface: a search page is forced to know about writes it will never do.
interface ListingStore {
  get(id: string): Promise<Listing | null>;
  search(q: Query): Promise<Listing[]>;
  create(l: NewListing): Promise<string>;
  update(id: string, patch: Partial<Listing>): Promise<void>;
  archive(id: string): Promise<void>;
  reindex(): Promise<void>;
}

// Segregated: each consumer depends only on what it calls.
interface ListingReader {
  get(id: string): Promise<Listing | null>;
  search(q: Query): Promise<Listing[]>;
}

interface ListingWriter {
  create(l: NewListing): Promise<string>;
  update(id: string, patch: Partial<Listing>): Promise<void>;
  archive(id: string): Promise<void>;
}

interface ListingMaintenance {
  reindex(): Promise<void>;
}
```

One class can still implement all three. The point is what the *consumers* are typed against — and
as a side effect, your test doubles shrink from six stubbed methods to two.

### D — Dependency Inversion Principle

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

This is the principle the whole creational group exists to serve, plus a good chunk of the
structural one.

- **[Factory Method](../01-creational/01-factory-method.md)** and
  **[Abstract Factory](../01-creational/02-abstract-factory.md)** — the point is precisely that the
  caller stops naming concrete classes.
- **[Builder](../01-creational/03-builder.md)** — callers describe *what* they want, not which
  constructor overload to call.
- **[Bridge](../02-structural/02-bridge.md)** — the abstraction depends on an implementor interface,
  never on a concrete implementor.
- **[Mediator](../03-behavioral/04-mediator.md)** — components depend on the mediator abstraction
  instead of on each other.
- **[Observer](../03-behavioral/06-observer.md)** — the publisher depends on a subscriber
  *interface*, and therefore on nothing concrete at all.

If you already use a DI container — `IServiceCollection` in .NET, or hand-rolled constructor
injection in a Node service — you are doing Dependency Inversion already, and the container is doing
Abstract Factory work on your behalf:

```csharp
// Program.cs / composition root. This is the ONLY place that names concrete types.
builder.Services.AddSingleton<ITaxPolicy, IndiaGstTaxPolicy>();
builder.Services.AddScoped<IListingRepository, SqlListingRepository>();
builder.Services.Decorate<IListingRepository, CachingListingRepository>();  // Decorator
builder.Services.AddScoped<IOrderNotifier, RabbitMqOrderNotifier>();        // Adapter over a broker
builder.Services.AddScoped<OrderProcessor>();
```

Everything downstream of that file depends only on interfaces. That is Dependency Inversion, and
the individual registrations are patterns from this catalogue doing the work.

### The one-screen summary

```
  PRINCIPLE                      PATTERNS THAT ARE ITS WORKED EXAMPLES
  -------------------------------------------------------------------------------
  S  Single Responsibility  ->   Strategy, Command, Visitor, Memento, Builder,
                                 Facade
  O  Open/Closed            ->   Strategy, Decorator, Chain of Responsibility,
                                 Factory Method, Abstract Factory, Observer,
                                 State, Visitor
  L  Liskov Substitution    ->   Decorator, Proxy, Adapter, Template Method
                                 (these *depend* on LSP being honoured)
  I  Interface Segregation  ->   Adapter, Facade, Strategy, Bridge
  D  Dependency Inversion   ->   Factory Method, Abstract Factory, Builder,
                                 Bridge, Mediator, Observer
```

Two older maxims sit underneath all of this, and they predate SOLID:

1. **Program to an interface, not an implementation.** Depend on what a thing *does*, not on what it
   *is*.
2. **Favour object composition over class inheritance.** Inheritance is fixed at compile time and
   binds you to a parent's internals; composition is rewireable at runtime. Roughly half the
   catalogue is "the composition version of something people used to do with inheritance" —
   Decorator instead of a subclass per feature combination, Strategy instead of a subclass per
   algorithm, Bridge instead of an N×M subclass explosion.

If you internalise those two sentences you will rediscover a good fraction of the 23 on your own.

---

## 🙂 Now the plain-English version

Skip the jargon for a second.

Imagine you have built a lot of kitchens. After the twentieth one, you notice things. The bin should
be under the chopping board, because that is where the peelings fall. The knives should be within
reach of the board but not above where a child can grab them. The fridge, sink, and hob should form
a triangle so you are not walking laps while cooking.

None of that is a kitchen. You cannot buy "the work triangle" at a shop and screw it to the wall.
But if you tell another kitchen fitter "keep the work triangle tight", they know exactly what you
mean, they know why it matters, and they know it will look different in a narrow galley kitchen than
in a big square one.

**That is a pattern.** A piece of hard-won experience, boiled down, given a name, general enough to
reapply, specific enough to be useful.

Software has the same thing. After enough programs, people noticed the same arrangements keep
working. "When the thing that varies is *which* calculation runs, put the calculation behind a small
interface and hand it in from outside" — that got the name Strategy. "When several parts of the
system need to know that something happened, but the part where it happened should not need a list
of them, let them subscribe" — that got the name Observer.

You will probably discover half of these on your own, by being annoyed enough times. Reading the
catalogue just means you get the names — and therefore the conversations — earlier.

And the most important plain-English rule: **do not go looking for places to use them.** Write the
simple thing first. When it starts to hurt in a specific way, come back to the catalogue and find
the shape that matches the hurt. Patterns are a diagnosis manual, not a shopping list.

---

## 📌 Quick reference card

```
  WHAT IT IS       A named, reusable arrangement of objects that solves a
                   recurring design problem and makes one axis of change cheap.

  WHAT IT IS NOT   Not an algorithm (recipe vs blueprint).
                   Not a library (nothing to install).
                   Not copy-paste code (the shape transfers, not the tokens).
                   Not a goal (indirection is a cost; buy it deliberately).

  WHERE FROM       Christopher Alexander, architecture -> "A Pattern Language".
                   1994, Gamma / Helm / Johnson / Vlissides ("Gang of Four"),
                   "Design Patterns: Elements of Reusable Object-Oriented
                   Software". 23 patterns. Examples in C++ and Smalltalk.

  WHY LEARN        Tested solutions + shared vocabulary. The vocabulary is the
                   part that pays off on day one.

  THE 3 GROUPS     Creational  (5)  = how objects get MADE
                   Structural  (7)  = how objects are COMPOSED
                   Behavioral (11)  = how objects TALK and split responsibility

  THE 3 LEVELS     Idiom          -> a few lines, one language (RAII, IDisposable)
                   Design pattern -> a few classes, any OO language (the GoF 23)
                   Architectural  -> the whole system (Layered, Hexagonal, CQRS,
                                     Event-Driven, Saga)

  THE 2 MAXIMS     Program to an interface, not an implementation.
                   Favour object composition over class inheritance.
```

---

## ➡️ Where to go next

- **[SOLID and other principles](../04-extras/01-solid-and-principles.md)** — the principles that
  the whole catalogue is built to serve, in depth.
- **[Strategy](../03-behavioral/08-strategy.md)** — the best first pattern. Small, obvious, and you
  will use it this week.
- **[Observer](../03-behavioral/06-observer.md)** — the second best first pattern, and the bridge to
  every message-broker system you already run.
- **[Decorator](../02-structural/04-decorator.md)** — once this clicks, the whole structural group
  clicks.
- **[Factory Method](../01-creational/01-factory-method.md)** — the gentlest entry into the
  creational group.
- **[Singleton](../01-creational/05-singleton.md)** — read it for the critique as much as for the
  pattern.

Read a chapter, then go and find the shape in code you already own. Recognition is the skill;
the catalogue is just the flashcards.
