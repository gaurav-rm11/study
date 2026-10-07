# 🧰 Your Stack Playbook — Patterns by Technology

Every other file in this folder is organised by *pattern*. This one is organised by *what you
actually type all day*: C#, TypeScript, SQL, RabbitMQ. The question it answers is not "what is the
Decorator pattern" but "I am staring at a `DelegatingHandler` — which pattern is that, and should I
be hand-rolling anything here at all?"

The short version, before anything else:

> **Most of the Gang of Four patterns are already inside your framework.** Your job is usually to
> *recognise* them and use the idiomatic hook, not to re-implement them. Hand-rolling a pattern that
> the runtime already provides is the single most common way engineers make a codebase worse while
> believing they are making it better.

Everything below is written against an automotive marketplace: car listings, dealers, pricing,
search, notifications. Those are the nouns in the examples.

```
                 ┌──────────────────────────────────────────────┐
                 │            Your daily surface                │
                 └──────────────────────────────────────────────┘
   TypeScript/Node        C# / .NET          SQL              RabbitMQ
   ─────────────────  ─────────────────  ─────────────  ─────────────────
   closures            DI container       Repository      pub/sub
   unions              middleware         Specification   command vs event
   Proxy               IEnumerable        Query Object    retry policies
   generators          IObservable        Identity Map    outbox
   RxJS                DelegatingHandler  pooling         dead-letter
   decorators          MediatR            lazy loading    idempotency
        │                    │                 │                │
        └────────────────────┴────────┬────────┴────────────────┘
                                      ▼
                        the same ~15 ideas, renamed
```

---

## 1. 🟦 C# / .NET — the framework already did it

### 1.1 The DI container is Factory + Abstract Factory + Singleton, all at once

This is the big one. `IServiceCollection` / `IServiceProvider` *is* a configurable object factory,
and the lifetimes are the object-lifecycle patterns.

| You were about to write | .NET already gives you |
| --- | --- |
| [Singleton](../01-creational/05-singleton.md) with a static instance | `AddSingleton<T>()` |
| [Factory Method](../01-creational/01-factory-method.md) | `AddTransient<TInterface, TImpl>()` |
| [Abstract Factory](../01-creational/02-abstract-factory.md) | a factory *interface* registered in DI |
| Service locator | constructor injection |

Idiomatic:

```csharp
// Program.cs
builder.Services.AddSingleton<IPriceIndexCache, RedisPriceIndexCache>();   // one for the process
builder.Services.AddScoped<IListingRepository, SqlListingRepository>();    // one per HTTP request
builder.Services.AddTransient<IValuationCalculator, ValuationCalculator>(); // new every time

// Constructor injection — no `new`, no static, no locator.
public sealed class ListingService
{
    private readonly IListingRepository _repo;
    private readonly IValuationCalculator _valuation;

    public ListingService(IListingRepository repo, IValuationCalculator valuation)
    {
        _repo = repo;
        _valuation = valuation;
    }

    public async Task<ListingView> GetAsync(int listingId, CancellationToken ct)
    {
        var listing = await _repo.GetAsync(listingId, ct)
                      ?? throw new ListingNotFoundException(listingId);
        var price = _valuation.Estimate(listing);
        return ListingView.From(listing, price);
    }
}
```

**When you *do* need a factory by hand:** when the thing you create depends on a *runtime* value the
container cannot know — a dealer id, a tenant, a message payload.

```csharp
public interface IPricingStrategyFactory
{
    IPricingStrategy For(ListingChannel channel);
}

public sealed class PricingStrategyFactory : IPricingStrategyFactory
{
    private readonly IReadOnlyDictionary<ListingChannel, IPricingStrategy> _strategies;

    // DI hands you every registered strategy; you index them.
    public PricingStrategyFactory(IEnumerable<IPricingStrategy> strategies)
        => _strategies = strategies.ToDictionary(s => s.Channel);

    public IPricingStrategy For(ListingChannel channel)
        => _strategies.TryGetValue(channel, out var s)
            ? s
            : throw new NotSupportedException($"No pricing strategy for {channel}");
}
```

Register all implementations and the factory:

```csharp
builder.Services.AddSingleton<IPricingStrategy, RetailPricingStrategy>();
builder.Services.AddSingleton<IPricingStrategy, AuctionPricingStrategy>();
builder.Services.AddSingleton<IPricingStrategy, CertifiedPreOwnedPricingStrategy>();
builder.Services.AddSingleton<IPricingStrategyFactory, PricingStrategyFactory>();
```

**Wrong move:** writing `public static readonly ListingService Instance = new();` in a project that
already has a DI container. You lose testability, you lose lifetime control, and you gain nothing.
A container-registered singleton is a [Singleton](../01-creational/05-singleton.md) without the
global-state tax.

### 1.2 Middleware = Chain of Responsibility

The ASP.NET Core request pipeline is a textbook
[Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md): each handler decides to
process, to pass along, or to short-circuit.

```csharp
public sealed class DealerRateLimitMiddleware
{
    private readonly RequestDelegate _next;   // <-- the "next link"
    private readonly IRateLimiter _limiter;

    public DealerRateLimitMiddleware(RequestDelegate next, IRateLimiter limiter)
    {
        _next = next;
        _limiter = limiter;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var dealerId = context.Request.Headers["X-Dealer-Id"].ToString();

        if (!string.IsNullOrEmpty(dealerId) && !await _limiter.TryAcquireAsync(dealerId))
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            context.Response.Headers.RetryAfter = "30";
            return;                    // short-circuit: the chain stops here
        }

        await _next(context);          // pass to the next handler
    }
}

// Program.cs — order IS the chain order.
app.UseExceptionHandler("/error");
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<DealerRateLimitMiddleware>();
app.MapControllers();
```

```
Request ─▶ ExceptionHandler ─▶ AuthN ─▶ AuthZ ─▶ RateLimit ─▶ Endpoint
               ▲                                    │
               └──────────── 429, chain ends ◀──────┘
```

**Wrong move:** building your own `IHandler { IHandler Next { get; set; } }` interface for
cross-cutting HTTP concerns. Use middleware. Keep hand-rolled chains for *domain* pipelines — listing
moderation, say — where the links are business rules, not HTTP concerns.

### 1.3 `IEnumerable<T>` and `yield` = Iterator

You will essentially never implement
[Iterator](../03-behavioral/03-iterator.md) by hand in C#.

```csharp
public IEnumerable<Listing> ExpiringSoon(IEnumerable<Listing> all, DateOnly today)
{
    foreach (var listing in all)
    {
        if (listing.ExpiresOn <= today.AddDays(7))
            yield return listing;      // lazy: nothing runs until you enumerate
    }
}
```

Async streaming — the same pattern, over I/O:

```csharp
public async IAsyncEnumerable<Listing> StreamDealerInventoryAsync(
    int dealerId,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var conn = new SqlConnection(_connectionString);
    await conn.OpenAsync(ct);

    await using var cmd = new SqlCommand(
        "SELECT Id, Make, Model, Year, PriceInr FROM Listings WHERE DealerId = @dealerId", conn);
    cmd.Parameters.AddWithValue("@dealerId", dealerId);

    await using var reader = await cmd.ExecuteReaderAsync(ct);
    while (await reader.ReadAsync(ct))
    {
        yield return new Listing(
            Id: reader.GetInt32(0),
            Make: reader.GetString(1),
            Model: reader.GetString(2),
            Year: reader.GetInt16(3),
            PriceInr: reader.GetDecimal(4));
    }
}
```

Consumed with `await foreach`, this never materialises 200k listings in memory.

**Wrong move:** implementing `IEnumerator<T>` with `MoveNext`/`Current`/`Reset` by hand. `yield`
generates that state machine for you. The only reason to write the interface manually is a very hot
loop where you want a struct enumerator with zero allocations — and you should measure first.

### 1.4 `event` and `IObservable<T>` = Observer

[Observer](../03-behavioral/06-observer.md) has two C# spellings.

```csharp
// (a) Classic events — in-process, synchronous, fine for small things.
public sealed class ListingPublisher
{
    public event EventHandler<ListingPublishedEventArgs>? Published;

    public void Publish(Listing listing)
    {
        _store.MarkLive(listing.Id);
        Published?.Invoke(this, new ListingPublishedEventArgs(listing));
    }
}
```

Events have real problems at scale: subscribers run synchronously on the publisher's thread, one
throwing subscriber breaks the rest, and forgetting to unsubscribe leaks the subscriber. For anything
that crosses a module boundary, prefer a mediator (below) or a real broker (section 4).

```csharp
// (b) IObservable<T> via System.Reactive — composable streams.
IObservable<PriceChanged> priceChanges = _priceFeed.Stream();

using var subscription = priceChanges
    .Where(p => p.Make == "Maruti")
    .Throttle(TimeSpan.FromSeconds(2))       // collapse bursts
    .DistinctUntilChanged(p => p.NewPriceInr)
    .Subscribe(
        onNext: p => _search.ReindexAsync(p.ListingId),
        onError: ex => _logger.LogError(ex, "Price feed failed"));
```

`IDisposable` from `Subscribe` is the unsubscribe handle — the pattern's "detach", made explicit.

### 1.5 `DelegatingHandler` = Decorator

An `HttpClient` handler chain wraps behaviour around a request without the caller knowing.
That is [Decorator](../02-structural/04-decorator.md) (and, viewed from the side, a chain again).

```csharp
public sealed class DealerApiAuthHandler : DelegatingHandler
{
    private readonly ITokenProvider _tokens;
    public DealerApiAuthHandler(ITokenProvider tokens) => _tokens = tokens;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokens.GetAsync(ct);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        return await base.SendAsync(request, ct);      // delegate to the wrapped handler
    }
}

public sealed class CorrelationIdHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        request.Headers.TryAddWithoutValidation(
            "X-Correlation-Id", Activity.Current?.Id ?? Guid.NewGuid().ToString("n"));
        return base.SendAsync(request, ct);
    }
}
```

Wiring:

```csharp
builder.Services.AddTransient<DealerApiAuthHandler>();
builder.Services.AddTransient<CorrelationIdHandler>();

builder.Services.AddHttpClient<IDealerFeedClient, DealerFeedClient>(c =>
{
    c.BaseAddress = new Uri("https://feeds.example-dealer-network.test/");
    c.Timeout = TimeSpan.FromSeconds(10);
})
.AddHttpMessageHandler<CorrelationIdHandler>()
.AddHttpMessageHandler<DealerApiAuthHandler>();
```

Decorating *your own* services is also idiomatic — just do it with a constructor that takes the inner
implementation:

```csharp
public sealed class CachingListingRepository : IListingRepository
{
    private readonly IListingRepository _inner;
    private readonly IMemoryCache _cache;

    public CachingListingRepository(IListingRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Listing?> GetAsync(int id, CancellationToken ct)
    {
        if (_cache.TryGetValue<Listing>(id, out var cached)) return cached;

        var listing = await _inner.GetAsync(id, ct);
        if (listing is not null)
            _cache.Set(id, listing, TimeSpan.FromMinutes(5));
        return listing;
    }

    public Task SaveAsync(Listing listing, CancellationToken ct)
    {
        _cache.Remove(listing.Id);
        return _inner.SaveAsync(listing, ct);
    }
}
```

The built-in container has no first-class `Decorate()` call, so register it explicitly:

```csharp
builder.Services.AddScoped<SqlListingRepository>();
builder.Services.AddScoped<IListingRepository>(sp =>
    new CachingListingRepository(
        sp.GetRequiredService<SqlListingRepository>(),
        sp.GetRequiredService<IMemoryCache>()));
```

### 1.6 `IHostedService` / `BackgroundService` = Template Method

`BackgroundService` defines the skeleton (start, run, stop, dispose) and leaves one hole for you:
[Template Method](../03-behavioral/09-template-method.md).

```csharp
public sealed class StaleListingSweeper : BackgroundService
{
    private readonly IServiceScopeFactory _scopes;
    private readonly ILogger<StaleListingSweeper> _logger;

    public StaleListingSweeper(IServiceScopeFactory scopes, ILogger<StaleListingSweeper> logger)
    {
        _scopes = scopes;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(15));

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                using var scope = _scopes.CreateScope();     // scoped services need a scope here
                var repo = scope.ServiceProvider.GetRequiredService<IListingRepository>();
                var expired = await repo.ExpireOlderThanAsync(DateTime.UtcNow.AddDays(-60), stoppingToken);
                _logger.LogInformation("Expired {Count} stale listings", expired);
            }
            catch (OperationCanceledException) { throw; }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Sweep failed; will retry next tick");
            }
        }
    }
}
```

Note the shape: the base class owns the lifecycle, you own one method. That is the whole pattern.

### 1.7 `ILogger` = Facade (with a bit of Strategy and Adapter)

`ILogger<T>` gives you one simple face over a pile of machinery — providers, scopes, filters,
formatters. [Facade](../02-structural/05-facade.md) for the caller; each provider is a
[Strategy](../03-behavioral/08-strategy.md); each provider that wraps Serilog or OpenTelemetry is an
[Adapter](../02-structural/01-adapter.md).

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
       {
           ["DealerId"] = dealerId,
           ["ListingId"] = listingId
       }))
{
    _logger.LogInformation("Repricing listing from {OldPrice} to {NewPrice}", old, @new);
}
```

Use structured message templates, never string interpolation, or your log backend cannot index the
fields.

### 1.8 The Options pattern = Builder + Prototype-ish configuration

```csharp
public sealed class SearchIndexOptions
{
    public const string SectionName = "SearchIndex";

    [Required] public string Endpoint { get; init; } = "";
    [Range(1, 1000)] public int BatchSize { get; init; } = 100;
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

builder.Services
    .AddOptions<SearchIndexOptions>()
    .Bind(builder.Configuration.GetSection(SearchIndexOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Then inject the right flavour:

- `IOptions<T>` — singleton, read once. Most cases.
- `IOptionsSnapshot<T>` — per scope, picks up config reloads. Scoped services only.
- `IOptionsMonitor<T>` — singleton with change notification (`OnChange`). Needed inside singletons
  and hosted services.

```csharp
public sealed class SearchIndexer
{
    private readonly IOptionsMonitor<SearchIndexOptions> _options;
    public SearchIndexer(IOptionsMonitor<SearchIndexOptions> options) => _options = options;

    public int CurrentBatchSize => _options.CurrentValue.BatchSize;
}
```

**Wrong move:** injecting `IConfiguration` everywhere and doing `config["SearchIndex:BatchSize"]`.
You lose validation, typing, and reload semantics.

### 1.9 MediatR = Mediator + Command

[Mediator](../03-behavioral/04-mediator.md) says objects should talk through a hub instead of
directly. [Command](../03-behavioral/02-command.md) says a request should be an object.
MediatR is both.

```csharp
// Command (a request object)
public sealed record RepriceListingCommand(int ListingId, decimal NewPriceInr, string ChangedBy)
    : IRequest<RepriceResult>;

// Handler
public sealed class RepriceListingHandler : IRequestHandler<RepriceListingCommand, RepriceResult>
{
    private readonly IListingRepository _repo;
    private readonly IPublishEndpoint _events;

    public RepriceListingHandler(IListingRepository repo, IPublishEndpoint events)
    {
        _repo = repo;
        _events = events;
    }

    public async Task<RepriceResult> Handle(RepriceListingCommand cmd, CancellationToken ct)
    {
        var listing = await _repo.GetAsync(cmd.ListingId, ct);
        if (listing is null) return RepriceResult.NotFound;
        if (cmd.NewPriceInr <= 0) return RepriceResult.Invalid("Price must be positive");

        var old = listing.PriceInr;
        listing.PriceInr = cmd.NewPriceInr;
        await _repo.SaveAsync(listing, ct);

        await _events.Publish(new ListingRepriced(listing.Id, old, cmd.NewPriceInr, cmd.ChangedBy), ct);
        return RepriceResult.Ok;
    }
}

// Controller stays thin
[HttpPost("listings/{id:int}/price")]
public async Task<IActionResult> Reprice(int id, RepriceRequest body, CancellationToken ct)
{
    var result = await _mediator.Send(
        new RepriceListingCommand(id, body.PriceInr, User.Identity?.Name ?? "system"), ct);

    return result.Kind switch
    {
        ResultKind.Ok       => NoContent(),
        ResultKind.NotFound => NotFound(),
        _                   => BadRequest(result.Error)
    };
}
```

Pipeline behaviours are Chain of Responsibility again, applied to commands:

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger) => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var sw = Stopwatch.StartNew();
        try { return await next(); }
        finally
        {
            _logger.LogInformation("{Request} handled in {Ms}ms",
                typeof(TRequest).Name, sw.ElapsedMilliseconds);
        }
    }
}
```

**Honest caveat.** MediatR is worth it when you genuinely want a uniform pipeline
(validation, logging, transactions, authorisation) across many use cases. In a small service it adds
indirection — a controller calling a service class directly is easier to follow. "Every call goes
through `_mediator.Send` " is not automatically good architecture; it is good only when the pipeline
is earning its keep.

### 1.10 Quick reference — C# built-ins

```
Pattern                    Idiomatic .NET                       Hand-roll when
─────────────────────────  ───────────────────────────────────  ────────────────────────
Singleton                  AddSingleton<T>()                    never (in DI apps)
Factory Method             AddTransient / Func<T> injection     runtime key decides type
Abstract Factory           a factory interface in DI            multi-provider families
Builder                    object initialisers, `with`          many optional steps + validation
Iterator                   yield / IAsyncEnumerable             zero-alloc hot path only
Observer                   event / IObservable<T>               never
Decorator                  DelegatingHandler, wrapping ctor     always fine, it's cheap
Chain of Responsibility    middleware, MediatR behaviours       domain rule pipelines
Mediator                   MediatR                              rarely
Command                    MediatR IRequest, delegates          undo/redo, queued work
Strategy                   interface + DI, or Func<>            always fine
Template Method            BackgroundService, base classes      prefer composition first
Adapter                    a thin wrapper class                 always fine
Facade                     a service class                      always fine
```

---

## 2. 🟨 TypeScript / Node — the language eats half the catalogue

A pattern exists to work around a language limitation. TypeScript has fewer of the limitations the
GoF book was written against, so several patterns collapse into a function.

### 2.1 Closures replace Strategy and Command

Java-style Strategy in TS is often noise:

```ts
// Ceremony you do not need
interface SortStrategy { sort(listings: Listing[]): Listing[] }
class PriceAscStrategy implements SortStrategy { sort(l: Listing[]) { /* ... */ return l } }
```

The TypeScript version:

```ts
type Listing = {
  id: number
  make: string
  model: string
  year: number
  priceInr: number
  kmDriven: number
  postedAt: string      // ISO
}

type SortStrategy = (a: Listing, b: Listing) => number

const sorts = {
  priceAsc:   (a, b) => a.priceInr - b.priceInr,
  priceDesc:  (a, b) => b.priceInr - a.priceInr,
  newest:     (a, b) => Date.parse(b.postedAt) - Date.parse(a.postedAt),
  lowestKm:   (a, b) => a.kmDriven - b.kmDriven,
} satisfies Record<string, SortStrategy>

export type SortKey = keyof typeof sorts

export function sortListings(listings: Listing[], key: SortKey): Listing[] {
  return [...listings].sort(sorts[key])
}
```

`satisfies` keeps the literal keys (so `SortKey` is a precise union) while still type-checking each
function. That is the whole [Strategy](../03-behavioral/08-strategy.md) pattern, in a map.

Command, likewise, is usually a closure plus a label:

```ts
type Command = { readonly label: string; execute(): void; undo(): void }

function makePriceCommand(listing: Listing, newPrice: number): Command {
  const previous = listing.priceInr           // captured state = the Memento
  return {
    label: `Reprice ${listing.make} ${listing.model} to ₹${newPrice}`,
    execute: () => { listing.priceInr = newPrice },
    undo:    () => { listing.priceInr = previous },
  }
}

class History {
  private readonly done: Command[] = []
  run(cmd: Command) { cmd.execute(); this.done.push(cmd) }
  undo() { this.done.pop()?.undo() }
}
```

### 2.2 Discriminated unions replace much of State

[State](../03-behavioral/07-state.md) with a class per state is heavy in TS. A tagged union plus an
exhaustive switch gives you compile-time safety that the class version does not.

```ts
type ListingState =
  | { status: 'draft';     editedBy: string }
  | { status: 'inReview';  submittedAt: string; reviewer?: string }
  | { status: 'live';      publishedAt: string; expiresAt: string }
  | { status: 'sold';      soldAt: string; salePriceInr: number }
  | { status: 'expired';   expiredAt: string }

type ListingEvent =
  | { type: 'submit' }
  | { type: 'approve'; at: string }
  | { type: 'reject'; reason: string }
  | { type: 'markSold'; priceInr: number }
  | { type: 'expire' }

function assertNever(x: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`)
}

export function transition(state: ListingState, event: ListingEvent): ListingState {
  switch (state.status) {
    case 'draft':
      return event.type === 'submit'
        ? { status: 'inReview', submittedAt: new Date().toISOString() }
        : state

    case 'inReview':
      if (event.type === 'approve') {
        const publishedAt = event.at
        const expiresAt = new Date(Date.parse(publishedAt) + 60 * 864e5).toISOString()
        return { status: 'live', publishedAt, expiresAt }
      }
      if (event.type === 'reject') {
        return { status: 'draft', editedBy: 'system' }
      }
      return state

    case 'live':
      if (event.type === 'markSold')
        return { status: 'sold', soldAt: new Date().toISOString(), salePriceInr: event.priceInr }
      if (event.type === 'expire')
        return { status: 'expired', expiredAt: new Date().toISOString() }
      return state

    case 'sold':
    case 'expired':
      return state                     // terminal

    default:
      return assertNever(state)        // add a state, this line fails to compile
  }
}
```

The `assertNever` trick is the point: add a sixth status and the compiler tells you every switch that
forgot it. A class-based State machine gives you no such guarantee.

Use classes for State when each state carries substantial *behaviour* (many methods), not just data.

### 2.3 `Proxy` is a built-in

JavaScript ships [Proxy](../02-structural/07-proxy.md) as a language primitive.

```ts
function auditReads<T extends object>(target: T, onRead: (prop: string) => void): T {
  return new Proxy(target, {
    get(obj, prop, receiver) {
      if (typeof prop === 'string') onRead(prop)
      return Reflect.get(obj, prop, receiver)
    },
  })
}

const listing = auditReads(rawListing, p => metrics.increment(`listing.field.${p}`))
```

A lazy-loading proxy for an expensive relation:

```ts
type DealerProfile = { id: number; name: string; city: string; rating: number }

function lazyDealer(dealerId: number, load: (id: number) => Promise<DealerProfile>) {
  let promise: Promise<DealerProfile> | undefined
  return {
    get value(): Promise<DealerProfile> {
      promise ??= load(dealerId)      // fetch once, cache the promise
      return promise
    },
  }
}
```

Be careful: `Proxy` traps are a real performance cost and they confuse debuggers. Use them for
framework-level concerns (validation, reactivity, mocking), not for everyday domain code.

### 2.4 Generators = Iterator

```ts
async function* paginateListings(
  fetchPage: (cursor?: string) => Promise<{ items: Listing[]; next?: string }>
): AsyncGenerator<Listing> {
  let cursor: string | undefined
  do {
    const page = await fetchPage(cursor)
    for (const item of page.items) yield item
    cursor = page.next
  } while (cursor)
}

// Consumer never knows pagination exists.
for await (const listing of paginateListings(fetchListingPage)) {
  if (listing.priceInr > 2_000_000) await flagForReview(listing.id)
}
```

Sync generators compose into pipelines, lazily:

```ts
function* filter<T>(src: Iterable<T>, pred: (x: T) => boolean): Generator<T> {
  for (const x of src) if (pred(x)) yield x
}
function* map<T, U>(src: Iterable<T>, fn: (x: T) => U): Generator<U> {
  for (const x of src) yield fn(x)
}
function* take<T>(src: Iterable<T>, n: number): Generator<T> {
  let i = 0
  for (const x of src) { if (i++ >= n) return; yield x }
}

const topCheapMarutis = [...take(
  map(filter(allListings, l => l.make === 'Maruti'), l => `${l.model} — ₹${l.priceInr}`),
  10)]
```

Nothing is computed until the spread pulls, and only ten items are ever processed past `take`.

### 2.5 RxJS = Observer, industrial grade

```ts
import { fromEvent, merge } from 'rxjs'
import { debounceTime, distinctUntilChanged, map, switchMap, catchError, startWith } from 'rxjs/operators'
import { of } from 'rxjs'

const input = document.querySelector<HTMLInputElement>('#search')!
const makeFilter = document.querySelector<HTMLSelectElement>('#make')!

const query$ = fromEvent(input, 'input').pipe(
  map(e => (e.target as HTMLInputElement).value.trim()),
  debounceTime(300),
  distinctUntilChanged()
)

const make$ = fromEvent(makeFilter, 'change').pipe(
  map(e => (e.target as HTMLSelectElement).value),
  startWith('')
)

const results$ = merge(query$, make$).pipe(
  switchMap(() => searchListings(input.value, makeFilter.value)),  // cancels the previous request
  catchError(err => { console.error(err); return of([] as Listing[]) })
)

const sub = results$.subscribe(renderResults)
// later: sub.unsubscribe()
```

`switchMap` is the part you cannot get from plain callbacks without pain: it cancels the in-flight
search when a newer keystroke arrives, killing the classic stale-results race.

### 2.6 Express / Koa middleware = Chain of Responsibility

```ts
import type { Request, Response, NextFunction } from 'express'

export function requireDealer(req: Request, res: Response, next: NextFunction) {
  const dealerId = req.header('x-dealer-id')
  if (!dealerId) return res.status(401).json({ error: 'dealer header missing' })
  ;(req as Request & { dealerId: string }).dealerId = dealerId
  next()                                   // pass along
}

export function timing(req: Request, res: Response, next: NextFunction) {
  const start = process.hrtime.bigint()
  res.on('finish', () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6
    console.log(JSON.stringify({ path: req.path, status: res.statusCode, ms }))
  })
  next()
}

app.use(timing)
app.use('/api/dealer', requireDealer, dealerRouter)
```

Koa's `async (ctx, next) => { await next() }` shape is the same chain, but each link wraps *both*
sides of the call — which makes it a Decorator as well.

### 2.7 NestJS decorators — Decorator, Adapter, and DI at once

```ts
import { Injectable, Controller, Get, Param, UseGuards, UseInterceptors } from '@nestjs/common'

@Injectable()
export class ListingService {
  constructor(private readonly repo: ListingRepository) {}
  findOne(id: number) { return this.repo.findById(id) }
}

@Controller('listings')
@UseGuards(DealerAuthGuard)             // Chain of Responsibility
@UseInterceptors(CacheInterceptor)      // Decorator
export class ListingController {
  constructor(private readonly listings: ListingService) {}

  @Get(':id')
  get(@Param('id') id: string) {
    return this.listings.findOne(Number(id))
  }
}
```

Note that TS decorators are *metadata*, not the GoF Decorator pattern — same word, different idea.
Nest's **interceptors** are the wrapping behaviour; the `@` syntax is just how you attach them.

### 2.8 Structural typing makes Adapter almost free

C# needs an explicit `: IDealerFeed`. TypeScript does not — if the shape matches, it fits.

```ts
interface DealerFeed {
  fetchInventory(dealerId: number): Promise<Listing[]>
}

// A third-party client with a different shape
class LegacyFeedSdk {
  async getStock(id: string): Promise<Array<{ vehicle_id: number; mk: string; mdl: string; price: string }>> {
    /* ... */ return []
  }
}

// The adapter is an object literal. No class, no `implements`.
function adaptLegacy(sdk: LegacyFeedSdk): DealerFeed {
  return {
    async fetchInventory(dealerId) {
      const raw = await sdk.getStock(String(dealerId))
      return raw.map(r => ({
        id: r.vehicle_id,
        make: r.mk,
        model: r.mdl,
        year: 0,
        priceInr: Number(r.price),
        kmDriven: 0,
        postedAt: new Date().toISOString(),
      }))
    },
  }
}
```

Because of this, in TypeScript [Adapter](../02-structural/01-adapter.md) is usually a mapping
function, not a class hierarchy. Keep it that way.

---

## 3. 🗄️ SQL / data layer — patterns with real trade-offs

### 3.1 Repository, honestly

The pitch: a Repository gives you a collection-like interface over persistence, so domain code does
not know about SQL.

The criticism, which is fair: **`DbSet<T>` in EF Core is already a repository, and `DbContext` is
already a Unit of Work.** Wrapping them in `IListingRepository` that forwards every call adds a layer
that only leaks — you end up with `IQueryable<T>` on the interface (so the abstraction is a lie) or
with forty bespoke methods (`GetByDealerAndStatusOrderedByPrice`).

Where a repository still earns its place:

1. Your data access is Dapper / raw ADO.NET — there is no built-in repository to duplicate.
2. You want a domain-shaped surface (`FindLiveListingsForDealer`) rather than a query-shaped one.
3. You need to swap or combine stores — SQL plus a search index, say.
4. You genuinely test against a fake and cannot use a real database in CI.

A repository worth writing (Dapper, intention-revealing, no `IQueryable` leak):

```csharp
public interface IListingRepository
{
    Task<Listing?> GetAsync(int id, CancellationToken ct);
    Task<IReadOnlyList<Listing>> FindAsync(ListingQuery query, CancellationToken ct);
    Task SaveAsync(Listing listing, CancellationToken ct);
    Task<int> ExpireOlderThanAsync(DateTime cutoffUtc, CancellationToken ct);
}

public sealed class SqlListingRepository : IListingRepository
{
    private readonly IDbConnectionFactory _connections;
    public SqlListingRepository(IDbConnectionFactory connections) => _connections = connections;

    public async Task<Listing?> GetAsync(int id, CancellationToken ct)
    {
        using var conn = _connections.Create();
        return await conn.QuerySingleOrDefaultAsync<Listing>(
            "SELECT Id, DealerId, Make, Model, Year, PriceInr, KmDriven, Status, PostedAt " +
            "FROM Listings WHERE Id = @id",
            new { id });
    }

    public async Task<IReadOnlyList<Listing>> FindAsync(ListingQuery query, CancellationToken ct)
    {
        var (sql, parameters) = ListingSqlBuilder.Build(query);   // see Query Object below
        using var conn = _connections.Create();
        var rows = await conn.QueryAsync<Listing>(sql, parameters);
        return rows.ToList();
    }

    public async Task SaveAsync(Listing listing, CancellationToken ct)
    {
        using var conn = _connections.Create();
        await conn.ExecuteAsync(
            @"UPDATE Listings
                 SET PriceInr = @PriceInr, Status = @Status, KmDriven = @KmDriven
               WHERE Id = @Id", listing);
    }

    public async Task<int> ExpireOlderThanAsync(DateTime cutoffUtc, CancellationToken ct)
    {
        using var conn = _connections.Create();
        return await conn.ExecuteAsync(
            "UPDATE Listings SET Status = 'expired' WHERE Status = 'live' AND PostedAt < @cutoffUtc",
            new { cutoffUtc });
    }
}
```

**Rule of thumb:** if your repository method bodies are one-liners that forward to EF, delete the
repository. If they encode a domain concept, keep it.

### 3.2 Unit of Work

One transaction boundary spanning several writes. With EF Core, `SaveChangesAsync` *is* it. Without
an ORM, make it explicit:

```csharp
public interface IUnitOfWork : IAsyncDisposable
{
    IDbConnection Connection { get; }
    IDbTransaction Transaction { get; }
    Task CommitAsync(CancellationToken ct);
    Task RollbackAsync(CancellationToken ct);
}

public sealed class SqlUnitOfWork : IUnitOfWork
{
    private readonly SqlConnection _conn;
    private readonly SqlTransaction _tx;
    private bool _committed;

    private SqlUnitOfWork(SqlConnection conn, SqlTransaction tx) { _conn = conn; _tx = tx; }

    public static async Task<SqlUnitOfWork> BeginAsync(string cs, CancellationToken ct)
    {
        var conn = new SqlConnection(cs);
        await conn.OpenAsync(ct);
        var tx = (SqlTransaction)await conn.BeginTransactionAsync(ct);
        return new SqlUnitOfWork(conn, tx);
    }

    public IDbConnection Connection => _conn;
    public IDbTransaction Transaction => _tx;

    public async Task CommitAsync(CancellationToken ct)
    {
        await _tx.CommitAsync(ct);
        _committed = true;
    }

    public Task RollbackAsync(CancellationToken ct) => _tx.RollbackAsync(ct);

    public async ValueTask DisposeAsync()
    {
        if (!_committed) { try { await _tx.RollbackAsync(); } catch { /* already gone */ } }
        await _tx.DisposeAsync();
        await _conn.DisposeAsync();
    }
}
```

Usage — listing update and outbox insert commit together or not at all:

```csharp
await using var uow = await SqlUnitOfWork.BeginAsync(_cs, ct);

await uow.Connection.ExecuteAsync(
    "UPDATE Listings SET PriceInr = @price WHERE Id = @id",
    new { price = newPrice, id = listingId }, uow.Transaction);

await uow.Connection.ExecuteAsync(
    "INSERT INTO Outbox (Id, Type, Payload, OccurredUtc) VALUES (@id, @type, @payload, @now)",
    new { id = Guid.NewGuid(), type = "ListingRepriced",
          payload = JsonSerializer.Serialize(evt), now = DateTime.UtcNow }, uow.Transaction);

await uow.CommitAsync(ct);
```

### 3.3 Specification — composable filters

[Specification](../03-behavioral/08-strategy.md)-style filters are Strategy applied to predicates.
They shine when the UI lets users combine filters freely (make + price band + city + fuel type).

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> ToExpression();
}

public static class SpecExtensions
{
    public static ISpecification<T> And<T>(this ISpecification<T> left, ISpecification<T> right)
        => new AndSpecification<T>(left, right);

    public static ISpecification<T> Or<T>(this ISpecification<T> left, ISpecification<T> right)
        => new OrSpecification<T>(left, right);
}

public sealed class PriceBetweenSpec : ISpecification<Listing>
{
    private readonly decimal _min, _max;
    public PriceBetweenSpec(decimal min, decimal max) { _min = min; _max = max; }
    public Expression<Func<Listing, bool>> ToExpression()
        => l => l.PriceInr >= _min && l.PriceInr <= _max;
}

public sealed class MakeIsSpec : ISpecification<Listing>
{
    private readonly string _make;
    public MakeIsSpec(string make) => _make = make;
    public Expression<Func<Listing, bool>> ToExpression() => l => l.Make == _make;
}

public sealed class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _l, _r;
    public AndSpecification(ISpecification<T> l, ISpecification<T> r) { _l = l; _r = r; }

    public Expression<Func<T, bool>> ToExpression()
    {
        var left = _l.ToExpression();
        var right = _r.ToExpression();
        var param = Expression.Parameter(typeof(T), "x");
        var body = Expression.AndAlso(
            Expression.Invoke(left, param),
            Expression.Invoke(right, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}
```

Usage:

```csharp
ISpecification<Listing> spec = new MakeIsSpec("Hyundai")
    .And(new PriceBetweenSpec(400_000m, 900_000m));

var results = await _db.Listings.Where(spec.ToExpression()).ToListAsync(ct);
```

Caveat: `Expression.Invoke` is not translatable by every LINQ provider. EF Core handles a lot, but
verify the generated SQL before you rely on it; if it fails, use a parameter-rebinding visitor or
drop to the Query Object below.

The same idea in TypeScript is embarrassingly simple:

```ts
type Spec<T> = (item: T) => boolean

const and = <T>(...specs: Spec<T>[]): Spec<T> => item => specs.every(s => s(item))
const or  = <T>(...specs: Spec<T>[]): Spec<T> => item => specs.some(s => s(item))

const makeIs = (make: string): Spec<Listing> => l => l.make === make
const priceBetween = (min: number, max: number): Spec<Listing> =>
  l => l.priceInr >= min && l.priceInr <= max

const cheapHyundai = and(makeIs('Hyundai'), priceBetween(400_000, 900_000))
```

### 3.4 Query Object — build SQL without string soup

```csharp
public sealed record ListingQuery(
    string? Make = null,
    string? City = null,
    decimal? MinPriceInr = null,
    decimal? MaxPriceInr = null,
    int? MaxKmDriven = null,
    string SortBy = "postedAt",
    int Page = 1,
    int PageSize = 20);

public static class ListingSqlBuilder
{
    private static readonly IReadOnlyDictionary<string, string> SortColumns =
        new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
        {
            ["postedAt"] = "PostedAt DESC",
            ["priceAsc"] = "PriceInr ASC",
            ["priceDesc"] = "PriceInr DESC",
            ["kmAsc"]    = "KmDriven ASC",
        };

    public static (string Sql, DynamicParameters Parameters) Build(ListingQuery q)
    {
        var where = new List<string> { "Status = 'live'" };
        var p = new DynamicParameters();

        if (q.Make is not null)        { where.Add("Make = @make");        p.Add("make", q.Make); }
        if (q.City is not null)        { where.Add("City = @city");        p.Add("city", q.City); }
        if (q.MinPriceInr is not null) { where.Add("PriceInr >= @minP");   p.Add("minP", q.MinPriceInr); }
        if (q.MaxPriceInr is not null) { where.Add("PriceInr <= @maxP");   p.Add("maxP", q.MaxPriceInr); }
        if (q.MaxKmDriven is not null) { where.Add("KmDriven <= @maxKm");  p.Add("maxKm", q.MaxKmDriven); }

        // Whitelist the ORDER BY — never interpolate user input into SQL.
        var order = SortColumns.TryGetValue(q.SortBy, out var col) ? col : "PostedAt DESC";

        p.Add("offset", (Math.Max(q.Page, 1) - 1) * q.PageSize);
        p.Add("limit", Math.Clamp(q.PageSize, 1, 100));

        var sql = $@"
            SELECT Id, DealerId, Make, Model, Year, PriceInr, KmDriven, City, Status, PostedAt
              FROM Listings
             WHERE {string.Join(" AND ", where)}
             ORDER BY {order}
            OFFSET @offset ROWS FETCH NEXT @limit ROWS ONLY";

        return (sql, p);
    }
}
```

Two things matter here: every *value* is a parameter, and every *identifier* (sort column) comes from
a whitelist. That is how a Query Object stays injection-safe.

### 3.5 Dialect handling = Abstract Factory

If you must support SQL Server and PostgreSQL:

```csharp
public interface ISqlDialect
{
    string Paginate(string sql, int offset, int limit);
    string QuoteIdentifier(string name);
    string UtcNow { get; }
}

public sealed class SqlServerDialect : ISqlDialect
{
    public string Paginate(string sql, int offset, int limit)
        => $"{sql} OFFSET {offset} ROWS FETCH NEXT {limit} ROWS ONLY";
    public string QuoteIdentifier(string name) => $"[{name}]";
    public string UtcNow => "SYSUTCDATETIME()";
}

public sealed class PostgresDialect : ISqlDialect
{
    public string Paginate(string sql, int offset, int limit)
        => $"{sql} LIMIT {limit} OFFSET {offset}";
    public string QuoteIdentifier(string name) => $"\"{name}\"";
    public string UtcNow => "NOW() AT TIME ZONE 'utc'";
}

public interface IDataAccessFactory                 // Abstract Factory
{
    IDbConnection CreateConnection();
    ISqlDialect Dialect { get; }
}
```

The factory hands out a *family* of related objects (connection + dialect) that must match. That
"must match" constraint is exactly what
[Abstract Factory](../01-creational/02-abstract-factory.md) exists for.

### 3.6 Identity Map, Lazy Loading, Object Pool — already in your ORM and driver

- **Identity Map**: EF Core's change tracker. Load listing 42 twice in one `DbContext` and you get
  the *same object*. That is why a long-lived `DbContext` in a web app is a bug — stale identity and
  unbounded memory. Scope it per request.
- **Lazy Loading = [Proxy](../02-structural/07-proxy.md)**: EF generates a proxy subclass whose
  navigation property fetches on first access. Convenient and dangerous: it causes N+1 queries. In a
  listings grid, loading `listing.Dealer` per row fires one query per row. Prefer explicit
  `.Include(l => l.Dealer)` or a projection.
- **Connection pooling = Object Pool**: ADO.NET pools connections by connection string. `new
  SqlConnection(cs)` is cheap; opening leases from the pool. This is why you should *not* cache a
  connection in a field — dispose it, give it back. Varying the connection string per request
  fragments the pool and is a classic production incident.

```
N+1 anatomy
───────────
SELECT * FROM Listings WHERE City='Pune'      -- 1 query, 50 rows
  for each row: SELECT * FROM Dealers WHERE Id=?   -- 50 more queries
                                                    = 51 round trips

Fixed with Include/JOIN:
SELECT l.*, d.* FROM Listings l JOIN Dealers d ON d.Id = l.DealerId WHERE l.City='Pune'
                                                    = 1 round trip
```

---

## 4. 🐇 RabbitMQ / messaging — patterns across a network

### 4.1 Observer vs pub/sub: what a broker changes

They look alike. They are not the same.

```
Observer (in-process)                 Pub/Sub (broker)
─────────────────────                 ────────────────
subject holds references              publisher knows nothing about subscribers
to its observers                      beyond an exchange name

synchronous, same thread              asynchronous, different process/machine

subscriber throws → publisher         subscriber throws → message nacked,
sees the exception                    publisher never knows

no delivery guarantee needed          at-least-once delivery, acks, retries,
(same memory)                         dead-letter queues

no ordering question                  ordering only per-queue, and not even
                                      that with concurrent consumers

no duplicates                         duplicates are NORMAL. plan for them.
```

The single most important consequence: **with a broker, "at least once" means your consumer will be
called twice with the same message.** Design for it (section 4.6) or you will double-charge a dealer.

### 4.2 Command messages vs Event messages

This distinction drives your whole topology.

| | Command | Event |
| --- | --- | --- |
| Name | imperative: `RepriceListing` | past tense: `ListingRepriced` |
| Intent | "do this" | "this happened" |
| Consumers | exactly one | zero to many |
| Coupling | sender knows the receiver | publisher knows nobody |
| Failure | sender may care | publisher does not care |
| Exchange | `direct` to a specific queue | `fanout` / `topic` |

```csharp
// Command — one owner, direct exchange, routing key names the handler's queue.
public sealed record RepriceListing(int ListingId, decimal NewPriceInr, string RequestedBy);

// Event — statement of fact, anyone may listen.
public sealed record ListingRepriced(
    int ListingId, decimal OldPriceInr, decimal NewPriceInr, DateTime OccurredUtc);
```

Topology setup with `RabbitMQ.Client`:

```csharp
using RabbitMQ.Client;

var factory = new ConnectionFactory
{
    HostName = "rabbit.internal",
    UserName = "marketplace",
    Password = secret,
    VirtualHost = "/listings",
    AutomaticRecoveryEnabled = true,
    NetworkRecoveryInterval = TimeSpan.FromSeconds(10)
};

using var connection = factory.CreateConnection("listing-service");
using var channel = connection.CreateModel();

// Events fan out by topic.
channel.ExchangeDeclare("listing.events", ExchangeType.Topic, durable: true, autoDelete: false);

// Commands go direct.
channel.ExchangeDeclare("listing.commands", ExchangeType.Direct, durable: true, autoDelete: false);

// A dead-letter exchange for poison messages.
channel.ExchangeDeclare("listing.dlx", ExchangeType.Fanout, durable: true, autoDelete: false);
channel.QueueDeclare("listing.dead", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("listing.dead", "listing.dlx", routingKey: "");

// The search-reindex consumer subscribes to price events.
var args = new Dictionary<string, object>
{
    ["x-dead-letter-exchange"] = "listing.dlx",
    ["x-message-ttl"] = 86_400_000          // 24h
};
channel.QueueDeclare("search.reindex", durable: true, exclusive: false, autoDelete: false, arguments: args);
channel.QueueBind("search.reindex", "listing.events", routingKey: "listing.repriced");
channel.QueueBind("search.reindex", "listing.events", routingKey: "listing.published");
```

Publishing an event:

```csharp
public void PublishRepriced(IModel channel, ListingRepriced evt)
{
    var body = JsonSerializer.SerializeToUtf8Bytes(evt);

    var props = channel.CreateBasicProperties();
    props.ContentType = "application/json";
    props.DeliveryMode = 2;                                  // persistent
    props.MessageId = Guid.NewGuid().ToString("n");          // for idempotency
    props.Timestamp = new AmqpTimestamp(DateTimeOffset.UtcNow.ToUnixTimeSeconds());
    props.Type = nameof(ListingRepriced);
    props.Headers = new Dictionary<string, object>
    {
        ["correlation-id"] = Activity.Current?.Id ?? ""
    };

    channel.BasicPublish(
        exchange: "listing.events",
        routingKey: "listing.repriced",
        mandatory: false,
        basicProperties: props,
        body: body);
}
```

Turn on publisher confirms if losing the event matters:

```csharp
channel.ConfirmSelect();
PublishRepriced(channel, evt);
if (!channel.WaitForConfirms(TimeSpan.FromSeconds(5)))
    throw new InvalidOperationException("Broker did not confirm publish");
```

### 4.3 The broker as Mediator

A message broker is [Mediator](../03-behavioral/04-mediator.md) at infrastructure scale. Without it,
every service dials every other service:

```
Without a broker                       With a broker
────────────────                       ─────────────
listing ───▶ search                    listing ──┐
   │  ╲    ╱   │                                 │
   │   ╲  ╱    │                       search ───┤
   ▼    ╳      ▼                                 ├──▶ [ exchange ]
pricing ─── notify                     pricing ──┤        (Mediator)
   ▲    ╱  ╲   ▲                                 │
   └───╱────╲──┘                       notify ───┘
  n² connections                        n connections
```

The trade-off is the same one Mediator always has: the hub becomes critical infrastructure, and
tracing a flow requires reading topology, not call stacks. Invest in correlation ids early.

### 4.4 Consumer pipeline = Chain of Responsibility

```csharp
public interface IMessageHandler
{
    Task<HandleResult> HandleAsync(MessageContext context, CancellationToken ct);
}

public sealed record MessageContext(
    string MessageId,
    string Type,
    ReadOnlyMemory<byte> Body,
    IDictionary<string, object> Headers,
    int DeliveryAttempt);

public enum HandleResult { Ack, Retry, DeadLetter }

// Links
public sealed class DeserializeLink : IMessageHandler
{
    private readonly IMessageHandler _next;
    private readonly ILogger _logger;
    public DeserializeLink(IMessageHandler next, ILogger logger) { _next = next; _logger = logger; }

    public async Task<HandleResult> HandleAsync(MessageContext ctx, CancellationToken ct)
    {
        try { JsonDocument.Parse(ctx.Body); }
        catch (JsonException ex)
        {
            _logger.LogError(ex, "Malformed message {Id}", ctx.MessageId);
            return HandleResult.DeadLetter;          // never retry bad JSON
        }
        return await _next.HandleAsync(ctx, ct);
    }
}

public sealed class IdempotencyLink : IMessageHandler
{
    private readonly IMessageHandler _next;
    private readonly IProcessedMessageStore _store;
    public IdempotencyLink(IMessageHandler next, IProcessedMessageStore store)
    { _next = next; _store = store; }

    public async Task<HandleResult> HandleAsync(MessageContext ctx, CancellationToken ct)
    {
        if (await _store.HasProcessedAsync(ctx.MessageId, ct))
            return HandleResult.Ack;                 // duplicate: succeed silently

        var result = await _next.HandleAsync(ctx, ct);
        if (result == HandleResult.Ack)
            await _store.MarkProcessedAsync(ctx.MessageId, ct);
        return result;
    }
}
```

Composition reads as a chain:

```csharp
IMessageHandler pipeline =
    new DeserializeLink(
        new IdempotencyLink(
            new RepriceHandler(repo, logger),
            processedStore),
        logger);
```

### 4.5 Retry / backoff = Strategy

```csharp
public interface IRetryPolicy
{
    bool ShouldRetry(int attempt, Exception ex);
    TimeSpan DelayFor(int attempt);
}

public sealed class ExponentialBackoffPolicy : IRetryPolicy
{
    private readonly int _maxAttempts;
    private readonly TimeSpan _baseDelay;
    private readonly Random _jitter = new();

    public ExponentialBackoffPolicy(int maxAttempts = 5, TimeSpan? baseDelay = null)
    {
        _maxAttempts = maxAttempts;
        _baseDelay = baseDelay ?? TimeSpan.FromSeconds(1);
    }

    public bool ShouldRetry(int attempt, Exception ex)
        => attempt < _maxAttempts && ex is not (ValidationException or JsonException);

    public TimeSpan DelayFor(int attempt)
    {
        var exponential = _baseDelay * Math.Pow(2, attempt - 1);
        var jitterMs = _jitter.Next(0, 500);              // avoid thundering herd
        return exponential + TimeSpan.FromMilliseconds(jitterMs);
    }
}

public sealed class NoRetryPolicy : IRetryPolicy
{
    public bool ShouldRetry(int attempt, Exception ex) => false;
    public TimeSpan DelayFor(int attempt) => TimeSpan.Zero;
}
```

Important: **do not `Thread.Sleep` inside a consumer to implement backoff.** You block a channel
thread and stall the queue. Use a *delay queue*: publish the failed message to a queue with a TTL and
a dead-letter exchange pointing back at the work queue.

```csharp
// A 30-second retry queue that returns messages to the main exchange when the TTL expires.
channel.QueueDeclare(
    queue: "search.reindex.retry.30s",
    durable: true, exclusive: false, autoDelete: false,
    arguments: new Dictionary<string, object>
    {
        ["x-message-ttl"] = 30_000,
        ["x-dead-letter-exchange"] = "listing.events",
        ["x-dead-letter-routing-key"] = "listing.repriced"
    });
```

```
main queue ──fail──▶ retry.30s ──TTL expires──▶ back to exchange ──▶ main queue
                          │
                   attempts exhausted
                          ▼
                    listing.dead (DLQ)  ──▶ human looks at it
```

### 4.6 The consumer, end to end

```csharp
public sealed class ReindexConsumer : BackgroundService
{
    private readonly IConnection _connection;
    private readonly IServiceScopeFactory _scopes;
    private readonly IRetryPolicy _retry;
    private readonly ILogger<ReindexConsumer> _logger;
    private IModel? _channel;

    public ReindexConsumer(
        IConnection connection,
        IServiceScopeFactory scopes,
        IRetryPolicy retry,
        ILogger<ReindexConsumer> logger)
    {
        _connection = connection;
        _scopes = scopes;
        _retry = retry;
        _logger = logger;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _channel = _connection.CreateModel();
        _channel.BasicQos(prefetchSize: 0, prefetchCount: 20, global: false);  // fairness + memory cap

        var consumer = new AsyncEventingBasicConsumer(_channel);
        consumer.Received += OnReceivedAsync;

        _channel.BasicConsume(queue: "search.reindex", autoAck: false, consumer: consumer);
        return Task.CompletedTask;
    }

    private async Task OnReceivedAsync(object sender, BasicDeliverEventArgs ea)
    {
        var messageId = ea.BasicProperties.MessageId ?? Guid.NewGuid().ToString("n");
        var attempt = GetDeathCount(ea.BasicProperties) + 1;

        try
        {
            var evt = JsonSerializer.Deserialize<ListingRepriced>(ea.Body.Span)
                      ?? throw new JsonException("null payload");

            using var scope = _scopes.CreateScope();
            var processed = scope.ServiceProvider.GetRequiredService<IProcessedMessageStore>();

            if (await processed.HasProcessedAsync(messageId, CancellationToken.None))
            {
                _channel!.BasicAck(ea.DeliveryTag, multiple: false);   // duplicate — done already
                return;
            }

            var indexer = scope.ServiceProvider.GetRequiredService<ISearchIndexer>();
            await indexer.ReindexListingAsync(evt.ListingId, CancellationToken.None);
            await processed.MarkProcessedAsync(messageId, CancellationToken.None);

            _channel!.BasicAck(ea.DeliveryTag, multiple: false);
        }
        catch (JsonException ex)
        {
            _logger.LogError(ex, "Poison message {Id}; dead-lettering", messageId);
            _channel!.BasicNack(ea.DeliveryTag, multiple: false, requeue: false);  // → DLX
        }
        catch (Exception ex) when (_retry.ShouldRetry(attempt, ex))
        {
            _logger.LogWarning(ex, "Attempt {Attempt} failed for {Id}; retrying", attempt, messageId);
            RepublishToRetryQueue(ea);
            _channel!.BasicAck(ea.DeliveryTag, multiple: false);   // ack the original; the copy lives on
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Giving up on {Id} after {Attempt} attempts", messageId, attempt);
            _channel!.BasicNack(ea.DeliveryTag, multiple: false, requeue: false);
        }
    }

    private static int GetDeathCount(IBasicProperties props)
    {
        if (props.Headers is null || !props.Headers.TryGetValue("x-death", out var raw)) return 0;
        if (raw is not List<object> deaths || deaths.Count == 0) return 0;
        if (deaths[0] is not Dictionary<string, object> first) return 0;
        return first.TryGetValue("count", out var c) && c is long n ? (int)n : 0;
    }

    private void RepublishToRetryQueue(BasicDeliverEventArgs ea)
        => _channel!.BasicPublish(
            exchange: "",
            routingKey: "search.reindex.retry.30s",
            basicProperties: ea.BasicProperties,
            body: ea.Body);

    public override void Dispose()
    {
        _channel?.Dispose();
        base.Dispose();
    }
}
```

Two details worth internalising:

- `BasicQos(prefetchCount: 20)` is your backpressure knob. Leave it unset and one consumer will grab
  the whole queue while its siblings idle.
- `requeue: true` on a nack puts the message straight back at the head of the queue — with a
  deterministic bug, that is an infinite hot loop. Almost always use `requeue: false` plus a DLX.

### 4.7 The Outbox pattern — the fix for "saved but never published"

Dual writes are the classic distributed bug: you commit to SQL, then the publish fails, and the world
disagrees about the price.

```
Naive                                  Outbox
─────                                  ──────
BEGIN TX                               BEGIN TX
  UPDATE Listings                        UPDATE Listings
COMMIT            ← crash here           INSERT INTO Outbox
publish to Rabbit  ← never happens     COMMIT              ← atomic: both or neither
                                       (relay publishes later, at least once)
```

Schema and relay:

```sql
CREATE TABLE Outbox (
    Id           UNIQUEIDENTIFIER PRIMARY KEY,
    Type         VARCHAR(200)     NOT NULL,
    RoutingKey   VARCHAR(200)     NOT NULL,
    Payload      NVARCHAR(MAX)    NOT NULL,
    OccurredUtc  DATETIME2        NOT NULL,
    PublishedUtc DATETIME2        NULL
);
CREATE INDEX IX_Outbox_Unpublished ON Outbox (OccurredUtc) WHERE PublishedUtc IS NULL;
```

```csharp
public sealed class OutboxRelay : BackgroundService
{
    private readonly IDbConnectionFactory _connections;
    private readonly IModel _channel;
    private readonly ILogger<OutboxRelay> _logger;

    public OutboxRelay(IDbConnectionFactory connections, IModel channel, ILogger<OutboxRelay> logger)
    {
        _connections = connections;
        _channel = channel;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(2));
        while (await timer.WaitForNextTickAsync(ct))
        {
            try { await DrainAsync(ct); }
            catch (Exception ex) { _logger.LogError(ex, "Outbox drain failed"); }
        }
    }

    private async Task DrainAsync(CancellationToken ct)
    {
        using var conn = _connections.Create();

        // READPAST lets multiple relay instances run without fighting over the same rows.
        var pending = await conn.QueryAsync<OutboxRow>(
            @"SELECT TOP (100) Id, Type, RoutingKey, Payload
                FROM Outbox WITH (UPDLOCK, READPAST)
               WHERE PublishedUtc IS NULL
               ORDER BY OccurredUtc");

        foreach (var row in pending)
        {
            var props = _channel.CreateBasicProperties();
            props.DeliveryMode = 2;
            props.ContentType = "application/json";
            props.MessageId = row.Id.ToString("n");     // stable id → consumers can dedupe
            props.Type = row.Type;

            _channel.BasicPublish("listing.events", row.RoutingKey, props,
                                  Encoding.UTF8.GetBytes(row.Payload));

            await conn.ExecuteAsync(
                "UPDATE Outbox SET PublishedUtc = SYSUTCDATETIME() WHERE Id = @id", new { id = row.Id });
        }
    }
}
```

The relay is at-least-once by construction: a crash between publish and the `UPDATE` republishes the
message. That is fine — the `MessageId` is stable, so the idempotent consumer drops the duplicate.
This is why 4.6 and 4.7 must be built together.

### 4.8 Memento and Event Sourcing

[Memento](../03-behavioral/05-memento.md) captures state so you can restore it. Event sourcing
inverts it: store the *changes*, derive the state.

```csharp
public abstract record ListingEvent(int ListingId, DateTime OccurredUtc);
public sealed record ListingCreated(int ListingId, DateTime OccurredUtc, string Make, string Model,
                                    decimal PriceInr) : ListingEvent(ListingId, OccurredUtc);
public sealed record PriceChanged(int ListingId, DateTime OccurredUtc, decimal NewPriceInr)
    : ListingEvent(ListingId, OccurredUtc);
public sealed record ListingSold(int ListingId, DateTime OccurredUtc, decimal SalePriceInr)
    : ListingEvent(ListingId, OccurredUtc);

public sealed record ListingSnapshot(int Id, string Make, string Model, decimal PriceInr, string Status)
{
    public static ListingSnapshot Replay(IEnumerable<ListingEvent> events)
    {
        ListingSnapshot? state = null;

        foreach (var e in events.OrderBy(x => x.OccurredUtc))
        {
            state = e switch
            {
                ListingCreated c => new ListingSnapshot(c.ListingId, c.Make, c.Model, c.PriceInr, "live"),
                PriceChanged p   => (state ?? throw new InvalidOperationException("no create event"))
                                        with { PriceInr = p.NewPriceInr },
                ListingSold s    => (state ?? throw new InvalidOperationException("no create event"))
                                        with { PriceInr = s.SalePriceInr, Status = "sold" },
                _ => state
            };
        }

        return state ?? throw new InvalidOperationException("Empty event stream");
    }
}
```

You get audit history for free — "why did this listing's price change three times last Tuesday" is a
query, not an archaeology project. The costs are real: replay gets slow (hence periodic snapshots,
which *are* Mementos), schema evolution of old events is painful, and every ad-hoc query needs a
projection. Event-source the handful of aggregates where history is the product; keep CRUD for the
rest.

---

## 5. 📅 A 30 / 60 / 90 day plan

The goal is not to memorise 23 patterns. It is to change what you *notice* while reading code.

### Days 1–30: recognition

| Week | Read | Do in real code |
| --- | --- | --- |
| 1 | [Strategy](../03-behavioral/08-strategy.md), [Factory Method](../01-creational/01-factory-method.md), [Singleton](../01-creational/05-singleton.md) | Find one `switch` on a type/enum that picks behaviour. Don't refactor. Write a three-line note: what varies, what stays. |
| 2 | [Adapter](../02-structural/01-adapter.md), [Facade](../02-structural/05-facade.md), [Decorator](../02-structural/04-decorator.md) | List every third-party client in your service. Mark which are wrapped and which leak their types into your domain. |
| 3 | [Observer](../03-behavioral/06-observer.md), [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) | Map your ASP.NET middleware order and your RabbitMQ topology on one page. Name the pattern at each hop. |
| 4 | [Template Method](../03-behavioral/09-template-method.md), [Iterator](../03-behavioral/03-iterator.md) | Find one method that loads a whole list into memory and convert it to `IEnumerable`/`IAsyncEnumerable` with `yield`. |

Week 4 deliverable: a one-page "patterns already in our codebase" note. You will be surprised how
many there are, unnamed.

**How to spot candidates.** Grep your own repo for these smells:

```
switch (type)          / if (x is TypeA)     → Strategy or polymorphism
new SomethingClient(   inside business logic → Factory + DI
bool flag1, bool flag2 in a signature        → Strategy or separate methods
"Manager", "Helper", "Util" class names      → no pattern, no cohesion — split it
a method > 80 lines with three try/catch     → Chain of Responsibility or Template Method
copy-pasted retry loops                      → Strategy (policy object)
if (cache != null) ... else ...              → Decorator
```

### Days 31–60: small, safe refactors

Each of these is a single PR, reviewable in fifteen minutes.

1. **Replace one switch with a strategy map.** Pick a pricing or notification-channel switch. In C#,
   register implementations and inject `IEnumerable<T>`. In TS, a `satisfies Record<K, F>` object.
   Keep the old tests green without touching them — that is your proof the refactor was behaviour
   preserving.

2. **Wrap one repository in a caching decorator.** No change to callers. Delete any ad-hoc `if
   (_cache.TryGetValue...)` sprinkled through the service.

3. **Extract one retry loop into a policy object.** You almost certainly have three slightly
   different retry loops. Find them, unify them, prove the differences were accidental.

4. **Make one RabbitMQ consumer idempotent.** Add a `ProcessedMessages` table keyed on `MessageId`
   with a TTL cleanup job. Then deliberately republish a message in a staging environment and confirm
   nothing double-fires.

5. **Convert one state field to a discriminated union** in the TypeScript front-end. Add
   `assertNever`. Enjoy the compiler finding the two places that forgot `expired`.

6. **Add an outbox to the one flow where a lost event would hurt most.** Usually pricing or booking.

### Days 61–90: judgment

Now the harder skill: knowing when *not* to.

- **Write the "no" memo.** Pick a place where a pattern would technically apply and write three
  paragraphs on why you are not applying it. This is the single most valuable exercise here.
- **Do a deliberate over-engineering post-mortem** on existing code — yours or the repo's. Find an
  abstraction with exactly one implementation that has never changed. Ask what it cost.
- **Learn the C++ and Java flavours** for contrast, since you are curious:
  - C++: RAII is a lifetime pattern with no GoF equivalent; `std::unique_ptr` makes ownership a type.
    Iterators are the foundation of the entire STL. `std::function` is Strategy. The pImpl idiom is
    Bridge, used to keep binary compatibility.
  - Java: the language forced heavier ceremony for years, which is why GoF patterns look so verbose
    in Java textbooks. Modern Java (records, sealed interfaces, pattern matching in `switch`) has
    quietly adopted the discriminated-union approach you will have learned in TypeScript.
- **Teach one pattern to a colleague** in ten minutes with code from your own repo. If you cannot
  find an example in your repo, you have probably picked a pattern you do not need.
- **Read one framework's source.** MediatR is small and readable. So is a middleware pipeline. You
  will see these patterns implemented by people who had to make them work for everyone.

Month 3 deliverable: a short document for your team — "patterns we use, patterns we deliberately
avoid, and why." That document is more valuable than knowing all 23.

---

## 6. ✅ Code-review checklist

### Signals a pattern is *missing*

| What you see | Likely missing | Ask in review |
| --- | --- | --- |
| A `switch`/`if-else` on a type that appears in 3+ places | Strategy / polymorphism | "If we add a fourth channel, how many files change?" |
| `new HttpClient()` or `new SqlConnection()` in a service method | Factory / DI | "How do we test this without a network?" |
| Copy-pasted try/catch-retry blocks | Strategy (policy) | "Are these three retry loops intentionally different?" |
| A method with 4+ boolean parameters | Strategy / Builder | "What does `Send(true, false, true)` mean at the call site?" |
| Business logic checking `status == "live"` in twelve places | State | "Where is the list of legal transitions?" |
| Every consumer reimplementing dedupe | a pipeline link | "Can this be one decorator around all consumers?" |
| A class that imports a vendor SDK type into its public API | Adapter | "What happens when we change vendors?" |
| Objects loaded and mutated with no transaction boundary | Unit of Work | "If step 3 throws, what is in the database?" |
| A publish right after a `SaveChanges` | Outbox | "What if the process dies between these two lines?" |
| `foreach` that fires a query per iteration | eager loading / projection | "How many round trips is this at 500 rows?" |

### Signals a pattern is *over-applied*

| What you see | Diagnosis |
| --- | --- |
| An interface with exactly one implementation, never mocked, never swapped | Speculative abstraction. Delete it; inline the class. |
| `IFooFactoryProvider` / `AbstractFooStrategyFactoryBean` | Ceremony. Name the concept, not the pattern. |
| A Repository whose every method is `=> _db.Set<T>()...` | Redundant layer over the ORM. |
| A mediator used for a call between two classes in the same file | Indirection with no payoff. |
| A decorator chain five deep for one HTTP call | Nobody can predict the behaviour. Flatten. |
| A Builder for a 3-field record | Use an object initialiser or `with`. |
| A Visitor over a type hierarchy that never grows | Pattern-matching `switch` is clearer. |
| Class names ending in the pattern name everywhere (`PriceStrategyFactoryImpl`) | The domain vocabulary has been replaced by pattern vocabulary. |
| A "for future flexibility" abstraction added in the same PR as the feature | YAGNI. Add it on the second requirement, not the first. |

### The three questions that settle most arguments

1. **What varies?** If nothing varies yet, you do not need the abstraction yet.
2. **Who else has to understand this?** A pattern that makes the author faster and every reader
   slower is a net loss.
3. **What is the cost of being wrong?** Inlining a premature abstraction is a cheap, mechanical
   refactor. Untangling a premature framework is not. When unsure, prefer the version that is easier
   to undo.

---

## 🗣️ In plain English

Your frameworks are full of solved problems. ASP.NET's middleware, `IEnumerable`, `HttpClient`
handlers, RxJS, Express's `next()`, RabbitMQ's exchanges — every one of them is a Gang of Four
pattern that somebody already implemented properly and tested against millions of users. Learning
patterns, for you, is mostly about learning to *see* them, so you stop writing a worse copy of what
you already have.

The handful you will genuinely write by hand are the domain-shaped ones: a strategy per pricing
channel, a decorator that adds caching, a state machine for the listing lifecycle, an idempotent
consumer, an outbox. Those solve problems your framework cannot know about.

And the most senior-sounding thing you can say in a code review is not "this should be a Visitor." It
is: "this could be a Visitor, but we only have two node types and they have not changed in two years,
so a switch is clearer. Let's revisit if a third one shows up."

---

### Where to go next

- [SOLID and principles](01-solid-and-principles.md) — the reasoning underneath every pattern here.
- [Strategy](../03-behavioral/08-strategy.md) and [Decorator](../02-structural/04-decorator.md) —
  the two you will use most.
- [Chain of Responsibility](../03-behavioral/01-chain-of-responsibility.md) — because your whole
  stack is made of pipelines.
- [Proxy](../02-structural/07-proxy.md) — to understand lazy loading before it bites you in
  production.
