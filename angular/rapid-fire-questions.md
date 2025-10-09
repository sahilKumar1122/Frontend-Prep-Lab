# Angular Rapid-Fire Questions & Answers (2025 Edition)

> **Quick answers to essential Angular concepts. For deep dives, follow the links to detailed explanations.**

## Table of Contents
- [Core Framework Fundamentals](#core-framework-fundamentals)
- [Component Lifecycle Hooks](#component-lifecycle-hooks)
- [Dependency Injection System](#dependency-injection-system)
- [Change Detection & Performance](#change-detection--performance)
- [RxJS & Reactive Programming](#rxjs--reactive-programming)
- [Template & View Queries](#template--view-queries)
- [Forms](#forms)
- [Routing](#routing)
- [Performance Optimization](#performance-optimization)
- [State Management](#state-management)
- [Testing](#testing)
- [Modern Angular (v16-v18 Features)](#modern-angular-v16-v18-features)
- [Angular CDK & Material](#angular-cdk--material)
- [Tooling & Ecosystem](#tooling--ecosystem)
- [Advanced / Architecture](#advanced--architecture)
- [Common Interview Trap Keywords](#common-interview-trap-keywords)

---

## Core Framework Fundamentals

### Component
**Q: What is a component?**
**A:** A TypeScript class decorated with `@Component` that controls a view template. It's the basic building block of Angular applications, encapsulating data, logic, and UI.

```typescript
@Component({
  selector: 'app-example',
  template: '<h1>{{ title }}</h1>',
  styleUrls: ['./example.component.scss']
})
export class ExampleComponent {
  title = 'Hello';
}
```

📚 **[Deep Dive →](./fundamentals.md#components)**

---

### Directive (Structural vs Attribute)
**Q: What's the difference between structural and attribute directives?**
**A:** 
- **Structural directives** (`*ngIf`, `*ngFor`, `*ngSwitch`) change DOM structure by adding/removing elements
- **Attribute directives** (`ngClass`, `ngStyle`, custom directives) change appearance or behavior of existing elements

```typescript
// Structural - changes DOM structure
<div *ngIf="isVisible">Content</div>

// Attribute - changes appearance
<div [ngClass]="{ 'active': isActive }">Content</div>
```

---

### Module (NgModule)
**Q: What is an NgModule?**
**A:** A container for cohesive blocks of code with `@NgModule` decorator. It organizes components, directives, pipes, and services into functional units. Includes declarations, imports, providers, and bootstrap configuration.

```typescript
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule],
  providers: [UserService],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

📚 **[Deep Dive →](./fundamentals.md#modules)**

---

### Pipe (Pure vs Impure)
**Q: What's the difference between pure and impure pipes?**
**A:** 
- **Pure pipes** (default): Execute only when input reference changes. Highly performant.
- **Impure pipes** (`pure: false`): Execute on every change detection cycle. Use sparingly.

```typescript
@Pipe({ name: 'purePipe', pure: true })  // Only recalculates on input change
@Pipe({ name: 'impurePipe', pure: false }) // Recalculates every CD cycle
```

---

### Template Syntax
**Q: What are the four types of template syntax?**
**A:**
1. **Interpolation**: `{{ expression }}` - One-way data binding from component to view
2. **Property Binding**: `[property]="expression"` - Bind to element properties
3. **Event Binding**: `(event)="handler()"` - Listen to events
4. **Two-way Binding**: `[(ngModel)]="property"` - Bidirectional data flow

```typescript
{{ title }}                    // Interpolation
[disabled]="isLoading"         // Property binding
(click)="handleClick()"        // Event binding
[(ngModel)]="username"         // Two-way binding
```

📚 **[Deep Dive →](./fundamentals.md#data-binding)**

---

### Change Detection
**Q: What is change detection?**
**A:** The mechanism Angular uses to synchronize component state with the view. When data changes, Angular checks the component tree and updates the DOM. Triggered by Zone.js monkey-patching async operations.

📚 **[Deep Dive →](./change-detection.md)**

---

### Zone.js
**Q: What does Zone.js do?**
**A:** Monkey-patches browser APIs (setTimeout, addEventListener, Promise, etc.) to track async operations and automatically trigger change detection. It creates an execution context that persists across async tasks.

```typescript
// Zone.js intercepts:
setTimeout(() => {})      // Patched
addEventListener()        // Patched
Promise.then()           // Patched
XMLHttpRequest           // Patched
// ~40 browser APIs total
```

📚 **[Deep Dive →](./change-detection.md)** | **[Modern Angular →](./modern-angular-features.md#zoneless-angular)**

---

### View Encapsulation
**Q: What are the three view encapsulation modes?**
**A:**
1. **Emulated** (default): Scopes styles using attribute selectors (`_ngcontent-xxx`)
2. **ShadowDom**: Uses native Shadow DOM for true encapsulation
3. **None**: No encapsulation, styles are global

```typescript
@Component({
  encapsulation: ViewEncapsulation.Emulated // Default
})
```

---

### Ivy Renderer
**Q: What is Ivy?**
**A:** Angular's modern compilation and rendering engine (default since v9). Provides smaller bundles, faster compilation, better debugging, and improved tree-shaking. Uses incremental DOM approach.

**Key improvements:**
- 40% smaller bundles
- Faster build times
- Better template type checking
- Improved debugging (access component in console: `ng.getComponent($0)`)

---

### Compilation (AOT vs JIT)
**Q: What's the difference between AOT and JIT?**
**A:**

| Aspect | AOT (Ahead-of-Time) | JIT (Just-in-Time) |
|--------|---------------------|-------------------|
| When | Build time | Runtime (browser) |
| Bundle Size | Smaller (no compiler) | Larger (+compiler) |
| Performance | Faster (pre-compiled) | Slower (compile in browser) |
| Errors | Caught at build | Caught at runtime |
| Production | ✅ Recommended | ❌ Not recommended |

```bash
ng build --aot=true   # AOT (default for production)
ng serve --aot=false  # JIT (default for dev)
```

---

### Metadata Reflection API
**Q: What is metadata reflection?**
**A:** Allows Angular to read decorator metadata at runtime using TypeScript's `Reflect.getMetadata()`. Used for dependency injection, component configuration, and decorator processing.

---

### Decorators
**Q: What are the main Angular decorators?**
**A:**
- `@Component` - Define a component
- `@Directive` - Define a directive  
- `@Pipe` - Define a pipe
- `@Injectable` - Mark class as available for DI
- `@NgModule` - Define a module
- `@Input` - Input property binding
- `@Output` - Output event emitter
- `@ViewChild/@ContentChild` - Query view/content children
- `@HostListener/@HostBinding` - Listen to host events/bind to host properties

---

## Component Lifecycle Hooks

### Execution Order
**Q: What is the execution order of lifecycle hooks?**
**A:**
1. **ngOnChanges** - When `@Input()` changes
2. **ngOnInit** - After first `ngOnChanges`, initialize component
3. **ngDoCheck** - Every change detection cycle (use sparingly)
4. **ngAfterContentInit** - After `<ng-content>` projected (once)
5. **ngAfterContentChecked** - After content checked (every CD)
6. **ngAfterViewInit** - After view initialized (once)
7. **ngAfterViewChecked** - After view checked (every CD)
8. **ngOnDestroy** - Before component destroyed (cleanup!)

📚 **[Complete Lifecycle Deep Dive →](./lifecycle-hooks.md)**

---

### ngOnChanges()
**Q: When does ngOnChanges fire?**
**A:** Fires when any `@Input()` property changes. Receives a `SimpleChanges` object with previous and current values. Fires before `ngOnInit` and whenever parent updates input bindings.

```typescript
ngOnChanges(changes: SimpleChanges): void {
  if (changes['userId']) {
    console.log('Previous:', changes['userId'].previousValue);
    console.log('Current:', changes['userId'].currentValue);
  }
}
```

📚 **[Deep Dive →](./lifecycle-hooks.md#ngonchanges)**

---

### ngOnInit()
**Q: What is ngOnInit used for?**
**A:** Initialize component after Angular sets `@Input()` properties. Perfect for API calls, subscriptions, and setup logic. Fires once after first `ngOnChanges`.

```typescript
ngOnInit(): void {
  this.loadData();           // ✅ API calls
  this.setupSubscriptions(); // ✅ RxJS subscriptions
  this.initializeForm();     // ✅ Form setup
}
```

📚 **[Constructor vs ngOnInit →](./constructor-vs-ngoninit.md)** | **[Lifecycle Deep Dive →](./lifecycle-hooks.md#ngOnInit)**

---

### ngDoCheck()
**Q: When should you use ngDoCheck?**
**A:** For custom change detection when Angular's default doesn't catch changes (e.g., array mutations, deep object changes). **Warning:** Runs on every CD cycle - use sparingly!

```typescript
ngDoCheck(): void {
  const hash = this.quickHash();
  if (this.lastHash !== hash) {
    this.customChangeDetection();
    this.lastHash = hash;
  }
}
```

📚 **[Deep Dive →](./lifecycle-hooks.md#ngDoCheck)**

---

### ngAfterContentInit()
**Q: When does ngAfterContentInit fire?**
**A:** After Angular projects external content into component's view via `<ng-content>`. `@ContentChild/@ContentChildren` queries are now available.

```typescript
@ContentChild(ChildComponent) child!: ChildComponent;

ngAfterContentInit(): void {
  console.log(this.child); // ✅ Now available
}
```

📚 **[Content Projection →](./content-projection.md)** | **[Lifecycle →](./lifecycle-hooks.md#ngAfterContentInit)**

---

### ngAfterContentChecked()
**Q: When does ngAfterContentChecked fire?**
**A:** After Angular checks projected content (every CD cycle). Use sparingly - avoid heavy operations or state changes here.

📚 **[Deep Dive →](./lifecycle-hooks.md#ngAfterContentChecked)**

---

### ngAfterViewInit()
**Q: What is ngAfterViewInit used for?**
**A:** After component's view and child views initialize. `@ViewChild/@ViewChildren` queries are now available. Perfect for DOM manipulation, third-party library initialization, measuring elements.

```typescript
@ViewChild('canvas') canvas!: ElementRef;

ngAfterViewInit(): void {
  this.chart = new Chart(this.canvas.nativeElement); // ✅ DOM ready
}
```

📚 **[Deep Dive →](./lifecycle-hooks.md#ngAfterViewInit)**

---

### ngAfterViewChecked()
**Q: When does ngAfterViewChecked fire?**
**A:** After Angular checks component's view and child views (every CD cycle). **Avoid state changes** here to prevent `ExpressionChangedAfterItHasBeenCheckedError`.

📚 **[Deep Dive →](./lifecycle-hooks.md#ngAfterViewChecked)**

---

### ngOnDestroy()
**Q: Why is ngOnDestroy critical?**
**A:** **Cleanup!** Unsubscribe observables, clear timers, remove event listeners, close WebSocket connections, destroy third-party libraries. **#1 cause of memory leaks** is skipping cleanup here.

```typescript
ngOnDestroy(): void {
  this.destroy$.next();          // ✅ Unsubscribe observables
  this.destroy$.complete();
  clearInterval(this.intervalId); // ✅ Clear timers
  window.removeEventListener();   // ✅ Remove listeners
  this.chart?.destroy();          // ✅ Destroy libraries
}
```

📚 **[Memory Leaks →](./debugging-memory-leaks.md)** | **[Lifecycle →](./lifecycle-hooks.md#ngOnDestroy)**

---

## Dependency Injection System

### Injector Hierarchy
**Q: How does Angular's hierarchical injector work?**
**A:** Angular creates a tree of injectors mirroring the component tree. When resolving a dependency, it searches:
1. Element injector (component/directive)
2. Parent element injectors (up the tree)
3. Module injector
4. Root injector

```typescript
// Search order:
Component providers → Parent Component → ... → Module → Root
```

📚 **[Complete DI Deep Dive →](./dependency-injection.md)**

---

### Provider Scopes
**Q: What are the three provider scopes?**
**A:**
- **root**: Singleton across entire app (tree-shakeable)
- **any**: One instance per lazy-loaded module
- **platform**: Shared across multiple Angular apps

```typescript
@Injectable({ providedIn: 'root' })  // ✅ Recommended (tree-shakeable)
@Injectable({ providedIn: 'any' })   // Lazy module scope
@Injectable({ providedIn: 'platform' }) // Multi-app sharing
```

📚 **[Deep Dive →](./dependency-injection.md#provider-scopes)**

---

### Injection Tokens
**Q: When do you need InjectionToken?**
**A:** For non-class dependencies (strings, objects, functions) that can't be used as types. Prevents naming collisions and enables type safety.

```typescript
export const API_URL = new InjectionToken<string>('api.url');

// Provide
providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]

// Inject
constructor(@Inject(API_URL) private apiUrl: string) {}
```

📚 **[Deep Dive →](./dependency-injection.md#injection-tokens)**

---

### Multi-Providers
**Q: What are multi-providers?**
**A:** Allow multiple providers for the same token, returning an array of instances. Used for extensible collections like HTTP interceptors, validators, or plugins.

```typescript
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }
]
// Result: array of [AuthInterceptor, LoggingInterceptor]
```

📚 **[Deep Dive →](./dependency-injection.md#multi-providers)**

---

### @Optional()
**Q: What does @Optional do?**
**A:** Tells Angular not to throw an error if dependency isn't found. Injects `null` instead.

```typescript
constructor(@Optional() private logger?: LoggerService) {
  if (this.logger) {
    this.logger.log('Available');
  }
}
```

---

### @Self()
**Q: What does @Self do?**
**A:** Only look for dependency in current component's injector. Don't search up the tree.

```typescript
@Component({
  providers: [LocalService]
})
export class MyComponent {
  constructor(@Self() private local: LocalService) {} // Must be in this component
}
```

---

### @SkipSelf()
**Q: What does @SkipSelf do?**
**A:** Skip current injector, start search from parent.

```typescript
constructor(@SkipSelf() private parent: ParentService) {} // Get from parent
```

---

### @Host()
**Q: What does @Host do?**
**A:** Stop injector search at host component (doesn't go beyond host boundary). Useful for directives querying host component.

```typescript
@Directive({ selector: '[appTooltip]' })
export class TooltipDirective {
  constructor(@Host() private host: MyComponent) {}
}
```

---

### Tree-Shakable Providers
**Q: What makes a provider tree-shakeable?**
**A:** Using `providedIn: 'root'` instead of module `providers` array. If service isn't used, it won't be in final bundle.

```typescript
// ✅ Tree-shakeable
@Injectable({ providedIn: 'root' })

// ❌ Not tree-shakeable
@NgModule({ providers: [MyService] })
```

---

### DI in Standalone Components
**Q: How does DI work in standalone components?**
**A:** Import providers directly via `importProvidersFrom()` or use `providers` in route config. No NgModule needed.

```typescript
@Component({
  standalone: true,
  providers: [LocalService],
  imports: [CommonModule]
})
```

📚 **[Modern Angular →](./modern-angular-features.md#standalone-components)**

---

## Change Detection & Performance

### ChangeDetectionStrategy.OnPush
**Q: What does OnPush do?**
**A:** Skips change detection unless:
1. `@Input()` reference changes
2. Event fires in component or children
3. Async pipe emits new value
4. Manually call `markForCheck()`

**Result:** Massive performance boost (90%+ CD reduction)

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
```

📚 **[Complete CD Deep Dive →](./change-detection.md)**

---

### ChangeDetectorRef
**Q: What is ChangeDetectorRef?**
**A:** API to manually control change detection for a component.

```typescript
constructor(private cdr: ChangeDetectorRef) {}

cdr.detectChanges()      // Run CD now
cdr.markForCheck()       // Schedule CD (OnPush bypass)
cdr.detach()            // Detach from CD tree
cdr.reattach()          // Reattach to CD tree
```

📚 **[Deep Dive →](./change-detection.md#changedetectorref)**

---

### markForCheck()
**Q: When do you use markForCheck?**
**A:** With OnPush to tell Angular "something changed, please check this component". Useful for async operations outside Angular's knowledge or manual state updates.

```typescript
ngOnInit(): void {
  this.externalService.subscribe(data => {
    this.data = data;
    this.cdr.markForCheck(); // Tell Angular to check
  });
}
```

---

### detectChanges()
**Q: What does detectChanges do?**
**A:** Immediately runs change detection for component and its children. Synchronous. Use when you need instant DOM update.

```typescript
this.value = newValue;
this.cdr.detectChanges(); // Update DOM now (synchronous)
```

---

### ApplicationRef.tick()
**Q: What is ApplicationRef.tick?**
**A:** Manually trigger change detection for entire app. Used in zoneless Angular or when running code outside Zone.js.

```typescript
constructor(private appRef: ApplicationRef) {}

this.appRef.tick(); // Check entire app
```

---

### NgZone.runOutsideAngular()
**Q: Why use runOutsideAngular?**
**A:** Run code outside Zone.js to prevent triggering change detection. Massive performance boost for heavy operations (animations, polling, canvas rendering).

```typescript
ngOnInit(): void {
  this.ngZone.runOutsideAngular(() => {
    setInterval(() => {
      // Heavy operation (no CD triggered)
      this.renderCanvas();
    }, 16); // 60 FPS
  });
}
```

**Performance:** 5000 CD calls/sec → 50 CD calls/sec (99% reduction)

📚 **[Performance →](./performance-optimization.md)** | **[CD Deep Dive →](./change-detection.md)**

---

### Pure Pipes vs Impure Pipes
**Q: How do pure and impure pipes affect performance?**
**A:**
- **Pure**: Only executes when input reference changes (fast)
- **Impure**: Executes every CD cycle (slow - can kill performance)

```typescript
{{ data | purePipe }}      // Executes when data reference changes
{{ data | impurePipe }}    // Executes 100+ times/second
```

**Rule:** Always use pure pipes unless you need to detect mutations.

---

### trackBy in *ngFor
**Q: Why is trackBy critical?**
**A:** Tells Angular how to identify items in a list. Without it, Angular destroys and recreates all DOM nodes on every change. With it, Angular reuses existing nodes.

```typescript
// ❌ Without trackBy: Destroys all 1000 DOM nodes on change
<div *ngFor="let item of items">{{ item.name }}</div>

// ✅ With trackBy: Only updates changed nodes
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>

trackById(index: number, item: any): number {
  return item.id; // Unique identifier
}
```

**Performance:** 800ms → 50ms render time (94% faster)

📚 **[Performance Deep Dive →](./performance-optimization.md)**

---

### CDK Virtual Scroll
**Q: When should you use Virtual Scroll?**
**A:** For lists with 100+ items. Renders only visible items (~15 DOM nodes) instead of all items (e.g., 10,000 nodes).

```typescript
<cdk-virtual-scroll-viewport [itemSize]="50" style="height: 500px">
  <div *cdkVirtualFor="let item of items">{{ item }}</div>
</cdk-virtual-scroll-viewport>
```

**Performance:** 10,000 DOM nodes → 15 DOM nodes (99.85% reduction)

📚 **[Performance →](./performance-optimization.md#virtual-scrolling)**

---

### signal, computed, effect
**Q: What are Signals?**
**A:** Fine-grained reactive primitives (Angular 16+). Alternative to Observables for synchronous state. Auto-track dependencies and trigger surgical DOM updates.

```typescript
count = signal(0);                           // Reactive value
double = computed(() => this.count() * 2);   // Derived value
effect(() => console.log(this.count()));     // Side effect

this.count.set(5);        // Update
this.count.update(n => n + 1); // Increment
```

**Benefits:** No subscriptions, no memory leaks, fine-grained CD, simpler than RxJS for state

📚 **[Modern Angular →](./modern-angular-features.md#signals)**

---

### Zoneless Angular
**Q: What is Zoneless Angular?**
**A:** Running Angular without Zone.js (experimental, v17+). Uses Signals and manual `markForCheck()` instead of automatic change detection.

```typescript
bootstrapApplication(AppComponent, {
  providers: [provideExperimentalZonelessChangeDetection()]
});
```

**Benefits:** -200KB bundle, more predictable performance, no monkey-patching
**Trade-offs:** Must use Signals or manual CD, not production-ready yet

📚 **[Modern Angular →](./modern-angular-features.md#zoneless-angular)**

---

## RxJS & Reactive Programming

### Observable
**Q: What is an Observable?**
**A:** A lazy, push-based stream that can emit multiple values over time. Core of reactive programming in Angular.

```typescript
const obs$ = new Observable(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});

obs$.subscribe(val => console.log(val)); // 1, 2
```

---

### Subject, BehaviorSubject, ReplaySubject, AsyncSubject
**Q: What's the difference between Subject types?**
**A:**

| Type | Description | Initial Value | Replay | Use Case |
|------|-------------|---------------|--------|----------|
| **Subject** | Basic multicast, no replay | No | No | Events, fire-and-forget |
| **BehaviorSubject** | Stores current value | Yes (required) | Last 1 | State management, current value needed |
| **ReplaySubject** | Replays N values | No | N values | History, late subscribers |
| **AsyncSubject** | Emits only final value | No | Last only (on complete) | Async result (like Promise) |

```typescript
const subject = new Subject<number>();
const behavior = new BehaviorSubject<number>(0);      // Requires initial value
const replay = new ReplaySubject<number>(3);          // Replay last 3
const async = new AsyncSubject<number>();             // Only final value
```

---

### RxJS Operators
**Q: What are the essential RxJS operators?**
**A:**

**Transformation:**
- `map()` - Transform each value
- `pluck()` - Extract property
- `switchMap()` - Switch to new observable (cancel previous)
- `mergeMap()` - Merge multiple observables (concurrent)
- `concatMap()` - Execute sequentially (one at a time)
- `exhaustMap()` - Ignore new while current executing

**Filtering:**
- `filter()` - Filter values
- `debounceTime()` - Wait for pause
- `distinctUntilChanged()` - Skip duplicates
- `take(n)` - Take first n values
- `takeUntil()` - Take until signal
- `first()`, `last()` - Take first/last

**Combination:**
- `combineLatest()` - Latest from all
- `forkJoin()` - Wait for all to complete (like Promise.all)
- `merge()` - Merge multiple streams
- `zip()` - Pair values by index

**Utility:**
- `tap()` - Side effects (debugging)
- `shareReplay()` - Multicast and cache
- `catchError()` - Error handling
- `retry()` - Retry on error

📚 **[RxJS Operators Deep Dive →](./rxjs-operators.md)**

---

### Cold vs Hot Observables
**Q: What's the difference?**
**A:**
- **Cold**: Creates new producer for each subscription (HTTP requests, `of()`, `from()`)
- **Hot**: Shares single producer across subscriptions (Subjects, DOM events, WebSockets)

```typescript
// Cold: Each subscriber gets separate HTTP request
const cold$ = http.get('/api/data');

// Hot: All subscribers share same stream
const hot$ = new Subject<number>();
```

---

### Memory Leaks
**Q: How do RxJS subscriptions cause memory leaks?**
**A:** Forgetting to unsubscribe keeps subscription, component, and closure scope in memory. Can accumulate hundreds of subscriptions over time.

```typescript
// ❌ Memory leak
ngOnInit(): void {
  this.service.getData().subscribe(data => this.data = data);
}

// ✅ Fixed with takeUntil
private destroy$ = new Subject<void>();

ngOnInit(): void {
  this.service.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => this.data = data);
}

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}
```

📚 **[Memory Leaks Deep Dive →](./debugging-memory-leaks.md)**

---

### pipe()
**Q: What is the pipe function?**
**A:** Chains multiple RxJS operators for clean, composable stream transformations.

```typescript
source$.pipe(
  map(x => x * 2),
  filter(x => x > 10),
  debounceTime(300),
  distinctUntilChanged()
).subscribe();
```

---

### Subscription Management
**Q: What are the best subscription management strategies?**
**A:**

**1. takeUntil (Best for multiple subscriptions)**
```typescript
private destroy$ = new Subject<void>();

ngOnInit(): void {
  obs1$.pipe(takeUntil(this.destroy$)).subscribe();
  obs2$.pipe(takeUntil(this.destroy$)).subscribe();
}

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}
```

**2. async pipe (Best for templates)**
```typescript
data$ = this.service.getData();
// Template: {{ data$ | async }}
```

**3. Manual unsubscribe**
```typescript
sub = obs$.subscribe();
ngOnDestroy(): void { this.sub.unsubscribe(); }
```

---

### take, takeWhile, first, last
**Q: When do you use each?**
**A:**
- `take(n)` - Take first n values then complete
- `takeWhile(predicate)` - Take while condition true
- `first()` - Take first value then complete
- `last()` - Wait for complete, emit last value

```typescript
source$.pipe(take(3))                    // First 3 values
source$.pipe(takeWhile(x => x < 10))     // While condition true
source$.pipe(first())                    // First value
source$.pipe(last())                     // Last value (waits for complete)
```

---

### combineLatest, forkJoin, zip, merge
**Q: What's the difference?**
**A:**

| Operator | Behavior | Use Case |
|----------|----------|----------|
| **combineLatest** | Emit when any source emits (needs all to emit once) | Form validation, dependent fields |
| **forkJoin** | Emit when all complete (like Promise.all) | Parallel HTTP requests |
| **zip** | Pair values by index | Synchronize streams |
| **merge** | Emit from any source (as they emit) | Combine independent streams |

```typescript
// Wait for all to complete
forkJoin([http1$, http2$]).subscribe(([data1, data2]) => {});

// Emit whenever any emits
combineLatest([field1$, field2$]).subscribe(([val1, val2]) => {});
```

📚 **[RxJS State Management →](./rxjs-state.md)**

---

### async pipe
**Q: Why is async pipe recommended?**
**A:** Automatically subscribes, unsubscribes, and triggers change detection. Prevents memory leaks and reduces boilerplate.

```typescript
// Component
data$ = this.service.getData();

// Template
<div *ngIf="data$ | async as data">{{ data.name }}</div>
```

**Benefits:** No manual subscription management, works with OnPush

---

## Template & View Queries

### @ViewChild()
**Q: What is @ViewChild?**
**A:** Query for single element or component in component's view. Available in `ngAfterViewInit()`.

```typescript
@ViewChild('myInput') input!: ElementRef;
@ViewChild(ChildComponent) child!: ChildComponent;

ngAfterViewInit(): void {
  this.input.nativeElement.focus();
  this.child.someMethod();
}
```

---

### @ViewChildren()
**Q: What is @ViewChildren?**
**A:** Query for multiple elements/components in view. Returns `QueryList<T>`.

```typescript
@ViewChildren(ItemComponent) items!: QueryList<ItemComponent>;

ngAfterViewInit(): void {
  this.items.forEach(item => item.activate());
  this.items.changes.subscribe(() => console.log('Items changed'));
}
```

---

### @ContentChild()
**Q: What is @ContentChild?**
**A:** Query for single element projected via `<ng-content>`. Available in `ngAfterContentInit()`.

```typescript
@ContentChild(HeaderComponent) header!: HeaderComponent;

ngAfterContentInit(): void {
  console.log(this.header); // Projected content
}
```

📚 **[Content Projection →](./content-projection.md)**

---

### @ContentChildren()
**Q: What is @ContentChildren?**
**A:** Query for multiple projected elements. Returns `QueryList<T>`.

```typescript
@ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
```

---

### ng-content (Content Projection)
**Q: What is ng-content?**
**A:** Projects content from parent into child component template. Enables flexible, reusable components.

```typescript
// Child
@Component({
  selector: 'app-card',
  template: '<div class="card"><ng-content></ng-content></div>'
})

// Parent
<app-card>
  <h1>Title</h1>
  <p>Content</p>
</app-card>
```

📚 **[Complete Content Projection Deep Dive →](./content-projection.md)**

---

### ng-template
**Q: What is ng-template?**
**A:** Defines template that's not rendered by default. Used with structural directives, `ngTemplateOutlet`, or programmatic rendering.

```typescript
<ng-template #loading>
  <div>Loading...</div>
</ng-template>

<div *ngIf="data; else loading">{{ data }}</div>
```

---

### ng-container
**Q: What is ng-container?**
**A:** Logical grouper that doesn't create DOM element. Useful for structural directives without extra wrapper.

```typescript
<ng-container *ngIf="condition">
  <h1>Title</h1>
  <p>Content</p>
</ng-container>
<!-- No wrapper div created -->
```

---

### TemplateRef and ViewContainerRef
**Q: What are TemplateRef and ViewContainerRef?**
**A:**
- **TemplateRef**: Reference to `<ng-template>` definition
- **ViewContainerRef**: Container where views can be attached (for dynamic rendering)

```typescript
@ViewChild('template') template!: TemplateRef<any>;
constructor(private vcr: ViewContainerRef) {}

ngAfterViewInit(): void {
  this.vcr.createEmbeddedView(this.template);
}
```

---

### Dynamic Component Creation
**Q: How do you create components dynamically?**
**A:** Use `ViewContainerRef.createComponent()` (Ivy) or `ComponentFactoryResolver` (legacy).

```typescript
constructor(private vcr: ViewContainerRef) {}

ngAfterViewInit(): void {
  const componentRef = this.vcr.createComponent(DynamicComponent);
  componentRef.instance.data = 'Hello';
}
```

---

## Forms

### Template-Driven vs Reactive Forms
**Q: What's the difference?**
**A:**

| Aspect | Template-Driven | Reactive Forms |
|--------|----------------|----------------|
| Setup | Template | Component class |
| Validation | Directives | Validators |
| Testing | Harder | Easier |
| Complexity | Simple forms | Complex forms |
| Data Flow | Asynchronous | Synchronous |
| Immutability | Mutable | Immutable |

```typescript
// Template-Driven
<input [(ngModel)]="name" required />

// Reactive
name = new FormControl('', Validators.required);
<input [formControl]="name" />
```

📚 **[Reactive Forms Deep Dive →](./reactive-forms.md)**

---

### FormGroup, FormControl, FormArray
**Q: What are the three form building blocks?**
**A:**
- **FormControl**: Single form field
- **FormGroup**: Group of controls (object)
- **FormArray**: Dynamic array of controls

```typescript
form = new FormGroup({
  name: new FormControl(''),
  email: new FormControl(''),
  addresses: new FormArray([
    new FormGroup({
      street: new FormControl(''),
      city: new FormControl('')
    })
  ])
});
```

📚 **[Deep Dive →](./reactive-forms.md)**

---

### FormBuilder
**Q: What is FormBuilder?**
**A:** Syntactic sugar for creating forms with less boilerplate.

```typescript
constructor(private fb: FormBuilder) {}

form = this.fb.group({
  name: ['', Validators.required],
  email: ['', [Validators.required, Validators.email]]
});
```

---

### Validators
**Q: What are built-in validators?**
**A:**
- `Validators.required` - Required field
- `Validators.email` - Email format
- `Validators.minLength(n)` - Min length
- `Validators.maxLength(n)` - Max length
- `Validators.min(n)` - Min value
- `Validators.max(n)` - Max value
- `Validators.pattern(regex)` - Regex pattern

```typescript
control = new FormControl('', [
  Validators.required,
  Validators.minLength(3),
  Validators.maxLength(20)
]);
```

---

### Custom Validators
**Q: How do you create custom validators?**
**A:** Function that returns `ValidationErrors | null`.

```typescript
// Sync validator
function forbiddenNameValidator(nameRe: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = nameRe.test(control.value);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Async validator
function uniqueEmailValidator(service: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return service.checkEmail(control.value).pipe(
      map(exists => exists ? { emailTaken: true } : null)
    );
  };
}
```

📚 **[Deep Dive →](./reactive-forms.md#custom-validators)**

---

### ControlValueAccessor
**Q: What is ControlValueAccessor?**
**A:** Interface to create custom form controls that integrate with Angular forms. Bridges custom component with form model.

```typescript
@Component({
  selector: 'app-custom-input',
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CustomInputComponent),
    multi: true
  }]
})
export class CustomInputComponent implements ControlValueAccessor {
  writeValue(value: any): void {}
  registerOnChange(fn: any): void {}
  registerOnTouched(fn: any): void {}
  setDisabledState(isDisabled: boolean): void {}
}
```

📚 **[Deep Dive →](./reactive-forms.md#controlvalueaccessor)**

---

### ngModelChange
**Q: What is ngModelChange?**
**A:** Event emitted when ngModel value changes. Fires before parent is updated.

```typescript
<input [(ngModel)]="name" (ngModelChange)="onNameChange($event)" />

onNameChange(value: string): void {
  console.log('New value:', value);
}
```

---

### UpdateOn
**Q: What is updateOn?**
**A:** Controls when form validation/value updates occur: `change` (default), `blur`, or `submit`.

```typescript
form = new FormGroup({
  name: new FormControl('', { updateOn: 'blur' })  // Validate on blur
});

// Or for entire form
form = new FormGroup({...}, { updateOn: 'submit' });
```

---

## Routing

### RouterModule
**Q: What is RouterModule?**
**A:** Module that provides routing functionality. Import `RouterModule.forRoot(routes)` in AppModule, `RouterModule.forChild(routes)` in feature modules.

```typescript
@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

📚 **[Routing Deep Dive →](./routing.md)**

---

### Route Guards
**Q: What are the main route guards?**
**A:**
- **CanActivate**: Can route be activated?
- **CanActivateChild**: Can child routes be activated?
- **CanDeactivate**: Can leave current route?
- **CanLoad**: Can lazy module be loaded?
- **Resolve**: Pre-fetch data before navigation

```typescript
// Functional guard (Angular 14+)
export const authGuard: CanActivateFn = (route, state) => {
  return inject(AuthService).isLoggedIn();
};

// Class-based guard (legacy)
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(): boolean {
    return this.auth.isLoggedIn();
  }
}
```

📚 **[Deep Dive →](./routing.md#guards)**

---

### Lazy Loading
**Q: What is lazy loading?**
**A:** Loading feature modules on-demand instead of at startup. Reduces initial bundle size.

```typescript
// Traditional (NgModule)
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
}

// Standalone
{
  path: 'admin',
  loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent)
}
```

📚 **[Performance →](./performance-optimization.md#lazy-loading)** | **[Routing →](./routing.md#lazy-loading)**

---

### Preloading Strategies
**Q: What are preloading strategies?**
**A:**
- **NoPreloading** (default): Don't preload
- **PreloadAllModules**: Preload all lazy modules after initial load
- **Custom**: Selective preloading logic

```typescript
RouterModule.forRoot(routes, {
  preloadingStrategy: PreloadAllModules
})
```

📚 **[Deep Dive →](./routing.md#preloading)**

---

### Route Parameters & Query Params
**Q: How do you access route parameters?**
**A:**

```typescript
// Route: '/user/:id'
constructor(private route: ActivatedRoute) {}

// Snapshot (one-time)
const id = this.route.snapshot.paramMap.get('id');

// Observable (reactive)
this.route.paramMap.subscribe(params => {
  const id = params.get('id');
});

// Query params: '/search?q=angular'
this.route.queryParamMap.subscribe(params => {
  const query = params.get('q');
});
```

---

### RouterOutlet, RouterLink, ActivatedRoute
**Q: What are the core routing directives?**
**A:**
- **RouterOutlet**: Placeholder where routed component renders
- **RouterLink**: Declarative navigation
- **ActivatedRoute**: Access route information

```typescript
<router-outlet></router-outlet>
<a routerLink="/users" [queryParams]="{ page: 1 }">Users</a>
```

---

### Nested Routes
**Q: How do nested routes work?**
**A:** Child routes render in child `<router-outlet>`.

```typescript
{
  path: 'admin',
  component: AdminComponent,
  children: [
    { path: 'users', component: UsersComponent },
    { path: 'settings', component: SettingsComponent }
  ]
}

// AdminComponent template
<router-outlet></router-outlet>
```

---

### navigateByUrl() vs navigate()
**Q: What's the difference?**
**A:**
- **navigateByUrl()**: Absolute path (string)
- **navigate()**: Relative path (array)

```typescript
router.navigateByUrl('/users/123');           // Absolute
router.navigate(['users', 123]);              // Relative to root
router.navigate(['../sibling'], { relativeTo: route }); // Relative to current
```

---

## Performance Optimization

### Lazy Loading
**Q: Why is lazy loading critical?**
**A:** Splits code into chunks loaded on-demand. Reduces initial bundle from 2MB to 500KB (typical).

📚 **[Performance Deep Dive →](./performance-optimization.md#lazy-loading)**

---

### Preloading Strategies
**Q: When should you preload?**
**A:** Preload likely-to-be-visited routes after initial load. Balance between eager and lazy.

```typescript
// Preload all
PreloadAllModules

// Custom: Preload admin only if user is admin
class CustomPreload implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

---

### OnPush Change Detection
**Q: How much faster is OnPush?**
**A:** 90-99% reduction in change detection calls. Critical for large apps.

📚 **[Change Detection →](./change-detection.md)** | **[Performance →](./performance-optimization.md)**

---

### trackBy in *ngFor
**Q: What's the performance impact?**
**A:** 
- **Without trackBy**: Destroys and recreates all DOM nodes (800ms for 1000 items)
- **With trackBy**: Only updates changed nodes (50ms for 1000 items)

**Improvement: 94% faster**

---

### Pure Pipes
**Q: Why use pure pipes?**
**A:** Execute only when input reference changes (vs. every CD cycle for impure).

---

### Code Splitting
**Q: What is code splitting?**
**A:** Break bundle into smaller chunks loaded on-demand. Achieved via lazy loading, dynamic imports.

---

### Tree Shaking
**Q: What is tree shaking?**
**A:** Dead code elimination during build. Removes unused code.

**Enablers:**
- ES modules
- `providedIn: 'root'`
- AOT compilation

---

### Route-level Code Preloading
**Q: What is route-level preloading?**
**A:** Load route code before navigation (preload) or after initial load (prefetch).

---

### Memoization
**Q: Why memoize values?**
**A:** Cache expensive calculations. Avoid recalculating in templates.

```typescript
// ❌ BAD: Recalculates every CD cycle
{{ expensiveFunction() }}

// ✅ GOOD: Calculate once
totalPrice = this.calculateTotal();
{{ totalPrice }}
```

---

### Caching (HTTP Interceptors)
**Q: How do you cache HTTP requests?**
**A:** Use `shareReplay()` or custom interceptor.

```typescript
// Interceptor caching
@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, HttpResponse<any>>();
  
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    if (this.cache.has(req.url)) {
      return of(this.cache.get(req.url)!);
    }
    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(req.url, event);
        }
      })
    );
  }
}
```

---

### Web Workers
**Q: When should you use Web Workers?**
**A:** For heavy computations (image processing, large data transformations) to avoid blocking UI thread.

```typescript
// Create worker
const worker = new Worker(new URL('./app.worker', import.meta.url));

worker.postMessage({ data: largeArray });
worker.onmessage = ({ data }) => {
  this.result = data;
};
```

---

### SSR + Hydration
**Q: What is hydration?**
**A:** Reusing server-rendered DOM instead of destroying and recreating. Huge performance boost for SSR apps.

```typescript
bootstrapApplication(AppComponent, {
  providers: [provideClientHydration()]
});
```

📚 **[Modern Angular →](./modern-angular-features.md#hydration)**

---

### Deferred Loading
**Q: What is deferred loading?**
**A:** Component-level lazy loading with declarative triggers (`@defer` in Angular 17+).

```typescript
@defer (on viewport) {
  <app-heavy-component />
} @placeholder {
  <div>Loading...</div>
}
```

📚 **[Modern Angular →](./modern-angular-features.md#deferred-loading)**

---

## State Management

### NgRx: Actions, Reducers, Selectors, Effects
**Q: What are the four core NgRx concepts?**
**A:**
- **Actions**: Events that describe what happened
- **Reducers**: Pure functions that update state
- **Selectors**: Query and derive state (memoized)
- **Effects**: Handle side effects (HTTP, WebSocket, etc.)

```typescript
// Action
export const loadUsers = createAction('[Users] Load');

// Reducer
const reducer = createReducer(
  initialState,
  on(loadUsers, state => ({ ...state, loading: true }))
);

// Selector (memoized)
export const selectUsers = createSelector(
  selectUserState,
  state => state.users
);

// Effect
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.api.getUsers())
  )
);
```

📚 **[NgRx Deep Dive →](./ngrx-state-management.md)**

---

### Akita / NGXS basics
**Q: How do Akita and NGXS differ from NgRx?**
**A:**

| Feature | NgRx | Akita | NGXS |
|---------|------|-------|------|
| Boilerplate | High | Low | Medium |
| Learning Curve | Steep | Gentle | Moderate |
| Community | Large | Small | Medium |

**Akita**: Object-oriented, less boilerplate
**NGXS**: Redux pattern with decorators
**NgRx**: Pure Redux, most popular

📚 **[Deep Dive →](./ngrx-state-management.md#alternatives)**

---

### Store DevTools
**Q: What are Store DevTools?**
**A:** Chrome extension for debugging state: time-travel, action replay, state inspection.

```typescript
imports: [
  StoreModule.forRoot(reducers),
  StoreDevtoolsModule.instrument({ maxAge: 25 })
]
```

---

### Immutability
**Q: Why is immutability critical in state management?**
**A:** Enables O(1) reference equality checks instead of O(n) deep equality. Powers OnPush change detection and selector memoization.

```typescript
// ❌ Mutable (breaks OnPush and memoization)
state.users.push(newUser);

// ✅ Immutable (enables optimization)
return { ...state, users: [...state.users, newUser] };
```

---

### Facade Pattern
**Q: What is the Facade pattern?**
**A:** Simplifies NgRx by hiding store complexity behind service API.

```typescript
@Injectable()
export class UserFacade {
  users$ = this.store.select(selectUsers);
  loading$ = this.store.select(selectLoading);
  
  constructor(private store: Store) {}
  
  loadUsers(): void {
    this.store.dispatch(loadUsers());
  }
}

// Component (much simpler!)
constructor(public facade: UserFacade) {}
ngOnInit() { this.facade.loadUsers(); }
```

---

### Signal-based State Management
**Q: How do Signals change state management?**
**A:** Built-in reactivity without RxJS, simpler API, fine-grained updates.

```typescript
// State store with Signals
@Injectable({ providedIn: 'root' })
export class UserStore {
  private state = signal({ users: [], loading: false });
  
  users = computed(() => this.state().users);
  loading = computed(() => this.state().loading);
  
  loadUsers() {
    this.state.update(s => ({ ...s, loading: true }));
    this.api.getUsers().subscribe(users => {
      this.state.update(s => ({ users, loading: false }));
    });
  }
}
```

📚 **[Modern Angular →](./modern-angular-features.md#signal-state)**

---

## Testing

### TestBed
**Q: What is TestBed?**
**A:** Creates isolated Angular testing module, compiles components, provides dependency injection for tests.

```typescript
TestBed.configureTestingModule({
  declarations: [MyComponent],
  imports: [HttpClientTestingModule],
  providers: [MyService]
});

fixture = TestBed.createComponent(MyComponent);
component = fixture.componentInstance;
```

📚 **[Testing Deep Dive →](./testing-strategy.md)**

---

### ComponentFixture
**Q: What is ComponentFixture?**
**A:** Wrapper around component that provides access to component instance, DOM, and change detection.

```typescript
fixture = TestBed.createComponent(MyComponent);
component = fixture.componentInstance;
element = fixture.nativeElement;
fixture.detectChanges();  // Trigger CD
```

---

### async vs fakeAsync
**Q: What's the difference?**
**A:**
- **async**: Waits for real async operations, use `whenStable()`
- **fakeAsync**: Synchronous control of time, use `tick()`

```typescript
// async - real async
it('test', async(() => {
  component.loadData();
  fixture.whenStable().then(() => {
    expect(component.data).toBeDefined();
  });
}));

// fakeAsync - synchronous control
it('test', fakeAsync(() => {
  component.loadData();
  tick();  // Advance virtual time
  expect(component.data).toBeDefined();
}));
```

📚 **[Deep Dive →](./testing-strategy.md#async-testing)**

---

### tick(), flush(), whenStable()
**Q: When do you use each?**
**A:**
- **tick(ms)**: Advance virtual time by specific amount (fakeAsync)
- **flush()**: Execute ALL pending timers (fakeAsync) - fails with infinite intervals
- **whenStable()**: Wait for all async operations to complete (async)

```typescript
tick(1000);          // Advance 1 second
flush();            // Execute all timers
fixture.whenStable(); // Wait for all async
```

---

### Spies and Mocks
**Q: How do you mock dependencies?**
**A:**

```typescript
// Create spy
const spy = jasmine.createSpyObj('MyService', ['getData']);

// Configure spy
spy.getData.and.returnValue(of({ data: 'test' }));

// Provide spy
TestBed.configureTestingModule({
  providers: [{ provide: MyService, useValue: spy }]
});

// Verify calls
expect(spy.getData).toHaveBeenCalledWith(123);
expect(spy.getData).toHaveBeenCalledTimes(1);
```

---

### HttpTestingController
**Q: How do you test HTTP requests?**
**A:** Use `HttpTestingController` to intercept and respond to HTTP calls.

```typescript
beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [HttpClientTestingModule]
  });
  httpMock = TestBed.inject(HttpTestingController);
});

it('should make HTTP request', () => {
  service.getData().subscribe();
  
  const req = httpMock.expectOne('/api/data');
  expect(req.request.method).toBe('GET');
  req.flush({ data: 'test' });
});

afterEach(() => {
  httpMock.verify(); // Verify no outstanding requests
});
```

📚 **[Deep Dive →](./testing-strategy.md#http-testing)**

---

### Jasmine / Jest
**Q: What's the difference?**
**A:**
- **Jasmine**: Default Angular testing framework, behavior-driven
- **Jest**: Facebook's framework, faster, snapshot testing, better parallelization

**Jest advantages:** Faster, better DX, watch mode, coverage built-in

---

### Karma / Vitest / Playwright
**Q: What are these tools?**
**A:**
- **Karma**: Test runner for browsers (deprecated)
- **Vitest**: Modern, fast test runner (Vite-based)
- **Playwright**: E2E testing framework (cross-browser)

```typescript
// Playwright E2E
test('should navigate', async ({ page }) => {
  await page.goto('http://localhost:4200');
  await page.click('[data-test="login-button"]');
  await expect(page).toHaveURL('/dashboard');
});
```

---

### Standalone Component Testing
**Q: How do you test standalone components?**
**A:** Import the component directly, no NgModule needed.

```typescript
TestBed.configureTestingModule({
  imports: [MyStandaloneComponent, HttpClientTestingModule]
});
```

📚 **[Modern Angular →](./modern-angular-features.md#testing-standalone)**

---

## Modern Angular (v16-v18 Features)

### Standalone Components / Directives / Pipes
**Q: What are standalone components?**
**A:** Self-contained components that don't need NgModule. Import dependencies directly.

```typescript
@Component({
  standalone: true,
  imports: [CommonModule, FormsModule, MyPipe],
  template: '<div>Hello</div>'
})
export class MyComponent {}
```

**Benefits:** Simpler DI, better tree-shaking, clearer dependencies, component-level code splitting

📚 **[Modern Angular Deep Dive →](./modern-angular-features.md#standalone-components)**

---

### bootstrapApplication()
**Q: What is bootstrapApplication?**
**A:** Bootstrap Angular app without AppModule (standalone approach).

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient()
  ]
});
```

---

### provideRouter() and route-level providers
**Q: How does routing work with standalone?**
**A:** Use `provideRouter()` instead of `RouterModule.forRoot()`.

```typescript
export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users.component'),
    providers: [UserService]  // Route-level providers
  }
];

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)]
});
```

---

### Signals API (signal, computed, effect)
**Q: What are the three Signal APIs?**
**A:**
- **signal()**: Create reactive value
- **computed()**: Derive value (memoized)
- **effect()**: Side effects when dependencies change

```typescript
count = signal(0);
double = computed(() => this.count() * 2);
effect(() => console.log('Count:', this.count()));

this.count.set(5);        // Update
this.count.update(n => n + 1); // Transform
```

📚 **[Modern Angular →](./modern-angular-features.md#signals)**

---

### Deferred Loading (@defer)
**Q: What is @defer?**
**A:** Declarative component-level lazy loading with triggers (Angular 17+).

```typescript
@defer (on viewport) {
  <app-heavy-chart />
} @loading {
  <div>Loading chart...</div>
} @error {
  <div>Failed to load</div>
} @placeholder {
  <div>Placeholder</div>
}

// Triggers: viewport, idle, hover, interaction, immediate, timer
```

📚 **[Deep Dive →](./modern-angular-features.md#deferred-loading)**

---

### Hydration (Server-Side Rendering)
**Q: What is hydration?**
**A:** Reusing server-rendered DOM instead of destroying and recreating. Massive performance boost.

```typescript
// Enable hydration
bootstrapApplication(AppComponent, {
  providers: [provideClientHydration()]
});
```

**Benefits:** 50% faster Time to Interactive, no flash of unstyled content, better SEO

📚 **[Deep Dive →](./modern-angular-features.md#hydration)**

---

### Zoneless Angular
**Q: How do you opt out of Zone.js?**
**A:** Use experimental zoneless change detection (Angular 17+).

```typescript
bootstrapApplication(AppComponent, {
  providers: [provideExperimentalZonelessChangeDetection()]
});
```

**Trade-offs:** -200KB, more predictable, but requires Signals or manual `markForCheck()`

📚 **[Deep Dive →](./modern-angular-features.md#zoneless)**

---

### Functional Guards and Resolvers
**Q: What are functional guards?**
**A:** Guards as functions instead of classes (Angular 14+, recommended).

```typescript
// Functional guard
export const authGuard: CanActivateFn = () => {
  return inject(AuthService).isLoggedIn();
};

// Functional resolver
export const userResolver: ResolveFn<User> = (route) => {
  return inject(UserService).getUser(route.params['id']);
};

// Usage
{ path: 'admin', canActivate: [authGuard], resolve: { user: userResolver } }
```

---

### Environment Providers
**Q: What are environment providers?**
**A:** Tree-shakeable provider functions for standalone apps.

```typescript
export function provideFeature() {
  return [
    FeatureService,
    { provide: CONFIG, useValue: config }
  ];
}

bootstrapApplication(AppComponent, {
  providers: [provideFeature()]
});
```

---

### Control Flow Syntax (@if, @for, @switch)
**Q: What is the new control flow syntax?**
**A:** Built-in syntax replacing structural directives (Angular 17+).

```typescript
// Old
<div *ngIf="condition">Content</div>
<div *ngFor="let item of items">{{ item }}</div>

// New
@if (condition) {
  <div>Content</div>
}

@for (item of items; track item.id) {
  <div>{{ item }}</div>
}

@switch (value) {
  @case ('a') { <div>A</div> }
  @case ('b') { <div>B</div> }
  @default { <div>Default</div> }
}
```

**Benefits:** Better performance, type safety, built-in trackBy

---

### Improved Template Diagnostics
**Q: What are improved template diagnostics?**
**A:** Better error messages, type checking, and IDE support for templates.

```typescript
// Now catches errors like:
{{ user.nme }}  // Error: Property 'nme' does not exist. Did you mean 'name'?
```

---

## Angular CDK & Material

### CDK Overlay
**Q: What is CDK Overlay?**
**A:** Service for displaying floating panels (dialogs, tooltips, dropdowns) with positioning and scroll management.

```typescript
constructor(private overlay: Overlay) {}

const overlayRef = this.overlay.create({
  positionStrategy: this.overlay.position()
    .global()
    .centerHorizontally()
    .centerVertically()
});

const portal = new ComponentPortal(MyComponent);
overlayRef.attach(portal);
```

---

### CDK Portal
**Q: What is CDK Portal?**
**A:** Piece of UI that can be dynamically rendered into another location.

```typescript
<ng-template cdk-portal #templatePortal="cdkPortal">
  <div>Content to render elsewhere</div>
</ng-template>

<div [cdkPortalOutlet]="templatePortal"></div>
```

---

### CDK DragDrop
**Q: How does CDK Drag and Drop work?**
**A:** Add `cdkDrag` and `cdkDropList` directives.

```typescript
<div cdkDropList (cdkDropListDropped)="drop($event)">
  <div *ngFor="let item of items" cdkDrag>{{ item }}</div>
</div>

drop(event: CdkDragDrop<string[]>) {
  moveItemInArray(this.items, event.previousIndex, event.currentIndex);
}
```

---

### CDK Virtual Scroll
**Q: Why use Virtual Scroll?**
**A:** Renders only visible items. 10,000 items = ~15 DOM nodes instead of 10,000.

```typescript
<cdk-virtual-scroll-viewport [itemSize]="50" style="height: 500px">
  <div *cdkVirtualFor="let item of items">{{ item }}</div>
</cdk-virtual-scroll-viewport>
```

**Performance: 99.85% DOM reduction**

---

### CDK A11y (Accessibility)
**Q: What accessibility features does CDK provide?**
**A:**
- **FocusTrap**: Trap focus within element
- **LiveAnnouncer**: Screen reader announcements
- **A11yModule**: Focus indicators, keyboard navigation

```typescript
constructor(private liveAnnouncer: LiveAnnouncer) {}

announce() {
  this.liveAnnouncer.announce('Data loaded', 'polite');
}
```

---

### CDK Observers
**Q: What are CDK Observers?**
**A:** Utilities for observing DOM changes:
- **IntersectionObserver**: Element visibility
- **MutationObserver**: DOM mutations
- **ResizeObserver**: Element size changes

---

### Angular Material Theming and Customization
**Q: How do you customize Material theme?**
**A:** Define custom color palette with `define-palette()` and theme with `define-light-theme()`.

```scss
@use '@angular/material' as mat;

$custom-primary: mat.define-palette(mat.$indigo-palette);
$custom-accent: mat.define-palette(mat.$pink-palette);

$custom-theme: mat.define-light-theme((
  color: (primary: $custom-primary, accent: $custom-accent)
));

@include mat.all-component-themes($custom-theme);
```

---

## Tooling & Ecosystem

### Angular CLI Commands
**Q: What are essential CLI commands?**
**A:**

```bash
ng new my-app              # Create new app
ng serve                   # Dev server
ng build                   # Production build
ng build --configuration=production  # AOT, minification
ng test                    # Run unit tests
ng e2e                     # Run E2E tests
ng generate component my-component  # Generate component
ng g c my-component --standalone  # Standalone component
ng add @angular/material   # Add package
ng update @angular/core    # Update Angular
ng deploy                  # Deploy to hosting
```

---

### AOT Compilation
**Q: What does AOT do?**
**A:** Compiles templates at build time instead of runtime. Smaller bundles, faster rendering, catches errors early.

```bash
ng build --aot=true  # Default for production
```

---

### TypeScript Decorators
**Q: What are decorators?**
**A:** Special syntax for adding metadata to classes, properties, methods. Core of Angular's DI and component system.

```typescript
@Component()      // Class decorator
class MyComponent {
  @Input() data;  // Property decorator
  @HostListener('click') onClick() {}  // Method decorator
}
```

---

### Linting (@angular-eslint)
**Q: What is @angular-eslint?**
**A:** ESLint plugin for Angular, replaced TSLint.

```bash
ng add @angular-eslint/schematics
ng lint
```

---

### Nx / Monorepo Management
**Q: What is Nx?**
**A:** Smart monorepo tool for Angular. Workspace management, code generation, affected builds, caching.

```bash
npx create-nx-workspace@latest
nx generate @nrwl/angular:app my-app
nx affected:build  # Build only affected projects
```

---

### SSR (@angular/platform-server)
**Q: How do you enable SSR?**
**A:**

```bash
ng add @angular/ssr
npm run build:ssr
npm run serve:ssr
```

---

### PWA (@angular/pwa)
**Q: How do you add PWA support?**
**A:**

```bash
ng add @angular/pwa
```

Adds: Service worker, manifest, icons, offline support

---

### Service Workers
**Q: What do service workers do?**
**A:** Cache assets, enable offline mode, push notifications, background sync.

```typescript
import { SwUpdate } from '@angular/service-worker';

constructor(updates: SwUpdate) {
  updates.versionUpdates.subscribe(event => {
    if (event.type === 'VERSION_READY') {
      updates.activateUpdate().then(() => location.reload());
    }
  });
}
```

---

### Environments Configuration
**Q: How do you configure environments?**
**A:**

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000'
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com'
};

// Usage
import { environment } from '../environments/environment';
```

---

### Polyfills
**Q: What are polyfills?**
**A:** Code that provides modern functionality in older browsers. Angular includes needed polyfills in `polyfills.ts`.

---

## Advanced / Architecture

### Monorepo Architecture with Nx
**Q: What is monorepo architecture?**
**A:** Single repository containing multiple projects (apps, libraries) with shared dependencies and tooling.

**Benefits:** Code sharing, consistent tooling, atomic changes, easier refactoring

---

### Micro Frontends with Module Federation
**Q: What is Module Federation?**
**A:** Load separately deployed Angular apps at runtime. Enables micro frontends.

```typescript
// Host app loads remote app
const remoteComponent = loadRemoteModule({
  remoteName: 'app2',
  exposedModule: './Component'
});
```

---

### Shared Libraries
**Q: How do you create shared libraries?**
**A:**

```bash
nx generate @nrwl/angular:library shared-ui
nx generate @nrwl/angular:library shared-data-access
```

---

### Feature Isolation
**Q: What is feature isolation?**
**A:** Each feature module is self-contained with own components, services, routing. Enables lazy loading and team autonomy.

---

### Dependency Boundaries
**Q: What are dependency boundaries?**
**A:** Rules about what can import what. Enforced by Nx.

```json
{
  "depConstraints": [
    {
      "sourceTag": "type:feature",
      "onlyDependOnLibsWithTags": ["type:ui", "type:data-access"]
    }
  ]
}
```

---

### API Layer Design
**Q: How do you design API layer?**
**A:** Separate data services from UI. Use DTOs, interceptors, error handling.

```typescript
// Data service
@Injectable({ providedIn: 'root' })
export class ApiService {
  get<T>(url: string): Observable<T> {
    return this.http.get<T>(url);
  }
}

// Feature service
@Injectable()
export class UserService {
  constructor(private api: ApiService) {}
  
  getUsers(): Observable<User[]> {
    return this.api.get<UserDto[]>('/users').pipe(
      map(dtos => dtos.map(dto => this.mapToUser(dto)))
    );
  }
}
```

---

### Error Handling & Global ErrorHandler
**Q: How do you implement global error handling?**
**A:** Implement `ErrorHandler` interface.

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  handleError(error: Error): void {
    console.error('Global error:', error);
    this.logger.logError(error);
    this.notifier.showError('An error occurred');
  }
}

// Provide
{ provide: ErrorHandler, useClass: GlobalErrorHandler }
```

---

### Feature Flags
**Q: How do you implement feature flags?**
**A:**

```typescript
@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  private flags = {
    newDashboard: false,
    betaFeatures: true
  };
  
  isEnabled(feature: string): boolean {
    return this.flags[feature] || false;
  }
}

// Usage
@if (featureFlags.isEnabled('newDashboard')) {
  <app-new-dashboard />
}
```

---

### Configurable Modules
**Q: What is forRoot/forChild pattern?**
**A:** Configure module with options.

```typescript
@NgModule({})
export class MyModule {
  static forRoot(config: Config): ModuleWithProviders<MyModule> {
    return {
      ngModule: MyModule,
      providers: [{ provide: CONFIG, useValue: config }]
    };
  }
}

// Usage
imports: [MyModule.forRoot({ apiUrl: 'http://...' })]
```

---

## Common Interview Trap Keywords

### Ivy
**Q: What is Ivy in one sentence?**
**A:** Angular's modern compilation and rendering engine that produces smaller bundles and enables better debugging.

---

### Renderer2
**Q: What is Renderer2?**
**A:** Angular's abstraction for DOM manipulation that works across platforms (browser, server, mobile).

```typescript
constructor(private renderer: Renderer2, private el: ElementRef) {}

ngOnInit() {
  this.renderer.setStyle(this.el.nativeElement, 'color', 'red');
  this.renderer.addClass(this.el.nativeElement, 'active');
}
```

---

### TemplateRef
**Q: What is TemplateRef?**
**A:** Reference to `<ng-template>` that can be rendered programmatically.

---

### ViewContainerRef
**Q: What is ViewContainerRef?**
**A:** Container where views can be attached dynamically (for dynamic component creation).

---

### InjectionToken
**Q: What is InjectionToken in one sentence?**
**A:** Type-safe token for injecting non-class dependencies (strings, objects, functions).

---

### markDirty()
**Q: What does markDirty do?**
**A:** Marks component as dirty and schedules change detection (used in reactive forms).

---

### signal vs observable
**Q: Key difference in one sentence?**
**A:** Signals are pull-based with automatic dependency tracking (no subscriptions), Observables are push-based with manual subscriptions.

---

### hydration mismatch
**Q: What causes hydration mismatch?**
**A:** Server-rendered HTML differs from client-rendered HTML (different data, conditionals, or random values).

**Fix:** Ensure consistent data, use `isPlatformBrowser()` for browser-only code.

---

### zone.js patching
**Q: What does Zone.js patch?**
**A:** Monkey-patches ~40 browser APIs (setTimeout, addEventListener, Promise, XHR, etc.) to track async operations and trigger change detection.

---

### control flow blocks
**Q: What are control flow blocks?**
**A:** New built-in syntax for conditionals and loops: `@if`, `@for`, `@switch` (Angular 17+).

---

### control value accessor
**Q: What is ControlValueAccessor in one sentence?**
**A:** Interface that bridges custom components with Angular forms (enables `ngModel` and `formControl` on custom inputs).

---

### service scope
**Q: What are the three service scopes?**
**A:** `root` (singleton), `any` (per lazy module), `platform` (across apps).

---

### multi-provider
**Q: What is multi-provider in one sentence?**
**A:** Allows multiple providers for same token, returning array of instances (HTTP interceptors, validators).

---

### async pipe unsubscribe behavior
**Q: What does async pipe do automatically?**
**A:** Subscribes when component initializes, unsubscribes when component destroys. Prevents memory leaks.

---

### event delegation
**Q: What is event delegation?**
**A:** Attaching single event listener to parent instead of multiple listeners to children. Angular uses this internally for performance.

---

### memory profiling
**Q: How do you profile memory in Angular?**
**A:** Chrome DevTools → Memory tab → Take heap snapshots → Compare before/after navigating to component multiple times → Look for detached DOM nodes and growing objects.

📚 **[Memory Leaks Deep Dive →](./debugging-memory-leaks.md)**

---

## Quick Reference Card

### When to Use What

**Change Detection:**
- Default: Simple components, frequent updates
- OnPush: List items, performance-critical components
- Detached: Fine-grained control, manual updates

**State Management:**
- Component state: Local component properties
- Service state: Shared across related components
- RxJS: Async data streams, complex transformations
- NgRx: Large apps, time-travel debugging, DevTools
- Signals: Simple state, fine-grained reactivity

**Forms:**
- Template-driven: Simple forms, rapid prototyping
- Reactive: Complex forms, dynamic validation, testing

**Routing:**
- Lazy loading: Large features (reduce initial bundle)
- Preloading: Likely-to-be-visited routes
- Guards: Authentication, authorization, unsaved changes
- Resolvers: Pre-fetch required data

**Testing:**
- Unit tests: Components, services, pipes (60%)
- Integration tests: Feature workflows (30%)
- E2E tests: Critical user journeys (10%)

---

## Performance Checklist

✅ Use `OnPush` change detection
✅ Implement `trackBy` in `*ngFor`
✅ Use pure pipes for transformations
✅ Lazy load feature modules
✅ Use CDK Virtual Scroll for large lists (100+ items)
✅ Run heavy operations outside Angular zone
✅ Unsubscribe from observables in `ngOnDestroy`
✅ Use `async` pipe instead of manual subscriptions
✅ Memoize expensive calculations
✅ Enable AOT compilation (production)
✅ Use `shareReplay()` for shared HTTP requests
✅ Implement code splitting
✅ Use deferred loading for below-the-fold content
✅ Enable SSR with hydration for public sites
✅ Profile with Chrome DevTools Performance tab

---

## Memory Leak Checklist

✅ Unsubscribe RxJS subscriptions in `ngOnDestroy`
✅ Remove event listeners (window, document, DOM)
✅ Clear timers (setInterval, setTimeout)
✅ Close WebSocket/SSE connections
✅ Destroy third-party libraries (charts, maps)
✅ Clean up service workers
✅ Avoid storing component references in services
✅ Limit cache sizes (implement LRU if needed)
✅ Use `takeUntil` pattern for multiple subscriptions
✅ Prefer `async` pipe over manual subscriptions

📚 **[Memory Leaks Deep Dive →](./debugging-memory-leaks.md)**

---

## Production Readiness Checklist

✅ AOT compilation enabled
✅ Production environment config
✅ Lazy loading implemented
✅ OnPush change detection where applicable
✅ Error handling (global ErrorHandler)
✅ HTTP interceptors (auth, errors, caching)
✅ Loading states for async operations
✅ Form validation (user-friendly messages)
✅ Accessibility (ARIA, keyboard navigation)
✅ SEO (meta tags, SSR if needed)
✅ Security (sanitization, CSP, HTTPS)
✅ Monitoring (error tracking, analytics)
✅ Performance budgets (bundle size limits)
✅ Test coverage (80%+ for critical paths)
✅ Documentation (README, API docs)

---

## Further Reading

📚 **Deep Dive Articles:**
- [Lifecycle Hooks Complete Guide](./lifecycle-hooks.md)
- [Change Detection Internals](./change-detection.md)
- [Dependency Injection Deep Dive](./dependency-injection.md)
- [RxJS Operators Masterclass](./rxjs-operators.md)
- [Content Projection Explained](./content-projection.md)
- [Performance Optimization Guide](./performance-optimization.md)
- [NgRx State Management](./ngrx-state-management.md)
- [Testing Strategy](./testing-strategy.md)
- [Memory Leak Debugging](./debugging-memory-leaks.md)
- [Modern Angular Features](./modern-angular-features.md)
- [Reactive Forms Guide](./reactive-forms.md)
- [Routing Deep Dive](./routing.md)
- [RxJS State Management](./rxjs-state.md)
- [Constructor vs ngOnInit](./constructor-vs-ngoninit.md)

📚 **[Back to Main README](./README.md)**

---

## Credits

This rapid-fire guide covers Angular concepts from fundamentals to advanced topics, with emphasis on modern best practices (Angular 16-18). For interview preparation, focus on understanding the "why" behind each concept, not just memorizing definitions.

**Last Updated:** 2025 (Angular 18+)

