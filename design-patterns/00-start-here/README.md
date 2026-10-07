# 🧭 Design Patterns — Start Here

Welcome. This folder is a complete, self-contained study kit for the 23 Gang of Four design patterns,
written for someone who ships real backend code rather than someone studying for a university exam.
Every pattern gets its own file, every file follows the same seven-part shape, and every code sample is
a complete program you can paste into a `.ts` or `.cs` file and run. Nothing here needs the internet —
the images live in `assets/`, the text lives in these files, and the links between them are relative.
Read at your own pace. The patterns are not a checklist to memorise; they are names for shapes you have
almost certainly already built by accident, and naming them is most of the benefit.

---

## 📁 How this folder is laid out

```
design-patterns/
├── 00-start-here/          ← you are here. Index, study order, lookup tables.
│   └── README.md
├── 01-creational/          ← 5 patterns about *making* objects
│   ├── 01-factory-method.md
│   ├── 02-abstract-factory.md
│   ├── 03-builder.md
│   ├── 04-prototype.md
│   └── 05-singleton.md
├── 02-structural/          ← 7 patterns about *composing* objects into bigger things
│   ├── 01-adapter.md
│   ├── 02-bridge.md
│   ├── 03-composite.md
│   ├── 04-decorator.md
│   ├── 05-facade.md
│   ├── 06-flyweight.md
│   └── 07-proxy.md
├── 03-behavioral/          ← 10 patterns about *how objects talk and share work*
│   ├── 01-chain-of-responsibility.md
│   ├── 02-command.md
│   ├── 03-iterator.md
│   ├── 04-mediator.md
│   ├── 05-memento.md
│   ├── 06-observer.md
│   ├── 07-state.md
│   ├── 08-strategy.md
│   ├── 09-template-method.md
│   └── 10-visitor.md
├── 04-extras/              ← the glue: principles, comparisons, the 23rd pattern
│   ├── 01-solid-and-principles.md
│   ├── 02-pattern-cheatsheet.md
│   ├── 03-antipatterns.md
│   ├── 04-patterns-in-dotnet-and-node.md
│   └── 05-interpreter-bonus.md
└── assets/                 ← every diagram and illustration, stored locally
```

**File naming:** every pattern file is `NN-slug.md`. The `NN` is the order the pattern appears in the
Refactoring.Guru catalog inside its category, *not* the order you should study it in. The study order is
below and it is deliberately different.

**Linking:** from `00-start-here/` a pattern file is `../01-creational/01-factory-method.md`.
From `04-extras/` a pattern file is `../03-behavioral/08-strategy.md`. Straightforward relative paths,
no build step, no site generator — it all works in VS Code's markdown preview, on GitHub, or in any
plain editor.

---

## 🧩 The 7-part structure every pattern file follows

Once you have read one file you have read them all, structurally. Here is the shape, so you know where to
jump when you only need one thing:

| Part | What it is | When you want it |
|---|---|---|
| **PART 1 — The pattern** | Intent, problem, solution, real-world analogy, structure diagram, pseudocode, applicability, how-to-implement, pros & cons, relations to other patterns — each followed by a plain-English restatement in my own words | First read, and whenever you want the canonical definition |
| **PART 2 — The official code** | The Refactoring.Guru reference implementation in **C#**, **TypeScript**, **C++** and **Java**, complete and runnable, with output | When you want to see the shape in a language you know, and compare how it changes across languages |
| **PART 3 — Learn by building** | An original, deliberately small example built from nothing, step by step, with the "before" code that hurt and the "after" code that does not | When the official example feels abstract |
| **PART 4 — In a real codebase** | How the pattern shows up in a TypeScript / C# service backed by SQL and RabbitMQ — repositories, handlers, consumers, DI registration | When you want to actually use it at work tomorrow |
| **PART 5 — Anti-patterns & when NOT to use it** | The ways this pattern goes wrong, the smell of over-application, and the simpler thing you should have written instead | Before you reach for it in a PR |
| **PART 6 — Famous real-world uses** | Where this pattern lives in .NET, Node, the DOM, ASP.NET Core, EF Core, RabbitMQ clients, and other code you already depend on | When you want to believe it is not academic |
| **PART 7 — Remember it** | A mnemonic, interview questions with model answers, and a short self-test with answers hidden below | The night before an interview, or a week later as revision |

A small map of how a file reads:

```
   ┌──────────────────────────────────────────────────────┐
   │ PART 1  theory        →  "what the book says"        │
   │ PART 2  official code →  "what it looks like x4"     │
   │ PART 3  build it      →  "what it feels like to need"│
   │ PART 4  at work       →  "where it goes in my repo"  │
   │ PART 5  don't         →  "how it hurts"              │
   │ PART 6  in the wild   →  "who already did it"        │
   │ PART 7  recall        →  "can I explain it out loud" │
   └──────────────────────────────────────────────────────┘
        theory ──────────────────────────────► practice
```

---

## 🎯 How to study this — the suggested order

Not alphabetical, not book order. This is ordered by **how soon a C#/TypeScript backend engineer will
actually use it**, and how much each one unlocks the next. The first six will change code you write this
month. The last six are worth knowing so you recognise them when you read someone else's code.

| # | Pattern | Why it sits here |
|---|---|---|
| 1 | [Strategy](../03-behavioral/08-strategy.md) | The single highest-value pattern for backend work: it is "pass a function/interface instead of writing an `if`", and it makes the other behavioral patterns obvious. |
| 2 | [Factory Method](../01-creational/01-factory-method.md) | Once you swap behaviour you immediately need to *choose* which implementation to build — this is that, and it is the spine of every DI container you already use. |
| 3 | [Observer](../03-behavioral/06-observer.md) | You are already doing it: C# `event`, Node `EventEmitter`, RxJS, and every RabbitMQ fanout exchange is this pattern with a network in the middle. |
| 4 | [Adapter](../02-structural/01-adapter.md) | The pattern you write most often without naming it — wrapping a third-party SDK, a legacy API, or a DTO shape you do not control. |
| 5 | [Decorator](../02-structural/04-decorator.md) | Caching, retry, logging, metrics, auth. ASP.NET Core middleware and Express middleware are decorators in a trench coat. |
| 6 | [Singleton](../01-creational/05-singleton.md) | Learn it early precisely so you learn *why you almost never write it by hand* — your DI container's singleton lifetime is the correct version. |
| 7 | [Builder](../01-creational/03-builder.md) | The cure for eleven-parameter constructors and half-initialised objects; also how every fluent API you like is shaped. |
| 8 | [Command](../03-behavioral/02-command.md) | Turns "do the thing" into an object you can queue, log, retry, and publish — which is exactly what a RabbitMQ message is. |
| 9 | [Template Method](../03-behavioral/09-template-method.md) | The inheritance-flavoured cousin of Strategy; you will meet it in base classes for jobs, importers, and controllers. |
| 10 | [Facade](../02-structural/05-facade.md) | How you stop a messy subsystem leaking into forty call sites; the honest name for most of your `Service` classes. |
| 11 | [State](../03-behavioral/07-state.md) | Listings, orders, leads and vehicles all have lifecycles; this is how you stop modelling them with a `switch` on a string column. |
| 12 | [Proxy](../02-structural/07-proxy.md) | Lazy loading, access control, and remote calls — EF Core lazy proxies and every generated gRPC/HTTP client are this. |
| 13 | [Composite](../02-structural/03-composite.md) | The moment you have categories inside categories, or filter groups inside filter groups, this saves you from recursion spaghetti. |
| 14 | [Iterator](../03-behavioral/03-iterator.md) | You consume it daily (`IEnumerable`, `yield return`, generators); learn to *implement* it and streaming/pagination gets much cleaner. |
| 15 | [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | Validation pipelines, request pipelines, approval flows — and the honest description of ASP.NET Core's middleware chain. |
| 16 | [Mediator](../03-behavioral/04-mediator.md) | Once several components talk to each other directly, this is the untangling move; MediatR made it a default in .NET shops. |
| 17 | [Abstract Factory](../01-creational/02-abstract-factory.md) | Factory Method's big sibling, for *families* of related objects — needed less often, so learn it after the simpler one. |
| 18 | [Bridge](../02-structural/02-bridge.md) | Rare but powerful: it is what you reach for when you are about to create a class-explosion from two independent axes of variation. |
| 19 | [Memento](../03-behavioral/05-memento.md) | Undo, snapshots, and "restore the draft" features; specialised, but nothing else does the job as cleanly. |
| 20 | [Flyweight](../02-structural/06-flyweight.md) | Pure memory optimisation. Learn it late, apply it rarely, and only with a profiler open. |
| 21 | [Prototype](../01-creational/04-prototype.md) | Cloning. It matters in C++ and in deep-copy-heavy code; in C#/TS you often reach for records, spread syntax or serialisation instead. |
| 22 | [Visitor](../03-behavioral/10-visitor.md) | The hardest to love and the easiest to misuse — but if you ever write a parser, linter or AST walker, it is the right answer. |

> **Where is the 23rd?** The Refactoring.Guru catalog documents 22 of the 23 GoF patterns — **Interpreter**
> is not on the site. It is covered separately, in my own words, in
> [../04-extras/05-interpreter-bonus.md](../04-extras/05-interpreter-bonus.md).

### Suggested pace

```
Week 1   Strategy, Factory Method, Observer          ← the three that change your daily code
Week 2   Adapter, Decorator, Singleton, Builder      ← the "I've been doing this already" week
Week 3   Command, Template Method, Facade, State     ← the ones that shape whole services
Week 4   Proxy, Composite, Iterator, CoR             ← pipelines, trees and streams
Week 5   Mediator, Abstract Factory, Bridge          ← decoupling at a larger scale
Week 6   Memento, Flyweight, Prototype, Visitor      ← the specialists
Any time ../04-extras/01-solid-and-principles.md     ← read this alongside week 1
```

Do not read a file straight through in one sitting. Read **PART 1** and **PART 3** first, write the
PART 3 example yourself without looking, then come back for PART 4 when you hit the problem at work.
PART 2's four-language code is a reference, not homework.

---

## 🏗️ Creational patterns — making objects

These five are about the moment an object comes into existence: who decides the concrete type, how the
arguments get supplied, and how many instances exist.

| Pattern | One-line intent | Link |
|---|---|---|
| Factory Method | Define an interface for creating an object, but let subclasses (or a lookup) decide which concrete class to instantiate. | [../01-creational/01-factory-method.md](../01-creational/01-factory-method.md) |
| Abstract Factory | Create whole *families* of related objects without naming their concrete classes, so the family stays internally consistent. | [../01-creational/02-abstract-factory.md](../01-creational/02-abstract-factory.md) |
| Builder | Construct a complex object step by step, so the same construction process can produce different representations — and no object is ever half-built. | [../01-creational/03-builder.md](../01-creational/03-builder.md) |
| Prototype | Create new objects by copying an existing instance rather than by calling a constructor. | [../01-creational/04-prototype.md](../01-creational/04-prototype.md) |
| Singleton | Ensure a class has exactly one instance and give the whole program a global access point to it. | [../01-creational/05-singleton.md](../01-creational/05-singleton.md) |

## 🧱 Structural patterns — composing objects

These seven are about wiring objects together into larger structures while keeping the structure
flexible and the seams clean.

| Pattern | One-line intent | Link |
|---|---|---|
| Adapter | Convert one interface into another so two classes that were never designed to meet can work together. | [../02-structural/01-adapter.md](../02-structural/01-adapter.md) |
| Bridge | Split one class into an abstraction and an implementation that can vary independently, instead of multiplying subclasses. | [../02-structural/02-bridge.md](../02-structural/02-bridge.md) |
| Composite | Treat individual objects and groups of objects through the same interface, so client code can ignore the difference. | [../02-structural/03-composite.md](../02-structural/03-composite.md) |
| Decorator | Attach extra behaviour to an object at runtime by wrapping it in another object with the same interface. | [../02-structural/04-decorator.md](../02-structural/04-decorator.md) |
| Facade | Put one simple front door on a complicated subsystem so callers need to know almost nothing about it. | [../02-structural/05-facade.md](../02-structural/05-facade.md) |
| Flyweight | Share the parts of state that are identical across many objects so millions of them fit in memory. | [../02-structural/06-flyweight.md](../02-structural/06-flyweight.md) |
| Proxy | Stand in for another object to control access to it — lazily, remotely, or with permission checks. | [../02-structural/07-proxy.md](../02-structural/07-proxy.md) |

## 🔀 Behavioral patterns — how objects talk

These ten are about responsibility and communication: who does what, and how work flows between objects.

| Pattern | One-line intent | Link |
|---|---|---|
| Chain of Responsibility | Pass a request along a chain of handlers until one of them deals with it. | [../03-behavioral/01-chain-of-responsibility.md](../03-behavioral/01-chain-of-responsibility.md) |
| Command | Turn a request into a standalone object so it can be passed around, queued, logged and undone. | [../03-behavioral/02-command.md](../03-behavioral/02-command.md) |
| Iterator | Walk through the elements of a collection without exposing how the collection is stored. | [../03-behavioral/03-iterator.md](../03-behavioral/03-iterator.md) |
| Mediator | Make components talk through one hub instead of talking to each other directly. | [../03-behavioral/04-mediator.md](../03-behavioral/04-mediator.md) |
| Memento | Capture and restore an object's internal state without breaking its encapsulation. | [../03-behavioral/05-memento.md](../03-behavioral/05-memento.md) |
| Observer | Let many objects subscribe to, and be notified about, events happening in another object. | [../03-behavioral/06-observer.md](../03-behavioral/06-observer.md) |
| State | Let an object change its behaviour when its internal state changes, as if it changed class. | [../03-behavioral/07-state.md](../03-behavioral/07-state.md) |
| Strategy | Define a family of interchangeable algorithms, encapsulate each one, and swap them at runtime. | [../03-behavioral/08-strategy.md](../03-behavioral/08-strategy.md) |
| Template Method | Put the skeleton of an algorithm in a base class and let subclasses fill in specific steps. | [../03-behavioral/09-template-method.md](../03-behavioral/09-template-method.md) |
| Visitor | Separate an algorithm from the object structure it operates on, so you can add operations without touching the classes. | [../03-behavioral/10-visitor.md](../03-behavioral/10-visitor.md) |

---

## ⭐ If you only learn five, learn these

Realistically, five patterns cover most of the value. Here they are, with the smallest honest example of
each so you can feel the shape immediately.

### 1. Strategy — the one that replaces `if/else` about *behaviour*

```typescript
// Pricing rules differ per channel. Don't branch — inject.

export interface PricingStrategy {
  readonly name: string;
  finalPrice(listPrice: number): number;
}

export class DealerPricing implements PricingStrategy {
  readonly name = 'dealer';
  finalPrice(listPrice: number): number {
    return Math.round(listPrice * 0.95);      // 5% dealer margin
  }
}

export class CertifiedPricing implements PricingStrategy {
  readonly name = 'certified';
  constructor(private readonly warrantyFee: number) {}
  finalPrice(listPrice: number): number {
    return Math.round(listPrice * 1.02) + this.warrantyFee;
  }
}

export class AuctionPricing implements PricingStrategy {
  readonly name = 'auction';
  finalPrice(listPrice: number): number {
    return Math.round(listPrice * 0.88);
  }
}

export class ListingPricer {
  constructor(private strategy: PricingStrategy) {}
  use(strategy: PricingStrategy): void { this.strategy = strategy; }
  quote(listPrice: number): string {
    return `${this.strategy.name}: ₹${this.strategy.finalPrice(listPrice).toLocaleString('en-IN')}`;
  }
}

const pricer = new ListingPricer(new DealerPricing());
console.log(pricer.quote(850_000));            // dealer: ₹807,500
pricer.use(new CertifiedPricing(12_000));
console.log(pricer.quote(850_000));            // certified: ₹879,000
pricer.use(new AuctionPricing());
console.log(pricer.quote(850_000));            // auction: ₹748,000
```

**Plain English:** instead of one function that knows every case, you have one small object per case and
a pointer to whichever one you want right now. Adding a case means adding a file, not editing a `switch`.

Full file: [../03-behavioral/08-strategy.md](../03-behavioral/08-strategy.md)

### 2. Factory Method — the one that decides *which* thing to build

```csharp
using System;
using System.Collections.Generic;

public interface IImageResizer
{
    string Resize(string path, int width);
}

public sealed class JpegResizer : IImageResizer
{
    public string Resize(string path, int width) => $"{path} -> jpeg @ {width}px (quality 82)";
}

public sealed class WebpResizer : IImageResizer
{
    public string Resize(string path, int width) => $"{path} -> webp @ {width}px (lossless off)";
}

public sealed class AvifResizer : IImageResizer
{
    public string Resize(string path, int width) => $"{path} -> avif @ {width}px (speed 6)";
}

public interface IResizerFactory
{
    IImageResizer Create(string format);
}

public sealed class ResizerFactory : IResizerFactory
{
    private readonly Dictionary<string, Func<IImageResizer>> _builders =
        new(StringComparer.OrdinalIgnoreCase)
        {
            ["jpeg"] = () => new JpegResizer(),
            ["webp"] = () => new WebpResizer(),
            ["avif"] = () => new AvifResizer(),
        };

    public IImageResizer Create(string format)
    {
        if (!_builders.TryGetValue(format, out var build))
            throw new NotSupportedException($"No resizer registered for '{format}'.");
        return build();
    }
}

public static class Demo
{
    public static void Main()
    {
        IResizerFactory factory = new ResizerFactory();
        foreach (var format in new[] { "jpeg", "webp", "avif" })
            Console.WriteLine(factory.Create(format).Resize("/photos/car-9931.png", 1280));
    }
}
```

**Plain English:** the calling code says *what kind* it wants as data, and one place turns that data into
a concrete object. The rest of your code only ever touches the interface.

Full file: [../01-creational/01-factory-method.md](../01-creational/01-factory-method.md)

### 3. Observer — the one you already use as `event` and `EventEmitter`

```typescript
type Listener<T> = (payload: T) => void;

export class EventChannel<T> {
  private listeners = new Set<Listener<T>>();

  subscribe(listener: Listener<T>): () => void {
    this.listeners.add(listener);
    return () => { this.listeners.delete(listener); };   // unsubscribe handle
  }

  publish(payload: T): void {
    for (const listener of [...this.listeners]) {
      try { listener(payload); }
      catch (err) { console.error('listener failed', err); }   // one bad subscriber must not kill the rest
    }
  }

  get count(): number { return this.listeners.size; }
}

interface ListingPublished { listingId: number; dealerId: number; price: number; }

const published = new EventChannel<ListingPublished>();

const offSearch = published.subscribe(e => console.log(`[search] index listing ${e.listingId}`));
published.subscribe(e => console.log(`[email] notify dealer ${e.dealerId}`));
published.subscribe(e => console.log(`[bus] publish listing.published for ${e.listingId}`));

published.publish({ listingId: 9931, dealerId: 77, price: 850_000 });
offSearch();                                   // search service goes away
published.publish({ listingId: 9932, dealerId: 77, price: 640_000 });
```

**Plain English:** the thing that knows something happened does not need to know who cares. It shouts;
whoever registered hears. Swap the in-process `Set` for a RabbitMQ fanout exchange and it is the same
pattern across a network.

Full file: [../03-behavioral/06-observer.md](../03-behavioral/06-observer.md)

### 4. Adapter — the one that makes a foreign shape fit yours

```typescript
// What our code wants to depend on:
export interface SmsSender {
  send(toE164: string, body: string): Promise<{ id: string }>;
}

// What the vendor SDK actually gives us (we don't control this):
class VendorGateway {
  dispatch(req: { msisdn: string; text: string; senderId: string }): Promise<{ refNo: number; ok: boolean }> {
    return Promise.resolve({ refNo: Math.floor(Math.random() * 1e9), ok: true });
  }
}

// The adapter: one small class, all the ugliness in one place.
export class VendorSmsAdapter implements SmsSender {
  constructor(
    private readonly gateway: VendorGateway,
    private readonly senderId: string,
  ) {}

  async send(toE164: string, body: string): Promise<{ id: string }> {
    const msisdn = toE164.replace(/^\+/, '');        // vendor wants no plus sign
    const res = await this.gateway.dispatch({ msisdn, text: body, senderId: this.senderId });
    if (!res.ok) throw new Error('SMS gateway rejected the message');
    return { id: String(res.refNo) };
  }
}

async function main() {
  const sender: SmsSender = new VendorSmsAdapter(new VendorGateway(), 'CARMKT');
  const { id } = await sender.send('+919812345678', 'Your test drive is confirmed.');
  console.log('sent', id);
}
main();
```

**Plain English:** you write the interface you wish the vendor had, then write one translator class.
When the vendor changes, or you switch vendors, exactly one file changes.

Full file: [../02-structural/01-adapter.md](../02-structural/01-adapter.md)

### 5. Decorator — the one that adds behaviour without editing the original

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public interface IVehicleRepository
{
    Task<string?> GetAsync(int id);
}

public sealed class SqlVehicleRepository : IVehicleRepository
{
    public async Task<string?> GetAsync(int id)
    {
        await Task.Delay(40);                       // pretend this is a real query
        return $"vehicle-{id}";
    }
}

// Decorator 1: caching.
public sealed class CachingVehicleRepository : IVehicleRepository
{
    private readonly IVehicleRepository _inner;
    private readonly Dictionary<int, string?> _cache = new();

    public CachingVehicleRepository(IVehicleRepository inner) => _inner = inner;

    public async Task<string?> GetAsync(int id)
    {
        if (_cache.TryGetValue(id, out var hit))
        {
            Console.WriteLine($"cache hit {id}");
            return hit;
        }
        var value = await _inner.GetAsync(id);
        _cache[id] = value;
        return value;
    }
}

// Decorator 2: timing.
public sealed class TimedVehicleRepository : IVehicleRepository
{
    private readonly IVehicleRepository _inner;
    public TimedVehicleRepository(IVehicleRepository inner) => _inner = inner;

    public async Task<string?> GetAsync(int id)
    {
        var started = DateTime.UtcNow;
        try { return await _inner.GetAsync(id); }
        finally { Console.WriteLine($"GetAsync({id}) took {(DateTime.UtcNow - started).TotalMilliseconds:F0}ms"); }
    }
}

public static class Demo
{
    public static async Task Main()
    {
        IVehicleRepository repo =
            new TimedVehicleRepository(
                new CachingVehicleRepository(
                    new SqlVehicleRepository()));

        Console.WriteLine(await repo.GetAsync(9931));
        Console.WriteLine(await repo.GetAsync(9931));   // second call is a cache hit
    }
}
```

The wrapping, drawn:

```
   caller
     │
     ▼
  ┌──────────────────────────┐
  │ TimedVehicleRepository   │  ← measures
  │  ┌─────────────────────┐ │
  │  │ CachingVehicleRepo  │ │  ← short-circuits
  │  │  ┌────────────────┐ │ │
  │  │  │ SqlVehicleRepo │ │ │  ← does the real work
  │  │  └────────────────┘ │ │
  │  └─────────────────────┘ │
  └──────────────────────────┘
```

**Plain English:** each layer implements the same interface, holds the next layer, does its own bit, and
delegates. You can reorder or remove layers without touching the class that does the real work.

Full file: [../02-structural/04-decorator.md](../02-structural/04-decorator.md)

---

## 🔎 Which pattern do I need?

Find your complaint in the left column. This is deliberately phrased the way the problem actually feels
at 4pm on a Thursday, not the way a textbook phrases it.

| My complaint | Probably want | Why |
|---|---|---|
| "My constructor has 11 parameters and half of them are optional." | [Builder](../01-creational/03-builder.md) | Named steps instead of positional arguments, and the object is only handed over when it is complete. |
| "I have a `switch` on a string that picks which calculation to run, and it keeps growing." | [Strategy](../03-behavioral/08-strategy.md) | Each branch becomes its own class; adding a case stops touching existing code. |
| "I have a `switch` on a *status column* and every method has the same switch." | [State](../03-behavioral/07-state.md) | The lifecycle becomes objects; each state knows its own allowed transitions. |
| "Every time we add a payment provider I edit five `if` blocks to build the right client." | [Factory Method](../01-creational/01-factory-method.md) | One registration point decides the concrete type; the rest of the code sees the interface. |
| "We need a *matched set* of objects — same theme, same region, same tenant — and they keep getting mixed up." | [Abstract Factory](../01-creational/02-abstract-factory.md) | One factory produces a whole family, so incompatible combinations become impossible. |
| "This third-party SDK has a horrible interface and it's now imported in 30 files." | [Adapter](../02-structural/01-adapter.md) | Wrap it once behind the interface you wish it had; 30 files depend on your shape instead. |
| "I want logging, retries, caching and metrics around this service, but the class is already 400 lines." | [Decorator](../02-structural/04-decorator.md) | Each concern becomes its own wrapper, composed at registration time. |
| "Calling this subsystem takes six steps in exactly the right order and everyone gets it wrong." | [Facade](../02-structural/05-facade.md) | One method that does the six steps correctly; callers stop knowing the order. |
| "Twelve components all need to know when a listing is published." | [Observer](../03-behavioral/06-observer.md) | Publish once, subscribe many; the publisher never learns who is listening. |
| "Every component talks to every other component and the dependency graph looks like wool." | [Mediator](../03-behavioral/04-mediator.md) | They all talk to one hub instead; N×N edges become N. |
| "The user wants an undo button." | [Memento](../03-behavioral/05-memento.md) | Snapshot state into an opaque object, restore it later, without exposing internals. |
| "I need to queue this operation, retry it, and log exactly what was requested." | [Command](../03-behavioral/02-command.md) | The request becomes a serialisable object — which is also exactly what you put on a RabbitMQ queue. |
| "Validation runs as one giant method and I can't reorder or skip a rule." | [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | Each rule is a link; the chain is assembled as data. |
| "Three importers are 90% identical and 10% different, and the 90% keeps drifting apart." | [Template Method](../03-behavioral/09-template-method.md) | The shared skeleton lives once in a base class; only the different steps are overridden. |
| "Loading this entity always drags in a 4MB blob we usually don't read." | [Proxy](../02-structural/07-proxy.md) | A stand-in object defers the expensive load until someone actually touches it. |
| "Categories contain categories contain categories, and my code is full of `if (isLeaf)`." | [Composite](../02-structural/03-composite.md) | Leaf and branch share one interface; the recursion stops being special-cased. |
| "I want to loop over this without materialising the whole result set in memory." | [Iterator](../03-behavioral/03-iterator.md) | `yield return` / generators hand out elements one at a time, hiding paging or streaming. |
| "We hold two million tiny objects and the process keeps hitting memory limits." | [Flyweight](../02-structural/06-flyweight.md) | Shared, immutable state is stored once; only the per-object differences are kept per instance. |
| "Building this object from scratch is expensive but I need many near-identical copies." | [Prototype](../01-creational/04-prototype.md) | Clone a configured instance instead of re-running the expensive setup. |
| "Config is loaded from disk and must be loaded exactly once for the whole process." | [Singleton](../01-creational/05-singleton.md) | One instance, one access point — but read PART 5 of that file first: your DI container usually does it better. |
| "Adding a new report over our object tree means editing every node class again." | [Visitor](../03-behavioral/10-visitor.md) | New operations become new visitor classes; the node classes stop changing. |
| "We have 3 renderers × 4 device types and I'm about to write 12 subclasses." | [Bridge](../02-structural/02-bridge.md) | Split the two axes into abstraction and implementation; 3 + 4 classes instead of 3 × 4. |
| "I need to evaluate user-supplied filter expressions like `price < 5L AND fuel = diesel`." | [Interpreter](../04-extras/05-interpreter-bonus.md) | A tiny grammar with one class per rule; the bonus file covers it since the catalog does not. |
| "My tests can't fake this because the class news-up its own dependencies." | [Factory Method](../01-creational/01-factory-method.md) + [Strategy](../03-behavioral/08-strategy.md) | Push creation and behaviour choice out to the edges so tests can inject doubles. |

If two rows fit, read both files' **PART 5** sections — the "when NOT to use it" part usually settles it
faster than the theory does.

---

## 🖼️ Images and offline use

Every diagram and illustration referenced by these files is stored **locally in `assets/`**. Nothing in
this folder loads an image over the network, so the whole thing works on a plane, on a train, behind a
corporate proxy, or in a repo with no internet access at all. Images are referenced with relative paths
like `../assets/decorator-structure.png` from a pattern file, or `assets/decorator-structure.png` from
this one. If an image ever fails to render, the surrounding text and the ASCII diagram beside it are
written to stand on their own — you should never *need* the picture to follow the explanation.

---

## 🔗 Related reading in this folder

- [../04-extras/01-solid-and-principles.md](../04-extras/01-solid-and-principles.md) — SOLID, composition
  over inheritance, program-to-an-interface, and the design principles that make the patterns feel
  inevitable rather than arbitrary. **Read this alongside week 1.**
- [../04-extras/04-interview-cheatsheet.md](../04-extras/04-interview-cheatsheet.md) — all 23 on a couple of
  pages, for revision and interviews.
- [../00-start-here/02-criticism-and-cautions.md](../00-start-here/02-criticism-and-cautions.md) — pattern abuse, god objects,
  premature abstraction, and how "design patterns" becomes a code smell of its own.
- [../04-extras/03-your-stack-playbook.md](../04-extras/03-your-stack-playbook.md) —
  where these patterns already live in .NET, ASP.NET Core, EF Core, Node and the npm ecosystem you use.
- [../04-extras/05-interpreter-bonus.md](../04-extras/05-interpreter-bonus.md) — the 23rd pattern, written
  from scratch because it is not in the source catalog.

---

## ⚠️ One warning before you start

Patterns are a vocabulary, not a target. The failure mode of studying them is going back to work and
"applying" them — turning a thirty-line function that was perfectly clear into six files and an
interface. Every pattern costs indirection, and indirection is only worth paying for when something is
actually changing.

The healthy sequence is: write the simple thing → feel the pain → recognise the pattern that names that
pain → apply it. Every file's **PART 5** exists to push back on the opposite instinct, and it is the part
most worth re-reading after you have been excited about a pattern for a week.

---

## 🙏 Credits

**PART 1 of every pattern file** — intent, problem, solution, real-world analogy, structure, pseudocode,
applicability, how-to-implement, pros & cons, and relations with other patterns — follows the catalog at
<https://refactoring.guru/design-patterns/catalog>, along with the reference code in PART 2. That site is
the best written, best illustrated treatment of these patterns anywhere, and it is the backbone of this
folder. The plain-English restatements, PART 3 through PART 7, and every example in this file are my own.

The catalog covers **22 of the 23 Gang of Four patterns** — **Interpreter** is not on the site, so it is
written from scratch in [../04-extras/05-interpreter-bonus.md](../04-extras/05-interpreter-bonus.md).

Now go read [Strategy](../03-behavioral/08-strategy.md). It is the best one to start with, and once it
clicks, six of the others click with it. 🚀
