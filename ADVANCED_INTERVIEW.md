# Citibank Front-End Architecture & Advanced Angular Engineer — Interview Preparation Guide

> **Companion to INTERVIEW.md** — This guide targets **principal-level and solutions-architect roles**, emphasising enterprise architecture decisions, clever algorithms, advanced RxJS/Signals patterns, and design trade-offs you must justify under pressure. Every answer is grounded in real patterns from `ARCHITECTURE.md` and the angularspace.com senior question bank.

---

## Sources

| Source | Coverage |
|--------|----------|
| `ARCHITECTURE.md` (1,996 lines) | Citibank equities-trading platform — production patterns |
| [angularspace.com/senior-angular-interview-questions](https://www.angularspace.com/senior-angular-interview-questions/) | 17 senior/principal questions with code samples |
| Angular 18+ Docs | Signals, Resource API, zoneless preview, `afterRenderEffect` |
| OWASP Top 10 | Security questions in Part 8 |

---

## How This Differs from INTERVIEW.md

| INTERVIEW.md | ADVANCED_INTERVIEW.md |
|--------------|-----------------------|
| Foundational (40 Q&A) | Principal / solutions-architect level |
| Conceptual explanations | Algorithm complexity + code trade-offs |
| Single-app patterns | Enterprise Nx monorepo + Module Federation |
| Basic lifecycle hooks | Signal-based replacements + `afterRenderEffect` |
| Basic RxJS | `scan`+`expand`, diamond problem, `shareReplay` semantics |
| Basic testing | 99% thresholds, k6 load testing, Playwright contracts |

---

## Part 1 — Senior Developer Mindset and Philosophy

### Q1: What separates a senior developer from a mid-level engineer?

**Answer (Citibank context):**

Technical depth is table-stakes. What differentiates a senior is **breadth of impact**:

1. **Initiative** — Identifies tech improvements without waiting to be asked. In `ARCHITECTURE.md`, the team proactively adopted Module Federation (ADR-001) before product mandated micro-frontends.
2. **Push-back with data** — A senior says "that feature will break our 99% coverage threshold on trading paths" and shows the k6 test regression, not just "that's a bad idea."
3. **Declarative thinking** — Writes `toSignal(merge(existing$, new$).pipe(scan(...)))` instead of three separate `subscribe` calls mutating a signal imperatively.
4. **Code review caliber** — Reviews for invariants (e.g., is `trackBy` always present on virtual scroll lists?), not just style.
5. **Product ownership** — Understands why `trade execution <50ms` is a business SLA, not an arbitrary number.

**Clever algorithm insight:** A senior considers the algorithmic complexity of template expressions. A naïve function call in a template runs every CD cycle — O(n × CD frequency). A pure pipe is O(1) after the first call with the same args because Angular caches the result.

---

### Q2: Declarative vs. imperative — why does it matter at scale?

**Imperative (fragile at scale):**
```typescript
// Three separate subscriptions → three separate CD triggers → three possible races
this.existingData$.subscribe(d => this.displayedData.set(d));
this.newData$.subscribe(d => this.displayedData.update(prev => [...prev, d]));
this.filter$.subscribe(f => this.displayedData.update(prev => prev.filter(f)));
```

**Declarative (single source of truth):**
```typescript
// One unified signal — impossible to get into a partial-update race condition
readonly displayedData = toSignal(
  merge(this.existingData$, this.newData$).pipe(
    scan((acc, curr) => Array.isArray(curr) ? curr : [...acc, curr], [] as Order[]),
    combineLatestWith(this.filter$),
    map(([data, filter]) => data.filter(filter))
  ),
  { initialValue: [] }
);
```

**Why it matters at Citibank:** Market data arrives at ~1,000 events/second. Imperative subscriptions create race conditions where a filter clears half-updated trade data. The declarative form is referentially transparent — the output is a pure function of the inputs.

---

### Q3: When do you reach for a state management library vs. services + signals?

| Criterion | Services + Signals | NgRx |
|-----------|--------------------|------|
| Team size | 1–3 devs | 4+ devs |
| Time-travel debugging needed | No | Yes |
| Complex async flows with cancellation | No | Yes (Effects) |
| Audit trail / compliance logging | No | Yes (action log) |
| Multiple micro-frontends sharing state | Fragile | Designed for it |

**Citibank answer:** The trading platform uses NgRx because `RiskEffects` must react to async portfolio risk events, cancel in-flight requests on portfolio change (`switchMap`), and the action log satisfies regulatory audit requirements. A service + signal pattern would require rebuilding all of that infrastructure manually.

---

## Part 2 — Advanced Component Communication Patterns

### Q4: Walk me through all five ways to communicate between components.

```
1. @Input() / @Output()             → parent-to-child / child-to-parent (simple tree)
2. Shared service with signals      → siblings / distant relatives (no prop-drilling)
3. Custom two-way binding           → shorthand for @Input() + @Output()
4. model() inputs (Angular 17.1+)   → syntactic sugar for two-way
5. ControlValueAccessor             → integrates custom UI controls with Angular Forms
```

**Custom two-way binding (the interview trap):**
```typescript
// Child component
@Component({ selector: 'app-price-input' })
export class PriceInputComponent {
  @Input()  cData!: number;
  @Output() cDataChange = new EventEmitter<number>();   // must be cData + Change
}

// Parent template — Angular desugars [(cData)]="parentPrice" into:
// [cData]="parentPrice" (cDataChange)="parentPrice = $event"
<app-price-input [(cData)]="parentPrice" />
```

**model() — Angular 17.1+ equivalent:**
```typescript
@Component({ selector: 'app-price-input' })
export class PriceInputComponent {
  readonly price = model<number>(0);   // two-way binding built-in
}
// Parent:
<app-price-input [(price)]="parentPrice" />
```

**ControlValueAccessor — for Forms integration:**
```typescript
@Component({
  selector: 'app-custom-input',
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: CustomInputComponent, multi: true }]
})
export class CustomInputComponent implements ControlValueAccessor {
  private onChange = (_: any) => {};
  private onTouched = () => {};

  writeValue(val: any): void { /* update internal display */ }
  registerOnChange(fn: any): void { this.onChange = fn; }
  registerOnTouched(fn: any): void { this.onTouched = fn; }
  setDisabledState?(isDisabled: boolean): void { /* toggle */ }
}
```

---

### Q5: NgZone — when and why do you opt out?

**Zone.js** monkey-patches every async API (setTimeout, Promise, XHR, fetch) and calls `ApplicationRef.tick()` after each. For a market data app processing 1,000 events/second this means 1,000 CD traversals per second.

**Opt out pattern (from ARCHITECTURE.md `MemoryOptimizedMarketDataService`):**
```typescript
constructor(private ngZone: NgZone) {
  // WebSocket events run outside Angular's zone — no CD triggered
  this.ngZone.runOutsideAngular(() => {
    this.marketSocket.on('price', (data: MarketQuote) => {
      this.priceCache.set(data.symbol, data);   // update LRU cache silently
    });
  });
}

// Only re-enter the zone when the UI actually needs to update
updateUI(data: Order[]): void {
  this.ngZone.run(() => this.orders.set(data));   // triggers CD once
}
```

**Rule of thumb:** `runOutsideAngular` for analytics, WebSocket listeners, `PerformanceObserver`, `requestAnimationFrame` animation loops. `ngZone.run()` only when you have a value the user must see immediately.

---

### Q6: InjectionToken — what is it and when do you reach for it?

**Problem:** `useValue: '/api'` cannot be injected as a typed dependency — `string` is not a DI token.

```typescript
// Library configuration pattern — replaces deprecated APP_INITIALIZER
export const API_ENDPOINT = new InjectionToken<string>('API_ENDPOINT', {
  providedIn: 'root',
  factory: () => '/default/api'   // fallback
});

// Feature module override
providers: [{ provide: API_ENDPOINT, useValue: environment.tradingApiUrl }]

// Consumer
@Injectable({ providedIn: 'root' })
export class TradingService {
  private apiUrl = inject(API_ENDPOINT);
}
```

**Modern replacement for `APP_INITIALIZER`:**
```typescript
// Old (deprecated)
providers: [{ provide: APP_INITIALIZER, useFactory: () => () => initApp(), multi: true }]

// New (Angular 17+)
providers: [provideAppInitializer(() => initApp())]
```

---

### Q7: Resolution modifiers — explain all four with real use cases.

```typescript
// optional — service may not be provided; returns null instead of throwing
const analytics = inject(AnalyticsService, { optional: true });

// self — only look in the current element's injector (not ancestors)
// Use case: directive that requires another directive on the SAME element
@Directive({ selector: '[requiredMarker]' })
export class RequiredMarkerDirective {
  private marker = inject(RequiredMarker, { self: true });  // throws if absent on same element
}

// skipSelf — skip current injector, start search at parent
// Use case: FormControlName finding its parent FormGroupDirective without self-circular reference
@Directive({ selector: '[formControlName]' })
export class FormControlNameDirective {
  private parent = inject(ControlContainer, { skipSelf: true });
}

// host — stop searching at the host component boundary
// Use case: projected content accessing only the host's services, not the app root
@Component({ selector: 'app-card' })
export class CardComponent {
  private theme = inject(ThemeService, { host: true });  // host's theme, not global
}
```

---

### Q8: `providers` vs. `viewProviders` — what is the difference?

```typescript
@Component({
  selector: 'app-payment',
  providers:     [PaymentService],  // visible to: template + children + projected content
  viewProviders: [PaymentService],  // visible to: template + children ONLY (NOT projected)
})
```

**Dynamic injection pattern (angularspace.com example):**
```typescript
const paymentInjector = Injector.create({
  providers: [{
    provide: PaymentService,
    useClass: usePaypal ? PaypalService : StripeService
  }],
  parent: this.injector
});

const ref = createComponent(PaymentButtonComponent, {
  environmentInjector: this.envInjector,
  elementInjector: paymentInjector
});
```

---

## Part 3 — Clever Algorithms in Angular

### Q9: Explain the `track` function algorithm in `@for()` and why `track $index` is dangerous.

**DOM diffing algorithm:**
`@for()` must reconcile old DOM nodes with a new array. Without `track`, Angular treats every element as potentially different → destroys and recreates all DOM nodes → O(n) destructions + O(n) creations = O(2n).

With `track item.id`: Angular runs a **hash map lookup** to match incoming items to existing DOM nodes → O(1) per item → O(n) total with no destructions for unchanged items.

```typescript
// WRONG — track $index
// If array = [A, B, C] and B is deleted → [A, C]
// Index 0→A (no-op), Index 1→C (C gets B's DOM node, state corrupted),
// Index 2 deleted. Focus and form state destroyed.

// WRONG — track item (object reference)
// HTTP response returns new object instances each time.
// { id: 1, symbol: 'AAPL' } !== { id: 1, symbol: 'AAPL' } → all nodes recreated

// CORRECT — stable identity key
@for (order of orders(); track order.id) {
  <app-order-row [order]="order" />
}
```

**Citibank context:** The `EquitiesTradingComponent` uses `trackByOrderId` in the CDK virtual scroll:
```typescript
trackByOrderId = (index: number, order: Order) => order.orderId;
```

---

### Q10: LRU Cache with TTL — how is it implemented in the trading platform?

From `ARCHITECTURE.md` — `MemoryOptimizedMarketDataService`:

```typescript
private readonly MAX_CACHE_SIZE = 1000;
private readonly TTL_MS = 5 * 60 * 1000;  // 5 minutes

// WeakMap for component-scoped subscriptions — auto-GC'd when component destroys
private readonly componentSubscriptions = new WeakMap<object, Subscription[]>();

// LRU eviction algorithm
private evictIfNeeded(): void {
  if (this.cache.size <= this.MAX_CACHE_SIZE) return;

  // Phase 1: remove all expired entries (TTL eviction) O(n)
  const now = Date.now();
  for (const [key, entry] of this.cache) {
    if (now - entry.timestamp > this.TTL_MS) this.cache.delete(key);
  }

  // Phase 2: if still over capacity, remove oldest entries (LRU eviction)
  if (this.cache.size > this.MAX_CACHE_SIZE) {
    const sortedKeys = [...this.cache.entries()]
      .sort((a, b) => a[1].timestamp - b[1].timestamp)        // O(n log n)
      .slice(0, this.cache.size - this.MAX_CACHE_SIZE)
      .map(([key]) => key);
    sortedKeys.forEach(k => this.cache.delete(k));
  }
}
```

**Complexity:** O(n log n) for LRU sort. In production, replace with a doubly-linked list + Map for O(1) LRU operations. The `WeakMap` ensures component subscriptions are garbage-collected automatically when a component is destroyed — no memory leak even without `ngOnDestroy`.

**Interview follow-up:** Why `WeakMap` and not `Map`?
> `WeakMap` keys are held weakly — when the component object is garbage-collected (unmounted), the entire entry is removed automatically. A `Map` would hold a strong reference, preventing GC = memory leak.

---

### Q11: CDK Virtual Scroll — what pre-loading algorithm is used?

From `ARCHITECTURE.md` — `TradeHistoryComponent`:

```typescript
onScrollIndexChange(index: number): void {
  this.currentScrollIndex = index;
  this.preloadDataAroundIndex(index);
}

private preloadDataAroundIndex(centerIndex: number): void {
  const BUFFER = 50;
  const start = Math.max(0, centerIndex - BUFFER);
  const end   = Math.min(this.totalItems - 1, centerIndex + BUFFER);

  // Only prefetch pages not already in cache — O(buffer / pageSize) requests max
  const pagesToLoad = new Set<number>();
  for (let i = start; i <= end; i++) {
    const page = Math.floor(i / this.PAGE_SIZE);
    if (!this.loadedPages.has(page)) pagesToLoad.add(page);
  }
  pagesToLoad.forEach(p => this.loadPage(p));
}
```

**Algorithm:** Sliding window of ±50 items around the current scroll index. Intersection with already-loaded pages is O(1) via Set. Only unloaded page ranges are fetched — prevents redundant API calls.

---

### Q12: Permission cache with TTL — explain the RbacService algorithm.

From `ARCHITECTURE.md` — `RbacService`:

```typescript
private readonly permissionCache = new Map<string, Permission[]>();
private readonly cacheExpiry     = new Map<string, number>();
private readonly CACHE_TTL_MS    = 15 * 60 * 1000;  // 15 minutes (Citibank security policy)

async getUserPermissions(userId: string): Promise<Permission[]> {
  // O(1) cache hit
  const expiry = this.cacheExpiry.get(userId);
  if (expiry && Date.now() < expiry) {
    return this.permissionCache.get(userId)!;
  }

  // Cache miss — fetch from IAM service
  const permissions = await this.iamService.getPermissions(userId).toPromise();
  this.permissionCache.set(userId, permissions!);
  this.cacheExpiry.set(userId, Date.now() + this.CACHE_TTL_MS);
  return permissions!;
}
```

**Security note:** 15-minute TTL balances performance (no IAM call per route) with security (permission revocations propagate within 15 minutes — meets Citibank SOC2 requirements). `canActivate` in `TradingAuthGuard` calls this synchronously after warm-up, making route transitions feel instant.

---

## Part 4 — Angular Signals Deep Dive

### Q13: What is the diamond problem in RxJS and how do signals solve it?

**Diamond problem** — when two observables derived from the same source both emit, a combinator fires twice:

```typescript
// Source
const price$ = this.marketData$.pipe(map(d => d.price));

// Two derived streams from the same source
const usd$ = price$.pipe(map(p => p * 1.0));     // branch A
const eur$ = price$.pipe(map(p => p * 0.93));    // branch B

// Combinator fires TWICE per source emission — once when usd$ updates, once when eur$ updates
combineLatest([usd$, eur$]).subscribe(([u, e]) => this.display(u, e));
// Output on one tick: display(100, 93.0) → display(100, 93.0)  // duplicate render!
```

**Signals batch synchronously — diamond problem eliminated:**
```typescript
readonly price   = signal(0);
readonly usdPrice = computed(() => this.price() * 1.0);
readonly eurPrice = computed(() => this.price() * 0.93);

// effect() runs ONCE after the synchronous batch completes — no duplicate
effect(() => {
  this.display(this.usdPrice(), this.eurPrice());
});
// When price.set(100): usdPrice and eurPrice both update in the same microtask,
// effect fires exactly once.
```

---

### Q14: `effect` and `untracked` — when and how?

| API | Purpose |
|-----|---------|
| `effect()` | Side effect that re-runs when any accessed signal changes |
| `untracked()` | Reads a signal without registering it as a dependency |
| `afterRenderEffect()` | Same as `effect` but guaranteed to run after DOM paint |

**`untracked` prevents infinite cycles:**
```typescript
// BAD — reading logCount inside effect that writes logCount → infinite loop
effect(() => {
  const data = this.marketData();             // tracked dependency
  this.logCount.update(n => n + 1);           // mutates signal → re-triggers effect
});

// GOOD — read logCount without tracking it
effect(() => {
  const data = this.marketData();             // re-runs when marketData changes
  const count = untracked(() => this.logCount());  // read without dependency
  this.auditService.log(data, count);
});
```

**`afterRenderEffect` for DOM focus (angularspace.com pattern):**
```typescript
readonly editMode  = signal(false);
readonly inputRef  = viewChild<ElementRef>('inputRef');

constructor() {
  afterRenderEffect(() => {
    const isEditing = this.editMode();           // dependency: re-run when editMode changes
    untracked(() => {
      if (isEditing) {
        this.inputRef()?.nativeElement?.focus();  // DOM operation after paint — safe
      }
    });
  });
}
```

---

### Q15: Replace all six lifecycle hooks with Signal equivalents.

| Old Hook | Signal Replacement | Notes |
|----------|---------------------|-------|
| `ngOnInit` | `constructor` + `inject()` | `inject()` available at construction time |
| `ngOnChanges` | `effect(() => { this.inputSignal() })` or `computed()` | Reacts to signal-based `input()` changes |
| `ngAfterViewInit` | `effect()` + `viewChild()` signal | `viewChild` becomes defined after first render |
| `ngAfterContentInit` | `effect()` + `contentChild()` signal | Same pattern as `viewChild` |
| `ngAfterViewChecked` | `afterRenderEffect()` | Runs after every render, not after every check |
| `ngOnDestroy` | `inject(DestroyRef).onDestroy(fn)` | Can be called from services — injector-independent |

**`ngOnDestroy` replacement example:**
```typescript
// Old
ngOnDestroy(): void {
  this.subscription.unsubscribe();
  this.cleanupWebSocket();
}

// New — no lifecycle interface needed, can be extracted to a utility
constructor() {
  const destroyRef = inject(DestroyRef);
  destroyRef.onDestroy(() => {
    this.subscription.unsubscribe();
    this.cleanupWebSocket();
  });
}
```

---

### Q16: How do you migrate an Observable-based service to signals incrementally?

```typescript
// Step 1 — preserve existing stream, add signal facade
@Injectable({ providedIn: 'root' })
export class MarketDataService {
  // Keep the observable for back-compatibility
  readonly marketData$ = this.ws.messages$.pipe(shareReplay(1));

  // Add signal facade (toSignal tears down subscription via DestroyRef automatically)
  readonly marketData = toSignal(this.marketData$, { initialValue: null });
}

// Step 2 — consumers migrate to signal API over time
// Old consumer:
this.marketDataService.marketData$.subscribe(d => this.data.set(d));

// New consumer:
readonly data = this.marketDataService.marketData;  // zero subscription management
```

**Citibank context:** `GlobalStateService` uses `selectSignal()` which is NgRx's built-in signal facade over the store stream — same incremental migration pattern baked into the framework.

---

## Part 5 — Advanced RxJS Patterns

### Q17: Higher-order mapping operators — selection algorithm.

```
The question is: what happens to the PREVIOUS inner observable when a NEW outer value arrives?

switchMap   → CANCEL previous inner    → Use for: search (only latest query matters)
mergeMap    → KEEP all inner parallel  → Use for: logging, fire-and-forget analytics
concatMap   → QUEUE inner sequential   → Use for: form submit (preserve order)
exhaustMap  → IGNORE new while active  → Use for: button mashing prevention
```

**Real-world pattern from `ARCHITECTURE.md` — `RiskEffects`:**
```typescript
loadPortfolioRisk$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TradingActions.loadPortfolio),
    switchMap(action =>
      this.riskService.getPortfolioRisk(action.portfolioId).pipe(
        map(risk => TradingActions.loadPortfolioSuccess({ risk })),
        catchError(err => of(TradingActions.loadPortfolioFailure({ error: err.message })))
      )
    )
  )
);
// switchMap: if user switches portfolios rapidly, previous risk calculation is cancelled.
// This prevents stale risk data from a slow previous request arriving after a newer one.
```

---

### Q18: `share()` vs. `shareReplay()` — explain the semantics and when to use each.

```
share()                         shareReplay({ bufferSize: 1, refCount: true })
─────────────────────────────   ──────────────────────────────────────────────
Hot observable                  Hot observable WITH replay buffer
No replay for late subscribers  Late subscribers get last N emissions
Disconnects with 0 subscribers  refCount:true → disconnects; false → stays alive
Good for: events, user gestures Good for: HTTP caching, config streams
```

**HTTP caching pattern:**
```typescript
// Without shareReplay — every subscriber triggers a new HTTP call
readonly config$ = this.http.get<Config>('/api/config');

// With shareReplay — single HTTP call, all subscribers get the cached response
readonly config$ = this.http.get<Config>('/api/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

**`refCount: false` use case — connection must stay alive across route navigations:**
```typescript
// e.g., WebSocket stays connected even when all current components unmount
readonly marketStream$ = this.ws.connect().pipe(
  shareReplay({ bufferSize: 1, refCount: false })  // never disconnects
);
```

**Interview trap:** If `bufferSize` is omitted from `shareReplay()`, it defaults to unlimited — a memory leak for long-running streams.

---

### Q19: `scan` + `expand` — recursive pagination algorithm.

This pattern fetches multiple pages in a single operator chain without recursion in imperative code.

```typescript
// Problem: fetch 3 pages of messages starting from a given offset,
// accumulate them into one array, prevent double-loading with exhaustMap.

paginatedMessages$ = this.paginationOffset$.pipe(
  // exhaustMap: ignore new offset triggers while current page load is in flight
  exhaustMap(offset =>
    this.api.getMessages(offset).pipe(
      // expand: recursively call itself with the next page until EMPTY is returned
      // (_, i) — _ is the last emission, i is the recursion depth (0-indexed)
      expand((_, i) => i < 2 ? this.api.getMessages(offset + (i + 1) * 20) : EMPTY),

      // scan: accumulate each page response into a growing array
      scan(
        (acc, curr) => ({ data: [...acc.data, ...curr.data] }),
        { data: [] as Message[] }
      )
    )
  )
);
// Result: one observable that emits three progressively complete arrays:
// [page1], [page1+page2], [page1+page2+page3]
```

**Complexity:** O(pages × items/page) total, all within a single subscription. The `exhaustMap` ensures that if the user scrolls rapidly, only the latest scroll position's page-load chain is active.

---

### Q20: `combineLatest` vs. `withLatestFrom` vs. `forkJoin` — choose the right one.

| Operator | Fires when | All must emit first | Completes when |
|----------|-----------|---------------------|----------------|
| `combineLatest` | ANY source emits | Yes | All sources complete |
| `withLatestFrom` | Only FIRST source emits | Second must have emitted at least once | First source completes |
| `forkJoin` | Only on completion | Yes | All sources complete |

```typescript
// combineLatest — live dashboard: update whenever ANY feed changes
combineLatest([price$, volume$, risk$]).subscribe(([p, v, r]) => this.updateDashboard(p, v, r));

// withLatestFrom — trade submit: get current portfolio state at submit time, don't react to portfolio changes
this.submitButton$.pipe(
  withLatestFrom(this.currentPortfolio$)
).subscribe(([_, portfolio]) => this.executeTrade(portfolio));

// forkJoin — app initializer: wait for ALL one-shot configs to resolve before rendering
forkJoin([this.loadConfig(), this.loadPermissions(), this.loadUserProfile()])
  .subscribe(([config, permissions, profile]) => this.bootstrap(config, permissions, profile));
```

---

### Q21: `throttleTime` + `sampleTime` for 60fps rendering — how does it work?

From `ARCHITECTURE.md` — `MarketDataGridComponent`:

```typescript
readonly displayedPrices = toSignal(
  this.marketData.getPrices().pipe(
    // throttleTime(100): allow first event, suppress for 100ms — rate-limits to max 10 updates/sec
    throttleTime(100),

    // sampleTime(16): emit the most recent value every 16ms ≈ 60fps (1000ms / 60 = 16.67ms)
    // If throttleTime already reduced to 10/sec, sampleTime just aligns to frame boundary
    sampleTime(16),

    // Combine with filter signal for final display list
    map(prices => prices.filter(this.activeFilter()))
  ),
  { initialValue: [] }
);
```

**Algorithmic rationale:** Without throttling, 1,000 price events/sec × CDK virtual scroll recalculation = browser frame drops. `throttleTime(100)` reduces to 10/sec. `sampleTime(16)` aligns updates to the 60fps render loop, preventing partial-frame flicker. Together they reduce CD work by ~98.4% while appearing real-time to the user.

---

## Part 6 — Architecture Patterns and Design

### Q22: Explain the Module Federation dynamic loading algorithm with fallback.

From `ARCHITECTURE.md` — `MicroFrontendLoaderService`:

```typescript
async loadRemoteModule(config: RemoteModuleConfig): Promise<Type<any>> {
  // Step 1: Inject remote entry script (adds window[remoteName] to global scope)
  await this.loadRemoteEntry(config.remoteEntry);

  // Step 2: Initialize the remote container (Webpack 5 Module Federation protocol)
  const container = (window as any)[config.remoteName];
  await container.init(__webpack_share_scopes__.default);

  // Step 3: Get the factory for the specific module
  const factory = await container.get(config.exposedModule);
  return factory();
}

private loadRemoteEntry(url: string): Promise<void> {
  // Idempotent — check if script already loaded to avoid duplicate injection
  if (document.querySelector(`script[src="${url}"]`)) return Promise.resolve();

  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = url;
    script.onload = () => resolve();
    script.onerror = () => reject(new Error(`Failed to load: ${url}`));
    document.head.appendChild(script);
  });
}
```

**Fallback component pattern:**
```typescript
loadWithFallback(config: RemoteModuleConfig): Observable<Type<any>> {
  return from(this.loadRemoteModule(config)).pipe(
    catchError(err => {
      this.telemetry.track('mfe_load_failure', { remote: config.remoteName, err });
      return of(this.createErrorComponent(config.remoteName));
    })
  );
}
```

**Key architecture decision (ADR-001):** `eager: false` in `webpack.config.js` means shared dependencies (Angular, RxJS, zone.js) are NOT bundled into each MFE's initial chunk — load on first use, dramatically reducing initial bundle size.

---

### Q23: Nx monorepo — how do you structure a multi-team Angular platform?

From `ARCHITECTURE.md`:

```
apps/
  shell/              → Host application — loads MFEs, handles auth routing
  trading-mfe/        → Trading micro-frontend (standalone app)
  portfolio-mfe/      → Portfolio micro-frontend
  risk-mfe/           → Risk management micro-frontend

libs/
  shared/
    ui-components/    → Design system: BaseComponent, FinancialCard, DataGrid
    data-access/      → Shared HTTP services, interceptors
    utils/            → Pure functions, date formatting, currency pipes
  domain/
    trading/          → Trading domain: NgRx state, models, trading-specific services
    portfolio/        → Portfolio domain
    risk/             → Risk domain

tools/
  generators/         → Custom Nx generators for scaffolding new MFE or domain lib
```

**Tag-based dependency enforcement:**
```json
// .eslintrc.json — prevents circular dependencies across domain boundaries
{ "sourceTag": "scope:trading", "onlyDependOnLibsWithTags": ["scope:shared"] }
```

**Interview insight:** The `libs/domain/trading` library is the anti-corruption layer — external teams cannot bypass it to call raw HTTP endpoints. All trading state mutations go through NgRx actions in this library.

---

### Q24: NgRx at enterprise scale — explain the complete data flow.

```
UI Component  →  dispatchAction()
                      ↓
              NgRx Action (plain object: { type, payload })
                      ↓
              ┌───────┴────────┐
              ↓                ↓
           Reducer          Effects
         (sync, pure)    (async, side effects)
              ↓                ↓
           New State    → new Actions dispatched
              ↓
           Selectors (memoized — recompute only when slice changes)
              ↓
           Component (selectSignal() → Signal<T>)
```

**Memoized selector performance:**
```typescript
// Selector only recomputes if orders or filter change — O(1) cache hit otherwise
export const selectFilteredOrders = createSelector(
  selectAllOrders,      // slice A
  selectActiveFilter,   // slice B
  (orders, filter) => orders.filter(o => o.status === filter)  // only recalculates when A or B change
);

// In component — returns an Angular Signal (not Observable)
readonly filteredOrders = this.store.selectSignal(selectFilteredOrders);
```

**NgRx Entity — O(log n) lookups with EntityAdapter:**
```typescript
export const orderAdapter = createEntityAdapter<Order>({
  selectId: order => order.orderId,   // primary key for O(log n) lookup
  sortComparer: (a, b) => b.timestamp - a.timestamp  // pre-sorted state
});
// adapter.removeOne(id, state) — O(log n), not O(n) linear scan
```

---

## Part 7 — Performance Engineering

### Q25: Explain the OnPush change detection strategy — when and why.

**Default CD:** Angular checks every component in the tree on every browser event → O(n) components checked.

**OnPush CD:** Component is skipped unless:
1. An `@Input()` reference changes
2. An event originates from within the component
3. The `async` pipe signals a new value
4. `ChangeDetectorRef.markForCheck()` is called manually
5. A Signal used in the template emits a new value

```typescript
@Component({
  selector: 'app-market-data-grid',
  changeDetection: ChangeDetectionStrategy.OnPush,  // skip unless inputs change
  template: `
    @for (price of displayedPrices(); track price.symbol) {
      <app-price-row [data]="price" />
    }
  `
})
export class MarketDataGridComponent {
  // Signal — Angular tracks this at the granular level, only marks this component dirty
  readonly displayedPrices = toSignal(
    this.marketData.getPrices().pipe(throttleTime(100), sampleTime(16)),
    { initialValue: [] }
  );
}
```

**Interview answer:** Combined with Signals, `OnPush` reduces the CD surface from the entire component tree to individual signal-tracked expressions. In `ARCHITECTURE.md`, this is the reason `MarketDataGridComponent` can handle 1,000 events/second while maintaining 60fps.

---

### Q26: Core Web Vitals — how do you instrument them in Angular?

From `ARCHITECTURE.md` — `PerformanceMonitoringService`:

```typescript
@Injectable({ providedIn: 'root' })
export class PerformanceMonitoringService {
  private observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      switch (entry.entryType) {
        case 'largest-contentful-paint':
          this.reportMetric('LCP', entry.startTime);  // target: <2,500ms
          break;
        case 'first-input':
          const fidEntry = entry as PerformanceEventTiming;
          this.reportMetric('FID', fidEntry.processingStart - fidEntry.startTime); // <100ms
          break;
        case 'layout-shift':
          const clsEntry = entry as any;
          if (!clsEntry.hadRecentInput)
            this.cumulativeCLS += clsEntry.value;                                  // <0.1
          break;
      }
    }
  });

  init(): void {
    this.observer.observe({ entryTypes: ['largest-contentful-paint', 'first-input', 'layout-shift'] });
  }
}
```

**Citibank performance targets (from ARCHITECTURE.md):**

| Metric | Target |
|--------|--------|
| First Contentful Paint | < 1,500ms |
| Largest Contentful Paint | < 2,500ms |
| First Input Delay | < 100ms |
| Cumulative Layout Shift | < 0.1 |
| Time to Interactive | < 3,500ms |
| Bundle size (initial) | < 250KB gzipped |
| Market data latency | < 10ms |
| Trade execution latency | < 50ms |

---

### Q27: What is lazy loading and how do you architect it for a micro-frontend platform?

**Three levels of lazy loading in the trading platform:**

**Level 1 — Route-based lazy loading:**
```typescript
export const appRoutes: Routes = [
  {
    path: 'trading',
    loadChildren: () => import('./features/trading/trading.routes')
      .then(m => m.TRADING_ROUTES)
  }
];
// Angular only downloads the trading bundle when the user navigates to /trading
```

**Level 2 — Micro-frontend lazy loading (Module Federation):**
```typescript
// From DynamicRouteService — routes are built from feature flags + permissions at runtime
const mfeRoute: Route = {
  path: 'portfolio',
  loadComponent: () =>
    this.mfeLoader.loadRemoteModule({
      remoteName: 'portfolioMfe',
      remoteEntry: 'https://portfolio.citibank.com/remoteEntry.js',
      exposedModule: './PortfolioComponent'
    })
};
this.router.resetConfig([...this.baseRoutes, mfeRoute]);
```

**Level 3 — Resource-based preloading (CDK virtual scroll):**
The `preloadDataAroundIndex(centerIndex)` algorithm (Part 3, Q11) lazy-loads data pages as the user scrolls, not all at once.

**Interview insight:** Module Federation enables independent deployments. The shell app at `citibank.com` never needs to redeploy when the trading team ships a new feature — they update `trading.citibank.com/remoteEntry.js` and all users get it on their next visit.

---

## Part 8 — Security, Authentication, and Compliance

### Q28: Describe a production-grade MFA authentication flow.

From `ARCHITECTURE.md` — `EnterpriseAuthenticationService`:

```typescript
async authenticateUser(credentials: UserCredentials): Promise<AuthResult> {
  // Step 1: Primary authentication (password/SSO)
  const primaryResult = await this.validateCredentials(credentials);

  // Step 2: MFA challenge (TOTP / SMS / hardware key)
  if (primaryResult.requiresMFA) {
    const mfaResult = await this.validateMFAChallenge(primaryResult.sessionToken);
    if (!mfaResult.success) throw new UnauthorizedError('MFA_FAILED');
  }

  // Step 3: Risk-based authentication (geolocation, device fingerprint, behavioural score)
  const riskScore = await this.riskEngine.evaluate({ credentials, context: this.getContext() });
  if (riskScore > RISK_THRESHOLD) await this.escalateToAdditionalVerification();

  // Step 4: Session establishment (short-lived JWT + refresh token rotation)
  const session = await this.sessionManager.createSession(primaryResult.userId);

  // Step 5: Audit log (SOC2 compliance — every login attempt recorded with context)
  await this.auditLogger.log({
    event: 'USER_LOGIN',
    userId: primaryResult.userId,
    timestamp: Date.now(),
    ipAddress: this.getClientIP(),
    riskScore
  });

  return { session, permissions: await this.rbacService.getUserPermissions(session.userId) };
}
```

**OWASP Top 10 mitigations built in:**
- Broken Access Control → `TradingAuthGuard` + `RbacService` on every protected route
- Cryptographic Failures → AES-256 at rest, TLS 1.3 in transit, `DataEncryptionService`
- Injection → `HttpClient` auto-encodes params; use parameterized queries in BFF
- Auth Failures → Risk-based auth, refresh token rotation, short JWT TTL (15 min)
- Security Logging → `auditLogger` on every auth event

---

### Q29: How does `TradingAuthGuard` enforce RBAC?

```typescript
@Injectable({ providedIn: 'root' })
export class TradingAuthGuard implements CanActivate {
  private rbacService = inject(RbacService);
  private router = inject(Router);

  async canActivate(route: ActivatedRouteSnapshot): Promise<boolean | UrlTree> {
    const requiredPermissions: Permission[] = route.data['requiredPermissions'] ?? [];
    const userPermissions = await this.rbacService.getUserPermissions(this.currentUserId);

    const hasAccess = requiredPermissions.every(p => userPermissions.includes(p));
    return hasAccess ? true : this.router.createUrlTree(['/unauthorized']);
  }
}

// Route configuration
{
  path: 'execute-trade',
  component: TradeExecutionComponent,
  canActivate: [TradingAuthGuard],
  data: { requiredPermissions: ['TRADE_EXECUTE', 'RISK_APPROVE'] }
}
```

**Algorithm:** `every()` is a short-circuit O(m) check where m = number of required permissions. The 15-minute cache in `RbacService` means the IAM HTTP call only happens once per session cycle, not per route transition.

---

## Part 9 — CRUD Application Architecture

### Q30: Design a full CRUD trading order system with NgRx and reactive forms.

**State shape:**
```typescript
export interface OrderState {
  entities: EntityState<Order>;       // NgRx Entity — normalized, O(log n) access
  loading: boolean;
  error: string | null;
  selectedOrderId: string | null;
}
```

**Reactive form with async validators (from `EquitiesTradingComponent`):**
```typescript
buildOrderForm(): FormGroup {
  return this.fb.group({
    symbol: ['', [Validators.required], [this.symbolExistsValidator()]],  // async: checks exchange API
    quantity: [null, [Validators.required, Validators.min(1)]],
    price: [null, [Validators.required, this.priceRangeValidator()]],     // custom: ±10% from market
    orderType: ['LIMIT', Validators.required]
  });
}

// Custom sync validator — checks price within ±10% of market
priceRangeValidator(): ValidatorFn {
  return (control): ValidationErrors | null => {
    const marketPrice = this.currentMarketPrice();
    if (!marketPrice || !control.value) return null;
    const deviation = Math.abs(control.value - marketPrice) / marketPrice;
    return deviation > 0.10 ? { priceOutOfRange: { deviation: (deviation * 100).toFixed(1) } } : null;
  };
}
```

**Optimistic update pattern:**
```typescript
// Dispatch optimistic update before HTTP call completes
submitOrder(order: Order): void {
  const tempId = `TEMP_${Date.now()}`;
  this.store.dispatch(TradingActions.submitOrderOptimistic({ order: { ...order, id: tempId } }));

  // Effect handles API call; on failure dispatches rollback action
}

// Reducer handles rollback
on(TradingActions.submitOrderFailure, (state, { tempId }) =>
  orderAdapter.removeOne(tempId, { ...state, error: 'Order submission failed' })
)
```

**CRUD + undo/redo:** Maintain a `history: OrderState[]` stack in the meta-reducer. `Ctrl+Z` dispatches `UndoAction` which pops the last state snapshot.

---

## Part 10 — CI/CD, Deployment, and Observability

### Q31: Walk through the Citibank CI/CD pipeline architecture.

From `ARCHITECTURE.md` — GitHub Actions:

```yaml
# Stage 1: Security Scan (OWASP dependency audit)
security-scan:
  runs-on: ubuntu-latest
  steps: [npm audit --audit-level=high, snyk test]

# Stage 2: Quality Gates (must pass before build)
quality-gates:
  needs: security-scan
  steps:
    - nx affected:lint --all
    - nx affected:test --all --coverage
    - Coverage threshold enforcement: 95% global / 99% critical trading paths

# Stage 3: Build (matrix — parallel per MFE)
build-and-test:
  needs: quality-gates
  strategy:
    matrix:
      app: [shell, trading-mfe, portfolio-mfe, risk-mfe]
  steps:
    - nx build ${{ matrix.app }} --prod --source-map=false
    - docker build -t ${{ matrix.app }}:${{ github.sha }} .

# Stage 4: Deploy (Azure Container Apps)
deploy:
  needs: build-and-test
  steps:
    - az containerapp update --name $app --image $IMAGE  # rolling, no downtime
```

**Azure Container Apps Bicep scaling config:**
```bicep
scale: {
  minReplicas: 3          // always 3 pods for HA — never cold-start for traders
  maxReplicas: 10
  rules: [{
    name: 'http-scaling'
    http: { metadata: { concurrentRequests: '30' } }
  }]
}
```

---

### Q32: How do you implement distributed tracing for micro-frontends?

**From `PerformanceMonitoringService` + `BusinessMetricsService`:**

```typescript
// Distributed trace header — passed through every MFE boundary
const correlationId = crypto.randomUUID();
this.http.post('/api/trade', order, {
  headers: { 'X-Correlation-ID': correlationId, 'X-Trace-Context': this.buildW3CTrace() }
});

// Business metrics streamed every second
readonly tradingMetrics$ = interval(1000).pipe(
  switchMap(() => this.getTradingMetrics()),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

**Canary deployment strategy:**
1. Deploy new version to 10% of Container App replicas
2. Monitor `BusinessMetricsService` error rate for 15 minutes
3. If error rate < 0.1%, promote to 100%
4. If degraded, `az containerapp revision deactivate` (instant rollback)

---

## Part 11 — Architecture Decision Records (ADRs)

### Q33: Why are ADRs critical in a Citibank engineering role?

**ADR-001: Angular + Module Federation (from ARCHITECTURE.md):**

```markdown
Status: Accepted
Context: 5 product teams shipping features to the same trading portal; 
         monolithic deployments cause coordination overhead and blast radius risk.
Decision: Module Federation with Nx monorepo. Each team owns an MFE with independent deploy pipeline.
Consequences (+): Independent deploys, team autonomy, tech stack flexibility per domain.
Consequences (-): Shared dependency version negotiation; remote loading adds ~50ms latency.
Alternatives Rejected: Single SPA (similar benefits, less Angular-native), iframes (poor UX).
```

**How to write an ADR in your role:**
1. **Context** — What problem forced a decision? What constraints exist (compliance, teams, performance)?
2. **Decision** — One sentence: "We will use X."
3. **Status** — Proposed / Accepted / Deprecated / Superseded
4. **Consequences** — Honest trade-offs, not just benefits. What becomes harder?
5. **Alternatives Rejected** — Shows you considered options; prevents later "why didn't we do Y?"

**Interview tip:** At Citibank, ADRs serve a second purpose — regulatory audit evidence that architecture changes were reviewed and approved by the Architecture Review Board (ARB).

---

## Self-Reinforcement Training

> Practice answering these five questions aloud without looking at notes. Aim for 3–5 minutes each.

1. **"Without looking at the code — explain how the `MemoryOptimizedMarketDataService` prevents memory leaks using only language concepts, no code."**
   *(Key terms: WeakMap, component reference, automatic GC, LRU eviction, TTL cache, manual vs automatic cleanup)*

2. **"You're reviewing a PR where a colleague used `mergeMap` for a live search feature. What do you tell them and why?"**
   *(Key: mergeMap runs parallel → race conditions on search results; should be `switchMap` to cancel previous request)*

3. **"A new junior engineer asks why Signals eliminated the diamond problem. Explain it in under 2 minutes using an analogy."**
   *(Key: synchronous batch = all derivatives update in one microtask; combineLatest fires once per upstream = two fires)*

4. **"Your team is debating `providers` vs `viewProviders`. Give the rule in one sentence and an example."**
   *(Key: viewProviders excludes projected `<ng-content>` — use when you want DI isolation from host consumers)*

5. **"The trading head asks why OnPush + Signals reduces render cost by 98%. Walk them through the math."**
   *(Key: Default = O(n) traversal per event. OnPush = dirty-check only. Signal = sub-expression granularity. 1000 events/sec × 16ms frame = 16 events amortized per frame. throttleTime(100) → 10/sec. sampleTime(16) → frame-aligned. 98% reduction = 1000 → 20 effective updates.)*

---

## Evaluation Rounds

### Round 1 — Baseline Assessment (~6.5 / 10)

**Evaluator Panel:**
- **Alex T.** — Citibank Front-End Angular Engineer (code quality, modern API usage)
- **Dr. Sarah K.** — Principal Front-End Architect (enterprise architecture, ADR rigor)
- **Marcus L.** — Fellow Senior Engineer (algorithms, practical trade-offs)

---

**Alex T. (Angular Engineer):**
> "You know the APIs: `toSignal`, `model()`, `effect()`. But I'm not hearing you justify *why* you reach for each. When I asked about `untracked`, you described what it does but missed the infinite-cycle prevention use case — that's the only real reason to use it. 
> Also, you described `ControlValueAccessor` correctly but didn't mention that `setDisabledState` is optional — important detail in an interview."

**Score: 6.5 / 10**
**Gaps:** `untracked` motivations, resolver modifier edge cases, `afterRenderEffect` vs `effect` distinction.

---

**Dr. Sarah K. (Principal Architect):**
> "You mentioned Module Federation but couldn't explain the `window[remoteName].init()` bootstrapping sequence. That's the most common failure point in production MFE deployments — remote entry 404, container not initialized, shared scope mismatch. 
> ADRs: You know they exist but can't articulate what makes a consequence section *honest* vs a PR exercise. At Citibank, the ARB will probe your consequences section harder than your decision."

**Score: 6.0 / 10**
**Gaps:** MFE bootstrapping algorithm, ADR quality, Nx tag dependency enforcement.

---

**Marcus L. (Fellow Senior Engineer):**
> "Your LRU algorithm was correct but you gave O(n log n) for the sort-and-evict path without offering the O(1) doubly-linked-list alternative. That's a principal-level oversight — you should know when your current implementation is good enough and when it needs upgrading. The `expand` + `scan` pagination pattern — you described it but couldn't trace the emission sequence step by step."

**Score: 6.8 / 10**
**Gaps:** Algorithm complexity alternatives, `expand` emission tracing, `track $index` deletion hazard.

**Round 1 Composite: 6.4 / 10**
**Action Plan:** Study Parts 3, 5, 6 deeply. Trace `expand` + `scan` by hand with a 3-page example. Memorise the MFE bootstrapping sequence.

---

### Round 2 — Post-Study Assessment (~8.8 / 10)

---

**Alex T. (Angular Engineer):**
> "Excellent recovery on lifecycle hooks — you rattled off all six replacements without hesitation and gave the `DestroyRef.onDestroy` pattern which is often missed. Your `ControlValueAccessor` now includes `setDisabledState` as optional. 
> One remaining gap: you said 'signals are synchronous' but didn't qualify that `effect()` is async (scheduled as a microtask) — important distinction when debugging why an effect hasn't fired yet in a test."

**Score: 8.5 / 10**

---

**Dr. Sarah K. (Principal Architect):**
> "The MFE bootstrapping sequence is now solid — you named `container.init(__webpack_share_scopes__.default)` and the idempotent script injection check. Good. 
> ADR consequences: much stronger. You now give *both* positive and negative consequences and name the alternative you rejected — that's ARB-ready quality. 
> Remaining: you didn't mention `eager: false` as a bundle size implication when discussing Module Federation. That's a deployment trade-off (slower initial remote load vs smaller shell bundle) that a principal must articulate."

**Score: 9.0 / 10**

---

**Marcus L. (Fellow Senior Engineer):**
> "The doubly-linked-list O(1) alternative for LRU — perfect. You correctly noted the current `Map.entries()` sort is O(n log n) and sufficient for MAX_CACHE_SIZE=1000 but would need upgrading at 100K entries. 
> `expand` + `scan` trace: you got pages 0→1→2 correctly. One miss: you said `exhaustMap` 'prevents double-load' without clarifying *what* triggers the double-load — a user scrolling rapidly before the first batch completes. Context matters."

**Score: 8.8 / 10**

**Round 2 Composite: 8.8 / 10**
**Action Plan:** Nail the `effect()` async scheduling caveat, `eager: false` trade-off, double-loading trigger for `exhaustMap`.

---

### Round 3 — Final Assessment (9.9 / 10) ✅

---

**Alex T. (Angular Engineer):**
> "Outstanding. You correctly stated that `effect()` runs as an asynchronous microtask — not synchronously like `computed()` — and explained the implication for unit tests (need `TestBed.flushEffects()` or `tick()`). Your `track` explanation was precise: you covered the deletion-shifts-indices hazard for `track $index`, the reference-change failure for `track item`, and the stable-identity solution with `track item.id`. Template performance: you quantified pipe memoization as O(1) vs function call as O(CD frequency). No gaps on the modern Angular API surface."

**Score: 9.9 / 10**

---

**Dr. Sarah K. (Principal Architect):**
> "Everything I needed to hear at a Citibank principal interview. You articulated `eager: false` as a deliberate trade-off: slower first-load of each remote (network round-trip to fetch shared deps) vs smaller initial shell bundle. You mentioned `strictVersion: true` in the shared config as a strong version contract enforcement. Your ADR discussion was mature — you named the *real* cost of Module Federation: team discipline around shared dependency versioning, and the mitigation (Nx tag enforcement + shared version policy in `package.json`). The Bicep `minReplicas: 3` justification — HA + cold-start elimination — was sharp. Full marks on architecture depth."

**Score: 9.9 / 10**

---

**Marcus L. (Fellow Senior Engineer):**
> "The recursive pagination trace was flawless — you described the three emissions (`[page1]`, `[page1+page2]`, `[page1+page2+page3]`) and why `exhaustMap` specifically prevents the trigger: 'a second scroll event arriving before the three-page chain completes would start a duplicate chain without `exhaustMap`'. 
> The WeakMap vs Map explanation for memory management was textbook — GC semantics, automatic cleanup, contrast with Map strong reference causing leak. The only thing I'll observe is that this is a very high ceiling for a 45-minute interview — you'll have to triage. Lead with the business impact of each pattern (why it matters for trading latency or compliance), then drill into the algorithm. Interviewers at this level respect prioritisation."

**Score: 9.8 / 10**

**Round 3 Composite: 9.87 / 10 ✅ (Exceeds 9.8 target)**

---

**Final Panel Recommendation:**
> "This candidate demonstrates principal-level command of Angular 18+ patterns, enterprise architecture thinking grounded in real financial-domain trade-offs, and the algorithm literacy expected of a solutions architect at Citibank. The self-awareness in triage advice (Marcus's note) reflects engineering maturity beyond individual contributor level. **Strongly recommend for Principal Front-End Architect / Senior Angular Solutions role.** ✅"

---

## Final Checklist — Must Answer Without Notes

Answer each in under 2 minutes. Check off when you can do it consistently:

- [ ] Name all four Resolution Modifiers (`optional`, `self`, `skipSelf`, `host`) and give a real use case for each
- [ ] Explain the diamond problem with a concrete signal example showing a single `effect` firing
- [ ] Trace `expand` + `scan` across 3 pages — name each emission value
- [ ] Choose the correct higher-order operator for: search, form submit, button mash, fire-and-forget
- [ ] Explain WeakMap vs Map for subscription tracking — include GC semantics
- [ ] Give the O(n log n) LRU sort complexity and the O(1) alternative data structure
- [ ] Explain `track $index` hazard with a deletion example (A, B, C → A, C)
- [ ] Describe the five-step Citibank MFA authentication flow from memory
- [ ] Explain `shareReplay({ bufferSize: 1, refCount: true })` — what `refCount: false` changes
- [ ] Name all six lifecycle hooks and their Signal replacements (include `ngAfterViewChecked → afterRenderEffect`)
- [ ] Explain `eager: false` in Module Federation — bundle size vs latency trade-off
- [ ] Quote three of the five Citibank performance targets with their numeric thresholds
- [ ] Explain why `effect()` is asynchronous and how it affects unit tests
- [ ] Describe the `providers` vs `viewProviders` difference in one sentence with a projected content example
- [ ] Write an ADR consequences section for a decision you made recently — include at least one negative consequence

---

*Last updated: ADVANCED_INTERVIEW.md v1.0 — Citibank Front-End Architecture & Advanced Angular*
