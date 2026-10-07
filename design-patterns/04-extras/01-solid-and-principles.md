# 🧭 SOLID and the Principles Behind the Patterns

Patterns are answers. Principles are the questions those answers keep answering.

If you only memorise the 22 Gang-of-Four patterns you end up with a bag of shapes and no idea
which one to reach for. If you understand the five or six forces that keep pushing designs in the
same direction, the patterns stop being trivia and start being *obvious*. Strategy is not a clever
trick someone invented; it is what falls out of the room when you apply Open/Closed plus
"encapsulate what varies" plus "favour composition over inheritance" to a piece of branching logic.

This chapter is the companion file. Read it once now, skim it again after you have worked through
a few pattern chapters, and read it properly a third time when you are about to redesign something
real at work.

Every principle below gets: the formal statement, the plain-English version, a violation you have
almost certainly shipped, the fix, and the patterns that embody it — linked so you can jump.

Code is in **C#** and **TypeScript**, because that is what you write. The examples come from an
automotive marketplace: listings, dealers, pricing, inspection reports, lead routing, message
queues. Same domain all the way through so you are never learning a new toy problem and a new
principle at the same time.

---

## 🗺️ Map of this chapter

```
 SOLID
 ├── S  Single Responsibility ..... one reason to change
 ├── O  Open/Closed ............... extend without editing
 ├── L  Liskov Substitution ....... subtypes must not lie
 ├── I  Interface Segregation ..... no fat interfaces
 └── D  Dependency Inversion ...... depend on abstractions

 Supporting principles (the GoF preface, restated)
 ├── Encapsulate what varies
 ├── Program to an interface, not an implementation
 └── Favour composition over inheritance

 Working discipline
 └── DRY / KISS / YAGNI  (and when each one is wrong)

 Finale: 22 patterns → principles table
```

---

## 🔤 A note before SOLID: these are heuristics, not laws

SOLID came out of Robert C. Martin's writing in the late 1990s and 2000s; the acronym itself was
coined later by Michael Feathers. The individual ideas are older and have separate parentage —
Liskov Substitution comes from Barbara Liskov's work on subtyping; Dependency Inversion is a
restatement of ideas that were floating around layered architecture for years.

What matters more than provenance: **all five are about the cost of change.** None of them makes
code faster. None makes it shorter — SOLID code is usually *longer*. What they buy you is the
ability to change one thing without a cascade, and to test a thing without booting the world.

So the honest framing is: apply them where change is likely, skip them where it is not. A
throwaway migration script does not need Dependency Inversion. Your pricing engine does.

---

# 1️⃣ S — Single Responsibility Principle

## Formal statement

> A class should have only one reason to change.

Later restated by Martin as: *gather together the things that change for the same reason, separate
the things that change for different reasons*, and later still as *a module should be responsible
to one, and only one, actor*.

## Plain English

If two different people, for two different reasons, would both come and ask you to edit this file,
it is doing too much. The finance team wants the tax rule changed. The frontend team wants the JSON
shape changed. The ops team wants a different log format. Three actors, three reasons, one class —
split it.

Note what SRP does **not** say. It does not say "a class should do one thing". Everything does one
thing if you describe it vaguely enough ("it handles listings"). The test is *reasons to change*,
and the practical proxy for that is *who asks for the change*.

## The violation

A `Listing` service that validates, prices, persists, publishes to RabbitMQ, and renders an email.

**C# — bad**

```csharp
public class ListingService
{
    private readonly SqlConnection _db;
    private readonly IModel _channel;   // RabbitMQ channel
    private readonly SmtpClient _smtp;

    public ListingService(SqlConnection db, IModel channel, SmtpClient smtp)
    {
        _db = db; _channel = channel; _smtp = smtp;
    }

    public void PublishListing(ListingDraft draft)
    {
        // Reason to change #1: the product team tweaks validation rules
        if (string.IsNullOrWhiteSpace(draft.RegistrationNumber))
            throw new ValidationException("Registration number required");
        if (draft.Year < 1990 || draft.Year > DateTime.UtcNow.Year + 1)
            throw new ValidationException("Year out of range");
        if (draft.Kilometers > 500_000)
            throw new ValidationException("Kilometers implausible");

        // Reason to change #2: pricing/finance changes the commission model
        var commission = draft.AskingPrice * 0.02m;
        if (draft.DealerTier == "Premium") commission *= 0.5m;
        var listedPrice = draft.AskingPrice + commission;

        // Reason to change #3: the DBA changes the schema
        using var cmd = new SqlCommand(
            "INSERT INTO Listings (Reg, Year, Km, Price, DealerId) " +
            "VALUES (@reg, @year, @km, @price, @dealer)", _db);
        cmd.Parameters.AddWithValue("@reg", draft.RegistrationNumber);
        cmd.Parameters.AddWithValue("@year", draft.Year);
        cmd.Parameters.AddWithValue("@km", draft.Kilometers);
        cmd.Parameters.AddWithValue("@price", listedPrice);
        cmd.Parameters.AddWithValue("@dealer", draft.DealerId);
        cmd.ExecuteNonQuery();

        // Reason to change #4: the platform team renames the exchange or adds headers
        var body = Encoding.UTF8.GetBytes(
            $"{{\"reg\":\"{draft.RegistrationNumber}\",\"price\":{listedPrice}}}");
        _channel.BasicPublish("listings.exchange", "listing.published", null, body);

        // Reason to change #5: marketing rewrites the email copy
        _smtp.Send(new MailMessage("noreply@market.example", draft.DealerEmail)
        {
            Subject = "Your listing is live",
            Body = $"Hi, your {draft.Year} car is listed at {listedPrice:C}."
        });
    }
}
```

Five actors. Every one of them makes you open the same file, and every one of them risks breaking
the other four's tests. You also cannot unit-test the pricing rule without a SQL connection, a
RabbitMQ channel and an SMTP server.

## The fix

Each reason to change gets its own home. The service becomes a *coordinator* — that is its single
responsibility.

**C# — good**

```csharp
public interface IListingValidator { void Validate(ListingDraft draft); }

public sealed class ListingValidator : IListingValidator
{
    public void Validate(ListingDraft draft)
    {
        if (string.IsNullOrWhiteSpace(draft.RegistrationNumber))
            throw new ValidationException("Registration number required");
        if (draft.Year < 1990 || draft.Year > DateTime.UtcNow.Year + 1)
            throw new ValidationException("Year out of range");
        if (draft.Kilometers > 500_000)
            throw new ValidationException("Kilometers implausible");
    }
}

public interface IPricingPolicy { decimal ListedPriceFor(ListingDraft draft); }

public sealed class CommissionPricingPolicy : IPricingPolicy
{
    public decimal ListedPriceFor(ListingDraft draft)
    {
        var rate = draft.DealerTier == "Premium" ? 0.01m : 0.02m;
        return draft.AskingPrice * (1 + rate);
    }
}

public interface IListingRepository { Task<long> AddAsync(Listing listing); }
public interface IEventBus { Task PublishAsync(string routingKey, object payload); }
public interface IDealerNotifier { Task ListingWentLiveAsync(Listing listing); }

public sealed class PublishListingHandler
{
    private readonly IListingValidator _validator;
    private readonly IPricingPolicy _pricing;
    private readonly IListingRepository _repo;
    private readonly IEventBus _bus;
    private readonly IDealerNotifier _notifier;

    public PublishListingHandler(
        IListingValidator validator, IPricingPolicy pricing,
        IListingRepository repo, IEventBus bus, IDealerNotifier notifier)
    {
        _validator = validator; _pricing = pricing;
        _repo = repo; _bus = bus; _notifier = notifier;
    }

    public async Task<long> HandleAsync(ListingDraft draft)
    {
        _validator.Validate(draft);

        var listing = Listing.From(draft, _pricing.ListedPriceFor(draft));
        var id = await _repo.AddAsync(listing);

        await _bus.PublishAsync("listing.published", new { id, listing.ListedPrice });
        await _notifier.ListingWentLiveAsync(listing);
        return id;
    }
}
```

**TypeScript — bad then good**

```ts
// ---------- BAD ----------
class ListingService {
  constructor(
    private db: Pool,
    private channel: amqp.Channel,
    private mailer: Transporter,
  ) {}

  async publish(draft: ListingDraft): Promise<void> {
    if (!draft.registrationNumber) throw new Error('Registration number required');
    if (draft.year < 1990) throw new Error('Year out of range');

    const commission = draft.dealerTier === 'Premium' ? 0.01 : 0.02;
    const listedPrice = draft.askingPrice * (1 + commission);

    await this.db.query(
      'INSERT INTO listings (reg, year, km, price, dealer_id) VALUES ($1,$2,$3,$4,$5)',
      [draft.registrationNumber, draft.year, draft.kilometers, listedPrice, draft.dealerId],
    );

    this.channel.publish(
      'listings.exchange',
      'listing.published',
      Buffer.from(JSON.stringify({ reg: draft.registrationNumber, listedPrice })),
    );

    await this.mailer.sendMail({
      to: draft.dealerEmail,
      subject: 'Your listing is live',
      text: `Listed at ${listedPrice}`,
    });
  }
}
```

```ts
// ---------- GOOD ----------
export interface ListingValidator { validate(draft: ListingDraft): void; }
export interface PricingPolicy { listedPriceFor(draft: ListingDraft): number; }
export interface ListingRepository { add(listing: Listing): Promise<number>; }
export interface EventBus { publish(routingKey: string, payload: unknown): Promise<void>; }
export interface DealerNotifier { listingWentLive(listing: Listing): Promise<void>; }

export class CommissionPricingPolicy implements PricingPolicy {
  listedPriceFor(draft: ListingDraft): number {
    const rate = draft.dealerTier === 'Premium' ? 0.01 : 0.02;
    return Math.round(draft.askingPrice * (1 + rate));
  }
}

export class PublishListingHandler {
  constructor(
    private readonly validator: ListingValidator,
    private readonly pricing: PricingPolicy,
    private readonly repo: ListingRepository,
    private readonly bus: EventBus,
    private readonly notifier: DealerNotifier,
  ) {}

  async handle(draft: ListingDraft): Promise<number> {
    this.validator.validate(draft);

    const listing = listingFrom(draft, this.pricing.listedPriceFor(draft));
    const id = await this.repo.add(listing);

    await this.bus.publish('listing.published', { id, price: listing.listedPrice });
    await this.notifier.listingWentLive(listing);
    return id;
  }
}
```

Now the finance team's change touches exactly one file, and `CommissionPricingPolicy` is testable
with no infrastructure at all.

## Patterns that embody SRP

- [Facade](../02-structural/05-facade.md) — one class whose only job is to be a simple front door,
  so the subsystem classes can each stay narrow.
- [Command](../03-behavioral/02-command.md) — one class per action, which is SRP taken to its
  logical extreme.
- [Mediator](../03-behavioral/04-mediator.md) — moves "who talks to whom" out of the colleagues
  and into a single object whose responsibility is coordination.
- [Visitor](../03-behavioral/10-visitor.md) — pulls an *operation* out of a class hierarchy so the
  hierarchy is not the place where every new report gets bolted on.
- [Builder](../01-creational/03-builder.md) — construction is a responsibility; if construction is
  complicated, it deserves its own class.
- [Iterator](../03-behavioral/03-iterator.md) — traversal is a responsibility separate from
  storage.

---

# 2️⃣ O — Open/Closed Principle

## Formal statement

> Software entities (classes, modules, functions) should be open for extension, but closed for
> modification.

Originally Bertrand Meyer's, in an inheritance-flavoured form; the version everyone uses today is
the polymorphic one — extend by *adding* a new implementation, not by editing the existing one.

## Plain English

Adding a new case should mean adding a new file, not editing an old one. If every new payment
method, every new dealer tier, every new export format makes you open the same `switch`, the design
is closed for extension and open for modification — exactly backwards.

## The violation

The growing switch. You know this one.

**TypeScript — bad**

```ts
type ExportFormat = 'csv' | 'xml' | 'json';

export class ListingExporter {
  export(listings: Listing[], format: ExportFormat): string {
    if (format === 'csv') {
      const header = 'id,reg,year,km,price';
      const rows = listings.map(
        (l) => `${l.id},${l.registrationNumber},${l.year},${l.kilometers},${l.listedPrice}`,
      );
      return [header, ...rows].join('\n');
    }
    if (format === 'xml') {
      const body = listings
        .map(
          (l) =>
            `<listing id="${l.id}"><reg>${l.registrationNumber}</reg>` +
            `<price>${l.listedPrice}</price></listing>`,
        )
        .join('');
      return `<listings>${body}</listings>`;
    }
    if (format === 'json') {
      return JSON.stringify(listings);
    }
    throw new Error(`Unsupported format: ${format}`);
  }
}
```

Every new format (a dealer feed, a vehicle-ads feed, a partner's fixed-width file) means editing
this class, re-testing all the existing branches, and risking a regression in CSV because you were
adding XML.

## The fix

One interface, one class per format, a registry to pick between them.

**TypeScript — good**

```ts
export interface ListingExportFormat {
  readonly id: string;
  readonly contentType: string;
  render(listings: Listing[]): string;
}

export class CsvExportFormat implements ListingExportFormat {
  readonly id = 'csv';
  readonly contentType = 'text/csv';

  render(listings: Listing[]): string {
    const header = 'id,reg,year,km,price';
    const rows = listings.map(
      (l) => `${l.id},${l.registrationNumber},${l.year},${l.kilometers},${l.listedPrice}`,
    );
    return [header, ...rows].join('\n');
  }
}

export class XmlExportFormat implements ListingExportFormat {
  readonly id = 'xml';
  readonly contentType = 'application/xml';

  render(listings: Listing[]): string {
    const body = listings
      .map(
        (l) =>
          `<listing id="${l.id}"><reg>${escapeXml(l.registrationNumber)}</reg>` +
          `<price>${l.listedPrice}</price></listing>`,
      )
      .join('');
    return `<listings>${body}</listings>`;
  }
}

export class ListingExporter {
  private readonly formats = new Map<string, ListingExportFormat>();

  constructor(formats: ListingExportFormat[]) {
    for (const f of formats) this.formats.set(f.id, f);
  }

  export(listings: Listing[], formatId: string): { body: string; contentType: string } {
    const format = this.formats.get(formatId);
    if (!format) throw new Error(`Unsupported format: ${formatId}`);
    return { body: format.render(listings), contentType: format.contentType };
  }
}

// Adding a partner feed = a new file. ListingExporter is never touched again.
export class PartnerFeedFormat implements ListingExportFormat {
  readonly id = 'partner-feed';
  readonly contentType = 'text/plain';
  render(listings: Listing[]): string {
    return listings.map((l) => `${l.registrationNumber.padEnd(12)}${l.listedPrice}`).join('\n');
  }
}
```

**C# — same shape, with DI doing the registration**

```csharp
// ---------- BAD ----------
public string Export(IReadOnlyList<Listing> listings, string format)
{
    switch (format)
    {
        case "csv":  /* ... */ break;
        case "xml":  /* ... */ break;
        case "json": /* ... */ break;
        default: throw new NotSupportedException(format);
    }
    // ...
}

// ---------- GOOD ----------
public interface IListingExportFormat
{
    string Id { get; }
    string ContentType { get; }
    string Render(IReadOnlyList<Listing> listings);
}

public sealed class CsvExportFormat : IListingExportFormat
{
    public string Id => "csv";
    public string ContentType => "text/csv";

    public string Render(IReadOnlyList<Listing> listings)
    {
        var sb = new StringBuilder("id,reg,year,km,price\n");
        foreach (var l in listings)
            sb.AppendLine($"{l.Id},{l.RegistrationNumber},{l.Year},{l.Kilometers},{l.ListedPrice}");
        return sb.ToString();
    }
}

public sealed class ListingExporter
{
    private readonly IReadOnlyDictionary<string, IListingExportFormat> _formats;

    // ASP.NET Core injects every registered IListingExportFormat here.
    public ListingExporter(IEnumerable<IListingExportFormat> formats)
        => _formats = formats.ToDictionary(f => f.Id, StringComparer.OrdinalIgnoreCase);

    public (string Body, string ContentType) Export(IReadOnlyList<Listing> listings, string formatId)
    {
        if (!_formats.TryGetValue(formatId, out var format))
            throw new NotSupportedException($"Unsupported format: {formatId}");
        return (format.Render(listings), format.ContentType);
    }
}

// Startup:
// services.AddSingleton<IListingExportFormat, CsvExportFormat>();
// services.AddSingleton<IListingExportFormat, XmlExportFormat>();
// services.AddSingleton<IListingExportFormat, PartnerFeedFormat>();  // new format, one line
```

## When *not* to apply it

Open/Closed has a cost: indirection. You cannot read the behaviour in one place any more. Apply it
along the axis where change actually happens. If you have had exactly two export formats for four
years and no plans for a third, the switch is fine. The rule of thumb many people use: write the
switch the first time, write it again the second time, refactor to polymorphism on the third —
because by the third you have evidence about *which* axis varies.

## Patterns that embody OCP

- [Strategy](../03-behavioral/08-strategy.md) — the canonical one. Swap the algorithm, never edit
  the context.
- [Decorator](../02-structural/04-decorator.md) — add behaviour by wrapping, never by editing.
- [Abstract Factory](../01-creational/02-abstract-factory.md) and
  [Factory Method](../01-creational/01-factory-method.md) — add a new product family or product by
  adding a creator.
- [Visitor](../03-behavioral/10-visitor.md) — open for new *operations* (closed for new element
  types; see the note in that chapter).
- [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) — add a handler to the
  chain; existing handlers do not change.
- [Observer](../03-behavioral/06-observer.md) — add a subscriber without touching the publisher.
- [State](../03-behavioral/07-state.md) — add a state class instead of another branch.
- [Template Method](../03-behavioral/09-template-method.md) — the algorithm skeleton is closed;
  the hooks are open.
- [Bridge](../02-structural/02-bridge.md) — add an implementor or an abstraction independently.

---

# 3️⃣ L — Liskov Substitution Principle

This is the one people get wrong, so it gets the long treatment.

## Formal statement

Barbara Liskov and Jeannette Wing's substitution requirement, in its usual paraphrase:

> If S is a subtype of T, then objects of type T may be replaced with objects of type S without
> altering any of the desirable properties of the program.

Unpacked into the rules that actually bite:

| Rule | Meaning |
| --- | --- |
| Preconditions may not be strengthened | The subtype cannot demand *more* of callers than the base did. |
| Postconditions may not be weakened | The subtype must deliver at least what the base promised. |
| Invariants must be preserved | Whatever the base guarantees at all times, the subtype guarantees too. |
| History constraint | The subtype cannot allow state changes the base type forbade (e.g. mutating something the base treated as immutable). |
| Exceptions | The subtype should not throw new exception types the base's contract did not admit. |

## Plain English

**A subclass must not surprise anyone holding a reference to the base class.**

That is the whole principle. If code written against `Vehicle` starts misbehaving when you hand it
a `Motorcycle`, the `Motorcycle` is lying about being a `Vehicle`. The compiler is happy; the
program is wrong.

The most useful diagnostic: **does any caller ever have to ask "which subclass is this?"** If you
see `if (x is Square)` or `instanceof`, or a comment like "don't call this one on a read-only
repo", LSP is already broken.

## Violation 1 — the classic: Rectangle / Square

Mathematically a square *is a* rectangle. In code, with mutable setters, it is not.

**C# — bad**

```csharp
public class Rectangle
{
    public virtual int Width  { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; }   // surprise!
    }

    public override int Height
    {
        get => base.Height;
        set { base.Height = value; base.Width = value; }   // surprise!
    }
}

// Written against Rectangle. Perfectly reasonable. Passes for Rectangle, fails for Square.
public static void ResizeAndCheck(Rectangle r)
{
    r.Width  = 5;
    r.Height = 4;
    Debug.Assert(r.Area == 20, $"Expected 20, got {r.Area}");   // Square gives 16
}
```

Nothing is wrong with `Square` in isolation. Nothing is wrong with `ResizeAndCheck` in isolation.
The bug is the *claim* that `Square` is substitutable for `Rectangle`. `Rectangle` has an implicit
invariant — "width and height vary independently" — and `Square` breaks it. That is a weakened
postcondition on the setter.

**TypeScript — the same trap**

```ts
class Rectangle {
  constructor(protected width: number, protected height: number) {}
  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
  get area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  override setWidth(w: number): void { this.width = w; this.height = w; }
  override setHeight(h: number): void { this.width = h; this.height = h; }
}

function resizeAndCheck(r: Rectangle): void {
  r.setWidth(5);
  r.setHeight(4);
  console.assert(r.area === 20, `expected 20, got ${r.area}`); // Square: 16
}
```

### The fix

Two options, both legitimate.

**Fix A — make them immutable.** The problem was the *setters*, not the geometry. With immutable
values, "resizing" returns a new object and there is no invariant to violate.

```csharp
public abstract class Shape
{
    public abstract int Area { get; }
}

public sealed class Rectangle : Shape
{
    public Rectangle(int width, int height) { Width = width; Height = height; }
    public int Width  { get; }
    public int Height { get; }
    public override int Area => Width * Height;

    public Rectangle WithWidth(int width)   => new Rectangle(width, Height);
    public Rectangle WithHeight(int height) => new Rectangle(Width, height);
}

public sealed class Square : Shape
{
    public Square(int side) { Side = side; }
    public int Side { get; }
    public override int Area => Side * Side;

    public Square WithSide(int side) => new Square(side);
}
```

`Square` no longer claims to be a `Rectangle`; both claim to be a `Shape`, and the `Shape` contract
("has an area") is one both can honestly keep.

**Fix B — drop the inheritance and keep the conversion.**

```ts
interface Shape { readonly area: number; }

class Rectangle implements Shape {
  constructor(readonly width: number, readonly height: number) {}
  get area(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(readonly side: number) {}
  get area(): number { return this.side * this.side; }
  toRectangle(): Rectangle { return new Rectangle(this.side, this.side); }
}
```

The lesson is not "squares are weird". It is: **`is-a` in English is not `is-a` in code. The code
relationship is "can honestly keep every promise the base type made", and mutability is where most
promises get broken.**

## Violation 2 — the realistic one: the read-only repository

This is the version you will actually hit, in some shape, at work.

**C# — bad**

```csharp
public interface IListingRepository
{
    Task<Listing?> GetAsync(long id);
    Task<long> AddAsync(Listing listing);
    Task UpdateAsync(Listing listing);
    Task DeleteAsync(long id);
}

public sealed class SqlListingRepository : IListingRepository
{
    // ... honest implementation, all four methods work ...
}

// A read-only replica, or a cached snapshot, or an archived-partition view.
public sealed class ReadOnlyListingRepository : IListingRepository
{
    private readonly IReadOnlyDictionary<long, Listing> _snapshot;

    public ReadOnlyListingRepository(IReadOnlyDictionary<long, Listing> snapshot)
        => _snapshot = snapshot;

    public Task<Listing?> GetAsync(long id)
        => Task.FromResult(_snapshot.TryGetValue(id, out var l) ? l : null);

    public Task<long> AddAsync(Listing listing)
        => throw new NotSupportedException("Read-only repository");   // ❌ LSP violation

    public Task UpdateAsync(Listing listing)
        => throw new NotSupportedException("Read-only repository");   // ❌

    public Task DeleteAsync(long id)
        => throw new NotSupportedException("Read-only repository");   // ❌
}
```

Now every caller that takes an `IListingRepository` has to either know which one it got, or accept
that a routine `UpdateAsync` may blow up at runtime. The classic symptom appears immediately:

```csharp
public async Task ArchiveAsync(IListingRepository repo, long id)
{
    if (repo is ReadOnlyListingRepository) return;   // ❌ the smell
    await repo.DeleteAsync(id);
}
```

`NotSupportedException` thrown from an interface method is *strengthening a precondition* ("you may
only call me if I happen to be the writable one"). That is exactly what LSP forbids.

The .NET base class library, to be fair, does this itself: `ICollection<T>.Add` on a fixed-size
array throws `NotSupportedException`. It is a known wart, not a model to copy.

**C# — good: split the interfaces so no implementation has to lie**

```csharp
public interface IReadListings
{
    Task<Listing?> GetAsync(long id);
    Task<IReadOnlyList<Listing>> SearchAsync(ListingQuery query);
}

public interface IWriteListings
{
    Task<long> AddAsync(Listing listing);
    Task UpdateAsync(Listing listing);
    Task DeleteAsync(long id);
}

// The SQL repo can honestly claim both.
public sealed class SqlListingRepository : IReadListings, IWriteListings
{
    // ... every method genuinely works ...
}

// The snapshot claims only what it can keep.
public sealed class SnapshotListingReader : IReadListings
{
    private readonly IReadOnlyDictionary<long, Listing> _snapshot;
    public SnapshotListingReader(IReadOnlyDictionary<long, Listing> snapshot) => _snapshot = snapshot;

    public Task<Listing?> GetAsync(long id)
        => Task.FromResult(_snapshot.TryGetValue(id, out var l) ? l : null);

    public Task<IReadOnlyList<Listing>> SearchAsync(ListingQuery query)
        => Task.FromResult<IReadOnlyList<Listing>>(
               _snapshot.Values.Where(query.Matches).ToList());
}

// Callers now declare exactly what they need, and the compiler enforces it.
public sealed class ListingSearchController
{
    private readonly IReadListings _listings;   // cannot accidentally write
    public ListingSearchController(IReadListings listings) => _listings = listings;
}

public sealed class ArchiveJob
{
    private readonly IWriteListings _listings;  // cannot be handed a snapshot
    public ArchiveJob(IWriteListings listings) => _listings = listings;
    public Task ArchiveAsync(long id) => _listings.DeleteAsync(id);   // no type check needed
}
```

Note how LSP and Interface Segregation are the same fix seen from two angles: the fat interface was
what *forced* someone to lie.

**TypeScript — bad then good**

```ts
// ---------- BAD ----------
interface ListingRepository {
  get(id: number): Promise<Listing | null>;
  add(listing: Listing): Promise<number>;
  update(listing: Listing): Promise<void>;
  remove(id: number): Promise<void>;
}

class CachedListingRepository implements ListingRepository {
  constructor(private readonly snapshot: Map<number, Listing>) {}
  async get(id: number) { return this.snapshot.get(id) ?? null; }
  async add(): Promise<number> { throw new Error('read-only'); }   // ❌
  async update(): Promise<void> { throw new Error('read-only'); }  // ❌
  async remove(): Promise<void> { throw new Error('read-only'); }  // ❌
}
```

```ts
// ---------- GOOD ----------
export interface ListingReader {
  get(id: number): Promise<Listing | null>;
  search(query: ListingQuery): Promise<Listing[]>;
}

export interface ListingWriter {
  add(listing: Listing): Promise<number>;
  update(listing: Listing): Promise<void>;
  remove(id: number): Promise<void>;
}

export class SqlListingRepository implements ListingReader, ListingWriter {
  constructor(private readonly db: Pool) {}
  async get(id: number): Promise<Listing | null> { /* ... */ return null; }
  async search(q: ListingQuery): Promise<Listing[]> { /* ... */ return []; }
  async add(l: Listing): Promise<number> { /* ... */ return 0; }
  async update(l: Listing): Promise<void> { /* ... */ }
  async remove(id: number): Promise<void> { /* ... */ }
}

export class SnapshotListingReader implements ListingReader {
  constructor(private readonly snapshot: Map<number, Listing>) {}
  async get(id: number): Promise<Listing | null> { return this.snapshot.get(id) ?? null; }
  async search(q: ListingQuery): Promise<Listing[]> {
    return [...this.snapshot.values()].filter((l) => matches(l, q));
  }
}

// Consumers narrow their dependency to the capability they need.
export class SearchService {
  constructor(private readonly listings: ListingReader) {}
}
```

## Violation 3 — the quiet one: weakened postconditions in a queue publisher

LSP breaks are not always exceptions. Often the subtype just does *less*.

```ts
// Base contract, documented and relied upon:
//   publish() resolves only after the broker has acknowledged the message.
export abstract class EventPublisher {
  abstract publish(routingKey: string, payload: unknown): Promise<void>;
}

export class RabbitMqPublisher extends EventPublisher {
  constructor(private readonly channel: ConfirmChannel) { super(); }

  async publish(routingKey: string, payload: unknown): Promise<void> {
    const body = Buffer.from(JSON.stringify(payload));
    await new Promise<void>((resolve, reject) => {
      this.channel.publish('listings.exchange', routingKey, body, { persistent: true },
        (err) => (err ? reject(err) : resolve()));
    });
  }
}

// ❌ Subtype keeps the signature but breaks the promise: it resolves before delivery.
export class BufferedPublisher extends EventPublisher {
  private readonly queue: Array<[string, unknown]> = [];

  async publish(routingKey: string, payload: unknown): Promise<void> {
    this.queue.push([routingKey, payload]);   // flushed "later", maybe never
  }
}
```

A caller that does `await publisher.publish(...)` and then commits its database transaction is
correct against `RabbitMqPublisher` and silently loses events against `BufferedPublisher`. No
exception, no test failure, just missing messages in production at 2am.

The fix is to be honest in the type system:

```ts
export interface EventPublisher {
  /** Resolves once the broker has acknowledged the message. */
  publish(routingKey: string, payload: unknown): Promise<void>;
}

export interface BufferedEventPublisher {
  /** Returns once the message is buffered locally. Call flush() for durability. */
  enqueue(routingKey: string, payload: unknown): void;
  flush(): Promise<void>;
}
```

Different promises, different types. Callers choose knowingly.

## The LSP checklist

Before you write `extends` or `implements`, ask:

```
 1. Can this type honour EVERY method of the base, for EVERY input the base accepts?
 2. Does it throw anything the base's contract does not admit?
 3. Does it do LESS than the base promised (weaker postcondition)?
 4. Does it require callers to do MORE first (stronger precondition)?
 5. Does it break an invariant the base maintained?
 6. Will any caller ever need to ask "which subtype is this?"

 Any "yes" to 2-6 (or "no" to 1) → do not inherit. Split the interface,
 or use composition, or model the relationship as conversion instead.
```

## Patterns that embody LSP

- [Strategy](../03-behavioral/08-strategy.md) — every strategy must be a genuine drop-in for the
  others; the pattern is worthless the moment one of them throws instead.
- [State](../03-behavioral/07-state.md) — each state must handle the full state interface, even if
  handling means a deliberate, documented no-op rather than an exception.
- [Template Method](../03-behavioral/09-template-method.md) — the base defines the contract and
  subclasses fill hooks; this is LSP's happiest home *and* the easiest place to break it by
  overriding something you should not have.
- [Composite](../02-structural/03-composite.md) — leaves and composites must both satisfy the
  component interface. The famous wrinkle: what does `Add(child)` mean on a leaf? Answering that
  honestly (either leaves are a separate type, or `Add` is a composite-only method) *is* LSP.
- [Proxy](../02-structural/07-proxy.md) and [Decorator](../02-structural/04-decorator.md) — both
  stand in for the real object. If the wrapper changes observable behaviour beyond its stated job,
  it is broken.
- [Adapter](../02-structural/01-adapter.md) — the whole point is that the adapted object is
  substitutable for the target interface.

---

# 4️⃣ I — Interface Segregation Principle

## Formal statement

> Clients should not be forced to depend upon interfaces that they do not use.

Equivalently: many small, client-specific interfaces are better than one general-purpose one.

## Plain English

Do not make me implement `SendFax()` because the interface was designed for a printer in 2004.
An interface should be shaped by what a *caller* needs, not by everything an implementation happens
to do.

## The violation

**C# — bad**

```csharp
public interface IVehicleListing
{
    // Used by everything
    long Id { get; }
    string RegistrationNumber { get; }
    decimal ListedPrice { get; }

    // Used only by the inspection module
    InspectionReport Inspection { get; }
    void AttachInspection(InspectionReport report);

    // Used only by the finance module
    LoanOffer[] LoanOffers { get; }
    void RecalculateEmi(decimal interestRate, int tenureMonths);

    // Used only by the search indexer
    string[] SearchKeywords { get; }
    void Reindex();

    // Used only by the dealer console
    DealerContact Contact { get; }
    void NotifyDealer(string message);
}
```

Every implementation — the real one, the test double, the projection used by the search worker —
must implement all of it. Your mocks get enormous. A change to `RecalculateEmi`'s signature
recompiles the search indexer, which does not care.

## The fix

**C# — good**

```csharp
public interface IListingIdentity
{
    long Id { get; }
    string RegistrationNumber { get; }
}

public interface IPricedListing : IListingIdentity
{
    decimal ListedPrice { get; }
}

public interface IInspectable
{
    InspectionReport? Inspection { get; }
    void AttachInspection(InspectionReport report);
}

public interface IFinanceable
{
    IReadOnlyList<LoanOffer> LoanOffers { get; }
    void RecalculateEmi(decimal interestRate, int tenureMonths);
}

public interface IIndexable
{
    IReadOnlyList<string> SearchKeywords { get; }
}

// One concrete class may still implement several — that is fine and normal.
public sealed class VehicleListing : IPricedListing, IInspectable, IFinanceable, IIndexable
{
    public long Id { get; init; }
    public string RegistrationNumber { get; init; } = "";
    public decimal ListedPrice { get; private set; }
    public InspectionReport? Inspection { get; private set; }
    public IReadOnlyList<LoanOffer> LoanOffers { get; private set; } = Array.Empty<LoanOffer>();
    public IReadOnlyList<string> SearchKeywords { get; private set; } = Array.Empty<string>();

    public void AttachInspection(InspectionReport report) => Inspection = report;

    public void RecalculateEmi(decimal interestRate, int tenureMonths)
    {
        // ...
    }
}

// Each client declares the slice it needs.
public sealed class SearchIndexer
{
    public void Index(IIndexable listing) { /* ... */ }
}

public sealed class InspectionWorkflow
{
    public void Complete(IInspectable listing, InspectionReport report)
        => listing.AttachInspection(report);
}
```

The test double for `SearchIndexer` now implements one property instead of eleven members.

**TypeScript — the structural-typing version**

TypeScript's structural typing makes ISP almost free: you can declare the slice at the point of
use, without the implementation knowing.

```ts
// ---------- BAD ----------
interface VehicleListing {
  id: number;
  registrationNumber: string;
  listedPrice: number;
  inspection: InspectionReport | null;
  attachInspection(r: InspectionReport): void;
  loanOffers: LoanOffer[];
  recalculateEmi(rate: number, tenureMonths: number): void;
  searchKeywords: string[];
  notifyDealer(message: string): void;
}

function indexListing(listing: VehicleListing): void {
  // needs exactly two fields, depends on nine
  searchEngine.put(listing.id, listing.searchKeywords);
}
```

```ts
// ---------- GOOD ----------
export interface ListingIdentity {
  readonly id: number;
  readonly registrationNumber: string;
}

export interface Indexable extends ListingIdentity {
  readonly searchKeywords: readonly string[];
}

export interface Inspectable {
  readonly inspection: InspectionReport | null;
  attachInspection(report: InspectionReport): void;
}

export interface Financeable {
  readonly loanOffers: readonly LoanOffer[];
  recalculateEmi(rate: number, tenureMonths: number): void;
}

export function indexListing(listing: Indexable): void {
  searchEngine.put(listing.id, [...listing.searchKeywords]);
}

// A test double is now trivial:
indexListing({ id: 1, registrationNumber: 'MH12AB1234', searchKeywords: ['hatchback', 'petrol'] });
```

Two TypeScript-specific notes:

- Because typing is structural, you rarely need `implements` at all. Declare the narrow interface
  next to the *consumer*, not next to the class. That is ISP in its purest form — the interface
  belongs to the client.
- Interfaces can be composed with `extends` or intersections (`Indexable & Inspectable`) at the
  call site, so there is no cost to keeping them small.

## Patterns that embody ISP

- [Adapter](../02-structural/01-adapter.md) — exists precisely to present the narrow interface a
  client wants over a wide one it does not.
- [Facade](../02-structural/05-facade.md) — a small interface onto a large subsystem.
- [Proxy](../02-structural/07-proxy.md) — same interface, narrower access; a protection proxy is
  ISP enforced at runtime.
- [Visitor](../03-behavioral/10-visitor.md) — each visitor is its own tiny interface over the
  hierarchy, instead of piling every operation onto the elements.
- [Bridge](../02-structural/02-bridge.md) — the implementor interface is deliberately minimal,
  containing only the primitives the abstraction needs.
- [Memento](../03-behavioral/05-memento.md) — the caretaker sees a deliberately empty interface;
  only the originator sees the wide one.

---

# 5️⃣ D — Dependency Inversion Principle

## Formal statement

> A. High-level modules should not depend on low-level modules. Both should depend on abstractions.
> B. Abstractions should not depend on details. Details should depend on abstractions.

## Plain English

Your business logic should not `import` your database driver. Define what you need as an interface
*in the business layer*, and let the infrastructure layer implement it. The arrow of dependency
points *inward*, toward the policy, away from the plumbing.

The word "inversion" is about that arrow. Without it, dependencies flow the way control flows:

```
  WITHOUT DIP  (dependency follows control)

   [ PricingEngine ]  ──uses──▶  [ SqlRateRepository ]  ──▶  [ System.Data.SqlClient ]
        (policy)                      (detail)

   Change the DB → recompile & retest the policy. Test the policy → need a database.

  WITH DIP  (dependency inverted at the boundary)

   [ PricingEngine ]  ──uses──▶  ( IRateSource )       ◀──implements──  [ SqlRateSource ]
        (policy)                  (abstraction,                              (detail)
                                   owned by the
                                   policy layer)

   The arrow from the detail now points INWARD. Policy compiles alone, tests alone.
```

The key detail everyone misses: **the interface belongs to the high-level module, not the
low-level one.** If `IRateSource` lives in your `Infrastructure` project, you have written an
interface, not inverted a dependency.

## The violation

**C# — bad**

```csharp
// In the Domain project. Note the using statements — the policy now depends on SQL and HTTP.
using System.Data.SqlClient;
using System.Net.Http;

public class PricingEngine
{
    private readonly string _connectionString;
    private readonly HttpClient _http = new HttpClient();

    public PricingEngine(string connectionString) => _connectionString = connectionString;

    public async Task<decimal> QuoteAsync(Listing listing)
    {
        // Low-level detail #1: SQL
        decimal baseRate;
        using (var conn = new SqlConnection(_connectionString))
        {
            await conn.OpenAsync();
            using var cmd = new SqlCommand(
                "SELECT Rate FROM DepreciationRates WHERE Make=@make AND Year=@year", conn);
            cmd.Parameters.AddWithValue("@make", listing.Make);
            cmd.Parameters.AddWithValue("@year", listing.Year);
            baseRate = (decimal)(await cmd.ExecuteScalarAsync() ?? 0m);
        }

        // Low-level detail #2: a third-party HTTP API
        var response = await _http.GetStringAsync(
            $"https://valuations.example.com/v2/market?reg={listing.RegistrationNumber}");
        var market = decimal.Parse(response);

        // The 5% of this method that is actually business policy:
        var depreciated = listing.AskingPrice * (1 - baseRate);
        return Math.Round((depreciated + market) / 2m, -2);
    }
}
```

To unit-test the one line of policy you need a SQL Server and a live third-party API. That is the
tell.

## The fix

**C# — good**

```csharp
// ---- Domain project: no infrastructure references at all ----
namespace Marketplace.Domain.Pricing;

public interface IDepreciationRates
{
    Task<decimal> RateForAsync(string make, int year, CancellationToken ct = default);
}

public interface IMarketValuations
{
    Task<decimal> MarketValueAsync(string registrationNumber, CancellationToken ct = default);
}

public sealed class PricingEngine
{
    private readonly IDepreciationRates _rates;
    private readonly IMarketValuations _valuations;

    public PricingEngine(IDepreciationRates rates, IMarketValuations valuations)
    {
        _rates = rates;
        _valuations = valuations;
    }

    public async Task<decimal> QuoteAsync(Listing listing, CancellationToken ct = default)
    {
        var rate   = await _rates.RateForAsync(listing.Make, listing.Year, ct);
        var market = await _valuations.MarketValueAsync(listing.RegistrationNumber, ct);

        var depreciated = listing.AskingPrice * (1 - rate);
        return Math.Round((depreciated + market) / 2m, -2);
    }
}

// ---- Infrastructure project: references Domain, not the other way round ----
namespace Marketplace.Infrastructure.Pricing;

public sealed class SqlDepreciationRates : IDepreciationRates
{
    private readonly string _connectionString;
    public SqlDepreciationRates(string connectionString) => _connectionString = connectionString;

    public async Task<decimal> RateForAsync(string make, int year, CancellationToken ct = default)
    {
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync(ct);
        using var cmd = new SqlCommand(
            "SELECT Rate FROM DepreciationRates WHERE Make=@make AND Year=@year", conn);
        cmd.Parameters.AddWithValue("@make", make);
        cmd.Parameters.AddWithValue("@year", year);
        var result = await cmd.ExecuteScalarAsync(ct);
        return result is decimal d ? d : 0m;
    }
}

public sealed class HttpMarketValuations : IMarketValuations
{
    private readonly HttpClient _http;
    public HttpMarketValuations(HttpClient http) => _http = http;

    public async Task<decimal> MarketValueAsync(string reg, CancellationToken ct = default)
    {
        var body = await _http.GetStringAsync($"/v2/market?reg={Uri.EscapeDataString(reg)}", ct);
        return decimal.Parse(body, CultureInfo.InvariantCulture);
    }
}
```

And the test needs nothing but memory:

```csharp
[Fact]
public async Task Quote_is_the_midpoint_of_depreciated_and_market_value()
{
    var engine = new PricingEngine(
        rates:      new StubRates(0.20m),
        valuations: new StubValuations(480_000m));

    var quote = await engine.QuoteAsync(new Listing
    {
        Make = "Maruti", Year = 2019, AskingPrice = 500_000m, RegistrationNumber = "MH12AB1234"
    });

    // depreciated = 400_000 ; market = 480_000 ; midpoint = 440_000
    Assert.Equal(440_000m, quote);
}

file sealed class StubRates : IDepreciationRates
{
    private readonly decimal _rate;
    public StubRates(decimal rate) => _rate = rate;
    public Task<decimal> RateForAsync(string make, int year, CancellationToken ct = default)
        => Task.FromResult(_rate);
}

file sealed class StubValuations : IMarketValuations
{
    private readonly decimal _value;
    public StubValuations(decimal value) => _value = value;
    public Task<decimal> MarketValueAsync(string reg, CancellationToken ct = default)
        => Task.FromResult(_value);
}
```

**TypeScript — bad then good**

```ts
// ---------- BAD ----------
import { Pool } from 'pg';
import fetch from 'node-fetch';

export class PricingEngine {
  private readonly pool = new Pool({ connectionString: process.env.DATABASE_URL });

  async quote(listing: Listing): Promise<number> {
    const { rows } = await this.pool.query(
      'SELECT rate FROM depreciation_rates WHERE make = $1 AND year = $2',
      [listing.make, listing.year],
    );
    const rate = rows[0]?.rate ?? 0;

    const res = await fetch(
      `https://valuations.example.com/v2/market?reg=${listing.registrationNumber}`,
    );
    const market = Number(await res.text());

    const depreciated = listing.askingPrice * (1 - rate);
    return Math.round((depreciated + market) / 200) * 100;
  }
}
```

```ts
// ---------- GOOD ----------
// src/domain/pricing.ts  — zero infrastructure imports
export interface DepreciationRates {
  rateFor(make: string, year: number): Promise<number>;
}

export interface MarketValuations {
  marketValue(registrationNumber: string): Promise<number>;
}

export class PricingEngine {
  constructor(
    private readonly rates: DepreciationRates,
    private readonly valuations: MarketValuations,
  ) {}

  async quote(listing: Listing): Promise<number> {
    const rate = await this.rates.rateFor(listing.make, listing.year);
    const market = await this.valuations.marketValue(listing.registrationNumber);

    const depreciated = listing.askingPrice * (1 - rate);
    return Math.round((depreciated + market) / 200) * 100;
  }
}

// src/infrastructure/pg-depreciation-rates.ts — imports the domain, not vice versa
import type { Pool } from 'pg';
import type { DepreciationRates } from '../domain/pricing';

export class PgDepreciationRates implements DepreciationRates {
  constructor(private readonly pool: Pool) {}

  async rateFor(make: string, year: number): Promise<number> {
    const { rows } = await this.pool.query<{ rate: number }>(
      'SELECT rate FROM depreciation_rates WHERE make = $1 AND year = $2',
      [make, year],
    );
    return rows[0]?.rate ?? 0;
  }
}

// src/infrastructure/http-market-valuations.ts
import type { MarketValuations } from '../domain/pricing';

export class HttpMarketValuations implements MarketValuations {
  constructor(private readonly baseUrl: string) {}

  async marketValue(registrationNumber: string): Promise<number> {
    const res = await fetch(
      `${this.baseUrl}/v2/market?reg=${encodeURIComponent(registrationNumber)}`,
    );
    if (!res.ok) throw new Error(`Valuation lookup failed: ${res.status}`);
    return Number(await res.text());
  }
}

// src/composition-root.ts — the ONE place that knows about both sides
export function buildPricingEngine(pool: Pool): PricingEngine {
  return new PricingEngine(
    new PgDepreciationRates(pool),
    new HttpMarketValuations('https://valuations.example.com'),
  );
}
```

## DIP vs Dependency Injection vs a DI container

These get conflated constantly. They are three things:

| | What it is |
| --- | --- |
| **Dependency Inversion (DIP)** | The design principle: policy defines the abstraction; details implement it. |
| **Dependency Injection (DI)** | The technique: pass collaborators in (usually via the constructor) instead of `new`-ing them inside. |
| **A DI container** | A tool (`Microsoft.Extensions.DependencyInjection`, `tsyringe`, `InversifyJS`) that wires the graph for you. Entirely optional. |

You can do DI by hand and get 100% of DIP's benefit — `buildPricingEngine` above is manual DI. The
container only saves typing. And you can use a container while still violating DIP, if the
interfaces live in the infrastructure layer.

## Patterns that embody DIP

- [Abstract Factory](../01-creational/02-abstract-factory.md) — the client depends on the abstract
  factory and abstract products; the concrete family is injected.
- [Factory Method](../01-creational/01-factory-method.md) — the framework calls back into a hook
  it does not know the implementation of.
- [Strategy](../03-behavioral/08-strategy.md) — the context depends on the strategy interface.
- [Bridge](../02-structural/02-bridge.md) — the abstraction depends on an implementor interface it
  defines.
- [Observer](../03-behavioral/06-observer.md) — the subject depends on an observer interface, not
  on the concrete subscribers.
- [Template Method](../03-behavioral/09-template-method.md) — the base algorithm depends on
  abstract hooks (this is the "Hollywood principle": don't call us, we'll call you).
- [Command](../03-behavioral/02-command.md) — the invoker depends only on the command interface.
- [Adapter](../02-structural/01-adapter.md) — how you make an existing third-party class satisfy
  an interface your domain defined.

---

# 🧩 Encapsulate What Varies

## Statement

> Identify the aspects of your application that vary and separate them from what stays the same.

This is the opening advice in the Gang of Four book's introduction, and arguably it is the parent
of all the others. Open/Closed is what you get when you apply it and then freeze the stable part.

## Plain English

Find the part of the code that keeps changing every sprint. Put a wall around it. Behind that wall,
change freely; in front of it, nothing notices.

## Two levels

**Encapsulation on a method** — extract the varying logic:

```ts
// BEFORE: the varying bit (how to decide "featured") is tangled into the stable bit (iterating)
export function featuredListings(all: Listing[]): Listing[] {
  return all.filter(
    (l) =>
      l.listedPrice > 300_000 &&
      l.photos.length >= 6 &&
      l.inspection !== null &&
      l.dealerTier === 'Premium',
  );
}

// AFTER: the rule has a name, a home, and a test of its own
export interface FeaturedRule {
  isFeatured(listing: Listing): boolean;
}

export class PremiumInspectedRule implements FeaturedRule {
  isFeatured(l: Listing): boolean {
    return (
      l.listedPrice > 300_000 && l.photos.length >= 6 &&
      l.inspection !== null && l.dealerTier === 'Premium'
    );
  }
}

export function featuredListings(all: Listing[], rule: FeaturedRule): Listing[] {
  return all.filter((l) => rule.isFeatured(l));
}
```

**Encapsulation on a class** — hide the field, expose the behaviour:

```csharp
// BAD: the invariant lives in whoever happens to touch the object
public class Listing
{
    public decimal ListedPrice;   // anyone can set this to -1
    public ListingStatus Status;  // anyone can jump Draft → Sold
}

// GOOD: the object owns its own rules; the representation can change freely
public sealed class Listing
{
    private decimal _listedPrice;

    public decimal ListedPrice => _listedPrice;
    public ListingStatus Status { get; private set; } = ListingStatus.Draft;

    public void Reprice(decimal newPrice)
    {
        if (newPrice <= 0) throw new ArgumentOutOfRangeException(nameof(newPrice));
        if (Status == ListingStatus.Sold)
            throw new InvalidOperationException("Cannot reprice a sold listing.");
        _listedPrice = newPrice;
    }

    public void Publish()
    {
        if (Status != ListingStatus.Draft)
            throw new InvalidOperationException($"Cannot publish from {Status}.");
        Status = ListingStatus.Live;
    }
}
```

## Patterns that embody it

Almost all of them, but most directly:
[Strategy](../03-behavioral/08-strategy.md) (varying algorithm),
[State](../03-behavioral/07-state.md) (varying behaviour by state),
[Factory Method](../01-creational/01-factory-method.md) and
[Abstract Factory](../01-creational/02-abstract-factory.md) (varying creation),
[Builder](../01-creational/03-builder.md) (varying construction steps),
[Template Method](../03-behavioral/09-template-method.md) (varying steps in a fixed algorithm),
[Bridge](../02-structural/02-bridge.md) (two independently varying dimensions),
[Decorator](../02-structural/04-decorator.md) (varying add-on responsibilities),
[Visitor](../03-behavioral/10-visitor.md) (varying operations),
[Memento](../03-behavioral/05-memento.md) (varying snapshot representation, hidden from the
caretaker),
[Flyweight](../02-structural/06-flyweight.md) (splitting the varying extrinsic state out of the
shared intrinsic state).

---

# 🔌 Program to an Interface, Not an Implementation

## Statement

> Program to an interface, not an implementation.

## Plain English

Declare your variables, parameters and fields as the most general type that does the job. The
concrete type should appear in exactly one place: where the object is created.

This is not "put an interface on everything". It is "do not let the concrete type leak into the
signature".

## The violation and the fix

```csharp
// BAD — the signature welds the caller to a concrete type and a concrete transport
public void RouteLeads(List<Lead> leads, RabbitMqLeadPublisher publisher)
{
    foreach (var lead in leads)
        publisher.PublishToRabbit(lead);
}

// GOOD — the caller states its need, not its supplier
public void RouteLeads(IEnumerable<Lead> leads, ILeadPublisher publisher)
{
    foreach (var lead in leads)
        publisher.Publish(lead);
}
```

Two independent wins there, worth separating:

1. `List<Lead>` → `IEnumerable<Lead>`: now you can pass an array, a LINQ query, a lazily-streamed
   result from a cursor. You never needed indexing.
2. `RabbitMqLeadPublisher` → `ILeadPublisher`: now you can swap RabbitMQ for an outbox table, or a
   no-op in tests, without touching this method.

```ts
// BAD
function routeLeads(leads: Lead[], publisher: RabbitMqLeadPublisher): Promise<void[]> {
  return Promise.all(leads.map((l) => publisher.publishToRabbit(l)));
}

// GOOD
export interface LeadPublisher {
  publish(lead: Lead): Promise<void>;
}

export async function routeLeads(
  leads: Iterable<Lead>,
  publisher: LeadPublisher,
): Promise<void> {
  for (const lead of leads) {
    await publisher.publish(lead);
  }
}
```

## The honest caveat

Do not create an `IFoo` for every `Foo` reflexively. An interface with exactly one implementation
that will never have another is pure ceremony — it doubles the file count and makes "go to
definition" useless. Create the abstraction when there is a *reason*: a second implementation, a
test seam at an I/O boundary, or a module boundary you want to keep clean.

The good news: in TypeScript, structural typing means you can often skip the `implements` keyword
and still get the benefit, and in C# you can start with a concrete class and extract the interface
later with one refactoring keystroke. So *not* having the interface yet is cheap to fix. Creating
fifty useless ones is not.

## Patterns that embody it

Every pattern that has a "Component", "Strategy", "Handler", "Observer", "Command", "State" or
"Implementor" role. Most explicitly:
[Strategy](../03-behavioral/08-strategy.md),
[Bridge](../02-structural/02-bridge.md),
[Adapter](../02-structural/01-adapter.md),
[Composite](../02-structural/03-composite.md),
[Iterator](../03-behavioral/03-iterator.md),
[Abstract Factory](../01-creational/02-abstract-factory.md),
[Prototype](../01-creational/04-prototype.md),
[Proxy](../02-structural/07-proxy.md).

---

# 🧱 Favour Composition Over Inheritance

## Statement

> Favour object composition over class inheritance.

## Plain English

Inheritance says *is-a* and gives you everything the parent has, forever, including the parts you
did not want. Composition says *has-a* and gives you only what you asked for, swappable at runtime.

## Why inheritance goes wrong: the combinatorial explosion

Suppose listings can be **promoted** (top of search), **certified** (inspected by us) and
**financed** (with pre-approved loan offers). Model that with inheritance:

```
                      Listing
                         │
        ┌────────────────┼────────────────┐
        │                │                │
  PromotedListing  CertifiedListing  FinancedListing
        │                │                │
        └───────┬────────┴────────┬───────┘
                │                 │
   PromotedCertifiedListing   PromotedFinancedListing
                │                 │
        CertifiedFinancedListing  │
                        │         │
              PromotedCertifiedFinancedListing

   3 flags → 7 subclasses.  4 flags → 15.  5 flags → 31.
   And nothing can change after construction.
```

With composition it is three small classes that stack in any order at runtime:

```
   new Promoted(
       new Certified(
           new Financed(
               new BasicListing(data))))

   3 flags → 3 classes.  n flags → n classes.
   Order and combination chosen at runtime.
```

**TypeScript — bad (inheritance) then good (composition)**

```ts
// ---------- BAD ----------
class BasicListing {
  constructor(protected readonly data: ListingData) {}
  displayPrice(): number { return this.data.listedPrice; }
  badges(): string[] { return []; }
}

class PromotedListing extends BasicListing {
  override displayPrice(): number { return super.displayPrice() + 999; }
  override badges(): string[] { return [...super.badges(), 'Promoted']; }
}

class CertifiedPromotedListing extends PromotedListing {
  override displayPrice(): number { return super.displayPrice() + 4999; }
  override badges(): string[] { return [...super.badges(), 'Certified']; }
}
// ...and now someone asks for certified-but-not-promoted. New class. And another. And another.
```

```ts
// ---------- GOOD ----------
export interface ListingView {
  displayPrice(): number;
  badges(): string[];
}

export class BasicListing implements ListingView {
  constructor(private readonly data: ListingData) {}
  displayPrice(): number { return this.data.listedPrice; }
  badges(): string[] { return []; }
}

abstract class ListingFeature implements ListingView {
  constructor(protected readonly inner: ListingView) {}
  displayPrice(): number { return this.inner.displayPrice(); }
  badges(): string[] { return this.inner.badges(); }
}

export class Promoted extends ListingFeature {
  override displayPrice(): number { return this.inner.displayPrice() + 999; }
  override badges(): string[] { return [...this.inner.badges(), 'Promoted']; }
}

export class Certified extends ListingFeature {
  override displayPrice(): number { return this.inner.displayPrice() + 4999; }
  override badges(): string[] { return [...this.inner.badges(), 'Certified']; }
}

export class Financed extends ListingFeature {
  override badges(): string[] { return [...this.inner.badges(), 'EMI available']; }
}

// Any combination, decided at runtime from a feature flag or a DB row:
export function decorate(base: ListingView, flags: ListingFlags): ListingView {
  let view = base;
  if (flags.certified) view = new Certified(view);
  if (flags.financed)  view = new Financed(view);
  if (flags.promoted)  view = new Promoted(view);
  return view;
}
```

That is [Decorator](../02-structural/04-decorator.md), arrived at from first principles.

**C# — the same idea**

```csharp
public interface IListingView
{
    decimal DisplayPrice();
    IReadOnlyList<string> Badges();
}

public sealed class BasicListing : IListingView
{
    private readonly Listing _listing;
    public BasicListing(Listing listing) => _listing = listing;
    public decimal DisplayPrice() => _listing.ListedPrice;
    public IReadOnlyList<string> Badges() => Array.Empty<string>();
}

public abstract class ListingFeature : IListingView
{
    protected readonly IListingView Inner;
    protected ListingFeature(IListingView inner) => Inner = inner;
    public virtual decimal DisplayPrice() => Inner.DisplayPrice();
    public virtual IReadOnlyList<string> Badges() => Inner.Badges();
}

public sealed class Promoted : ListingFeature
{
    public Promoted(IListingView inner) : base(inner) { }
    public override decimal DisplayPrice() => Inner.DisplayPrice() + 999m;
    public override IReadOnlyList<string> Badges()
        => Inner.Badges().Append("Promoted").ToList();
}

public sealed class Certified : ListingFeature
{
    public Certified(IListingView inner) : base(inner) { }
    public override decimal DisplayPrice() => Inner.DisplayPrice() + 4999m;
    public override IReadOnlyList<string> Badges()
        => Inner.Badges().Append("Certified").ToList();
}
```

## When inheritance IS right

"Favour" is not "never". Inheritance earns its keep when:

- The relationship is genuinely substitutable (passes the LSP checklist above).
- The base is stable and you control it — you are not inheriting across a library boundary.
- You are sharing a *contract*, not just code. Sharing code alone is what composition is for.
- The hierarchy is shallow — one or two levels. Depth is where fragility lives.

[Template Method](../03-behavioral/09-template-method.md) is the pattern that is *supposed* to use
inheritance, and it works. Note that Strategy is Template Method redone with composition, and that
the trade-off is real: Template Method is less code, Strategy is more flexible.

## Patterns that embody composition over inheritance

[Decorator](../02-structural/04-decorator.md),
[Strategy](../03-behavioral/08-strategy.md),
[Bridge](../02-structural/02-bridge.md),
[Composite](../02-structural/03-composite.md),
[Proxy](../02-structural/07-proxy.md),
[State](../03-behavioral/07-state.md),
[Adapter](../02-structural/01-adapter.md) (the object-adapter form),
[Command](../03-behavioral/02-command.md).

---

# ✂️ DRY, KISS, YAGNI — the working discipline

These are not design principles in the SOLID sense; they are habits for the hour-to-hour work. All
three are frequently misapplied, so each gets its counterweight.

## DRY — Don't Repeat Yourself

**Statement (Hunt & Thomas, *The Pragmatic Programmer*):** Every piece of knowledge must have a
single, unambiguous, authoritative representation within a system.

**Plain English:** Facts should live in one place. Note "knowledge", not "characters".

**The trap:** DRY is about *knowledge*, not *text*. Two functions that look identical but change
for different reasons are not duplication — they are a coincidence, and merging them creates a
coupling that will hurt later.

```csharp
// These LOOK identical. Do NOT merge them.

// Changes when the tax authority changes the rate on vehicle sales.
public decimal SalesTax(decimal amount) => amount * 0.18m;

// Changes when finance changes the platform's own service fee.
public decimal ServiceFee(decimal amount) => amount * 0.18m;
```

The day the tax rate moves to 20% and the service fee stays at 18%, the merged version breaks in a
way that is hard to spot. This is the "incidental duplication" trap, and it produces worse code
than the duplication it removed.

**Real duplication looks like this** — the same *rule* stated in several places:

```ts
// BAD: the "what makes a listing publishable" rule, stated three times, drifting apart
// api/publish.ts
if (!listing.photos.length || !listing.price || !listing.registrationNumber) throw new Error();
// admin/bulkPublish.ts
if (listing.photos.length < 1 || listing.price <= 0) throw new Error();   // already drifted
// worker/republish.ts
if (!listing.photos.length) throw new Error();                            // drifted further

// GOOD: one authoritative statement
export function publishabilityErrors(listing: Listing): string[] {
  const errors: string[] = [];
  if (listing.photos.length < 1) errors.push('At least one photo is required');
  if (listing.listedPrice <= 0) errors.push('Price must be positive');
  if (!listing.registrationNumber) errors.push('Registration number is required');
  return errors;
}
```

**Counterweight:** *"Duplication is far cheaper than the wrong abstraction."* — Sandi Metz. When in
doubt, wait for the third occurrence.

## KISS — Keep It Simple

**Plain English:** The simplest thing that could possibly work, and no simpler.

The pattern-specific hazard: patterns are how smart people write complicated code. A `Builder` for
a two-field object, an `AbstractFactory` when you have one family, a `Visitor` for a hierarchy with
one operation — each adds files, indirection and onboarding cost for no benefit.

```csharp
// KISS violation: a pattern applied because it was on the shelf
public interface IListingIdGenerationStrategy { long Next(); }
public sealed class SequentialIdStrategy : IListingIdGenerationStrategy { /* ... */ }
public sealed class ListingIdGeneratorFactory { /* ... */ }
public sealed class ListingIdGeneratorFactoryProvider { /* ... */ }

// KISS: the database already does this
// INSERT ... ; SELECT SCOPE_IDENTITY();
```

**Counterweight:** simple is not the same as short or clever. A 30-line explicit `switch` is
simpler than a 6-line regex-driven dispatch table nobody can debug.

## YAGNI — You Aren't Gonna Need It

**Plain English:** Do not build it until you actually need it. Not when you think you will need it
— when you need it.

```ts
// YAGNI violation, seen in every codebase: a pluggable everything, one implementation each
export interface NotificationChannel { send(m: Message): Promise<void>; }
export interface NotificationChannelFactory { create(type: string): NotificationChannel; }
export interface NotificationTemplateEngine { render(t: string, d: unknown): string; }
export interface NotificationRetryPolicy { shouldRetry(attempt: number, e: Error): boolean; }
// ...three years later, still only EmailChannel exists.

// YAGNI: ship this, extract an interface the day a second channel appears
export async function emailDealer(to: string, subject: string, body: string): Promise<void> {
  await mailer.sendMail({ to, subject, text: body });
}
```

**Counterweight — and this is important for a backend engineer:** YAGNI applies to *speculative
generality*, not to things that are expensive to retrofit. Some things you genuinely cannot bolt on
later at reasonable cost:

- Security boundaries and authorisation checks.
- Idempotency keys on message consumers (RabbitMQ delivers at-least-once; you *will* get
  duplicates, and retrofitting idempotency into a system that has already double-charged people is
  a bad week).
- Database schema decisions that need a migration on a hundred-million-row table.
- Anything that becomes a public API contract someone else depends on.

For those, design up front. For "we might want to support a second payment gateway someday",
YAGNI.

## The three in tension

```
   DRY   pulls toward  →  abstraction, sharing, indirection
   KISS  pulls toward  →  concreteness, fewer moving parts
   YAGNI pulls toward  →  doing less now, deciding later

   They disagree on purpose. The judgement is knowing which force
   the current code needs more of — and that is what experience is.
```

---

# 📊 The 22 patterns mapped to the principles they most embody

`●` = the pattern is largely *about* this principle. `○` = it supports it as a side effect.

Abbreviations: **SRP** Single Responsibility · **OCP** Open/Closed · **LSP** Liskov ·
**ISP** Interface Segregation · **DIP** Dependency Inversion · **EWV** Encapsulate What Varies ·
**PTI** Program To an Interface · **CoI** Composition over Inheritance.

## Creational

| Pattern | SRP | OCP | LSP | ISP | DIP | EWV | PTI | CoI | The one-line reason |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | --- |
| [Factory Method](../01-creational/01-factory-method.md) | ○ | ● | ○ | | ● | ● | ● | | Creation varies; subclasses decide, callers depend on the product interface. |
| [Abstract Factory](../01-creational/02-abstract-factory.md) | ○ | ● | ○ | ○ | ● | ● | ● | ○ | A whole product family is what varies; the client sees only abstractions. |
| [Builder](../01-creational/03-builder.md) | ● | ○ | | | | ● | ○ | ● | Construction is its own responsibility; steps vary, the director stays fixed. |
| [Prototype](../01-creational/04-prototype.md) | ○ | ○ | ○ | | | ● | ○ | ● | Copy an existing object instead of knowing its class or its constructor. |
| [Singleton](../01-creational/05-singleton.md) | ○ | | | | | | | | Controls instantiation — and fights DIP and testability; use sparingly. |

## Structural

| Pattern | SRP | OCP | LSP | ISP | DIP | EWV | PTI | CoI | The one-line reason |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | --- |
| [Adapter](../02-structural/01-adapter.md) | ● | ○ | ● | ● | ● | ○ | ● | ● | Makes a foreign class substitutable for the interface your code owns. |
| [Bridge](../02-structural/02-bridge.md) | ● | ● | ○ | ● | ● | ● | ● | ● | Two dimensions vary independently; the abstraction owns a minimal implementor interface. |
| [Composite](../02-structural/03-composite.md) | ○ | ● | ● | ○ | | ● | ● | ● | Leaf and composite must be interchangeable; trees built by containment. |
| [Decorator](../02-structural/04-decorator.md) | ● | ● | ● | ○ | ○ | ● | ● | ● | Add responsibilities by wrapping — the poster child for composition over inheritance. |
| [Facade](../02-structural/05-facade.md) | ● | ○ | | ● | ○ | ● | ○ | ● | One narrow interface onto a wide subsystem; decoupling is its whole job. |
| [Flyweight](../02-structural/06-flyweight.md) | ● | | ○ | | | ● | ○ | ● | Separates intrinsic (shared) state from extrinsic (varying) state. |
| [Proxy](../02-structural/07-proxy.md) | ● | ● | ● | ● | ○ | ● | ● | ● | Same interface, extra concern (lazy load, cache, access control) added around it. |

## Behavioural

| Pattern | SRP | OCP | LSP | ISP | DIP | EWV | PTI | CoI | The one-line reason |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | --- |
| [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | ● | ● | ● | ○ | ○ | ● | ● | ● | Each handler has one job; add handlers without editing the chain. |
| [Command](../03-behavioral/02-command.md) | ● | ● | ● | ● | ● | ● | ● | ● | One class per action; the invoker knows only the command interface. |
| [Iterator](../03-behavioral/03-iterator.md) | ● | ● | ● | ● | ○ | ● | ● | ○ | Traversal is separated from storage, behind a two-method interface. |
| [Mediator](../03-behavioral/04-mediator.md) | ● | ● | | ○ | ● | ● | ● | ● | Coordination becomes one object's responsibility instead of an N×N mesh. |
| [Memento](../03-behavioral/05-memento.md) | ● | | | ● | | ● | ○ | ● | Snapshot representation is encapsulated; the caretaker sees an opaque token. |
| [Observer](../03-behavioral/06-observer.md) | ● | ● | ● | ● | ● | ● | ● | ● | The subject depends on an observer interface; subscribers added without edits. |
| [State](../03-behavioral/07-state.md) | ● | ● | ● | ○ | ○ | ● | ● | ● | Behaviour-by-state is what varies; a new state is a new class, not a new branch. |
| [Strategy](../03-behavioral/08-strategy.md) | ● | ● | ● | ● | ● | ● | ● | ● | The complete set: every principle in this chapter, in one small pattern. |
| [Template Method](../03-behavioral/09-template-method.md) | ○ | ● | ● | ○ | ● | ● | ○ | | Fixed skeleton, varying hooks — inversion of control via inheritance. |
| [Visitor](../03-behavioral/10-visitor.md) | ● | ● | ○ | ● | ○ | ● | ● | ○ | Operations move out of the elements; open for new operations. |

### Reading the table

- **Strategy, Command and Observer score on everything.** That is not an accident — they are the
  three patterns that are purest "polymorphism applied to a varying thing", which is what most of
  these principles are describing from different angles.
- **Singleton scores on almost nothing,** and actively works against DIP (global access instead of
  injection) and against testability. It is in the book, and it is the one to reach for last. If
  you need exactly one instance, register it as a singleton *lifetime* in your DI container and
  inject it like anything else.
- **Template Method is the odd one out on Composition over Inheritance** — it is the pattern that
  deliberately uses inheritance. That is fine, and it is the reason "favour" is not "always".
- **Adapter, Proxy and Decorator all score high on LSP** because all three are impersonations. An
  impersonation that changes observable behaviour beyond its stated job is a bug by definition.

---

## 🎯 What to actually do with this

A short, honest workflow:

1. **When something hurts, name the principle it violates.** "This is hard to test" is usually DIP.
   "Every change touches this file" is usually SRP or OCP. "I had to add an `instanceof`" is
   always LSP.
2. **Then, and only then, look for a pattern.** The principle tells you what is wrong; the pattern
   is a known-good shape for the fix. Going the other way — picking a pattern first — is how you
   get `ListingIdGeneratorFactoryProvider`.
3. **Prefer the smallest fix that removes the pain.** Extracting an interface is smaller than
   introducing Abstract Factory. Try the small one first.
4. **Re-read the LSP checklist before every `extends`.** It is the cheapest bug-prevention ritual
   in this whole document.

Next, if you have not already: [Strategy](../03-behavioral/08-strategy.md) shows all eight of these
principles working together in about eighty lines of code, and is the best single pattern to
internalise first.
