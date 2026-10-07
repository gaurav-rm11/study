# 🗺️ Pattern Relationships Map — How the 23 Patterns Talk to Each Other

Patterns are not 23 independent inventions. They are a *vocabulary*, and like any
vocabulary the words overlap, rhyme, and get confused with each other. Half the pain of
learning design patterns is not "what is a Decorator" — it is "why is this a Decorator and
not a Proxy, when the class diagram is literally the same picture?"

This chapter is the atlas for that. Three things live here:

1. **A combination map** — which patterns naturally sit next to each other in real systems.
2. **A confusion table** — every "often confused with" pair in the catalogue, with a
   *they look alike because…*, a *they differ because…*, and a one-line decision rule.
3. **The intent argument** — the single most important idea in this whole folder: several
   patterns are *structurally identical*. The class diagram does not name the pattern. The
   intent does.

Read it after you have met the patterns individually. Come back to it every time you are
about to argue with a colleague in a pull request.

---

## 🧭 Table of contents

- [The big picture map](#-the-big-picture-map)
- [The confusion atlas](#-the-confusion-atlas-every-often-confused-with-pair)
  - [Strategy vs State](#1-strategy-vs-state)
  - [Strategy vs Bridge](#2-strategy-vs-bridge)
  - [Adapter vs Decorator vs Proxy vs Facade](#3-adapter-vs-decorator-vs-proxy-vs-facade)
  - [Factory Method vs Abstract Factory vs Builder](#4-factory-method-vs-abstract-factory-vs-builder)
  - [Composite vs Decorator](#5-composite-vs-decorator)
  - [Chain of Responsibility vs Decorator](#6-chain-of-responsibility-vs-decorator)
  - [Mediator vs Observer](#7-mediator-vs-observer)
  - [Command vs Strategy](#8-command-vs-strategy)
  - [Template Method vs Strategy](#9-template-method-vs-strategy)
  - [Proxy vs Decorator](#10-proxy-vs-decorator)
  - [Flyweight vs Singleton](#11-flyweight-vs-singleton)
  - [Visitor vs Iterator](#12-visitor-vs-iterator)
- [Structurally identical, different intent](#-structurally-identical-different-intent)
- [Patterns that commonly appear together](#-patterns-that-commonly-appear-together)
- [A decision flowchart](#-a-decision-flowchart)
- [Plain English summary](#-plain-english-summary)

---

## 🌐 The big picture map

Read the arrows as "commonly used with / naturally leads to". This is not a class diagram;
it is a *social network* of patterns.

```
                        ┌──────────────────────────────────────────┐
                        │            CREATIONAL CLUSTER            │
                        │                                          │
   "I need one of X"    │   Factory Method ──refines──> Abstract   │
   but which X?         │        │                      Factory    │
                        │        │                        │        │
                        │        │ often implemented      │ often  │
                        │        │ with                   │ a      │
                        │        v                        v        │
                        │    Prototype <--clone--    Singleton     │
                        │        ^                                 │
                        │        │  "build it step by step"        │
                        │     Builder                              │
                        └────────┬─────────────────────────────────┘
                                 │ builds
                                 v
        ┌────────────────────────────────────────────────────────────┐
        │                    STRUCTURAL CLUSTER                       │
        │                                                             │
        │     Composite <──iterate──> Iterator                        │
        │        │  ^                                                 │
        │        │  └── decorated by ── Decorator ──stack──> Decorator│
        │        │                          ^                         │
        │        │                          │ same shape as           │
        │        │                          v                         │
        │      Facade ──hides──> subsystem  Proxy ──guards──> Subject │
        │        ^                          ^                         │
        │        │                          │                         │
        │      Adapter ──wraps──> foreign API                         │
        │                                                             │
        │      Bridge ──abstraction × implementation grid             │
        │        ^                                                    │
        │        │ implementations often created by Abstract Factory  │
        │                                                             │
        │      Flyweight ──shared instances──> handed out by Factory  │
        └────────────────────────────────────────────────────────────┘
                                 │ operated on by
                                 v
        ┌────────────────────────────────────────────────────────────┐
        │                   BEHAVIORAL CLUSTER                        │
        │                                                             │
        │   Strategy <──sibling──> State <──sibling──> Template Method│
        │      ^                     │                                │
        │      │                     │ transitions logged by          │
        │      │                     v                                │
        │   Command ──history──> Memento  ──undo/redo                 │
        │      │                                                      │
        │      │ queued / retried / logged                            │
        │      v                                                      │
        │   Chain of Responsibility ──pipeline──> handler ──> handler │
        │                                                             │
        │   Observer <──tames fan-out──> Mediator                     │
        │                                                             │
        │   Visitor ──walks──> Composite (via Iterator)               │
        │                                                             │
        │   Interpreter/Template/Iterator: Composite's natural friends│
        └────────────────────────────────────────────────────────────┘
```

A second view — the same information as a "who wraps whom" layering, which is how you will
actually see it in a C# service or a Node app:

```
  caller
    │
    v
 ┌────────────────────┐   Facade          — one friendly door into many services
 │  PricingFacade     │
 └────────┬───────────┘
          v
 ┌────────────────────┐   Proxy           — caching / auth / lazy-load, SAME interface
 │ CachingPriceProxy  │
 └────────┬───────────┘
          v
 ┌────────────────────┐   Decorator       — adds behaviour, SAME interface, stackable
 │ AuditedPriceSvc    │
 └────────┬───────────┘
          v
 ┌────────────────────┐   Adapter         — converts a foreign interface to ours
 │ LegacySoapAdapter  │
 └────────┬───────────┘
          v
   3rd-party SOAP API
```

Every box above has the *same* outward interface except the Facade (new, simpler
interface) and the Adapter (converts a different one). That single observation is 80% of
the structural-pattern confusion, and we unpack it below.

---

## 🔍 The confusion atlas: every "often confused with" pair

The quick-scan table first. Every row is expanded afterwards with code.

| # | Pair | Look alike because… | Differ because… | Decision rule |
|---|------|--------------------|-----------------|---------------|
| 1 | [Strategy](../03-behavioral/08-strategy.md) vs [State](../03-behavioral/07-state.md) | Identical diagram: a context delegates to a swappable object behind one interface. | Strategy objects are chosen by the *client* and never change each other; State objects choose *their own successor* and encode a lifecycle. | **Does the swapped object decide what comes next? → State. Does the caller decide? → Strategy.** |
| 2 | [Strategy](../03-behavioral/08-strategy.md) vs [Bridge](../02-structural/02-bridge.md) | Both hold a reference to an interface and delegate to it. | Strategy swaps one *algorithm* at runtime; Bridge splits a whole *abstraction hierarchy* from a whole *implementation hierarchy* so both can grow independently. | **One varying method → Strategy. Two whole hierarchies multiplying → Bridge.** |
| 3 | [Adapter](../02-structural/01-adapter.md) vs [Decorator](../02-structural/04-decorator.md) | Both wrap an object and forward calls. | Adapter *changes* the interface without changing behaviour; Decorator *keeps* the interface and changes behaviour. | **Interface changed → Adapter. Behaviour changed → Decorator.** |
| 3b | [Proxy](../02-structural/07-proxy.md) vs [Facade](../02-structural/05-facade.md) | Both stand in front of something and simplify the caller's life. | Proxy has the *same* interface as one subject and controls access to it; Facade invents a *new, smaller* interface over *many* objects. | **Same interface, one subject → Proxy. New interface, many subjects → Facade.** |
| 4 | [Factory Method](../01-creational/01-factory-method.md) vs [Abstract Factory](../01-creational/02-abstract-factory.md) | Both hide `new` behind a method. | Factory Method is one method producing one product, varied by subclassing; Abstract Factory is an *object* with several methods producing a *family* of matched products. | **One product → Factory Method. A matched family → Abstract Factory.** |
| 4b | [Abstract Factory](../01-creational/02-abstract-factory.md) vs [Builder](../01-creational/03-builder.md) | Both are objects you call to get objects. | Abstract Factory returns a finished product immediately and cares *which family*; Builder assembles one complex product over many steps and cares *how it is put together*. | **Which variant? → Abstract Factory. How many steps? → Builder.** |
| 5 | [Composite](../02-structural/03-composite.md) vs [Decorator](../02-structural/04-decorator.md) | Both are recursive: an object with the same interface as its children. | Composite has *many* children and exists to treat a tree uniformly; Decorator has exactly *one* child and exists to add behaviour. | **Many children, tree semantics → Composite. Exactly one child, added behaviour → Decorator.** |
| 6 | [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) vs [Decorator](../02-structural/04-decorator.md) | Both build a linked list of objects that each get a turn around the call. | A Decorator link *always* passes the call on; a Chain link *may stop* the call — handling it and returning, or dropping it. | **Any link allowed to not forward? → Chain of Responsibility. Everyone always forwards? → Decorator.** |
| 7 | [Mediator](../03-behavioral/04-mediator.md) vs [Observer](../03-behavioral/06-observer.md) | Both decouple objects that would otherwise call each other directly. | Observer is a one-way broadcast where the publisher does not care who listens; Mediator is a hub that *knows* the participants and orchestrates a protocol between them. | **Does the middle object contain the "if A then B" logic? → Mediator. Is the middle just a mailing list? → Observer.** |
| 8 | [Command](../03-behavioral/02-command.md) vs [Strategy](../03-behavioral/08-strategy.md) | Both wrap "something to do" into an object with one method. | Strategy encapsulates *how* to do a step inside a known operation; Command encapsulates *a whole request with its arguments* so it can be queued, logged, retried, undone. | **Do you need to store, queue or undo it? → Command. Is it just a pluggable "how"? → Strategy.** |
| 9 | [Template Method](../03-behavioral/09-template-method.md) vs [Strategy](../03-behavioral/08-strategy.md) | Both let you vary one step of an algorithm. | Template Method varies it by *inheritance*, fixed at compile time, one subclass per variation; Strategy varies it by *composition*, swappable at runtime, and the varying piece can be reused elsewhere. | **Chosen at runtime or shared across contexts? → Strategy. Fixed per subclass and sharing a lot of code? → Template Method.** |
| 10 | [Proxy](../02-structural/07-proxy.md) vs [Decorator](../02-structural/04-decorator.md) | Structurally indistinguishable: same interface, holds one wrappee, forwards. | Decorator *adds* responsibilities the caller opted into and stacks freely; Proxy *controls access* to a subject it often owns or creates, and is usually invisible to the caller. | **Is the wrapper about access/lifetime/location? → Proxy. Is it about extra behaviour the caller asked for? → Decorator.** |
| 11 | [Flyweight](../02-structural/06-flyweight.md) vs [Singleton](../01-creational/05-singleton.md) | Both hand back shared instances instead of new ones. | Singleton guarantees *exactly one* instance globally, usually for identity/coordination; Flyweight keeps *many* immutable shared instances keyed by intrinsic state, purely to save memory. | **One instance by design → Singleton. Many shared immutable instances to save memory → Flyweight.** |
| 12 | [Visitor](../03-behavioral/10-visitor.md) vs [Iterator](../03-behavioral/03-iterator.md) | Both are "go over a structure and do something to each element". | Iterator provides *traversal* and leaves the operation to the caller; Visitor provides *type-dispatched operations* and often relies on a traversal (sometimes an Iterator) to reach elements. | **Need to walk it? → Iterator. Need a different operation per concrete element type, added without editing those types? → Visitor.** |

Now the detail.

---

### 1. Strategy vs State

**They look alike because…** if you drew both on a whiteboard you would draw the same
three boxes: a `Context` holding a field of type `IThing`, and two or more concrete
`IThing`s. There is no structural difference at all.

**They differ because…** of *who owns the transition*. In Strategy, the client picks the
algorithm and the algorithm has no opinion about what should run next. In State, the state
objects themselves drive the machine forward: `PendingState.Approve()` puts the context
into `ApprovedState`. State encodes a *lifecycle*; Strategy encodes a *choice*.

A second tell: Strategy implementations are usually **stateless and interchangeable in any
order**. State implementations usually are **not** interchangeable — you cannot go from
`Sold` back to `Draft` without a rule saying so.

Automotive marketplace example. A listing has a lifecycle — that is State:

```csharp
// ---- State: the listing's lifecycle. Each state knows its legal successors. ----
public interface IListingState
{
    string Name { get; }
    void Submit(Listing listing);
    void Approve(Listing listing);
    void MarkSold(Listing listing);
}

public sealed class DraftState : IListingState
{
    public string Name => "Draft";
    public void Submit(Listing l)   => l.TransitionTo(new PendingReviewState());
    public void Approve(Listing l)  => throw new InvalidOperationException("Submit first.");
    public void MarkSold(Listing l) => throw new InvalidOperationException("Not live yet.");
}

public sealed class PendingReviewState : IListingState
{
    public string Name => "PendingReview";
    public void Submit(Listing l)   => throw new InvalidOperationException("Already submitted.");
    public void Approve(Listing l)  => l.TransitionTo(new LiveState());
    public void MarkSold(Listing l) => throw new InvalidOperationException("Not live yet.");
}

public sealed class LiveState : IListingState
{
    public string Name => "Live";
    public void Submit(Listing l)   => throw new InvalidOperationException("Already live.");
    public void Approve(Listing l)  { /* idempotent no-op */ }
    public void MarkSold(Listing l) => l.TransitionTo(new SoldState());
}

public sealed class SoldState : IListingState
{
    public string Name => "Sold";
    public void Submit(Listing l)   => throw new InvalidOperationException("Sold is terminal.");
    public void Approve(Listing l)  => throw new InvalidOperationException("Sold is terminal.");
    public void MarkSold(Listing l) { /* idempotent no-op */ }
}

public sealed class Listing
{
    private IListingState _state = new DraftState();
    public string StateName => _state.Name;

    public void TransitionTo(IListingState next) => _state = next;   // states call this
    public void Submit()   => _state.Submit(this);
    public void Approve()  => _state.Approve(this);
    public void MarkSold() => _state.MarkSold(this);
}
```

Notice `TransitionTo` — the states are steering the context. That is the fingerprint of
State.

Now pricing. Nothing about a pricing rule says what pricing rule comes next. That is
Strategy:

```typescript
// ---- Strategy: how to price a listing. Caller picks; strategies are order-independent.
interface PricingStrategy {
  readonly name: string;
  price(car: Car): number;
}

class MarketAveragePricing implements PricingStrategy {
  readonly name = "market-average";
  constructor(private readonly marketData: MarketDataPort) {}
  price(car: Car): number {
    const comps = this.marketData.comparables(car.make, car.model, car.year);
    const avg = comps.reduce((s, c) => s + c.askingPrice, 0) / Math.max(comps.length, 1);
    return Math.round(avg * this.conditionFactor(car));
  }
  private conditionFactor(car: Car): number {
    return car.condition === "excellent" ? 1.06 : car.condition === "poor" ? 0.88 : 1.0;
  }
}

class DepreciationCurvePricing implements PricingStrategy {
  readonly name = "depreciation-curve";
  price(car: Car): number {
    const age = new Date().getFullYear() - car.year;
    const residual = Math.pow(0.85, age);           // 15% per year, compounding
    const kmPenalty = Math.min(car.odometerKm / 200_000, 0.35);
    return Math.round(car.originalMsrp * residual * (1 - kmPenalty));
  }
}

class DealerFloorPricing implements PricingStrategy {
  readonly name = "dealer-floor";
  constructor(private readonly floor: number) {}
  price(car: Car): number {
    return Math.max(this.floor, Math.round(car.originalMsrp * 0.4));
  }
}

class PriceQuoteService {
  constructor(private strategy: PricingStrategy) {}
  useStrategy(s: PricingStrategy): void { this.strategy = s; }   // the CALLER decides
  quote(car: Car): { amount: number; method: string } {
    return { amount: this.strategy.price(car), method: this.strategy.name };
  }
}
```

The service swaps strategies; a strategy never swaps itself.

> **Decision rule:** *Does the swapped object decide what comes next? → State. Does the
> caller decide? → Strategy.*

---

### 2. Strategy vs Bridge

**They look alike because…** both are "object A holds a reference to interface B and
delegates work to it". In a single snapshot of code they are the same line:
`private readonly IRenderer _renderer;`

**They differ because…** of *scale and reason*. Strategy is tactical: one operation inside
one class has several implementations. Bridge is architectural: you have two dimensions of
variation that would otherwise multiply into a combinatorial class explosion, so you split
the hierarchy in two and let them vary independently.

The classic smell that calls for Bridge is a class list like:

```
  WebCarCard, WebBikeCard, WebTruckCard,
  MobileCarCard, MobileBikeCard, MobileTruckCard,
  EmailCarCard, EmailBikeCard, EmailTruckCard        // 3 × 3 = 9 and growing
```

Two dimensions: *vehicle kind* and *render target*. Bridge turns 3 × 3 into 3 + 3:

```csharp
// ---- Bridge: the IMPLEMENTOR side (render targets) ----
public interface IListingRenderer
{
    string Heading(string text);
    string KeyValue(string key, string value);
    string Cta(string label, string url);
}

public sealed class HtmlRenderer : IListingRenderer
{
    public string Heading(string t) => $"<h2>{System.Net.WebUtility.HtmlEncode(t)}</h2>";
    public string KeyValue(string k, string v) => $"<dt>{k}</dt><dd>{v}</dd>";
    public string Cta(string label, string url) => $"<a class=\"cta\" href=\"{url}\">{label}</a>";
}

public sealed class PlainTextRenderer : IListingRenderer
{
    public string Heading(string t) => t.ToUpperInvariant() + "\n" + new string('=', t.Length);
    public string KeyValue(string k, string v) => $"{k,-14}: {v}";
    public string Cta(string label, string url) => $"{label} -> {url}";
}

public sealed class AmpRenderer : IListingRenderer
{
    public string Heading(string t) => $"<h2 class=\"amp-h\">{t}</h2>";
    public string KeyValue(string k, string v) => $"<div class=\"amp-kv\"><b>{k}</b> {v}</div>";
    public string Cta(string label, string url) => $"<amp-link href=\"{url}\">{label}</amp-link>";
}

// ---- Bridge: the ABSTRACTION side (what kind of listing) ----
public abstract class ListingCard
{
    protected readonly IListingRenderer R;           // the bridge
    protected ListingCard(IListingRenderer renderer) => R = renderer;
    public abstract string Render();
}

public sealed class CarCard : ListingCard
{
    private readonly Car _car;
    public CarCard(Car car, IListingRenderer r) : base(r) => _car = car;
    public override string Render() => string.Join("\n",
        R.Heading($"{_car.Year} {_car.Make} {_car.Model}"),
        R.KeyValue("Odometer", $"{_car.OdometerKm:N0} km"),
        R.KeyValue("Fuel", _car.FuelType),
        R.Cta("View listing", $"/cars/{_car.Id}"));
}

public sealed class CommercialVehicleCard : ListingCard
{
    private readonly Truck _truck;
    public CommercialVehicleCard(Truck t, IListingRenderer r) : base(r) => _truck = t;
    public override string Render() => string.Join("\n",
        R.Heading($"{_truck.Year} {_truck.Make} {_truck.Model}"),
        R.KeyValue("Payload", $"{_truck.PayloadKg:N0} kg"),
        R.KeyValue("Axles", _truck.AxleCount.ToString()),
        R.Cta("Request quote", $"/commercial/{_truck.Id}"));
}
```

Adding a fourth render target now costs one class, not four. That "n + m instead of n × m"
outcome is Bridge's whole reason to exist. Strategy never promises that; it just makes one
method pluggable.

> **Decision rule:** *One varying method → Strategy. Two whole hierarchies multiplying →
> Bridge.*

---

### 3. Adapter vs Decorator vs Proxy vs Facade

These four are the wrapper family and they cause more pull-request arguments than the
other nineteen combined. Put them in one table on the two axes that matter:

```
                    │  Interface the wrapper exposes
                    │  SAME as wrappee        DIFFERENT from wrappee
 ───────────────────┼──────────────────────────────────────────────────
  Wraps ONE object  │  Decorator (adds)        Adapter (converts)
                    │  Proxy     (controls)    
 ───────────────────┼──────────────────────────────────────────────────
  Wraps MANY objects│  (rare)                  Facade (simplifies)
```

Four wrappers around the same idea — a vehicle-history report provider — in TypeScript:

```typescript
// The interface OUR application speaks.
interface VehicleHistoryProvider {
  fetch(vin: string): Promise<HistoryReport>;
}

// ---------------------------------------------------------------- ADAPTER
// The vendor speaks a different language. We convert it. Behaviour unchanged.
class VinCheckAdapter implements VehicleHistoryProvider {
  constructor(private readonly vendor: VinCheckSoapClient) {}

  async fetch(vin: string): Promise<HistoryReport> {
    const raw = await this.vendor.RequestReportByChassis({
      ChassisNumber: vin,
      IncludeAccidentBlocks: "Y",
      Locale: "en-IN",
    });
    return {
      vin,
      accidents: raw.AccidentBlocks.map(b => ({
        date: new Date(b.EventDate),
        severity: b.SeverityCode === "1" ? "minor" : b.SeverityCode === "2" ? "moderate" : "severe",
        description: b.FreeText ?? "",
      })),
      odometerReadings: raw.OdoBlocks.map(o => ({ date: new Date(o.At), km: Number(o.Value) })),
      titleBranded: raw.TitleFlag === "BRANDED",
    };
  }
}

// -------------------------------------------------------------- DECORATOR
// Same interface. Adds behaviour the caller opted into. Stacks freely.
class RedactingHistoryProvider implements VehicleHistoryProvider {
  constructor(private readonly inner: VehicleHistoryProvider) {}
  async fetch(vin: string): Promise<HistoryReport> {
    const report = await this.inner.fetch(vin);
    return {
      ...report,
      accidents: report.accidents.map(a => ({
        ...a,
        description: a.description.replace(/\b[A-Z][a-z]+ [A-Z][a-z]+\b/g, "[name redacted]"),
      })),
    };
  }
}

class MetricsHistoryProvider implements VehicleHistoryProvider {
  constructor(
    private readonly inner: VehicleHistoryProvider,
    private readonly metrics: MetricsSink,
  ) {}
  async fetch(vin: string): Promise<HistoryReport> {
    const started = performance.now();
    try {
      const r = await this.inner.fetch(vin);
      this.metrics.timing("history.fetch.ok", performance.now() - started);
      return r;
    } catch (err) {
      this.metrics.timing("history.fetch.fail", performance.now() - started);
      throw err;
    }
  }
}

// ------------------------------------------------------------------ PROXY
// Same interface. Controls ACCESS: caching, and it owns the cache lifetime.
class CachingHistoryProxy implements VehicleHistoryProvider {
  private readonly cache = new Map<string, { at: number; report: HistoryReport }>();

  constructor(
    private readonly real: VehicleHistoryProvider,
    private readonly ttlMs = 24 * 60 * 60 * 1000,
  ) {}

  async fetch(vin: string): Promise<HistoryReport> {
    const hit = this.cache.get(vin);
    if (hit && Date.now() - hit.at < this.ttlMs) return hit.report;   // real object never called
    const report = await this.real.fetch(vin);
    this.cache.set(vin, { at: Date.now(), report });
    return report;
  }
}

// ----------------------------------------------------------------- FACADE
// NEW, smaller interface over MANY collaborators. Not a VehicleHistoryProvider.
class ListingInspectionFacade {
  constructor(
    private readonly history: VehicleHistoryProvider,
    private readonly valuation: ValuationService,
    private readonly photos: PhotoQualityService,
    private readonly compliance: ComplianceRuleEngine,
  ) {}

  /** One call for the whole "is this listing OK to publish" question. */
  async inspect(listing: Listing): Promise<InspectionVerdict> {
    const [report, value, photoScore] = await Promise.all([
      this.history.fetch(listing.vin),
      this.valuation.estimate(listing.car),
      this.photos.score(listing.photoUrls),
    ]);
    const issues = this.compliance.evaluate({ listing, report, value, photoScore });
    return {
      publishable: issues.every(i => i.severity !== "blocking"),
      issues,
      suggestedPrice: value.amount,
    };
  }
}

// Composition at the edge of the app:
const provider: VehicleHistoryProvider =
  new CachingHistoryProxy(                 // controls access
    new MetricsHistoryProvider(            // adds behaviour
      new RedactingHistoryProvider(        // adds behaviour
        new VinCheckAdapter(soapClient)),  // converts the interface
      metrics));
```

Read that final composition top to bottom and you can name each wrapper by what it *does*,
even though `CachingHistoryProxy`, `MetricsHistoryProvider` and `RedactingHistoryProvider`
have byte-for-byte the same shape.

**They look alike because…** all four sit between a caller and a callee and all four
forward work inward.

**They differ because…**

- **Adapter** exists because two interfaces do not match. It is a translator. Behaviour
  should be unchanged.
- **Decorator** exists to add behaviour while preserving the contract, so wrappers stack.
- **Proxy** exists to control *access* — lazy creation, caching, remoting, permission
  checks, reference counting. It usually controls the subject's lifetime too.
- **Facade** exists to hide a *subsystem* behind a smaller, task-shaped interface. It
  introduces a new interface rather than preserving one.

> **Decision rules:**
> - *Interface changed → Adapter. Behaviour changed → Decorator.*
> - *Same interface, one subject, access control → Proxy. New interface, many subjects →
>   Facade.*

---

### 4. Factory Method vs Abstract Factory vs Builder

**They look alike because…** all three answer "how do I get an object without calling
`new` at the call site?"

**They differ because…** of *what varies*:

| | What varies | Mechanism | Returns |
|---|---|---|---|
| [Factory Method](../01-creational/01-factory-method.md) | Which single concrete product | Subclass overrides one method | One product, immediately |
| [Abstract Factory](../01-creational/02-abstract-factory.md) | Which whole *family* of products | Swap the factory object | Several related products, each immediately |
| [Builder](../01-creational/03-builder.md) | How the product is *assembled* | Call steps in sequence, then `Build()` | One complex product, at the end |

Factory Method — subclass decides the concrete product:

```csharp
public abstract class ListingImporter
{
    // The template of the import, fixed here.
    public ImportResult Import(Stream feed)
    {
        var parser = CreateParser();            // <-- the factory method
        var rows = parser.Parse(feed);
        var listings = rows.Select(ToListing).ToList();
        return new ImportResult(listings.Count, listings);
    }

    protected abstract IFeedParser CreateParser();   // subclasses fill this in
    protected abstract Listing ToListing(FeedRow row);
}

public sealed class DealerCsvImporter : ListingImporter
{
    protected override IFeedParser CreateParser() => new CsvFeedParser(delimiter: ',');
    protected override Listing ToListing(FeedRow row) => new Listing
    {
        Vin = row["vin"],
        Make = row["make"],
        Model = row["model"],
        Year = int.Parse(row["year"]),
        AskingPrice = decimal.Parse(row["price"]),
    };
}

public sealed class OemXmlImporter : ListingImporter
{
    protected override IFeedParser CreateParser() => new XmlFeedParser(rootElement: "Vehicles");
    protected override Listing ToListing(FeedRow row) => new Listing
    {
        Vin = row["ChassisNo"],
        Make = row["Brand"],
        Model = row["Variant"],
        Year = int.Parse(row["ModelYear"]),
        AskingPrice = decimal.Parse(row["ExShowroom"]),
    };
}
```

(Notice this example is *also* a Template Method — `Import` is the template. The two
patterns are near-inseparable; see the combinations section.)

Abstract Factory — one object producing a matched family:

```csharp
public interface IMarketplaceUiKit
{
    IPriceFormatter PriceFormatter();
    IDistanceFormatter DistanceFormatter();
    IDateFormatter DateFormatter();
    IDisclaimerBlock Disclaimer();
}

public sealed class IndiaUiKit : IMarketplaceUiKit
{
    public IPriceFormatter PriceFormatter()     => new IndianRupeeLakhFormatter();
    public IDistanceFormatter DistanceFormatter() => new KilometreFormatter();
    public IDateFormatter DateFormatter()       => new DayMonthYearFormatter();
    public IDisclaimerBlock Disclaimer()        => new IndiaRtoDisclaimer();
}

public sealed class UsUiKit : IMarketplaceUiKit
{
    public IPriceFormatter PriceFormatter()     => new UsDollarFormatter();
    public IDistanceFormatter DistanceFormatter() => new MileFormatter();
    public IDateFormatter DateFormatter()       => new MonthDayYearFormatter();
    public IDisclaimerBlock Disclaimer()        => new UsLemonLawDisclaimer();
}
```

The *point* is consistency: you can never accidentally pair rupees with miles, because the
kit hands out a matched set.

Builder — many steps, one product, validation at the end:

```typescript
class SearchQueryBuilder {
  private readonly filters: string[] = [];
  private readonly params: Record<string, unknown> = {};
  private sort: string = "relevance DESC";
  private page = 1;
  private size = 20;

  make(make: string): this      { this.filters.push("make = @make"); this.params.make = make; return this; }
  model(model: string): this    { this.filters.push("model = @model"); this.params.model = model; return this; }
  yearBetween(from: number, to: number): this {
    this.filters.push("year BETWEEN @yearFrom AND @yearTo");
    this.params.yearFrom = from; this.params.yearTo = to; return this;
  }
  priceUnder(max: number): this { this.filters.push("asking_price <= @maxPrice"); this.params.maxPrice = max; return this; }
  withinKm(km: number, lat: number, lng: number): this {
    this.filters.push("geo_distance(@lat, @lng, latitude, longitude) <= @radiusKm");
    this.params.lat = lat; this.params.lng = lng; this.params.radiusKm = km; return this;
  }
  sortBy(expr: "price-asc" | "price-desc" | "newest" | "relevance"): this {
    this.sort = { "price-asc": "asking_price ASC", "price-desc": "asking_price DESC",
                  "newest": "listed_at DESC", "relevance": "relevance DESC" }[expr];
    return this;
  }
  paginate(page: number, size: number): this { this.page = page; this.size = Math.min(size, 100); return this; }

  build(): { sql: string; params: Record<string, unknown> } {
    if (this.page < 1) throw new Error("page must be >= 1");            // validation lives HERE
    const where = this.filters.length ? `WHERE ${this.filters.join(" AND ")}` : "";
    const offset = (this.page - 1) * this.size;
    return {
      sql: `SELECT id, vin, make, model, year, asking_price
            FROM listings ${where}
            ORDER BY ${this.sort}
            OFFSET ${offset} ROWS FETCH NEXT ${this.size} ROWS ONLY`,
      params: this.params,
    };
  }
}

const { sql, params } = new SearchQueryBuilder()
  .make("Maruti Suzuki")
  .yearBetween(2018, 2023)
  .priceUnder(900_000)
  .withinKm(50, 19.076, 72.8777)
  .sortBy("price-asc")
  .paginate(1, 24)
  .build();
```

> **Decision rules:**
> - *One product varied by subclass → Factory Method.*
> - *A matched family varied by swapping a factory object → Abstract Factory.*
> - *One product that needs many optional steps and end-of-construction validation →
>   Builder.*

---

### 5. Composite vs Decorator

**They look alike because…** both are wrappers implementing the same interface as the thing
they wrap, and both recurse.

**They differ because…** of *arity and intent*. A Composite holds a **collection** of
children and exists so clients can treat "one thing" and "a group of things" identically. A
Decorator holds **exactly one** wrappee and exists to add behaviour. Composite is about
*structure*; Decorator is about *behaviour*.

```typescript
// ---- COMPOSITE: many children, tree semantics ----
interface PackageItem {
  priceInPaise(): number;
  describe(indent?: string): string;
}

class AddOn implements PackageItem {                    // leaf
  constructor(private readonly label: string, private readonly paise: number) {}
  priceInPaise(): number { return this.paise; }
  describe(indent = ""): string { return `${indent}- ${this.label}: ₹${this.paise / 100}`; }
}

class Bundle implements PackageItem {                   // composite: MANY children
  private readonly children: PackageItem[] = [];
  constructor(private readonly label: string, private readonly discountPct = 0) {}
  add(item: PackageItem): this { this.children.push(item); return this; }
  priceInPaise(): number {
    const gross = this.children.reduce((s, c) => s + c.priceInPaise(), 0);
    return Math.round(gross * (1 - this.discountPct / 100));
  }
  describe(indent = ""): string {
    return [`${indent}+ ${this.label} (₹${this.priceInPaise() / 100})`,
            ...this.children.map(c => c.describe(indent + "  "))].join("\n");
  }
}

// ---- DECORATOR: exactly one wrappee, adds behaviour ----
class GstApplied implements PackageItem {               // ONE child
  constructor(private readonly inner: PackageItem, private readonly ratePct = 18) {}
  priceInPaise(): number { return Math.round(this.inner.priceInPaise() * (1 + this.ratePct / 100)); }
  describe(indent = ""): string {
    return `${this.inner.describe(indent)}\n${indent}  (incl. ${this.ratePct}% GST)`;
  }
}

const listingPackage = new GstApplied(                  // decorator on top
  new Bundle("Premium seller pack", 10)                 // composite underneath
    .add(new AddOn("Featured placement, 30 days", 249900))
    .add(new AddOn("Professional photoshoot", 149900))
    .add(new Bundle("Verification pack")
      .add(new AddOn("RC verification", 29900))
      .add(new AddOn("Inspection report", 99900))));
```

That last block is the clean way to see it: the Composite builds the tree, the Decorator
wraps the tree.

> **Decision rule:** *Many children, tree semantics → Composite. Exactly one child, added
> behaviour → Decorator.*

---

### 6. Chain of Responsibility vs Decorator

**They look alike because…** both produce a linked chain of same-interface objects, each of
which can run code before and after passing control inward. Express/ASP.NET middleware is
usually described as both, and people argue about it endlessly.

**They differ because…** a Decorator's contract is that it *always* delegates — it is
adding to a call that will definitely happen. A Chain link is allowed to **not** forward:
handle it and return, or reject and stop. "Short-circuiting is legal" is the whole
distinction.

```csharp
// ---- CHAIN OF RESPONSIBILITY: links may stop the request ----
public abstract class ListingGuard
{
    private ListingGuard? _next;
    public ListingGuard Then(ListingGuard next) { _next = next; return next; }

    public GuardResult Check(Listing listing)
    {
        var mine = Evaluate(listing);
        if (!mine.Passed) return mine;                       // <-- SHORT CIRCUIT
        return _next is null ? GuardResult.Ok : _next.Check(listing);
    }

    protected abstract GuardResult Evaluate(Listing listing);
}

public sealed class VinFormatGuard : ListingGuard
{
    protected override GuardResult Evaluate(Listing l) =>
        l.Vin.Length == 17 && l.Vin.All(char.IsLetterOrDigit)
            ? GuardResult.Ok
            : GuardResult.Fail("VIN must be 17 alphanumeric characters.");
}

public sealed class PhotoCountGuard : ListingGuard
{
    protected override GuardResult Evaluate(Listing l) =>
        l.PhotoUrls.Count >= 4 ? GuardResult.Ok : GuardResult.Fail("At least 4 photos required.");
}

public sealed class PriceSanityGuard : ListingGuard
{
    protected override GuardResult Evaluate(Listing l) =>
        l.AskingPrice > 10_000m && l.AskingPrice < 100_000_000m
            ? GuardResult.Ok
            : GuardResult.Fail("Asking price is outside the plausible range.");
}

public sealed class DuplicateVinGuard : ListingGuard
{
    private readonly IListingRepository _repo;
    public DuplicateVinGuard(IListingRepository repo) => _repo = repo;
    protected override GuardResult Evaluate(Listing l) =>
        _repo.ExistsActiveWithVin(l.Vin) ? GuardResult.Fail("An active listing already uses this VIN.")
                                         : GuardResult.Ok;
}

// wiring
var guards = new VinFormatGuard();
guards.Then(new PhotoCountGuard())
      .Then(new PriceSanityGuard())
      .Then(new DuplicateVinGuard(repo));

var verdict = guards.Check(listing);   // stops at the first failure
```

Compare with the Decorator from section 3: `MetricsHistoryProvider` has no branch that
returns without calling `inner.fetch`. It cannot refuse.

Middleware pipelines, honestly, are a hybrid: they are shaped like Decorators, but because
any middleware may return a 401 without calling `next()`, they behave like a Chain. Call
them a chain-shaped pipeline and move on; the useful question in review is "can this link
swallow the request, and is that documented?"

> **Decision rule:** *Any link allowed to not forward? → Chain of Responsibility. Everyone
> always forwards? → Decorator.*

---

### 7. Mediator vs Observer

**They look alike because…** both put a third object between components so those components
do not reference each other. Both reduce an n × n web to something manageable.

**They differ because…** of *where the knowledge lives*. In Observer the subject knows
nothing about subscribers beyond "they implement the notify interface" — it broadcasts and
walks away. In Mediator the hub deliberately *knows the participants and the protocol*: "if
the price field changes, recompute the EMI widget and disable the Submit button until the
recalculation returns."

Observer — fan-out, sender does not care who listens:

```typescript
type Handler<T> = (event: T) => void | Promise<void>;

class DomainEvents {
  private readonly handlers = new Map<string, Handler<any>[]>();

  on<T>(eventName: string, handler: Handler<T>): () => void {
    const list = this.handlers.get(eventName) ?? [];
    list.push(handler);
    this.handlers.set(eventName, list);
    return () => {                                     // unsubscribe
      const cur = this.handlers.get(eventName) ?? [];
      this.handlers.set(eventName, cur.filter(h => h !== handler));
    };
  }

  async emit<T>(eventName: string, event: T): Promise<void> {
    for (const h of this.handlers.get(eventName) ?? []) {
      try { await h(event); }
      catch (err) { console.error(`handler for ${eventName} failed`, err); }  // one bad sub must not kill the rest
    }
  }
}

// publisher has no idea these exist
events.on<ListingPublished>("listing.published", e => searchIndex.upsert(e.listingId));
events.on<ListingPublished>("listing.published", e => rabbit.publish("listing.published", e));
events.on<ListingPublished>("listing.published", e => sellerMailer.confirmLive(e.sellerId, e.listingId));
```

Mediator — the hub owns the rules:

```typescript
class ListingFormMediator {
  constructor(
    private readonly priceField: NumberField,
    private readonly yearField: NumberField,
    private readonly emiWidget: EmiWidget,
    private readonly submitButton: Button,
    private readonly valuation: ValuationService,
  ) {
    // The mediator wires everything; the widgets know only the mediator.
    priceField.onChange = () => this.notify("price");
    yearField.onChange  = () => this.notify("year");
  }

  private async notify(source: "price" | "year"): Promise<void> {
    // THIS is the protocol logic that makes it a Mediator, not an event bus.
    if (source === "price" || source === "year") {
      this.submitButton.disabled = true;
      this.emiWidget.showSpinner();

      const price = this.priceField.value;
      const year = this.yearField.value;

      if (price <= 0 || year < 1950) {
        this.emiWidget.showMessage("Enter a valid price and year");
        this.submitButton.disabled = true;
        return;
      }

      const emi = await this.valuation.estimateEmi(price, year);
      this.emiWidget.render(emi);
      this.submitButton.disabled = false;
    }
  }
}
```

A rule of thumb that has never let me down: **if you delete the middle object, does the
business rule disappear?** If yes it was a Mediator (it held logic). If the components just
stop being notified but no rule is lost, it was an Observer (it held wiring).

> **Decision rule:** *Does the middle object contain the "if A then B" logic? → Mediator. Is
> the middle just a mailing list? → Observer.*

---

### 8. Command vs Strategy

**They look alike because…** both are a small object with essentially one method, injected
somewhere and invoked.

**They differ because…** a Strategy is *parameterless in spirit* — it receives the data as
arguments and answers a question inside someone else's algorithm. A Command is a **fully
bound request**: it carries its receiver and its arguments, so it can be put in a list,
serialised, sent through RabbitMQ, retried, logged, and often undone.

The tell: if you can meaningfully write `List<ICommand> history` and replay it later, it is
a Command. `List<IPricingStrategy> history` is meaningless.

```csharp
public interface ICommand
{
    string Describe();
    Task ExecuteAsync(CancellationToken ct = default);
    Task UndoAsync(CancellationToken ct = default);
}

public sealed class ReducePriceCommand : ICommand
{
    private readonly IListingRepository _repo;
    private readonly Guid _listingId;
    private readonly decimal _newPrice;
    private decimal _previousPrice;            // captured so we can undo

    public ReducePriceCommand(IListingRepository repo, Guid listingId, decimal newPrice)
    {
        _repo = repo; _listingId = listingId; _newPrice = newPrice;
    }

    public string Describe() => $"Reduce price of {_listingId} to {_newPrice:C}";

    public async Task ExecuteAsync(CancellationToken ct = default)
    {
        var listing = await _repo.GetAsync(_listingId, ct);
        _previousPrice = listing.AskingPrice;
        listing.AskingPrice = _newPrice;
        await _repo.SaveAsync(listing, ct);
    }

    public async Task UndoAsync(CancellationToken ct = default)
    {
        var listing = await _repo.GetAsync(_listingId, ct);
        listing.AskingPrice = _previousPrice;
        await _repo.SaveAsync(listing, ct);
    }
}

public sealed class CommandBus
{
    private readonly Stack<ICommand> _done = new();
    private readonly Stack<ICommand> _undone = new();

    public async Task SendAsync(ICommand cmd, CancellationToken ct = default)
    {
        await cmd.ExecuteAsync(ct);
        _done.Push(cmd);
        _undone.Clear();
    }

    public async Task UndoAsync(CancellationToken ct = default)
    {
        if (_done.Count == 0) return;
        var cmd = _done.Pop();
        await cmd.UndoAsync(ct);
        _undone.Push(cmd);
    }

    public async Task RedoAsync(CancellationToken ct = default)
    {
        if (_undone.Count == 0) return;
        var cmd = _undone.Pop();
        await cmd.ExecuteAsync(ct);
        _done.Push(cmd);
    }

    public IEnumerable<string> AuditTrail() => _done.Reverse().Select(c => c.Describe());
}
```

Everything in `CommandBus` — the stacks, the audit trail, the redo — is impossible with a
Strategy, because a Strategy does not carry its arguments.

> **Decision rule:** *Do you need to store, queue, log or undo it? → Command. Is it just a
> pluggable "how"? → Strategy.*

---

### 9. Template Method vs Strategy

**They look alike because…** both exist to vary one step of an otherwise fixed algorithm.
Both let you write "the outline is the same, this bit differs".

**They differ because…** Template Method uses **inheritance** (the variation is baked into
a subclass at compile time, and the subclass cannot be reused by a different algorithm);
Strategy uses **composition** (the variation is a separate object, swappable at runtime,
reusable across contexts, testable in isolation).

Same requirement, both ways:

```csharp
// ---- TEMPLATE METHOD: inheritance. The skeleton is in the base class. ----
public abstract class FeedExportJob
{
    public async Task<ExportSummary> RunAsync(IReadOnlyList<Listing> listings, Stream output)
    {
        var filtered = Filter(listings);                 // hook
        await WriteHeaderAsync(output);                  // hook
        foreach (var listing in filtered)
            await WriteRowAsync(output, listing);        // hook
        await WriteFooterAsync(output);                  // hook with default
        return new ExportSummary(filtered.Count, GetType().Name);
    }

    protected virtual IReadOnlyList<Listing> Filter(IReadOnlyList<Listing> all) => all;
    protected abstract Task WriteHeaderAsync(Stream s);
    protected abstract Task WriteRowAsync(Stream s, Listing l);
    protected virtual Task WriteFooterAsync(Stream s) => Task.CompletedTask;
}

public sealed class GoogleVehicleAdsExport : FeedExportJob
{
    protected override IReadOnlyList<Listing> Filter(IReadOnlyList<Listing> all) =>
        all.Where(l => l.PhotoUrls.Count >= 3 && l.AskingPrice > 0).ToList();

    protected override async Task WriteHeaderAsync(Stream s) =>
        await s.WriteLineAsync("vehicle_id,make,model,year,price,image_link");

    protected override async Task WriteRowAsync(Stream s, Listing l) =>
        await s.WriteLineAsync($"{l.Id},{l.Make},{l.Model},{l.Year},{l.AskingPrice},{l.PhotoUrls[0]}");
}

public sealed class PartnerXmlExport : FeedExportJob
{
    protected override Task WriteHeaderAsync(Stream s) => s.WriteLineAsync("<vehicles>");
    protected override Task WriteRowAsync(Stream s, Listing l) =>
        s.WriteLineAsync($"  <vehicle vin=\"{l.Vin}\" price=\"{l.AskingPrice}\" />");
    protected override Task WriteFooterAsync(Stream s) => s.WriteLineAsync("</vehicles>");
}
```

```csharp
// ---- STRATEGY: composition. The skeleton is in a concrete class that HOLDS the varying part.
public interface IFeedFormat
{
    Task WriteHeaderAsync(Stream s);
    Task WriteRowAsync(Stream s, Listing l);
    Task WriteFooterAsync(Stream s);
}

public interface IListingFilter
{
    IReadOnlyList<Listing> Apply(IReadOnlyList<Listing> all);
}

public sealed class FeedExporter                        // NOT abstract, never subclassed
{
    private readonly IFeedFormat _format;
    private readonly IListingFilter _filter;

    public FeedExporter(IFeedFormat format, IListingFilter filter)
    {
        _format = format; _filter = filter;
    }

    public async Task<ExportSummary> RunAsync(IReadOnlyList<Listing> listings, Stream output)
    {
        var filtered = _filter.Apply(listings);
        await _format.WriteHeaderAsync(output);
        foreach (var l in filtered) await _format.WriteRowAsync(output, l);
        await _format.WriteFooterAsync(output);
        return new ExportSummary(filtered.Count, _format.GetType().Name);
    }
}

// Now format and filter vary INDEPENDENTLY — 3 formats × 4 filters = 7 classes, not 12.
var exporter = new FeedExporter(new CsvFeedFormat(), new HasPhotosAndPriceFilter());
```

Choose Template Method when the variations genuinely share a lot of code and are fixed per
type. Choose Strategy when you need runtime choice, independent variation of two axes, or
easy unit tests. In modern C# and TypeScript, Strategy wins most of the time — but
Template Method remains excellent for framework base classes where you *want* to constrain
the shape (think a `BackgroundService` subclass, or a base integration-test fixture).

> **Decision rule:** *Chosen at runtime, or reused across contexts, or two axes varying? →
> Strategy. Fixed per subclass with lots of shared code? → Template Method.*

---

### 10. Proxy vs Decorator

This is the hardest pair in the catalogue, because there is **no structural difference at
all**. Same interface, one wrappee, forwards calls. If you only look at the class diagram
you cannot tell them apart, ever.

**They look alike because…** they *are* alike, structurally. Identical.

**They differ because…** of intent, and three practical consequences fall out of that
intent:

| | Decorator | Proxy |
|---|---|---|
| Purpose | Add responsibilities | Control access |
| Who creates the wrappee | The client, and passes it in | Often the proxy itself (lazy init) |
| Caller awareness | Caller deliberately composed it | Caller usually does not know it exists |
| Stacking | Designed to stack, order matters | Usually one, sometimes a small fixed set |
| Does it always call inward? | Yes | Not necessarily — a cache hit or a denied permission stops it |

```csharp
// ---- DECORATOR: caller opted in, always forwards, stacks ----
public sealed class RetryingListingRepository : IListingRepository
{
    private readonly IListingRepository _inner;
    private readonly int _attempts;

    public RetryingListingRepository(IListingRepository inner, int attempts = 3)
    {
        _inner = inner;                              // client GAVE us the inner
        _attempts = attempts;
    }

    public async Task<Listing> GetAsync(Guid id, CancellationToken ct = default)
    {
        for (var i = 1; ; i++)
        {
            try { return await _inner.GetAsync(id, ct); }     // always reaches inward
            catch (SqlException) when (i < _attempts)
            {
                await Task.Delay(TimeSpan.FromMilliseconds(100 * Math.Pow(2, i)), ct);
            }
        }
    }
    // ... other members forward too
}

// ---- PROXY: controls access, may never touch the subject, creates it itself ----
public sealed class AuthorizingListingRepositoryProxy : IListingRepository
{
    private readonly Lazy<IListingRepository> _real;     // proxy OWNS creation
    private readonly ICurrentUser _user;

    public AuthorizingListingRepositoryProxy(Func<IListingRepository> factory, ICurrentUser user)
    {
        _real = new Lazy<IListingRepository>(factory);
        _user = user;
    }

    public async Task<Listing> GetAsync(Guid id, CancellationToken ct = default)
    {
        if (!_user.HasPermission("listings.read"))
            throw new UnauthorizedAccessException();          // subject NEVER touched
        return await _real.Value.GetAsync(id, ct);
    }
    // ... other members gated too
}
```

If you find yourself unable to decide, ask: *would a reviewer be surprised to learn this
wrapper is in the pipeline?* A caching or auth wrapper that got silently injected by the DI
container is a Proxy. A retry or audit wrapper the developer consciously composed is a
Decorator. And if it genuinely is both — a caching wrapper that the caller explicitly
composed — name it for the dominant concern and stop arguing.

> **Decision rule:** *Access, lifetime or location → Proxy. Extra behaviour the caller asked
> for → Decorator.*

---

### 11. Flyweight vs Singleton

**They look alike because…** in both, calling for "an object" hands you back one that
already exists rather than a fresh allocation. Both involve a registry or a static factory.

**They differ because…** Singleton is about **identity and coordination** — there is
conceptually exactly one of this thing in the process, and code depends on that. Flyweight
is about **memory** — there may be millions of logical objects, but the parts that repeat
are shared, and the parts that differ are passed in as extrinsic state.

```typescript
// ---- FLYWEIGHT: many shared immutable instances, keyed by intrinsic state ----
interface VehicleSpec {                         // intrinsic: identical for every car of this model
  readonly make: string;
  readonly model: string;
  readonly variant: string;
  readonly fuelType: string;
  readonly engineCc: number;
  readonly seats: number;
  readonly bodyStyle: string;
}

class VehicleSpecFactory {
  private readonly pool = new Map<string, VehicleSpec>();

  get(make: string, model: string, variant: string, fuelType: string,
      engineCc: number, seats: number, bodyStyle: string): VehicleSpec {
    const key = `${make}|${model}|${variant}|${fuelType}|${engineCc}|${seats}|${bodyStyle}`;
    let spec = this.pool.get(key);
    if (!spec) {
      spec = Object.freeze({ make, model, variant, fuelType, engineCc, seats, bodyStyle });
      this.pool.set(key, spec);
    }
    return spec;                                 // MANY distinct instances live here
  }

  get pooledCount(): number { return this.pool.size; }
}

class ListingRow {                               // extrinsic state lives per listing
  constructor(
    readonly id: string,
    readonly spec: VehicleSpec,                  // shared
    readonly vin: string,                        // unique
    readonly odometerKm: number,                 // unique
    readonly askingPrice: number,                // unique
    readonly sellerId: string,                   // unique
  ) {}
}

// 1,000,000 listings but maybe 4,000 distinct specs -> the specs cost ~4,000 objects, not 1,000,000.
```

```csharp
// ---- SINGLETON: exactly one, by design, because identity matters ----
public sealed class FeatureFlagRegistry
{
    private static readonly Lazy<FeatureFlagRegistry> Instance =
        new(() => new FeatureFlagRegistry(), LazyThreadSafetyMode.ExecutionAndPublication);

    public static FeatureFlagRegistry Current => Instance.Value;

    private readonly ConcurrentDictionary<string, bool> _flags = new();
    private FeatureFlagRegistry() { }

    public bool IsEnabled(string key) => _flags.TryGetValue(key, out var v) && v;
    public void Set(string key, bool value) => _flags[key] = value;
}
```

Two more practical differences worth knowing:

1. **Mutability.** Flyweights must be immutable (they are shared across contexts that know
   nothing about each other). Singletons are often mutable — that is usually their problem.
2. **Testing.** A Flyweight factory is trivially replaceable per test. A classic Singleton
   is global state and is the reason people prefer a DI container's *singleton lifetime*,
   which gives you one-per-container instead of one-per-process. In C# and modern
   TypeScript, prefer `services.AddSingleton<T>()` over a hand-rolled static `Instance`.

> **Decision rule:** *One instance because identity demands it → Singleton. Many shared
> immutable instances because memory demands it → Flyweight.*

---

### 12. Visitor vs Iterator

**They look alike because…** both show up when you have a collection or tree and want to
"do something to everything in it". Tutorials for both start with "walk the structure".

**They differ because…** Iterator's product is a **sequence** — it answers "what is next?"
and the caller decides what to do. Visitor's product is **double dispatch** — it answers
"what operation applies to *this concrete type*?" and lets you add whole new operations
without touching the element classes.

Iterator is about *access*. Visitor is about *extension* along the operations axis.

```typescript
// ---- ITERATOR: traversal, caller supplies the operation ----
interface InventoryNode {
  accept<T>(visitor: InventoryVisitor<T>): T;      // for the Visitor below
}

class Dealership implements InventoryNode, Iterable<InventoryNode> {
  readonly children: InventoryNode[] = [];
  constructor(readonly name: string) {}
  add(node: InventoryNode): this { this.children.push(node); return this; }

  *[Symbol.iterator](): Iterator<InventoryNode> {   // depth-first traversal
    yield this;
    for (const child of this.children) {
      if (Symbol.iterator in Object(child)) {
        yield* (child as unknown as Iterable<InventoryNode>);
      } else {
        yield child;
      }
    }
  }

  accept<T>(v: InventoryVisitor<T>): T { return v.visitDealership(this); }
}

class CarUnit implements InventoryNode {
  constructor(readonly vin: string, readonly price: number, readonly year: number) {}
  accept<T>(v: InventoryVisitor<T>): T { return v.visitCar(this); }
}

class BikeUnit implements InventoryNode {
  constructor(readonly vin: string, readonly price: number, readonly cc: number) {}
  accept<T>(v: InventoryVisitor<T>): T { return v.visitBike(this); }
}

// Using the ITERATOR: the caller writes the logic inline.
function countUnits(root: Dealership): number {
  let n = 0;
  for (const node of root) if (!(node instanceof Dealership)) n++;
  return n;
}
```

```typescript
// ---- VISITOR: operations added without editing CarUnit / BikeUnit ----
interface InventoryVisitor<T> {
  visitDealership(d: Dealership): T;
  visitCar(c: CarUnit): T;
  visitBike(b: BikeUnit): T;
}

class TotalValueVisitor implements InventoryVisitor<number> {
  visitDealership(d: Dealership): number {
    return d.children.reduce((sum, c) => sum + c.accept(this), 0);
  }
  visitCar(c: CarUnit): number { return c.price; }
  visitBike(b: BikeUnit): number { return b.price; }
}

class RoadTaxVisitor implements InventoryVisitor<number> {   // different rules per TYPE
  visitDealership(d: Dealership): number {
    return d.children.reduce((sum, c) => sum + c.accept(this), 0);
  }
  visitCar(c: CarUnit): number { return c.price * (c.price > 1_000_000 ? 0.14 : 0.09); }
  visitBike(b: BikeUnit): number { return b.price * (b.cc > 350 ? 0.12 : 0.07); }
}

class ExportRowVisitor implements InventoryVisitor<string[]> {
  visitDealership(d: Dealership): string[] { return d.children.flatMap(c => c.accept(this)); }
  visitCar(c: CarUnit): string[] { return [`car,${c.vin},${c.year},${c.price}`]; }
  visitBike(b: BikeUnit): string[] { return [`bike,${b.vin},${b.cc},${b.price}`]; }
}

const total = root.accept(new TotalValueVisitor());
const tax   = root.accept(new RoadTaxVisitor());
const csv   = root.accept(new ExportRowVisitor()).join("\n");
```

Adding a fourth operation costs one new visitor class and zero edits to `CarUnit`. Adding a
fourth *element type* costs an edit to every visitor — that is Visitor's famous trade-off,
and it is why Visitor suits stable type hierarchies (AST nodes, inventory kinds) and not
churning ones.

> **Decision rule:** *Need to walk it? → Iterator. Need a different operation per concrete
> type, added without editing those types? → Visitor.*

---

## 🧬 Structurally identical, different intent

This is the section to internalise. **A design pattern is named by its intent, not by its
class diagram.** Several patterns produce the exact same picture, and no amount of staring
at UML will separate them.

The identical-shape clusters:

```
  SHAPE A: "Context holds one interface reference and delegates"
  ┌──────────┐        ┌───────────────┐
  │ Context  │──────> │  <<interface>>│ <──── ConcreteA, ConcreteB
  └──────────┘        └───────────────┘
     Strategy  ·  State  ·  Bridge  ·  Command(receiver)
     Distinguished by: who chooses the concrete? does it self-transition?
                       is it one method or two hierarchies?

  SHAPE B: "Wrapper implements the same interface as its wrappee and forwards"
  ┌─────────────┐      ┌───────────────┐
  │  Wrapper    │─────>│  <<interface>>│ <──── RealThing
  │(implements ─┼──────┘               │
  │  the iface) │      └───────────────┘
  └─────────────┘
     Decorator  ·  Proxy  ·  Chain of Responsibility link  ·  (Composite with one child)
     Distinguished by: adds behaviour? controls access? may refuse to forward?
                       holds many children?

  SHAPE C: "Base class calls abstract methods the subclass fills in"
  ┌──────────────┐
  │  AbstractX   │  publicMethod() { stepA(); stepB(); }
  │  stepA() abs │
  │  stepB() abs │ <──── ConcreteX1, ConcreteX2
  └──────────────┘
     Template Method  ·  Factory Method
     Distinguished by: does the hook RETURN an object (Factory Method)
                       or PERFORM a step (Template Method)?
```

Why does this happen? Because object-oriented design has a small number of mechanisms —
delegation, composition, inheritance, polymorphism — and 23 patterns must be built from
them. The mechanisms repeat; the *problems* do not.

**Practical consequences you should actually change your behaviour over:**

1. **Name your classes after the intent, not the shape.** `CachingPriceProxy`,
   `AuditedPriceDecorator`, `LegacySoapAdapter`. A reader who sees three same-shaped
   wrappers in a constructor chain learns everything from the names. This is the single
   highest-leverage habit in this chapter.

2. **Stop arguing about the "correct" name in code review when the shape is ambiguous.**
   If a wrapper both caches and adds logging, the argument has no factual answer. Decide
   what its dominant responsibility is, name it that, and move on. (Better: split it — one
   concern per wrapper.)

3. **Write the intent in a doc comment.** One line: `/// Proxy: gates access by permission;
   the real repository is created lazily and never touched for unauthorised callers.` That
   sentence is worth more than the pattern name in the class name.

4. **Do not "convert" a Strategy into a State to satisfy a reviewer.** If the code has the
   Strategy intent — the caller picks, the pickees are order-independent — then it *is* a
   Strategy, even if someone points at a State diagram and says "same picture". They are
   right about the picture and wrong about the pattern.

5. **The reverse is also true: shape is not sufficient evidence.** A class called
   `PricingStrategy` whose implementations set `context.strategy = new OtherThing()` is a
   State machine wearing a Strategy's name, and it will surprise everyone who reads it.
   Rename it.

A short worked example of the same code, three intents:

```typescript
// Shape: identical. Intent: three different patterns. Names do the work.

// 1. DECORATOR — adds behaviour, caller composed it deliberately.
class ThrottledSearch implements SearchPort {
  constructor(private readonly inner: SearchPort, private readonly limiter: RateLimiter) {}
  async search(q: Query): Promise<Results> {
    await this.limiter.acquire();
    return this.inner.search(q);
  }
}

// 2. PROXY — controls access; may not call inward at all.
class TenantScopedSearchProxy implements SearchPort {
  constructor(private readonly real: SearchPort, private readonly tenant: TenantContext) {}
  async search(q: Query): Promise<Results> {
    if (!this.tenant.canSearch()) return { items: [], total: 0 };   // real object untouched
    return this.real.search(q.scopedTo(this.tenant.id));
  }
}

// 3. CHAIN LINK — may handle and stop, or pass on.
class CachedOrLiveSearch implements SearchPort {
  constructor(private readonly next: SearchPort, private readonly cache: ResultCache) {}
  async search(q: Query): Promise<Results> {
    const hit = this.cache.get(q.key());
    if (hit) return hit;                                             // STOPS here
    const res = await this.next.search(q);
    this.cache.set(q.key(), res);
    return res;
  }
}
```

Three classes, one shape, three names, and any reader of the composition line knows exactly
what the pipeline does.

---

## 🤝 Patterns that commonly appear together

Patterns are not competitors. In production code they arrive in packs. Here are the pairings
you will actually meet.

### Abstract Factory + Bridge

[Abstract Factory](../01-creational/02-abstract-factory.md) +
[Bridge](../02-structural/02-bridge.md)

Bridge gives you two hierarchies — abstraction and implementation. Something has to choose
which implementation an abstraction gets, and choosing a *matched set* of implementations is
exactly what Abstract Factory does.

```csharp
// The Bridge implementors from section 2, now supplied as a family.
public interface IChannelKit
{
    IListingRenderer Renderer();
    IImageTransformer Images();
    ITrackingLinkBuilder Links();
}

public sealed class WebChannelKit : IChannelKit
{
    public IListingRenderer Renderer()    => new HtmlRenderer();
    public IImageTransformer Images()     => new WebpTransformer(maxWidth: 1600);
    public ITrackingLinkBuilder Links()   => new UtmLinkBuilder("web");
}

public sealed class EmailChannelKit : IChannelKit
{
    public IListingRenderer Renderer()    => new AmpRenderer();
    public IImageTransformer Images()     => new JpegTransformer(maxWidth: 600);
    public ITrackingLinkBuilder Links()   => new UtmLinkBuilder("email");
}

// Abstraction side picks up a whole consistent implementation family in one line.
ListingCard card = new CarCard(car, kit.Renderer());
```

You cannot accidentally render AMP markup with 1600px WebP images, because the kit ships a
coherent set. That is Abstract Factory earning its keep on top of Bridge.

### Builder + Composite

[Builder](../01-creational/03-builder.md) + [Composite](../02-structural/03-composite.md)

Composites are trees, and trees are annoying to construct by hand. Builder gives you a
fluent API whose nesting mirrors the tree.

```typescript
class BundleBuilder {
  private readonly stack: Bundle[] = [];
  private root?: Bundle;

  openBundle(label: string, discountPct = 0): this {
    const b = new Bundle(label, discountPct);
    if (this.stack.length) this.stack[this.stack.length - 1].add(b);
    else this.root = b;
    this.stack.push(b);
    return this;
  }
  item(label: string, paise: number): this {
    if (!this.stack.length) throw new Error("openBundle() first");
    this.stack[this.stack.length - 1].add(new AddOn(label, paise));
    return this;
  }
  closeBundle(): this {
    if (!this.stack.length) throw new Error("nothing open");
    this.stack.pop();
    return this;
  }
  build(): Bundle {
    if (this.stack.length) throw new Error(`${this.stack.length} bundle(s) left open`);
    if (!this.root) throw new Error("empty");
    return this.root;
  }
}

const pack = new BundleBuilder()
  .openBundle("Premium seller pack", 10)
    .item("Featured placement, 30 days", 249900)
    .openBundle("Verification pack")
      .item("RC verification", 29900)
      .item("Inspection report", 99900)
    .closeBundle()
  .closeBundle()
  .build();
```

The builder also enforces structural invariants (unbalanced open/close) that the Composite
itself does not check.

### Composite + Iterator + Visitor

[Composite](../02-structural/03-composite.md) +
[Iterator](../03-behavioral/03-iterator.md) +
[Visitor](../03-behavioral/10-visitor.md)

This trio is almost a single compound pattern. Composite builds the tree; Iterator provides
a uniform way to walk it; Visitor provides type-specific operations over it. Section 12's
inventory example is exactly this. In C#, `IEnumerable<T>` plus `yield return` gives you the
Iterator half for free:

```csharp
public sealed class InventoryFolder : IInventoryNode, IEnumerable<IInventoryNode>
{
    private readonly List<IInventoryNode> _children = new();
    public string Name { get; }
    public InventoryFolder(string name) => Name = name;
    public InventoryFolder Add(IInventoryNode node) { _children.Add(node); return this; }

    public IEnumerator<IInventoryNode> GetEnumerator()
    {
        yield return this;
        foreach (var child in _children)
        {
            if (child is IEnumerable<IInventoryNode> subtree)
                foreach (var n in subtree) yield return n;
            else
                yield return child;
        }
    }
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    public T Accept<T>(IInventoryVisitor<T> v) => v.VisitFolder(this);
    public IReadOnlyList<IInventoryNode> Children => _children;
}

// LINQ over a Composite, courtesy of Iterator:
var expensive = root.Where(n => n is VehicleUnit u && u.Price > 2_000_000m).ToList();

// Type-dispatched operation, courtesy of Visitor:
var taxDue = root.Accept(new RoadTaxVisitor());
```

### Command + Memento for undo

[Command](../03-behavioral/02-command.md) + [Memento](../03-behavioral/05-memento.md)

Command gives you the *unit of work* to undo; Memento gives you the *state to restore*.
Storing the previous value inside the command (as in section 8) works for simple edits, but
for a multi-field edit the clean answer is to snapshot the aggregate.

```csharp
// ---- MEMENTO: an opaque snapshot the originator can restore from ----
public sealed class ListingMemento
{
    internal ListingMemento(string json, DateTimeOffset takenAt)
    {
        Json = json; TakenAt = takenAt;
    }
    internal string Json { get; }
    public DateTimeOffset TakenAt { get; }
}

public sealed partial class Listing
{
    public ListingMemento Snapshot() =>
        new(JsonSerializer.Serialize(this), DateTimeOffset.UtcNow);

    public void Restore(ListingMemento memento)
    {
        var prior = JsonSerializer.Deserialize<Listing>(memento.Json)
                    ?? throw new InvalidOperationException("Corrupt memento.");
        AskingPrice = prior.AskingPrice;
        Description = prior.Description;
        PhotoUrls    = prior.PhotoUrls.ToList();
        Features     = prior.Features.ToList();
    }
}

// ---- COMMAND that uses it ----
public sealed class BulkEditListingCommand : ICommand
{
    private readonly Listing _listing;
    private readonly Action<Listing> _edit;
    private ListingMemento? _before;

    public BulkEditListingCommand(Listing listing, Action<Listing> edit)
    {
        _listing = listing; _edit = edit;
    }

    public string Describe() => $"Bulk edit of listing {_listing.Id}";

    public Task ExecuteAsync(CancellationToken ct = default)
    {
        _before = _listing.Snapshot();        // Memento captured BEFORE mutation
        _edit(_listing);
        return Task.CompletedTask;
    }

    public Task UndoAsync(CancellationToken ct = default)
    {
        if (_before is not null) _listing.Restore(_before);
        return Task.CompletedTask;
    }
}
```

Command alone gives you a history; Memento alone gives you snapshots; together they give
you undo/redo that survives commands touching many fields at once.

### Observer + Mediator

[Observer](../03-behavioral/06-observer.md) + [Mediator](../03-behavioral/04-mediator.md)

They are not rivals — they layer. A very common arrangement: components publish events
(Observer), and a Mediator is the one subscriber that contains the cross-component rules.
Components stay ignorant of each other; the protocol lives in exactly one testable place.

```typescript
class ListingWorkflowMediator {
  constructor(
    private readonly events: DomainEvents,
    private readonly search: SearchIndex,
    private readonly queue: RabbitPublisher,
    private readonly mailer: SellerMailer,
    private readonly moderation: ModerationQueue,
  ) {
    // Mediator SUBSCRIBES (Observer) and owns the orchestration (Mediator).
    events.on<ListingApproved>("listing.approved", e => this.onApproved(e));
    events.on<ListingRejected>("listing.rejected", e => this.onRejected(e));
    events.on<PriceReduced>("listing.priceReduced", e => this.onPriceReduced(e));
  }

  private async onApproved(e: ListingApproved): Promise<void> {
    await this.search.upsert(e.listingId);
    await this.queue.publish("listing.live", { listingId: e.listingId });
    await this.mailer.confirmLive(e.sellerId, e.listingId);
  }

  private async onRejected(e: ListingRejected): Promise<void> {
    await this.search.remove(e.listingId);
    await this.mailer.explainRejection(e.sellerId, e.listingId, e.reasons);
    if (e.reasons.some(r => r.code === "SUSPECTED_FRAUD")) {
      await this.moderation.escalate(e.sellerId);          // the cross-cutting RULE
    }
  }

  private async onPriceReduced(e: PriceReduced): Promise<void> {
    await this.search.upsert(e.listingId);
    if (e.percentDrop >= 10) {
      await this.queue.publish("listing.priceDropAlert", {
        listingId: e.listingId, percentDrop: e.percentDrop,
      });
    }
  }
}
```

The `if SUSPECTED_FRAUD then escalate` line is the Mediator part. The `events.on(...)` lines
are the Observer part.

### Flyweight + Factory

[Flyweight](../02-structural/06-flyweight.md) +
[Factory Method](../01-creational/01-factory-method.md) /
[Abstract Factory](../01-creational/02-abstract-factory.md)

Flyweight is not really usable without a factory — the *whole mechanism* is "ask the factory,
get a shared instance if one exists". The pooling logic must live in one place or sharing
breaks. You saw this in `VehicleSpecFactory` in section 11. A thread-safe C# version:

```csharp
public sealed class VehicleSpecFactory
{
    private readonly ConcurrentDictionary<SpecKey, VehicleSpec> _pool = new();

    public VehicleSpec Get(string make, string model, string variant,
                           string fuelType, int engineCc, int seats)
    {
        var key = new SpecKey(make, model, variant, fuelType, engineCc, seats);
        return _pool.GetOrAdd(key, k => new VehicleSpec(
            k.Make, k.Model, k.Variant, k.FuelType, k.EngineCc, k.Seats));
    }

    public int PooledCount => _pool.Count;

    private readonly record struct SpecKey(
        string Make, string Model, string Variant, string FuelType, int EngineCc, int Seats);
}
```

`record struct` gives you value equality for the key for free — exactly what a flyweight
pool needs.

### Proxy + Decorator chains

[Proxy](../02-structural/07-proxy.md) + [Decorator](../02-structural/04-decorator.md)

Because they share a shape, they compose into a single pipeline with no friction. The order
matters and is worth thinking about explicitly:

```
  caller
    │
    v
 [ AuthorizationProxy ]   fail fast — never pay for anything below if denied
    │
    v
 [ CachingProxy ]         short-circuit before doing expensive work
    │
    v
 [ MetricsDecorator ]     measure only the calls that actually reach the source
    │
    v
 [ RetryDecorator ]       retry the real call, not the cache lookup
    │
    v
 [ SqlListingRepository ] the real thing
```

```csharp
// Composition root (ASP.NET Core). Read outward-in: the LAST wrapper is the OUTERMOST.
services.AddScoped<IListingRepository>(sp =>
{
    IListingRepository repo = new SqlListingRepository(sp.GetRequiredService<DbContextFactory>());
    repo = new RetryingListingRepository(repo, attempts: 3);
    repo = new MetricsListingRepository(repo, sp.GetRequiredService<IMetrics>());
    repo = new CachingListingRepositoryProxy(repo, sp.GetRequiredService<IMemoryCache>());
    repo = new AuthorizingListingRepositoryProxy(() => repo, sp.GetRequiredService<ICurrentUser>());
    return repo;
});
```

Two ordering rules that will save you: **short-circuiting wrappers go outermost** (no point
authorising after you have already hit the database), and **metrics go just outside the
thing you want to measure** (metrics above the cache measures cache hits as "fast database
calls", which is a lie your dashboards will tell for months).

### State + Strategy

[State](../03-behavioral/07-state.md) + [Strategy](../03-behavioral/08-strategy.md)

They coexist happily because they answer different questions. A listing's State determines
*what is allowed*; a Strategy determines *how a permitted operation is carried out*. Very
often the current State chooses the Strategy:

```csharp
public interface IListingState
{
    string Name { get; }
    bool CanReducePrice { get; }
    IPricingStrategy PricingStrategy { get; }     // the State picks the Strategy
    void Approve(Listing listing);
    void MarkSold(Listing listing);
}

public sealed class LiveState : IListingState
{
    public string Name => "Live";
    public bool CanReducePrice => true;
    public IPricingStrategy PricingStrategy => new MarketAveragePricing();
    public void Approve(Listing l) { }
    public void MarkSold(Listing l) => l.TransitionTo(new SoldState());
}

public sealed class StaleState : IListingState                 // live > 90 days, no enquiries
{
    public string Name => "Stale";
    public bool CanReducePrice => true;
    public IPricingStrategy PricingStrategy => new AggressiveDiscountPricing(discountPct: 8);
    public void Approve(Listing l) { }
    public void MarkSold(Listing l) => l.TransitionTo(new SoldState());
}

public sealed class DraftState : IListingState
{
    public string Name => "Draft";
    public bool CanReducePrice => false;
    public IPricingStrategy PricingStrategy => new DepreciationCurvePricing();
    public void Approve(Listing l) => l.TransitionTo(new LiveState());
    public void MarkSold(Listing l) => throw new InvalidOperationException("Not live yet.");
}
```

Two same-shaped patterns in one file, doing genuinely different jobs, and completely
readable because the names and the responsibilities are distinct.

---

## 🧮 A decision flowchart

When you are mid-implementation and unsure which name applies:

```
  Are you deciding WHICH OBJECT TO CREATE?
    ├─ One product, varied by subclass ............ Factory Method
    ├─ A matched family of products ............... Abstract Factory
    ├─ Many optional construction steps ........... Builder
    ├─ Copying an existing configured object ...... Prototype
    └─ Exactly one instance, process-wide ......... Singleton   (prefer DI lifetime)

  Are you ARRANGING objects / changing how they're reached?
    ├─ Two interfaces don't match ................. Adapter
    ├─ Two hierarchies multiplying (n × m) ........ Bridge
    ├─ Tree; leaf and branch treated the same ..... Composite
    ├─ Add behaviour, same interface, stackable ... Decorator
    ├─ Hide a subsystem behind a new small API .... Facade
    ├─ Many identical immutable parts, memory ..... Flyweight
    └─ Control access / lifetime / location ....... Proxy

  Are you deciding HOW OBJECTS INTERACT or how an algorithm varies?
    ├─ Several handlers, any may stop the request . Chain of Responsibility
    ├─ A request as a storable/undoable object .... Command
    ├─ Uniform traversal of a collection .......... Iterator
    ├─ A hub owning cross-component rules ......... Mediator
    ├─ Snapshot + restore without leaking internals Memento
    ├─ Broadcast to unknown subscribers ........... Observer
    ├─ Object behaviour changes with lifecycle .... State
    ├─ Swap one algorithm at runtime .............. Strategy
    ├─ Fixed skeleton, subclass fills in steps .... Template Method
    └─ New operations over a stable type tree ..... Visitor
```

And the three questions that resolve most arguments:

1. **Who decides?** Caller → Strategy/Command. The object itself → State. A hub → Mediator.
2. **What is preserved?** The interface → Decorator/Proxy/Chain. A new interface → Facade.
   A different interface converted → Adapter.
3. **What is the axis of change?** New types → Visitor is a bad fit. New operations →
   Visitor is a good fit. New products → Abstract Factory. New steps → Builder.

---

## 🗣️ Plain English summary

If you remember nothing else from this chapter, remember these:

- **Patterns are named by intent, not shape.** Decorator and Proxy draw the same picture.
  So do Strategy, State and Bridge. What separates them is *why you did it*, and the only
  place that "why" survives code review is in the class name and one doc comment.

- **The "who decides" question settles most pairs.** If the swapped-in object decides what
  runs next, it is State. If the caller decides, it is Strategy. If a hub encodes the
  cross-component rules, it is Mediator. If nobody encodes rules and it is just a mailing
  list, it is Observer.

- **The "same interface or different" question settles the wrappers.** Same interface, adds
  behaviour → Decorator. Same interface, controls access → Proxy. Same interface, may
  refuse to forward → Chain of Responsibility. Different interface, converts → Adapter. New
  smaller interface over many things → Facade.

- **Creational patterns differ by what varies.** Which single product (Factory Method),
  which family (Abstract Factory), how it is assembled (Builder), copy an existing one
  (Prototype), only ever one (Singleton).

- **Patterns arrive in packs.** Composite wants an Iterator and a Visitor. Command wants a
  Memento. Bridge wants an Abstract Factory. Flyweight is unusable without a Factory. Proxy
  and Decorator stack in the same pipeline. State picks Strategies. Seeing a pattern usually
  means one or two of its friends are about to show up.

- **Do not force the vocabulary.** The point of knowing these names is to communicate
  faster with other engineers, not to score points. If the shape is ambiguous, pick the
  dominant intent, name the class for it, write one line of comment, and ship.

---

### 🔗 Jump back to a pattern

**Creational:**
[Factory Method](../01-creational/01-factory-method.md) ·
[Abstract Factory](../01-creational/02-abstract-factory.md) ·
[Builder](../01-creational/03-builder.md) ·
[Prototype](../01-creational/04-prototype.md) ·
[Singleton](../01-creational/05-singleton.md)

**Structural:**
[Adapter](../02-structural/01-adapter.md) ·
[Bridge](../02-structural/02-bridge.md) ·
[Composite](../02-structural/03-composite.md) ·
[Decorator](../02-structural/04-decorator.md) ·
[Facade](../02-structural/05-facade.md) ·
[Flyweight](../02-structural/06-flyweight.md) ·
[Proxy](../02-structural/07-proxy.md)

**Behavioral:**
[Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) ·
[Command](../03-behavioral/02-command.md) ·
[Iterator](../03-behavioral/03-iterator.md) ·
[Mediator](../03-behavioral/04-mediator.md) ·
[Memento](../03-behavioral/05-memento.md) ·
[Observer](../03-behavioral/06-observer.md) ·
[State](../03-behavioral/07-state.md) ·
[Strategy](../03-behavioral/08-strategy.md) ·
[Template Method](../03-behavioral/09-template-method.md) ·
[Visitor](../03-behavioral/10-visitor.md)

**Extras:**
[SOLID and principles](01-solid-and-principles.md) ·
[Start here](../00-start-here/README.md)
