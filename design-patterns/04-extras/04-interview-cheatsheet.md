# 🧠 Interview & Revision Cheat-Sheet

This is the page you open on the train, twenty minutes before the call. Everything here is
compressed on purpose. If a line does not make sense, click through to the full chapter —
every pattern name links to it.

How to use it:

- **Day before an interview:** read the big table, then the "one sentence per pattern" list at the bottom.
- **An hour before:** do the 20-question quiz. Anything you miss, open that chapter.
- **Five minutes before:** read the whiteboard section. Those five patterns are ~80% of what gets asked.

> Plain English up front: a design pattern is not a library and not a rule. It is a *name* for a
> shape of code that keeps reappearing. The value in an interview is mostly the shared vocabulary —
> being able to say "that's a Strategy" and have the other person immediately know what you mean.

---

## 📋 All 22 patterns on one page

| # | Pattern | Category | One-sentence intent | Mnemonic |
|---|---------|----------|---------------------|----------|
| 1 | [Factory Method](../01-creational/01-factory-method.md) | Creational | Let a subclass decide which concrete class to instantiate, behind a method the base class calls. | "Virtual constructor" |
| 2 | [Abstract Factory](../01-creational/02-abstract-factory.md) | Creational | Create whole families of related objects without naming their concrete types. | "Factory of factories" |
| 3 | [Builder](../01-creational/03-builder.md) | Creational | Assemble a complex object step by step so the same steps can produce different results. | "Sandwich counter" |
| 4 | [Prototype](../01-creational/04-prototype.md) | Creational | Create new objects by copying an existing configured instance. | "Clone, don't build" |
| 5 | [Singleton](../01-creational/05-singleton.md) | Creational | Guarantee one instance of a class and give everyone a global way to reach it. | "There can be only one" |
| 6 | [Adapter](../02-structural/01-adapter.md) | Structural | Translate one interface into another so incompatible classes can work together. | "Travel plug" |
| 7 | [Bridge](../02-structural/02-bridge.md) | Structural | Split an abstraction from its implementation so both can vary independently. | "Two dials, not one" |
| 8 | [Composite](../02-structural/03-composite.md) | Structural | Treat a tree of objects and a single object through the same interface. | "Folder is a file" |
| 9 | [Decorator](../02-structural/04-decorator.md) | Structural | Add behaviour to one object at runtime by wrapping it in another of the same type. | "Russian doll" |
| 10 | [Facade](../02-structural/05-facade.md) | Structural | Put one simple entry point in front of a messy subsystem. | "Reception desk" |
| 11 | [Flyweight](../02-structural/06-flyweight.md) | Structural | Share the parts of state that are identical across many objects to save memory. | "One font, many letters" |
| 12 | [Proxy](../02-structural/07-proxy.md) | Structural | Stand in for another object to control access, defer cost, or add bookkeeping. | "Same face, gatekeeper" |
| 13 | [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | Behavioral | Pass a request along a chain until some handler deals with it. | "Pass the parcel" |
| 14 | [Command](../03-behavioral/02-command.md) | Behavioral | Turn a request into an object so it can be queued, logged, retried, or undone. | "Request in a box" |
| 15 | [Iterator](../03-behavioral/03-iterator.md) | Behavioral | Walk a collection's elements without exposing how it stores them. | "Next, please" |
| 16 | [Mediator](../03-behavioral/04-mediator.md) | Behavioral | Route all communication between colleagues through one hub instead of many direct links. | "Air traffic control" |
| 17 | [Memento](../03-behavioral/05-memento.md) | Behavioral | Capture an object's internal state so it can be restored later without breaking encapsulation. | "Save game" |
| 18 | [Observer](../03-behavioral/06-observer.md) | Behavioral | Let many objects subscribe to one object's state changes and get notified automatically. | "Newsletter" |
| 19 | [State](../03-behavioral/07-state.md) | Behavioral | Let an object change its behaviour by swapping the state object it delegates to. | "Traffic light" |
| 20 | [Strategy](../03-behavioral/08-strategy.md) | Behavioral | Make a family of algorithms interchangeable behind one interface. | "Pick the tool" |
| 21 | [Template Method](../03-behavioral/09-template-method.md) | Behavioral | Fix the skeleton of an algorithm in a base method and let subclasses fill in steps. | "Fill in the blanks" |
| 22 | [Visitor](../03-behavioral/10-visitor.md) | Behavioral | Add a new operation over a fixed object structure without editing its classes. | "Guest with a clipboard" |

Related reading: [SOLID and principles](01-solid-and-principles.md) · [start here](../00-start-here/README.md)

---

## 🗺️ The three categories in one ASCII map

```
                     WHAT PROBLEM ARE YOU SOLVING?
                                 |
        +------------------------+------------------------+
        |                        |                        |
   CREATIONAL               STRUCTURAL               BEHAVIORAL
 "who makes it, and      "how do objects fit     "who talks to whom,
  how much do I know      together / how do I      and how does work
  about the concrete      reshape an interface"    get split up"
  type?"
        |                        |                        |
 Factory Method           Adapter   Facade         Chain   Command
 Abstract Factory         Bridge    Flyweight      Iterator Mediator
 Builder                  Composite Proxy          Memento  Observer
 Prototype                Decorator                State    Strategy
 Singleton                                         Template Visitor
```

A quick sorting rule for interviews: if the question is about `new`, it is creational.
If it is about *wrapping* or *shaping* an object, it is structural. If it is about *who calls whom
at runtime*, it is behavioral.

---

## ⚔️ The classic comparison questions (with model answers)

These come up constantly. Each answer is written the way you would actually say it out loud:
one crisp sentence, then the distinguishing test, then an example.

<details>
<summary><strong>1. Strategy vs State</strong></summary>

**Say:** Strategy swaps an algorithm chosen from outside; State swaps behaviour the object chooses
for itself as part of a lifecycle.

**The test:** *who decides the next object?* In Strategy, the caller injects it and it never changes
on its own. In State, the current state object (or the context) decides which state comes next, and
states know about each other.

**Example:** picking `FixedPriceCalculator` vs `AuctionPriceCalculator` for a listing is Strategy —
the caller knows which one. A listing going `Draft → PendingApproval → Live → Sold` is State —
`PendingApproval` knows the only legal next steps are `Live` or `Rejected`.

Chapters: [Strategy](../03-behavioral/08-strategy.md) · [State](../03-behavioral/07-state.md)
</details>

<details>
<summary><strong>2. Decorator vs Proxy</strong></summary>

**Say:** They have the same shape — both implement the target interface and hold a reference to it —
but Decorator *adds* behaviour, while Proxy *controls access* to behaviour that already exists.

**The test:** *who creates the wrapped object?* A Decorator is handed the thing it wraps and you can
stack several. A Proxy usually owns or creates its subject (lazily, remotely, or after a permission
check) and you rarely stack proxies.

**Example:** `GzipStream` wrapping a `FileStream` is Decorator. An EF Core lazy-loading proxy that
fetches `listing.Dealer` on first access is Proxy.

Chapters: [Decorator](../02-structural/04-decorator.md) · [Proxy](../02-structural/07-proxy.md)
</details>

<details>
<summary><strong>3. Adapter vs Facade</strong></summary>

**Say:** Adapter changes an interface you cannot change, to one your code already expects.
Facade invents a *new, simpler* interface over several classes.

**The test:** *does a target interface already exist?* Adapter has a fixed target it must match.
Facade is free to design whatever API is convenient.

**Example:** wrapping a legacy `PriceQuoteSoapClient` so it satisfies your `IPriceProvider` is
Adapter. A `ListingPublishService.Publish(id)` that internally hits validation, image processing,
search indexing and a RabbitMQ publish is Facade.

Chapters: [Adapter](../02-structural/01-adapter.md) · [Facade](../02-structural/05-facade.md)
</details>

<details>
<summary><strong>4. Adapter vs Bridge</strong></summary>

**Say:** Adapter is applied *after the fact* to reconcile two things that already exist.
Bridge is designed *up front* so two dimensions can vary independently.

**The test:** *was there a choice?* If you were forced into it by someone else's API, it's an Adapter.
If you deliberately split "what it does" from "how it does it", it's a Bridge.

**Example:** `NotificationChannel` (abstraction: `PriceDropAlert`, `TestDriveReminder`) bridged to
`IMessageTransport` (implementor: SMS, email, push) is Bridge. You planned both axes.

Chapters: [Adapter](../02-structural/01-adapter.md) · [Bridge](../02-structural/02-bridge.md)
</details>

<details>
<summary><strong>5. Factory Method vs Abstract Factory</strong></summary>

**Say:** Factory Method creates *one* product and varies by subclassing. Abstract Factory creates a
*family* of related products and varies by composition — you pass the factory in.

**The test:** *how many create methods on the interface?* One → Factory Method. Several that must be
consistent with each other → Abstract Factory.

**Example:** `CreateExporter()` overridden per report type is Factory Method. An
`IStorageFactory` returning a matched `IBlobStore` + `IThumbnailer` + `ISignedUrlBuilder` for
either S3 or Azure is Abstract Factory.

Chapters: [Factory Method](../01-creational/01-factory-method.md) · [Abstract Factory](../01-creational/02-abstract-factory.md)
</details>

<details>
<summary><strong>6. Builder vs Abstract Factory</strong></summary>

**Say:** Abstract Factory returns a finished product in one call. Builder assembles the product over
many calls and you get it at the end.

**The test:** *is the construction multi-step and order-sensitive?* Then Builder. Do you need
consistency across several product types? Then Abstract Factory.

**Example:** `new SearchQueryBuilder().Make("Honda").MaxPrice(800000).Sort(Price.Asc).Build()`
is Builder. It also solves the telescoping-constructor problem.

Chapters: [Builder](../01-creational/03-builder.md) · [Abstract Factory](../01-creational/02-abstract-factory.md)
</details>

<details>
<summary><strong>7. Template Method vs Strategy</strong></summary>

**Say:** Same goal — vary part of an algorithm — but Template Method does it with *inheritance at
compile time*, Strategy with *composition at runtime*.

**The test:** *can it change while the program runs?* Strategy yes, Template Method no.
Also: Template Method inverts control (base class calls you); Strategy does not.

**Example:** an abstract `FeedImporter.Import()` that calls `Fetch()`, `Parse()`, `Validate()`,
`Persist()` with only `Parse()` abstract is Template Method. Pull `Parse` out into an
`IFeedParser` you inject, and you have converted it to Strategy.

Chapters: [Template Method](../03-behavioral/09-template-method.md) · [Strategy](../03-behavioral/08-strategy.md)
</details>

<details>
<summary><strong>8. Observer vs Mediator</strong></summary>

**Say:** Observer is one-to-many broadcast of *state change*. Mediator is many-to-many *coordination*
funnelled through a hub that contains the interaction logic.

**The test:** *where does the logic live?* Observer's subject knows nothing about what observers do.
A Mediator knows all the colleagues and encodes the rules between them.

**Example:** `PriceChanged` firing to three subscribers is Observer. A form controller that disables
"Submit" when the make dropdown is empty and clears the model dropdown when make changes is Mediator.

Chapters: [Observer](../03-behavioral/06-observer.md) · [Mediator](../03-behavioral/04-mediator.md)
</details>

<details>
<summary><strong>9. Observer vs Pub/Sub (message broker)</strong></summary>

**Say:** Observer is in-process, synchronous by default, and the subject holds direct references to
its observers. Pub/Sub puts a broker in the middle so publisher and subscriber never see each other,
and it is typically asynchronous and durable.

**The test:** *is there a third party holding the list?* Broker → pub/sub.

**Example:** C# `event` / RxJS `Subject` is Observer. RabbitMQ with a topic exchange is pub/sub —
and it buys you retries, dead-lettering and cross-service delivery that Observer cannot give you.

Chapter: [Observer](../03-behavioral/06-observer.md)
</details>

<details>
<summary><strong>10. Command vs Strategy</strong></summary>

**Say:** Both wrap behaviour in an object, but Strategy is "how to do a step" and is usually
stateless and reusable; Command is "a specific request, with its arguments baked in", designed to be
stored, queued, undone or replayed.

**The test:** *does it carry its own parameters and a receiver?* Command yes. Does it have `Undo()`?
Definitely Command.

**Example:** `ISortStrategy` vs `RelistVehicleCommand { ListingId = 4711 }` sitting on a queue.

Chapters: [Command](../03-behavioral/02-command.md) · [Strategy](../03-behavioral/08-strategy.md)
</details>

<details>
<summary><strong>11. Chain of Responsibility vs Decorator</strong></summary>

**Say:** Structurally near-identical linked wrappers. Decorator guarantees every wrapper runs and the
call reaches the core. Chain of Responsibility explicitly allows a link to *stop* the chain.

**The test:** *may a link short-circuit?* Yes → Chain. Also, Decorator preserves the target
interface; chain handlers usually share a handler interface, not the target's.

**Example:** ASP.NET Core middleware is both, honestly — but the auth middleware returning 401
without calling `next()` is the Chain behaviour.

Chapters: [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) · [Decorator](../02-structural/04-decorator.md)
</details>

<details>
<summary><strong>12. Composite vs Decorator</strong></summary>

**Say:** Both are recursive and share the component interface. Composite has *many* children and is
about tree structure. Decorator has exactly *one* child and is about added behaviour.

**The test:** count the children. `List<IComponent>` → Composite. Single `IComponent _inner` → Decorator.

Chapters: [Composite](../02-structural/03-composite.md) · [Decorator](../02-structural/04-decorator.md)
</details>

<details>
<summary><strong>13. Prototype vs Factory</strong></summary>

**Say:** A factory builds from a recipe; a prototype copies a live, already-configured instance.

**The test:** *is the configuration expensive or only known at runtime?* Then cloning wins — you keep
the tuning that a constructor would not know about.

**Example:** a `SearchFilter` template configured by a dealer, cloned per request and then tweaked.

Chapters: [Prototype](../01-creational/04-prototype.md) · [Factory Method](../01-creational/01-factory-method.md)
</details>

<details>
<summary><strong>14. Flyweight vs caching / object pooling</strong></summary>

**Say:** Flyweight is about *splitting* state into shared intrinsic and per-use extrinsic, so the
shared part can be interned. A cache just avoids recomputation; a pool lends out mutable objects and
takes them back.

**The test:** *did you have to change the object's shape (pass extrinsic state in as arguments)?*
That reshaping is the pattern. Flyweights are immutable and shared simultaneously; pooled objects are
mutable and lent exclusively.

Chapter: [Flyweight](../02-structural/06-flyweight.md)
</details>

<details>
<summary><strong>15. Memento vs event sourcing vs just serializing</strong></summary>

**Say:** Memento stores a *snapshot* the originator can restore, while keeping the internals opaque to
the caretaker. Event sourcing stores the *deltas* and rebuilds by replay.

**The test:** encapsulation. A Memento is deliberately a black box to everyone but the originator;
a serialized DTO is readable by anyone, which is the thing Memento is trying to avoid.

Chapter: [Memento](../03-behavioral/05-memento.md)
</details>

<details>
<summary><strong>16. Visitor vs pattern matching / switch</strong></summary>

**Say:** Visitor uses double dispatch to add operations to a closed type hierarchy without touching
those types. In C# 9+ you can often get the same result with a `switch` expression on type patterns —
simpler, but it loses the compiler's exhaustiveness help unless the hierarchy is sealed.

**The test:** the *expression problem*. Adding new **operations** often → Visitor. Adding new
**types** often → do not use Visitor, it makes that case worse.

Chapter: [Visitor](../03-behavioral/10-visitor.md)
</details>

---

## 🎯 "Name that pattern" — 20 scenarios

Read the scenario, say the pattern out loud, then expand. No peeking.

<details>
<summary><strong>1.</strong> Your export feature writes CSV, XLSX and PDF. The caller passes a format enum and gets back the right writer, and callers never see the concrete writer classes.</summary>

**Factory Method** (or a simple static factory). If the choice also drags along a matching
formatter and a matching header renderer that must agree, it becomes **Abstract Factory**.
</details>

<details>
<summary><strong>2.</strong> A photo upload pipeline: resize, watermark, then strip EXIF — and ops want to turn watermarking off per dealer without redeploying.</summary>

**Decorator**. Each step implements `IImageProcessor` and wraps the next. Composition happens from
config at startup.
</details>

<details>
<summary><strong>3.</strong> A vehicle listing moves through Draft, PendingApproval, Live, Sold and Archived, and each stage allows different operations.</summary>

**State**. Each state class implements the same operations and throws or no-ops for illegal ones.
</details>

<details>
<summary><strong>4.</strong> You must call a third-party valuation service whose SDK returns XML with different field names than your domain model.</summary>

**Adapter**. Your `IValuationProvider` is the target; the adapter translates.
</details>

<details>
<summary><strong>5.</strong> A rendering engine draws 200,000 map pins. Every pin has an x/y and a dealer id, but only 12 distinct icon bitmaps exist.</summary>

**Flyweight**. Icons are intrinsic and shared; x/y and dealer id are extrinsic and passed in at draw time.
</details>

<details>
<summary><strong>6.</strong> An admin screen needs "undo last 10 actions" over bulk edits.</summary>

**Command** (with an undo stack). If each action also snapshots the entity's prior state, the snapshot
itself is a **Memento**.
</details>

<details>
<summary><strong>7.</strong> An HTTP request passes through logging, then auth, then rate limiting; any of them can reject it early.</summary>

**Chain of Responsibility**.
</details>

<details>
<summary><strong>8.</strong> A config object is expensive to load from the database and everyone needs the same instance.</summary>

**Singleton** — but in modern C#/TS you express this as a DI container registration with singleton
lifetime, not a static `Instance` property.
</details>

<details>
<summary><strong>9.</strong> A `Package` on the marketplace can contain vehicles, accessories, or other packages, and you need a total price for any of them.</summary>

**Composite**. `TotalPrice()` on the leaf returns its own; on the composite it sums children.
</details>

<details>
<summary><strong>10.</strong> A search query object has 14 optional filters and you refuse to write a 14-argument constructor.</summary>

**Builder** (fluent). In C# an `init`-only record with object initializers covers many of these cases.
</details>

<details>
<summary><strong>11.</strong> A UI component must re-render whenever the price filter changes, and so must the map, the count badge and the URL.</summary>

**Observer**.
</details>

<details>
<summary><strong>12.</strong> You need per-call retry, timeout and metrics around an expensive remote client, without the client knowing.</summary>

**Proxy** (a protection/smart proxy). If you kept adding independently stackable concerns, you have
drifted into **Decorator** territory — and that is fine.
</details>

<details>
<summary><strong>13.</strong> Import jobs for three dealer feed formats share the same steps — download, parse, validate, upsert — and only parsing differs.</summary>

**Template Method**. Inject the parser instead and it becomes **Strategy**.
</details>

<details>
<summary><strong>14.</strong> A checkout page where changing the finance option enables/disables four other controls and recalculates two totals.</summary>

**Mediator**.
</details>

<details>
<summary><strong>15.</strong> You want `for (const listing of results)` to work over a custom paged result set that fetches lazily.</summary>

**Iterator** (in JS, a generator implementing `Symbol.iterator`; in C#, `IEnumerable<T>` + `yield return`).
</details>

<details>
<summary><strong>16.</strong> You need to run tax, discount and shipping calculations over a fixed hierarchy of order-line types, and new calculations get added every quarter.</summary>

**Visitor**. Fixed types, growing operations — the exact case it is for.
</details>

<details>
<summary><strong>17.</strong> Your app must support both S3 and Azure Blob, and each needs a matched blob store, thumbnailer and signed-URL builder that must not be mixed.</summary>

**Abstract Factory**.
</details>

<details>
<summary><strong>18.</strong> A test needs 50 listings that differ in two fields from a carefully built baseline object.</summary>

**Prototype** (clone the baseline and mutate). Test-data builders combine this with **Builder**.
</details>

<details>
<summary><strong>19.</strong> One `PublishListing(id)` call should trigger validation, image processing, search indexing and a RabbitMQ event — and controllers should not know about any of that.</summary>

**Facade**.
</details>

<details>
<summary><strong>20.</strong> You have `Report` types (Sales, Inventory) and `Renderer` types (HTML, PDF, CSV) and you refuse to write six classes.</summary>

**Bridge**. Two independent axes, composed instead of multiplied.
</details>

---

## 🔍 Spot the pattern in a real API

Naming a pattern in a framework you actually use is the single most convincing thing you can do in an
interview. These are all real, widely-known APIs.

| API | Pattern | Why |
|-----|---------|-----|
| .NET `Stream` wrappers — `GZipStream`, `CryptoStream`, `BufferedStream` over a `FileStream` | Decorator | Each is a `Stream` that holds a `Stream` and adds behaviour; they stack. |
| Java `InputStream` / `BufferedInputStream` / `DataInputStream` | Decorator | The textbook example, same shape as above. |
| `IEnumerable<T>` / `IEnumerator<T>`, and JS `Symbol.iterator` | Iterator | Traversal decoupled from storage. |
| C# `yield return` / JS `function*` | Iterator | Compiler-generated state machine implementing the iterator contract. |
| .NET `IObservable<T>` / `IObserver<T>`, RxJS `Subject`, C# `event` | Observer | Subscribe, get pushed notifications. |
| `Array.prototype.sort(comparator)` and .NET `IComparer<T>` | Strategy | The comparison algorithm is passed in. |
| `Array.prototype.map(fn)` | Strategy | Same idea, function as the strategy object. |
| `HttpClient` + `DelegatingHandler` pipeline | Chain of Responsibility / Decorator | Each handler may act, then call the inner handler. |
| ASP.NET Core middleware (`app.Use(async (ctx, next) => ...)`) | Chain of Responsibility | A link can short-circuit by not calling `next`. |
| Express.js middleware | Chain of Responsibility | Same. |
| `StringBuilder`, `UriBuilder`, `HostBuilder`/`WebApplicationBuilder` | Builder | Step-by-step assembly then `Build()`/`ToString()`. |
| `HttpClientFactory` (`CreateClient(name)`) | Factory Method | Named creation hiding the concrete configuration. |
| `ILoggerFactory.CreateLogger<T>()` | Factory Method | Same. |
| `DbProviderFactory` (`CreateConnection`, `CreateCommand`, `CreateParameter`) | Abstract Factory | A family of provider-matched objects. |
| EF Core lazy-loading proxies; WCF/gRPC generated clients | Proxy | Same interface, remote or deferred work behind it. |
| JS `Proxy` object (`new Proxy(target, handler)`) | Proxy | Literally the pattern, built into the language. |
| `System.Text.Json` `JsonSerializerOptions` converters | Strategy | Per-type serialization algorithm plugged in. |
| .NET `ICloneable`, JS `structuredClone`, C# `record` `with` expressions | Prototype | Copy-based creation. |
| DOM tree / React component tree | Composite | Nodes and containers share one interface. |
| `System.Threading.Tasks.Task` continuations, `Func<T>` / `Action` handed to a queue | Command | Behaviour reified as an object to run later. |
| .NET `ArrayPool<T>` / `String` interning | Flyweight-adjacent | Interning is genuine Flyweight; pooling is not. |
| `Math`, `Console`, `Path` — static utility holders | *Not* Singleton | No instance, no lifecycle control. Static class ≠ Singleton. |

A useful line to have ready: **"the most commonly used patterns in code I write are Strategy,
Decorator, Adapter, Facade and Iterator — and in C# three of them are already in the framework."**

---

## 🪤 Common interview traps

<details>
<summary><strong>Is Singleton an anti-pattern?</strong></summary>

**Answer honestly, with nuance.** The *requirement* ("exactly one instance") is legitimate. The
*classic implementation* (a static `Instance` property that callers reach for directly) is what people
call an anti-pattern, for three reasons:

1. **Hidden dependency.** A class that calls `Config.Instance` has a dependency that does not appear in
   its constructor. You cannot see it, and you cannot substitute it in tests.
2. **Global mutable state.** Shared across threads, ordering bugs, test pollution between runs.
3. **Lifetime is decided by the class, not the application.** You lose the ability to have one per
   tenant, or one per request, later.

**What to say you do instead:** register the type with singleton lifetime in the DI container and
inject the interface. You keep "one instance" and lose all three problems.

```csharp
// The requirement, done properly
services.AddSingleton<IPricingRules, PricingRules>();

public sealed class ListingService
{
    private readonly IPricingRules _rules;           // visible, substitutable
    public ListingService(IPricingRules rules) => _rules = rules;
}
```

Bonus point: mention that a DI singleton must be **thread-safe and stateless-ish**, and that injecting
a scoped service into a singleton is the classic captive-dependency bug.
</details>

<details>
<summary><strong>Why is double-checked locking broken without <code>volatile</code>?</strong></summary>

Because without a memory barrier, another thread can observe a **non-null reference to a
not-yet-fully-constructed object**. Writing the reference field and writing the object's own fields can
be reordered (by the compiler, the JIT, or the CPU), so thread B sees `_instance != null`, skips the
lock, and uses an object whose fields are still default values.

```csharp
// BROKEN without volatile
private static Singleton _instance;
public static Singleton Instance
{
    get
    {
        if (_instance == null)                 // (1) no lock — can see a torn publish
        {
            lock (Gate)
            {
                if (_instance == null)
                    _instance = new Singleton(); // (2) write to field vs writes inside ctor
            }                                    //     may be observed out of order
        }
        return _instance;
    }
}
```

Two correct fixes:

```csharp
// Fix A: volatile gives the needed acquire/release semantics
private static volatile Singleton _instance;

// Fix B: don't hand-roll it at all
private static readonly Lazy<Singleton> _lazy =
    new(() => new Singleton(), LazyThreadSafetyMode.ExecutionAndPublication);
public static Singleton Instance => _lazy.Value;
```

Extra credit: on the CLR, a plain `static readonly` field initialized in a static initializer is
already thread-safe and lazy enough for almost every case — the type initializer runs once under a
lock the runtime manages. And note this whole class of bug simply does not exist in single-threaded
JavaScript, which is why the pattern is a C#/Java/C++ interview question and not a JS one.
</details>

<details>
<summary><strong>Strategy vs State when the class diagrams are identical</strong></summary>

They *are* identical. Context holds an interface, concrete classes implement it, context delegates.
The difference is **intent and who controls transitions**, and you should say exactly that.

| | Strategy | State |
|---|---|---|
| Who chooses the object | The client, from outside | The object itself, as part of a lifecycle |
| Does it change during one operation's life | Usually no | Yes, that is the point |
| Do the implementations know each other | No | Usually yes (`Live` knows about `Sold`) |
| Number of implementations used per context instance | One at a time, often forever | Many over time, in sequence |
| Typical vocabulary | "algorithm", "policy", "rule" | "status", "mode", "phase" |

**The killer line:** "If the concrete classes reference each other or set `context.State = ...`,
it's State. If they are mutually oblivious and the caller picks one, it's Strategy."
</details>

<details>
<summary><strong>"Is a static class a Singleton?"</strong></summary>

No. A static class has no instance, so it cannot implement an interface, cannot be passed as a
parameter, cannot be mocked, and cannot have its lifetime managed. A Singleton is a normal object with
a controlled lifetime. Say "a static class is a namespace for functions; a Singleton is an object with
a policy".
</details>

<details>
<summary><strong>"Isn't every pattern just an interface?"</strong></summary>

Half-true and worth engaging with rather than dismissing. Many patterns share the mechanism
(polymorphism) and differ in *intent*. That is precisely why the names are useful: `IComparer<T>`,
`IImageProcessor` and `IListingState` are all "just interfaces", but saying Strategy, Decorator and
State tells your reader what the interface is *for*.
</details>

<details>
<summary><strong>"Do patterns matter in functional / JS code?"</strong></summary>

Some collapse into language features. Strategy is a function argument. Command is a closure.
Iterator is a generator. Template Method is a higher-order function with hooks. Say that out loud —
it shows you understand the pattern rather than the class diagram:

```typescript
// Strategy, without a single class
type PriceRule = (base: number, ctx: { dealerTier: string }) => number;

const flat: PriceRule = (b) => b;
const tiered: PriceRule = (b, c) => (c.dealerTier === "gold" ? b * 0.95 : b);

function quote(base: number, ctx: { dealerTier: string }, rule: PriceRule) {
  return Math.round(rule(base, ctx));
}
```

Others — Composite, Observer, Proxy, Visitor — remain structurally the same because they are about
object graphs, not about dispatch.
</details>

<details>
<summary><strong>"Have you ever removed a pattern?"</strong></summary>

Answer yes and have a story. Over-patterning is a real smell: an `IFooFactoryProvider` with one
implementation, an abstract base with one subclass, a Strategy interface that never got a second
strategy. The honest position is **add a pattern when the second case arrives**, not in anticipation
of it. Interviewers reward this.
</details>

<details>
<summary><strong>"Which pattern do you dislike?"</strong></summary>

Safe, defensible answer: Visitor. It is powerful but the double-dispatch boilerplate is heavy, it
breaks the moment someone adds a type to the hierarchy, and in C# a `switch` expression over sealed
types usually reads better. Name the tradeoff, don't just say "it's ugly".
</details>

<details>
<summary><strong>"Singleton + multi-threading + lazy = ?"</strong></summary>

They are fishing for `Lazy<T>`, or for `static readonly`, or for the `volatile` discussion above.
Give all three and say which you would actually ship (DI singleton).
</details>

---

## ⏱️ Five-minute whiteboard versions

The five that get asked most. Each one: the diagram, minimum viable code, and the sentence to say.

### 1. Singleton

```
  +---------------------------+
  |        Singleton          |
  +---------------------------+
  | - static instance         |
  | - private ctor            |
  +---------------------------+
  | + static Instance : get   |
  +---------------------------+
             ^
             | everyone calls the same accessor
```

```csharp
public sealed class ExchangeRates
{
    private static readonly Lazy<ExchangeRates> _lazy = new(() => new ExchangeRates());
    public static ExchangeRates Instance => _lazy.Value;

    private readonly IReadOnlyDictionary<string, decimal> _rates;
    private ExchangeRates()
        => _rates = new Dictionary<string, decimal> { ["INR"] = 1m, ["USD"] = 83.2m };

    public decimal ToInr(string currency, decimal amount) => amount * _rates[currency];
}
```

**Say:** "Private constructor, static accessor, thread-safe lazy init. In production I'd express this
as `AddSingleton` in the DI container so it stays testable."

---

### 2. Factory Method

```
   Creator (abstract)                Product (interface)
   +-----------------+               +----------------+
   | + Process()     |-------------->| + Export(rows) |
   | # CreateExporter|: abstract     +----------------+
   +-----------------+                      ^
          ^                                 |
    +-----+------+                  +-------+--------+
 CsvCreator   PdfCreator         CsvExporter    PdfExporter
```

```csharp
public interface IExporter { byte[] Export(IReadOnlyList<Listing> rows); }

public abstract class ReportJob
{
    protected abstract IExporter CreateExporter();   // the factory method

    public byte[] Run(IReadOnlyList<Listing> rows)   // fixed workflow
    {
        var exporter = CreateExporter();
        return exporter.Export(rows);
    }
}

public sealed class CsvReportJob : ReportJob
{
    protected override IExporter CreateExporter() => new CsvExporter();
}
```

**Say:** "The base class owns the workflow but defers the `new` to a subclass — a virtual constructor.
It satisfies the open/closed principle: a new format is a new subclass, not an edit."

---

### 3. Observer

```
   Subject                      Observer (interface)
   +---------------+            +------------------+
   | + Subscribe() |----------->| + OnNext(event)  |
   | + Notify()    |  0..*      +------------------+
   +---------------+                    ^
                               +--------+--------+
                           EmailAlerts      SearchIndexer
```

```typescript
type Listener<T> = (payload: T) => void;

export class Emitter<T> {
  private listeners = new Set<Listener<T>>();

  subscribe(fn: Listener<T>): () => void {
    this.listeners.add(fn);
    return () => this.listeners.delete(fn);   // unsubscribe handle
  }

  emit(payload: T): void {
    for (const fn of [...this.listeners]) {   // copy: a listener may unsubscribe mid-emit
      try {
        fn(payload);
      } catch (err) {
        console.error("observer threw", err); // one bad listener must not kill the rest
      }
    }
  }
}

const priceDrops = new Emitter<{ listingId: number; newPrice: number }>();
const off = priceDrops.subscribe(({ listingId }) => console.log("notify watchers of", listingId));
priceDrops.emit({ listingId: 4711, newPrice: 749000 });
off();
```

**Say:** "Subscribe returns an unsubscribe function — that's the leak fix most people forget. I copy
the listener set before iterating and swallow-and-log listener exceptions so one subscriber can't take
down the notification."

---

### 4. Strategy

```
   Context                        Strategy (interface)
   +---------------+              +------------------+
   | - strategy    |------------->| + Calculate(x)   |
   | + Execute()   |              +------------------+
   +---------------+                      ^
                            +-------------+-------------+
                      FlatFee        PercentageFee   TieredFee
```

```csharp
public interface IListingFeeStrategy
{
    decimal Calculate(decimal salePrice);
}

public sealed class FlatFee : IListingFeeStrategy
{
    public decimal Calculate(decimal salePrice) => 999m;
}

public sealed class PercentageFee(decimal rate) : IListingFeeStrategy
{
    public decimal Calculate(decimal salePrice) => Math.Round(salePrice * rate, 2);
}

public sealed class TieredFee : IListingFeeStrategy
{
    public decimal Calculate(decimal salePrice) => salePrice switch
    {
        < 300_000m  => 499m,
        < 1_000_000m => 1_499m,
        _            => 2_999m
    };
}

public sealed class Checkout(IListingFeeStrategy fee)
{
    public decimal Total(decimal salePrice) => salePrice + fee.Calculate(salePrice);
}
```

**Say:** "One interface, several algorithms, chosen by the caller and injected. It replaces a switch
that would otherwise grow every time pricing changes."

---

### 5. Decorator

```
  IImageProcessor
        ^
        |------------------------------+
   BaseProcessor                 ProcessorDecorator (holds IImageProcessor)
                                        ^
                              +---------+---------+
                          Watermark            ExifStrip

  new ExifStrip(new Watermark(new Resize(new BaseProcessor())))
```

```typescript
export interface ImageProcessor {
  process(buffer: Buffer): Promise<Buffer>;
}

export class PassThrough implements ImageProcessor {
  async process(buffer: Buffer): Promise<Buffer> {
    return buffer;
  }
}

abstract class ProcessorDecorator implements ImageProcessor {
  constructor(protected readonly inner: ImageProcessor) {}
  abstract process(buffer: Buffer): Promise<Buffer>;
}

export class Watermark extends ProcessorDecorator {
  constructor(inner: ImageProcessor, private readonly text: string) {
    super(inner);
  }
  async process(buffer: Buffer): Promise<Buffer> {
    const base = await this.inner.process(buffer);
    return applyWatermark(base, this.text);   // your imaging library
  }
}

export class Timed extends ProcessorDecorator {
  constructor(inner: ImageProcessor, private readonly label: string) {
    super(inner);
  }
  async process(buffer: Buffer): Promise<Buffer> {
    const started = performance.now();
    try {
      return await this.inner.process(buffer);
    } finally {
      console.info(`${this.label} took ${(performance.now() - started).toFixed(1)}ms`);
    }
  }
}

// Composition decided by config, not by inheritance
const pipeline: ImageProcessor = new Timed(
  new Watermark(new PassThrough(), "carwale.com"),
  "upload-pipeline"
);
```

**Say:** "Same interface in, same interface out, one inner reference. That's what makes it stackable
in any order at runtime — the thing inheritance can't do."

---

## 🧾 If you remember only one sentence per pattern

1. **[Factory Method](../01-creational/01-factory-method.md)** — A subclass decides which concrete class gets created, behind a method the base class calls.
2. **[Abstract Factory](../01-creational/02-abstract-factory.md)** — One factory object creates a whole matched family of products, so you can't mix S3 with an Azure thumbnailer.
3. **[Builder](../01-creational/03-builder.md)** — Build a complicated object step by step and call `Build()` at the end, instead of a constructor with fourteen arguments.
4. **[Prototype](../01-creational/04-prototype.md)** — Copy an existing configured object rather than constructing a new one from scratch.
5. **[Singleton](../01-creational/05-singleton.md)** — Exactly one instance, reachable globally; legitimate as a requirement, usually better served by a DI singleton than a static property.
6. **[Adapter](../02-structural/01-adapter.md)** — A translator that makes someone else's interface look like the one your code already expects.
7. **[Bridge](../02-structural/02-bridge.md)** — Split "what it does" from "how it does it" so the two can vary without producing a class per combination.
8. **[Composite](../02-structural/03-composite.md)** — A tree where a branch and a leaf answer to the same interface, so callers never check which one they hold.
9. **[Decorator](../02-structural/04-decorator.md)** — A wrapper with the same interface as the thing it wraps, adding behaviour, stackable at runtime.
10. **[Facade](../02-structural/05-facade.md)** — One friendly method in front of a subsystem that takes five calls in the right order to use correctly.
11. **[Flyweight](../02-structural/06-flyweight.md)** — Share the identical (intrinsic) part of state across thousands of instances and pass the varying (extrinsic) part in as arguments.
12. **[Proxy](../02-structural/07-proxy.md)** — A stand-in with the same interface that controls access: lazy, remote, protected or instrumented.
13. **[Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md)** — Hand the request down a line of handlers until one deals with it or stops it.
14. **[Command](../03-behavioral/02-command.md)** — Turn "do this thing with these arguments" into an object you can queue, log, retry or undo.
15. **[Iterator](../03-behavioral/03-iterator.md)** — Walk a collection one element at a time without the caller knowing how it is stored.
16. **[Mediator](../03-behavioral/04-mediator.md)** — Put one hub in the middle so N components coordinate through it instead of holding N² references to each other.
17. **[Memento](../03-behavioral/05-memento.md)** — Take an opaque snapshot of an object's state that only that object knows how to restore.
18. **[Observer](../03-behavioral/06-observer.md)** — Subscribers register with a subject and get pushed a notification whenever its state changes.
19. **[State](../03-behavioral/07-state.md)** — The object delegates to a state object and swaps it as its lifecycle advances, so the behaviour changes with the status.
20. **[Strategy](../03-behavioral/08-strategy.md)** — Interchangeable algorithms behind one interface, chosen by the caller.
21. **[Template Method](../03-behavioral/09-template-method.md)** — The base class owns the algorithm's skeleton and subclasses fill in the named gaps.
22. **[Visitor](../03-behavioral/10-visitor.md)** — Add new operations over a fixed set of types via double dispatch, without editing those types.

---

## ✅ Final checklist for the interview

- [ ] I can name all three categories and sort any pattern into one in under five seconds.
- [ ] I can draw Strategy, Decorator, Observer, Factory Method and Singleton from memory.
- [ ] I have a real example from my own codebase for at least four patterns.
- [ ] I can name a pattern in a framework I use daily (`Stream`, `IComparer<T>`, middleware).
- [ ] I can defend Singleton *and* explain why I'd use DI instead.
- [ ] I can explain the `volatile` / double-checked-locking bug in two sentences.
- [ ] I can separate Strategy from State using the "who transitions" test.
- [ ] I have an honest answer to "when did a pattern make your code worse?"

Next: [SOLID and principles](01-solid-and-principles.md) · back to [how to use this](../00-start-here/README.md)
