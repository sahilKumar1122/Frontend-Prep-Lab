# Modern Angular Features - Deep Dive

## Table of Contents

### Part 1: Angular Signals
- [What Signals Are - Under the Hood](#what-signals-are---under-the-hood)
- [signal(), computed(), and effect() Internals](#signal-computed-and-effect-internals)
- [What Problem Signals Solve](#what-problem-signals-solve)
- [Signals vs Observables - Deep Comparison](#signals-vs-observables---deep-comparison)
- [Mixing Signals and RxJS](#mixing-signals-and-rxjs)

### Part 2: Standalone Components
- [What They Are - Deep Dive](#what-they-are---deep-dive)
- [Compiler Dependency Resolution](#compiler-dependency-resolution)
- [Routing and Lazy Loading](#routing-and-lazy-loading)
- [Migration Decision Framework](#migration-decision-framework)

### Part 3: Zoneless Angular
- [What Zone.js Does Today](#what-zonejs-does-today)
- [Zoneless Change Detection Model](#zoneless-change-detection-model)
- [runOutsideAngular() vs Zoneless](#runoutsideangular-vs-zoneless)
- [Trade-offs of Going Zoneless](#trade-offs-of-going-zoneless)

### Part 4: Deferred Loading & Hydration
- [Deferred Loading (Angular 17+)](#deferred-loading-angular-17)
- [Server-Side Hydration](#server-side-hydration)
- [Debugging Hydration Mismatches](#debugging-hydration-mismatches)

### Part 5: Real-World Architecture
- [Dashboard with Real-Time Widgets](#dashboard-with-real-time-widgets)
- [Hybrid System Architecture](#hybrid-system-architecture)

### Part 6: Production Decision
- [New Enterprise Angular 18 App](#new-enterprise-angular-18-app)
- [Technical Justification](#technical-justification)
- [Team Guidelines](#team-guidelines)
- [Rollout Plan](#rollout-plan)

[← Back to Fundamentals](./fundamentals.md)

---

## Angular Signals

### Question: Explain modern Angular features (signals, standalone components, zoneless CD, deferred loading, hydration) with internal mechanics, comparisons, migration reasoning, and production-level architectural decisions.

**Answer:**

Angular has fundamentally evolved. These aren't just "new features" - they represent architectural shifts in how Angular handles reactivity, modularity, and performance. Let's start with the most fundamental change: Signals.

---

### **What Signals Are - Under the Hood**

```typescript
// Signal is NOT just a wrapper around a value
// It's a reactive primitive with dependency tracking

// Simplified internal implementation
class Signal<T> {
  private _value: T;
  private _version = 0;
  private _subscribers = new Set<Computation>();
  
  constructor(initialValue: T) {
    this._value = initialValue;
  }
  
  // Reading a signal tracks dependencies
  get(): T {
    // If currently executing a computation/effect,
    // add this signal as a dependency
    const currentComputation = CURRENT_COMPUTATION;
    if (currentComputation) {
      this._subscribers.add(currentComputation);
      currentComputation.addDependency(this);
    }
    
    return this._value;
  }
  
  // Writing a signal notifies dependents
  set(newValue: T): void {
    if (this._value !== newValue) {
      this._value = newValue;
      this._version++;
      
      // Notify all subscribers
      this._subscribers.forEach(computation => {
        computation.markDirty();
      });
      
      // Schedule change detection
      scheduleChangeDetection();
    }
  }
  
  update(updateFn: (value: T) => T): void {
    this.set(updateFn(this._value));
  }
}

// Computed signal - automatically tracks dependencies
class ComputedSignal<T> extends Signal<T> {
  private _computation: () => T;
  private _dirty = true;
  private _dependencies = new Set<Signal<any>>();
  
  constructor(computation: () => T) {
    super(undefined as T);
    this._computation = computation;
  }
  
  get(): T {
    if (this._dirty) {
      // Set as current computation
      const previousComputation = CURRENT_COMPUTATION;
      CURRENT_COMPUTATION = this;
      
      // Clear old dependencies
      this._dependencies.forEach(dep => {
        dep._subscribers.delete(this);
      });
      this._dependencies.clear();
      
      // Execute computation - automatically tracks dependencies
      this._value = this._computation();
      this._dirty = false;
      
      // Restore previous computation
      CURRENT_COMPUTATION = previousComputation;
    }
    
    return this._value;
  }
  
  markDirty(): void {
    if (!this._dirty) {
      this._dirty = true;
      // Propagate to subscribers
      this._subscribers.forEach(computation => {
        computation.markDirty();
      });
    }
  }
}

// Effect - runs when dependencies change
class Effect {
  private _computation: () => void;
  private _dependencies = new Set<Signal<any>>();
  
  constructor(computation: () => void) {
    this._computation = computation;
    this.run();
  }
  
  run(): void {
    const previousComputation = CURRENT_COMPUTATION;
    CURRENT_COMPUTATION = this;
    
    // Clear old dependencies
    this._dependencies.forEach(dep => {
      dep._subscribers.delete(this);
    });
    this._dependencies.clear();
    
    // Run effect - tracks dependencies
    this._computation();
    
    CURRENT_COMPUTATION = previousComputation;
  }
  
  markDirty(): void {
    // Re-run effect
    this.run();
  }
}

// Global tracking context
let CURRENT_COMPUTATION: Computation | null = null;
```

**Key Difference from Observables:**

```typescript
// Observable: Push-based (producer pushes values)
const value$ = new BehaviorSubject(0);

// To get value, must subscribe
value$.subscribe(v => {
  console.log(v); // Pushed to subscriber
});

// To update
value$.next(1); // Pushes to all subscribers

// Observable chain execution:
// Producer → Observable → Operators → Subscriber
// Subscriber pulls data through the chain

// Signal: Pull-based (consumer pulls values)
const value = signal(0);

// To get value, just call it
console.log(value()); // Pulled by consumer

// To update
value.set(1); // Notifies dependents, but they pull when ready

// Signal execution:
// Consumer reads signal → Automatic dependency tracking
// Signal changes → Marks dependents dirty
// Next render → Dependents pull new values

// Critical architectural difference:
{
  observables: {
    model: "Push",
    execution: "Eager (pushes immediately)",
    subscriptions: "Manual subscribe/unsubscribe",
    memoryLeaks: "Possible if not unsubscribed",
    changeDetection: "Async pipe calls markForCheck()"
  },
  
  signals: {
    model: "Pull",
    execution: "Lazy (pulls when needed)",
    subscriptions: "Automatic dependency tracking",
    memoryLeaks: "Not possible (no subscriptions)",
    changeDetection: "Built-in (no markForCheck needed)"
  }
}
```

### **signal(), computed(), and effect() Internals**

```typescript
// Real usage
@Component({
  template: `
    <div>Count: {{ count() }}</div>
    <div>Double: {{ double() }}</div>
    <button (click)="increment()">Increment</button>
  `
})
export class CounterComponent {
  // signal - writable reactive value
  count = signal(0);
  
  // computed - derived reactive value (automatically tracks dependencies)
  double = computed(() => this.count() * 2);
  
  constructor() {
    // effect - side effect that runs when dependencies change
    effect(() => {
      console.log('Count changed to:', this.count());
      // Automatically re-runs when count changes
    });
  }
  
  increment(): void {
    this.count.update(c => c + 1);
    // Automatically:
    // 1. Updates count signal
    // 2. Marks double as dirty
    // 3. Schedules effect to run
    // 4. Schedules change detection
    // 5. Template reads double() - recomputes
  }
}

// What happens step by step:

// Step 1: Template renders, reads count()
{
  action: "Read count()",
  internal: [
    "count.get() is called",
    "Template is current computation",
    "count._subscribers.add(template)",
    "Returns count._value"
  ]
}

// Step 2: Template reads double()
{
  action: "Read double()",
  internal: [
    "double.get() is called",
    "double is dirty (not computed yet)",
    "Set double as CURRENT_COMPUTATION",
    "Execute computation: () => this.count() * 2",
    "count.get() is called during computation",
    "count._subscribers.add(double)",
    "double._dependencies.add(count)",
    "Computation returns 0",
    "double._dirty = false",
    "double._subscribers.add(template)",
    "Returns 0"
  ]
}

// Step 3: User clicks button, increment() called
{
  action: "count.update(c => c + 1)",
  internal: [
    "count.set(1) is called",
    "count._value changes: 0 → 1",
    "count._version++",
    "Notify subscribers:",
    "  - double.markDirty()",
    "    - double._dirty = true",
    "    - double notifies template",
    "  - effect.markDirty()",
    "    - effect.run()",
    "    - console.log('Count changed to: 1')",
    "scheduleChangeDetection()",
    "  - Angular schedules CD for next microtask"
  ]
}

// Step 4: Change detection runs
{
  action: "Change Detection",
  internal: [
    "Template checks: need to update?",
    "Template reads count()",
    "  - Returns 1 (changed from 0)",
    "  - Mark DOM update needed",
    "Template reads double()",
    "  - double is dirty",
    "  - Recompute: count() * 2",
    "  - count.get() returns 1",
    "  - Result: 2",
    "  - double._dirty = false",
    "  - Returns 2 (changed from 0)",
    "  - Mark DOM update needed",
    "Update DOM:",
    "  - Count: 0 → 1",
    "  - Double: 0 → 2"
  ]
}
```

### **What Problem Signals Solve**

```typescript
// Problem 1: Zone.js overhead

// With Zone.js + Observables:
@Component({
  template: `<div>{{ count$ | async }}</div>`
})
export class OldWay {
  count$ = new BehaviorSubject(0);
  
  increment(): void {
    this.count$.next(this.count$.value + 1);
    // Zone.js:
    // 1. Patches button click
    // 2. Runs ApplicationRef.tick()
    // 3. Checks ENTIRE component tree
    // 4. Async pipe gets new value
    // 5. Calls markForCheck()
    // 6. CD runs again for this component
    // Result: Potentially 2 CD cycles
  }
}

// With Signals (no Zone.js needed):
@Component({
  template: `<div>{{ count() }}</div>`
})
export class NewWay {
  count = signal(0);
  
  increment(): void {
    this.count.update(c => c + 1);
    // Signals:
    // 1. Updates signal
    // 2. Marks template as dirty
    // 3. Schedules targeted CD
    // 4. Only updates this component
    // Result: 1 targeted CD cycle
  }
}

// Problem 2: Manual subscription management

// With Observables:
@Component({})
export class OldWay implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  count = 0;
  
  constructor(private service: DataService) {}
  
  ngOnInit(): void {
    this.service.count$
      .pipe(takeUntil(this.destroy$))
      .subscribe(count => {
        this.count = count;
        // Manual subscription
        // Manual unsubscribe needed
        // Memory leak if forgotten
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// With Signals:
@Component({})
export class NewWay {
  count = this.service.count;
  // No subscription needed
  // No unsubscribe needed
  // No memory leaks possible
  
  constructor(private service: DataService) {}
}

// Problem 3: Change detection noise

// With Observables + OnPush:
@Component({
  template: `
    <div>A: {{ a$ | async }}</div>
    <div>B: {{ b$ | async }}</div>
    <div>C: {{ c$ | async }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OldWay {
  a$ = this.service.a$;
  b$ = this.service.b$;
  c$ = this.service.c$;
  
  // If 'a' changes:
  // - a$ emits
  // - async pipe calls markForCheck()
  // - ENTIRE component re-evaluated
  // - All three async pipes execute
  // - Even though b and c didn't change
}

// With Signals:
@Component({
  template: `
    <div>A: {{ a() }}</div>
    <div>B: {{ b() }}</div>
    <div>C: {{ c() }}</div>
  `
})
export class NewWay {
  a = this.service.a;
  b = this.service.b;
  c = this.service.c;
  
  // If 'a' changes:
  // - a signal notifies
  // - Only {{ a() }} DOM node updates
  // - b() and c() not even called
  // - Fine-grained reactivity
}
```

### **Signals vs Observables - Deep Comparison**

```typescript
// Architecture comparison

interface Comparison {
  observables: {
    paradigm: "Functional Reactive Programming (FRP)",
    model: "Push-based stream processing",
    composition: "Operators (map, filter, switchMap, etc.)",
    timeModel: "Explicit time dimension (over time)",
    asyncSupport: "Native (HTTP, WebSocket, intervals)",
    errorHandling: "Built-in (catchError, retry)",
    backpressure: "Supported (buffer, throttle, debounce)",
    coldVsHot: "Both (separate concern)",
    multicast: "Supported (share, shareReplay)",
    completion: "Can complete",
    cancellation: "Via unsubscribe",
    
    bestFor: [
      "HTTP requests",
      "WebSocket streams",
      "User input events (debounced)",
      "Complex async flows",
      "Event sourcing",
      "Time-based operations"
    ]
  },
  
  signals: {
    paradigm: "Synchronous reactive state",
    model: "Pull-based value access",
    composition: "computed() for derivations",
    timeModel: "Current value (point in time)",
    asyncSupport: "None (must wrap in signal)",
    errorHandling: "Try/catch in computed",
    backpressure: "N/A (no streams)",
    coldVsHot: "Always 'hot' (current value)",
    multicast: "Automatic (all dependents notified)",
    completion: "Never completes",
    cancellation: "Automatic (dependency tracking)",
    
    bestFor: [
      "Component state",
      "Derived state (computed)",
      "Form state",
      "UI state",
      "Synchronous updates",
      "Fine-grained reactivity"
    ]
  }
}

// Practical comparison:

// Scenario 1: HTTP Request
{
  // ✅ Observable - Natural fit
  data$ = this.http.get('/api/data').pipe(
    retry(3),
    catchError(err => of(null)),
    shareReplay(1)
  );
  
  // ❌ Signal - Awkward
  data = signal(null);
  
  ngOnInit(): void {
    this.http.get('/api/data').subscribe(
      data => this.data.set(data),
      err => this.data.set(null)
    );
  }
}

// Scenario 2: Derived State
{
  // ❌ Observable - Verbose
  firstName$ = new BehaviorSubject('John');
  lastName$ = new BehaviorSubject('Doe');
  fullName$ = combineLatest([
    this.firstName$,
    this.lastName$
  ]).pipe(
    map(([first, last]) => `${first} ${last}`)
  );
  
  // ✅ Signal - Natural fit
  firstName = signal('John');
  lastName = signal('Doe');
  fullName = computed(() => 
    `${this.firstName()} ${this.lastName()}`
  );
}

// Scenario 3: User Input Debouncing
{
  // ✅ Observable - Natural fit
  searchQuery$ = new Subject<string>();
  results$ = this.searchQuery$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(query => this.api.search(query))
  );
  
  // ❌ Signal - Need RxJS anyway
  searchQuery = signal('');
  results = signal([]);
  
  // Still need Observable for debounce
  private searchQuery$ = toObservable(this.searchQuery);
  
  ngOnInit(): void {
    this.searchQuery$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(query => this.api.search(query))
    ).subscribe(results => this.results.set(results));
  }
}
```

### **Mixing Signals and RxJS**

```typescript
// Bridge utilities (Angular provides these)

// 1. toObservable - Signal to Observable
@Component({})
export class SignalToObservable {
  count = signal(0);
  
  // Convert signal to observable
  count$ = toObservable(this.count);
  
  ngOnInit(): void {
    // Now can use RxJS operators
    this.count$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      map(c => c * 2)
    ).subscribe(doubled => {
      console.log('Doubled:', doubled);
    });
  }
}

// 2. toSignal - Observable to Signal
@Component({})
export class ObservableToSignal {
  // HTTP observable
  private data$ = this.http.get<Data>('/api/data');
  
  // Convert to signal
  data = toSignal(this.data$, { initialValue: null });
  
  // Now can use in template without async pipe
  // template: `<div>{{ data()?.name }}</div>`
}

// 3. Hybrid Architecture (Best Practice)
@Component({
  template: `
    <div>
      <!-- Signals for UI state -->
      <div>Count: {{ count() }}</div>
      <div>Double: {{ double() }}</div>
      
      <!-- Observables for async data -->
      <div>User: {{ user$ | async | json }}</div>
      <div>Search: {{ searchResults() | json }}</div>
    </div>
    
    <input [formControl]="searchControl" />
  `
})
export class HybridComponent implements OnInit {
  // UI state - Signals
  count = signal(0);
  double = computed(() => this.count() * 2);
  
  // Async data - Observables
  user$ = this.userService.getCurrentUser();
  
  // Hybrid: Search with debounce
  searchControl = new FormControl('');
  private searchQuery$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(query => this.api.search(query))
  );
  
  // Convert final result to signal
  searchResults = toSignal(this.searchQuery$, { initialValue: [] });
  
  ngOnInit(): void {
    // Use observables for complex async flows
    // Convert to signals for template consumption
  }
}

// Guidelines for mixing:
{
  useSignals: [
    "Component state (forms, UI, toggles)",
    "Derived state (computed values)",
    "Synchronous updates",
    "Values read in templates"
  ],
  
  useObservables: [
    "HTTP requests",
    "WebSockets",
    "Complex async flows",
    "Time-based operations (debounce, throttle)",
    "Event streams that can complete"
  ],
  
  bridge: [
    "Use toSignal() to convert observables to signals for templates",
    "Use toObservable() when you need RxJS operators on signals",
    "Keep async logic in observables",
    "Expose final state as signals"
  ]
}
```

---

## Standalone Components

### **What They Are - Deep Dive**

```typescript
// Traditional NgModule-based component
@Component({
  selector: 'app-user-profile',
  template: `<div>{{ user.name }}</div>`
})
export class UserProfileComponent {
  @Input() user!: User;
}

@NgModule({
  declarations: [UserProfileComponent],
  imports: [CommonModule, FormsModule],
  exports: [UserProfileComponent]
})
export class UserModule {}

// Standalone component
@Component({
  selector: 'app-user-profile',
  standalone: true,  // ← Key difference
  imports: [CommonModule, FormsModule],  // ← Direct imports
  template: `<div>{{ user.name }}</div>`
})
export class UserProfileComponent {
  @Input() user!: User;
}

// No NgModule needed!
```

### **Compiler Dependency Resolution**

```typescript
// How Angular resolves dependencies:

// NgModule-based:
{
  compilationContext: "NgModule",
  
  resolutionFlow: [
    "1. Component declares dependencies via NgModule",
    "2. NgModule imports other modules",
    "3. Compiler builds dependency graph from module imports",
    "4. Component gets access to all declarations/exports in scope",
    "5. Implicit dependency tree (hard to track)"
  ],
  
  example: {
    component: "UserProfileComponent",
    module: "UserModule",
    availableDependencies: [
      "All declarations in UserModule",
      "All exports from imported modules (CommonModule, FormsModule)",
      "All exports from parent modules (implicit)"
    ],
    problem: "Can't tell what component actually uses"
  }
}

// Standalone:
{
  compilationContext: "Component itself",
  
  resolutionFlow: [
    "1. Component explicitly imports dependencies",
    "2. Compiler reads component's imports array",
    "3. Builds minimal dependency graph",
    "4. Component gets only what it imports",
    "5. Explicit dependency tree (clear and traceable)"
  ],
  
  example: {
    component: "UserProfileComponent",
    imports: [CommonModule, FormsModule],
    availableDependencies: [
      "Only CommonModule exports",
      "Only FormsModule exports",
      "Nothing else"
    ],
    benefit: "Clear what component depends on"
  }
}

// Internal compilation difference:

// NgModule compilation:
class NgModuleCompiler {
  compile(module: Type<any>): NgModuleFactory {
    // 1. Collect all declarations
    const declarations = module.declarations;
    
    // 2. Collect all imports (recursive)
    const imports = this.collectImports(module.imports);
    
    // 3. Build compilation scope
    const scope = {
      declarations: [...declarations],
      exports: [...this.collectExports(imports)]
    };
    
    // 4. Each component gets entire scope
    declarations.forEach(component => {
      this.compileComponent(component, scope);
    });
  }
}

// Standalone compilation:
class StandaloneCompiler {
  compile(component: Type<any>): ComponentFactory {
    // 1. Read component's imports directly
    const imports = component.imports || [];
    
    // 2. Build minimal scope (only what's imported)
    const scope = {
      exports: this.collectExports(imports)
    };
    
    // 3. Component gets only its imports
    this.compileComponent(component, scope);
    
    // Result: Smaller scope, clearer dependencies
  }
}
```

### **Routing and Lazy Loading**

```typescript
// Traditional module-based routing
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module')
      .then(m => m.UsersModule)
  }
];

// users.module.ts
@NgModule({
  declarations: [UsersComponent, UserDetailComponent],
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: UsersComponent },
      { path: ':id', component: UserDetailComponent }
    ])
  ]
})
export class UsersModule {}

// Standalone routing (cleaner!)
const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/users.component')
      .then(m => m.UsersComponent)
  },
  {
    path: 'users/:id',
    loadComponent: () => import('./users/user-detail.component')
      .then(m => m.UserDetailComponent)
  }
];

// No module needed!
// Each component loaded independently

// Or with child routes:
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.routes')
      .then(m => m.USERS_ROUTES)
  }
];

// users.routes.ts
export const USERS_ROUTES: Routes = [
  {
    path: '',
    component: UsersComponent
  },
  {
    path: ':id',
    component: UserDetailComponent
  }
];

// Benefits:
{
  traditional: {
    chunkSize: "Module + all components",
    overhead: "Module metadata",
    flexibility: "All or nothing"
  },
  
  standalone: {
    chunkSize: "Only needed component",
    overhead: "Minimal",
    flexibility: "Component-level code splitting"
  }
}
```

### **Migration Decision Framework**

```typescript
// Should you migrate?

interface MigrationDecision {
  migrateIf: {
    newProject: "YES - Start standalone from day 1",
    
    smallApp: "YES - Easy migration, big benefits",
    criteria: [
      "< 50 components",
      "< 10 modules",
      "< 1 week migration time"
    ],
    
    mediumApp: "MAYBE - Incremental migration",
    criteria: [
      "50-200 components",
      "10-30 modules",
      "Can migrate feature by feature",
      "Team has bandwidth"
    ],
    
    largeApp: "CAREFULLY - Incremental migration",
    criteria: [
      "> 200 components",
      "> 30 modules",
      "Complex module dependencies",
      "High regression risk",
      "Limited migration bandwidth"
    ],
    approach: "Migrate new features only, keep old modules"
  },
  
  dontMigrateIf: [
    "App is in maintenance mode",
    "Team unfamiliar with standalone",
    "Near major deadline",
    "Angular version < 15",
    "Too much technical debt already"
  ]
}

// Migration strategy for large app:

// Phase 1: New features only
{
  action: "Write all new features as standalone",
  duration: "Ongoing",
  risk: "Low",
  benefit: "Prevents more NgModules"
}

// Phase 2: Leaf modules (no dependencies)
{
  action: "Convert modules with no dependents",
  examples: ["Utility modules", "Widget modules"],
  duration: "1-2 sprints",
  risk: "Low",
  benefit: "Quick wins, learn migration process"
}

// Phase 3: Feature modules (bottom-up)
{
  action: "Convert feature modules starting from leaves",
  strategy: "Bottom-up (dependencies first)",
  duration: "3-6 months",
  risk: "Medium",
  benefit: "Reduced module complexity"
}

// Phase 4: Core/Shared modules
{
  action: "Convert shared/core last",
  reason: "Highest dependency count",
  duration: "2-4 months",
  risk: "High",
  benefit: "Complete migration"
}

// Our decision for 500-component app:
{
  decision: "Incremental migration",
  reasoning: [
    "New features: 100% standalone (preventing debt)",
    "Old features: Migrate when touching code",
    "Core modules: Keep for now (too risky)",
    "Timeline: 12-18 months to full migration"
  ],
  
  results: {
    after6Months: "30% standalone, 70% NgModule",
    after12Months: "60% standalone, 40% NgModule",
    after18Months: "90% standalone, 10% NgModule",
    benefits: [
      "Clearer dependencies",
      "Faster builds",
      "Smaller bundles",
      "Easier testing"
    ]
  }
}
```

---

## Zoneless Angular

### **What Zone.js Does Today**

```typescript
// Zone.js monkey-patches global APIs

// Before Zone.js:
window.addEventListener('click', () => {
  this.count++;
  // No change detection!
  // UI not updated
});

// With Zone.js:
{
  whatItPatches: [
    "addEventListener/removeEventListener",
    "setTimeout/clearTimeout",
    "setInterval/clearInterval",
    "Promise.then",
    "requestAnimationFrame",
    "XMLHttpRequest",
    "fetch",
    "MutationObserver",
    "... ~40 more APIs"
  ],
  
  howItWorks: [
    "1. Patch browser APIs at app bootstrap",
    "2. Wrap callbacks in Zone context",
    "3. Track when async operations complete",
    "4. Trigger ApplicationRef.tick()",
    "5. Run change detection on entire tree"
  ]
}

// Actual Zone.js patch (simplified):
const originalAddEventListener = EventTarget.prototype.addEventListener;

EventTarget.prototype.addEventListener = function(
  type: string,
  listener: EventListener,
  options?: any
) {
  const wrappedListener = function(...args: any[]) {
    // Enter Angular zone
    Zone.current.run(() => {
      // Run original listener
      listener.apply(this, args);
      
      // After listener completes:
      if (Zone.current === ngZone) {
        // Trigger change detection
        ApplicationRef.tick();
      }
    });
  };
  
  originalAddEventListener.call(this, type, wrappedListener, options);
};

// Cost of Zone.js:
{
  overhead: {
    memory: "~200KB for Zone.js library",
    cpu: "Wrapper execution on every event",
    complexity: "Monkey-patching can cause issues",
    debugging: "Harder to debug (wrapped calls)"
  },
  
  problem: "Triggers CD on EVERY async operation",
  
  example: {
    mouseMove: "60 times per second",
    typing: "10-20 times per second",
    polling: "Every interval tick",
    result: "Excessive CD checks"
  }
}
```

### **Zoneless Change Detection Model**

```typescript
// New model (Angular 17+): No Zone.js needed with Signals

// How it works without Zone.js:

// 1. Signals track dependencies automatically
@Component({
  template: `
    <div>Count: {{ count() }}</div>
    <button (click)="increment()">+</button>
  `
})
export class ZonelessComponent {
  count = signal(0);
  
  increment(): void {
    this.count.update(c => c + 1);
    // Signal notifies template
    // Schedules targeted CD
    // No Zone.js needed!
  }
}

// 2. Event handlers are still wired by Angular
{
  how: [
    "Angular still creates event listeners",
    "But doesn't rely on Zone.js to detect them",
    "Signals provide direct change notification path"
  ]
}

// 3. Manual CD for non-Signal code
@Component({
  template: `<div>{{ value }}</div>`
})
export class ManualCDComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateValue(): void {
    setTimeout(() => {
      this.value = 1;
      
      // Without Zone.js, must manually trigger CD
      this.cdr.markForCheck();
    }, 1000);
  }
}

// Zoneless configuration:
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection()
    // Disables Zone.js
    // Requires Signals or manual CD
  ]
});
```

### **runOutsideAngular() vs Zoneless**

```typescript
// runOutsideAngular - With Zone.js

@Component({})
export class WithZoneComponent {
  constructor(private ngZone: NgZone) {}
  
  startAnimation(): void {
    // Run outside Angular's zone
    this.ngZone.runOutsideAngular(() => {
      const animate = () => {
        // Heavy animation logic
        this.updateCanvas();
        
        // Zone.js won't trigger CD here
        requestAnimationFrame(animate);
      };
      animate();
    });
    
    // Manually trigger CD when needed
    setTimeout(() => {
      this.ngZone.run(() => {
        this.updateUI();
      });
    }, 1000);
  }
}

// Purpose: Opt out of Zone.js for specific code
// Zone.js still loaded and active
// Memory: ~200KB for Zone.js
// Use case: Performance-critical loops

// Zoneless - No Zone.js at all

@Component({})
export class ZonelessComponent {
  // No NgZone injected - doesn't exist!
  
  startAnimation(): void {
    // No runOutsideAngular needed
    const animate = () => {
      // Heavy animation logic
      this.updateCanvas();
      
      // CD won't trigger automatically anyway
      requestAnimationFrame(animate);
    };
    animate();
    
    // Manually trigger CD when needed
    setTimeout(() => {
      this.cdr.markForCheck();
    }, 1000);
  }
  
  constructor(private cdr: ChangeDetectorRef) {}
}

// Purpose: Remove Zone.js entirely
// Zone.js not loaded
// Memory: Save ~200KB
// Use case: Fully Signal-based apps

// Comparison:
{
  withZoneJs: {
    approach: "runOutsideAngular()",
    loaded: "Zone.js library (200KB)",
    default: "Automatic CD on all async",
    optOut: "runOutsideAngular() for specific code",
    manualCD: "ngZone.run() to re-enter"
  },
  
  zoneless: {
    approach: "provideExperimentalZonelessChangeDetection()",
    loaded: "No Zone.js",
    default: "No automatic CD",
    optOut: "N/A (always manual)",
    manualCD: "markForCheck() or Signals"
  }
}
```

### **Trade-offs of Going Zoneless**

```typescript
interface ZonelessTradeoffs {
  benefits: {
    performance: [
      "No Zone.js overhead (~200KB)",
      "No monkey-patching overhead",
      "More predictable performance",
      "Targeted change detection only"
    ],
    
    simplicity: [
      "Fewer magic behaviors",
      "Explicit change detection",
      "Easier debugging (no Zone context)"
    ],
    
    compatibility: [
      "Better for Web Workers",
      "Better for SSR",
      "Better for micro-frontends"
    ]
  },
  
  costs: {
    migration: [
      "Must convert state to Signals",
      "Or add manual markForCheck() everywhere",
      "Third-party libraries may break"
    ],
    
    complexity: [
      "Must understand when CD needs triggering",
      "Easy to forget markForCheck()",
      "Debugging: 'Why didn't UI update?'"
    ],
    
    compatibility: [
      "Some third-party libraries expect Zone.js",
      "NgRx effects may need updates",
      "Custom decorators may break"
    ]
  }
}

// Decision matrix:

const shouldGoZoneless = {
  yes: [
    "New app starting fresh",
    "All state management with Signals",
    "Team comfortable with explicit CD",
    "Performance is critical",
    "No Zone.js dependent libraries"
  ],
  
  no: [
    "Existing large app",
    "Heavy Observable usage",
    "Third-party libraries depend on Zone.js",
    "Team unfamiliar with Signals",
    "Migration risk too high"
  ],
  
  maybe: [
    "Medium-sized app",
    "Can migrate incrementally",
    "Willing to add markForCheck() calls",
    "Performance issues with current approach"
  ]
};

// Our production decision (e-commerce app):
{
  decision: "Not yet zoneless",
  
  reasoning: [
    "Large existing codebase (500+ components)",
    "Heavy RxJS usage (HTTP, WebSockets)",
    "Third-party libraries (charts, maps)",
    "Migration risk vs benefit unclear",
    "Team learning curve"
  ],
  
  approach: [
    "Use Signals for new features",
    "Keep Zone.js for now",
    "Monitor Angular's zoneless maturity",
    "Reevaluate in 6-12 months"
  ]
}
```

---

## Deferred Loading & Hydration

### **Deferred Loading (Angular 17+)**

```typescript
// Traditional lazy loading
const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module')
      .then(m => m.DashboardModule)
  }
];
// Loads on route navigation only

// Deferred loading (new!)
@Component({
  template: `
    <div>
      <h1>Dashboard</h1>
      
      <!-- Load immediately -->
      <app-header />
      
      <!-- Defer loading until visible -->
      @defer (on viewport) {
        <app-heavy-chart [data]="chartData" />
      } @placeholder {
        <div class="skeleton">Loading chart...</div>
      } @loading (minimum 500ms) {
        <div class="spinner"></div>
      } @error {
        <div class="error">Failed to load chart</div>
      }
      
      <!-- Defer loading on interaction -->
      @defer (on interaction) {
        <app-comments />
      } @placeholder {
        <button>Load Comments</button>
      }
      
      <!-- Defer loading on idle -->
      @defer (on idle) {
        <app-analytics-widget />
      }
      
      <!-- Defer loading on timer -->
      @defer (on timer(3s)) {
        <app-ads />
      }
    </div>
  `
})
export class DashboardComponent {}

// Differences:
{
  traditionalLazyLoading: {
    trigger: "Route navigation",
    granularity: "Module or component level",
    api: "Router configuration",
    when: "User navigates",
    bundleStrategy: "Route-based chunks"
  },
  
  deferredLoading: {
    trigger: "Multiple (viewport, interaction, idle, timer, immediate, hover)",
    granularity: "Template block level",
    api: "@defer directive in template",
    when: "Configurable conditions",
    bundleStrategy: "Component-level chunks"
  }
}

// How it works internally:
{
  compilation: [
    "1. Compiler identifies @defer blocks",
    "2. Extracts deferred component into separate chunk",
    "3. Generates loading logic based on trigger",
    "4. Creates intersection observer (viewport)",
    "5. Creates event listeners (interaction)",
    "6. Uses requestIdleCallback (idle)"
  ],
  
  runtime: [
    "1. Render placeholder initially",
    "2. Monitor trigger condition",
    "3. When triggered, dynamically import chunk",
    "4. Show loading state during import",
    "5. Render actual component when ready",
    "6. Handle errors if import fails"
  ]
}

// Real-world example:
@Component({
  template: `
    <!-- Above fold - load immediately -->
    <app-hero />
    <app-featured-products />
    
    <!-- Below fold - defer until visible -->
    @defer (on viewport) {
      <app-product-grid [products]="products" />
    } @placeholder {
      <div class="grid-skeleton"></div>
    }
    
    <!-- Way below - defer until idle -->
    @defer (on idle) {
      <app-newsletter-signup />
      <app-footer />
    }
    
    <!-- Modal - defer until interaction -->
    <button (click)="showModal = true">Open Modal</button>
    @defer (when showModal) {
      <app-modal />
    }
  `
})
export class HomePageComponent {}

// Bundle impact:
{
  before: {
    mainBundle: "500 KB (everything)",
    initialLoad: "500 KB",
    timeToInteractive: "3.2s"
  },
  
  after: {
    mainBundle: "200 KB (above fold only)",
    deferredChunks: [
      "product-grid.chunk.js - 150 KB",
      "newsletter.chunk.js - 100 KB",
      "modal.chunk.js - 50 KB"
    ],
    initialLoad: "200 KB",
    timeToInteractive: "1.5s",
    improvement: "53% faster"
  }
}
```

### **Server-Side Hydration**

```typescript
// Problem SSR solves:

// Traditional CSR (Client-Side Rendering):
{
  flow: [
    "1. Browser downloads HTML (empty <app-root>)",
    "2. Downloads JavaScript bundle",
    "3. Executes JavaScript",
    "4. Renders application",
    "5. Makes API calls",
    "6. Re-renders with data"
  ],
  
  userExperience: {
    timeToFirstByte: "100ms",
    timeToFirstPaint: "2000ms",  // Blank screen
    timeToInteractive: "3500ms",
    seo: "Poor (crawlers see empty page)"
  }
}

// SSR (Server-Side Rendering):
{
  flow: [
    "1. Server renders full HTML",
    "2. Browser displays HTML immediately",
    "3. Downloads JavaScript bundle",
    "4. Hydrates application",
    "5. Application interactive"
  ],
  
  userExperience: {
    timeToFirstByte: "300ms",
    timeToFirstPaint: "400ms",  // Instant content!
    timeToInteractive: "2000ms",
    seo: "Excellent (crawlers see full HTML)"
  }
}

// Hydration problem:

// Traditional hydration (Angular <16):
{
  process: [
    "1. Server renders HTML",
    "2. Client downloads app",
    "3. Client destroys server HTML",  // ⚠️ Flash of content!
    "4. Client re-renders from scratch",
    "5. Application interactive"
  ],
  
  issues: [
    "Flicker during hydration",
    "Wasted work (re-rendering everything)",
    "Slower time to interactive"
  ]
}

// New hydration (Angular 16+):
{
  process: [
    "1. Server renders HTML",
    "2. Server adds hydration metadata",
    "3. Client downloads app",
    "4. Client reuses server HTML",  // ✅ No flash!
    "5. Client attaches event listeners",
    "6. Application interactive"
  ],
  
  benefits: [
    "No flicker",
    "Faster hydration",
    "Lower CPU usage",
    "Better user experience"
  ]
}

// Enable hydration:
// main.ts
import { provideClientHydration } from '@angular/platform-browser';

bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration()
  ]
});

// server.ts
import { ngExpressEngine } from '@nguniversal/express-engine';

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModule
}));

// How it works internally:
{
  server: [
    "1. Render component tree to HTML string",
    "2. Add ngServerContext attributes to DOM nodes",
    "3. Serialize component state to JSON",
    "4. Embed state in HTML as <script> tag",
    "5. Send HTML to client"
  ],
  
  client: [
    "1. Parse HTML (already rendered)",
    "2. Read ngServerContext attributes",
    "3. Match component tree to DOM nodes",
    "4. Reuse existing DOM instead of re-creating",
    "5. Restore component state from embedded JSON",
    "6. Attach event listeners",
    "7. Application interactive"
  ]
}

// Example server HTML:
/*
<app-user-profile ngh="0">
  <div ngh="1">
    <h1 ngh="2">John Doe</h1>
    <p ngh="3">john@example.com</p>
  </div>
</app-user-profile>

<script id="ng-state" type="application/json">
  {
    "0": { "userId": 123 },
    "1": {},
    "2": { "text": "John Doe" },
    "3": { "text": "john@example.com" }
  }
</script>
*/
```

### **Debugging Hydration Mismatches**

```typescript
// Mismatch occurs when server and client HTML differ

// Common causes:

// 1. Browser-only APIs
@Component({
  template: `<div>Width: {{ windowWidth }}</div>`
})
export class BadComponent {
  windowWidth = window.innerWidth;  // ⚠️ Error on server!
  // window is undefined on server
}

// Fix:
export class GoodComponent {
  windowWidth = 0;
  
  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}
  
  ngOnInit(): void {
    if (isPlatformBrowser(this.platformId)) {
      this.windowWidth = window.innerWidth;
    }
  }
}

// 2. Random data
@Component({
  template: `<div>{{ randomId }}</div>`
})
export class BadComponent {
  randomId = Math.random();  // ⚠️ Differs server vs client!
}

// Fix:
export class GoodComponent {
  randomId = '';
  
  ngOnInit(): void {
    // Generate on client only
    if (isPlatformBrowser(this.platformId)) {
      this.randomId = Math.random().toString();
    }
  }
}

// 3. Dates/Timezones
@Component({
  template: `<div>{{ now | date }}</div>`
})
export class BadComponent {
  now = new Date();  // ⚠️ Different timezone server vs client
}

// Fix: Use fixed date or UTC
export class GoodComponent {
  now = new Date('2024-01-01T00:00:00Z');  // Fixed date
}

// 4. Direct DOM manipulation
@Component({})
export class BadComponent implements AfterViewInit {
  @ViewChild('container') container!: ElementRef;
  
  ngAfterViewInit(): void {
    // ⚠️ Server HTML doesn't include this change
    this.container.nativeElement.innerHTML = '<p>Dynamic</p>';
  }
}

// Fix: Use Angular templates
@Component({
  template: `
    <div #container>
      <p *ngIf="showDynamic">Dynamic</p>
    </div>
  `
})
export class GoodComponent {
  showDynamic = true;
}

// Debugging mismatches:

// Enable hydration debugging
bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(
      withDebugHydration()  // Shows detailed errors
    )
  ]
});

// Console errors:
/*
ERROR: NG0500: During hydration Angular expected <div> but found <p>.
  Expected: <div ngh="2">Expected content</div>
  Actual:   <p ngh="2">Actual content</p>
  
  Component: UserProfileComponent
  Template: user-profile.component.html:15
  
  This usually means:
  - Server and client rendered different content
  - Check for browser-only APIs (window, document)
  - Check for random data (Math.random(), Date.now())
  - Check for direct DOM manipulation
*/

// Hydration error handling:
@Component({})
export class AppComponent {
  constructor(private errorHandler: ErrorHandler) {
    // Global hydration error handler
    if (isPlatformBrowser(this.platformId)) {
      window.addEventListener('hydration-error', (event) => {
        console.error('Hydration mismatch:', event);
        
        // Track in analytics
        this.analytics.track('hydration-error', {
          component: event.component,
          expected: event.expected,
          actual: event.actual
        });
      });
    }
  }
}
```

---

## Signals vs Observables - Real Use Cases

### **Dashboard with Real-Time Widgets**

```typescript
// Scenario: Trading dashboard with multiple real-time widgets

@Component({
  template: `
    <div class="dashboard">
      <!-- Widget 1: Current prices (real-time) -->
      <div class="widget">
        <h3>Current Prices</h3>
        <div *ngFor="let price of currentPrices()">
          {{ price.symbol }}: ${{ price.value }}
        </div>
      </div>
      
      <!-- Widget 2: Portfolio value (derived) -->
      <div class="widget">
        <h3>Portfolio Value</h3>
        <div>Total: ${{ portfolioValue() }}</div>
        <div>Change: {{ portfolioChange() }}</div>
      </div>
      
      <!-- Widget 3: Market news (async loading) -->
      <div class="widget">
        <h3>Market News</h3>
        <div *ngFor="let article of news$ | async">
          {{ article.title }}
        </div>
      </div>
      
      <!-- Widget 4: Trade form (UI state) -->
      <div class="widget">
        <h3>Place Trade</h3>
        <form [formGroup]="tradeForm">
          <input formControlName="symbol" />
          <input formControlName="quantity" />
          <div>Total: ${{ tradeTotal() }}</div>
        </form>
      </div>
    </div>
  `
})
export class TradingDashboardComponent implements OnInit {
  // ✅ Signals for real-time data
  private priceUpdates = signal<PriceUpdate[]>([]);
  currentPrices = computed(() => this.priceUpdates());
  
  // ✅ Signals for derived state
  portfolioValue = computed(() => {
    const prices = this.currentPrices();
    return this.calculatePortfolioValue(prices);
  });
  
  portfolioChange = computed(() => {
    const current = this.portfolioValue();
    const previous = this.previousValue();
    return ((current - previous) / previous * 100).toFixed(2) + '%';
  });
  
  private previousValue = signal(10000);
  
  // ✅ Observable for async data loading
  news$ = this.newsService.getMarketNews().pipe(
    retry(3),
    catchError(() => of([])),
    shareReplay(1)
  );
  
  // ✅ Signals for form state
  tradeForm = new FormGroup({
    symbol: new FormControl(''),
    quantity: new FormControl(0)
  });
  
  // Convert form to signal for reactive total
  private symbol = toSignal(
    this.tradeForm.get('symbol')!.valueChanges,
    { initialValue: '' }
  );
  
  private quantity = toSignal(
    this.tradeForm.get('quantity')!.valueChanges,
    { initialValue: 0 }
  );
  
  tradeTotal = computed(() => {
    const symbol = this.symbol();
    const quantity = this.quantity();
    const price = this.getPriceForSymbol(symbol);
    return (price * (quantity || 0)).toFixed(2);
  });
  
  constructor(
    private websocket: WebSocketService,
    private newsService: NewsService
  ) {}
  
  ngOnInit(): void {
    // ✅ Observable for WebSocket (push-based stream)
    this.websocket.priceUpdates$.subscribe(update => {
      // Convert to signal
      this.priceUpdates.update(prices => {
        const index = prices.findIndex(p => p.symbol === update.symbol);
        if (index >= 0) {
          const updated = [...prices];
          updated[index] = update;
          return updated;
        }
        return [...prices, update];
      });
    });
  }
  
  private getPriceForSymbol(symbol: string): number {
    const price = this.currentPrices().find(p => p.symbol === symbol);
    return price?.value || 0;
  }
  
  private calculatePortfolioValue(prices: PriceUpdate[]): number {
    // Calculate based on holdings
    return prices.reduce((total, price) => {
      const holding = this.getHolding(price.symbol);
      return total + (price.value * holding);
    }, 0);
  }
  
  private getHolding(symbol: string): number {
    // Get from portfolio
    return 0;
  }
}

// Architecture decision:
{
  useSignals: {
    realTimeData: "currentPrices",
    reason: "Frequent updates, need fine-grained reactivity",
    benefit: "Only affected DOM nodes update"
  },
  
  useDerivedSignals: {
    computedValues: "portfolioValue, portfolioChange, tradeTotal",
    reason: "Automatically recompute when dependencies change",
    benefit: "No manual recalculation, automatic optimization"
  },
  
  useObservables: {
    asyncData: "news$, websocket.priceUpdates$",
    reason: "Async streams with error handling",
    benefit: "Native retry, catchError, shareReplay"
  },
  
  bridgePattern: {
    convert: "Observable → Signal (toSignal)",
    where: "WebSocket data, form values",
    benefit: "Expose final state as signals, keep async logic in observables"
  }
}
```

### **Hybrid System Architecture**

```typescript
// Best practices for mixing Signals and Observables

// 1. Data Layer - Observables
@Injectable({ providedIn: 'root' })
export class DataService {
  // HTTP - Observable (native fit)
  private api = inject(HttpClient);
  
  getUsers(): Observable<User[]> {
    return this.api.get<User[]>('/api/users').pipe(
      retry(3),
      catchError(err => {
        console.error(err);
        return of([]);
      }),
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
  
  // WebSocket - Observable (stream)
  private ws = inject(WebSocketService);
  
  priceUpdates$ = this.ws.connect('prices').pipe(
    retryWhen(errors =>
      errors.pipe(
        delay(1000),
        take(10)
      )
    )
  );
}

// 2. State Layer - Signals
@Injectable({ providedIn: 'root' })
export class StateService {
  // Writable signals for state
  private _users = signal<User[]>([]);
  private _selectedUserId = signal<number | null>(null);
  
  // Read-only signals exposed
  users = this._users.asReadonly();
  selectedUserId = this._selectedUserId.asReadonly();
  
  // Computed signals for derived state
  selectedUser = computed(() => {
    const id = this._selectedUserId();
    if (!id) return null;
    return this._users().find(u => u.id === id) || null;
  });
  
  // Bridge: Observable → Signal
  constructor(private dataService: DataService) {
    // Load users and convert to signal
    this.dataService.getUsers().subscribe(users => {
      this._users.set(users);
    });
  }
  
  // Actions
  selectUser(id: number): void {
    this._selectedUserId.set(id);
  }
  
  addUser(user: User): void {
    this._users.update(users => [...users, user]);
  }
}

// 3. Component Layer - Consume Signals
@Component({
  template: `
    <div>
      <!-- Signals in template -->
      <div *ngFor="let user of users()">
        {{ user.name }}
      </div>
      
      <div *ngIf="selectedUser() as user">
        <h2>{{ user.name }}</h2>
        <p>{{ user.email }}</p>
      </div>
    </div>
  `
})
export class UserListComponent {
  // Inject state service
  private state = inject(StateService);
  
  // Expose signals
  users = this.state.users;
  selectedUser = this.state.selectedUser;
  
  // Actions
  selectUser(id: number): void {
    this.state.selectUser(id);
  }
}

// 4. Complex async flows - Keep Observable
@Component({})
export class SearchComponent {
  // Form input - FormControl
  searchControl = new FormControl('');
  
  // Search flow - Observable (debounce, switchMap)
  private searchQuery$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(query => (query?.length || 0) > 2)
  );
  
  // Results - Observable → Signal
  searchResults = toSignal(
    this.searchQuery$.pipe(
      switchMap(query => this.api.search(query)),
      catchError(() => of([]))
    ),
    { initialValue: [] }
  );
  
  // Template consumes signal
  // template: `<div *ngFor="let result of searchResults()">...</div>`
}

// Architecture summary:
{
  layers: {
    data: "Observables (HTTP, WebSocket, streams)",
    state: "Signals (application state)",
    ui: "Signals (component state, derived values)"
  },
  
  bridging: {
    dataToState: "Subscribe and set signal",
    stateToUI: "Direct signal consumption",
    complexAsync: "Keep Observable, convert final result to signal"
  },
  
  benefits: [
    "Clear separation of concerns",
    "Observables for what they're good at (async)",
    "Signals for what they're good at (synchronous state)",
    "Best of both worlds"
  ]
}
```

---

## Production Architecture Decision

### **New Enterprise Angular 18 App**

```typescript
// Scenario: Building new enterprise app from scratch
// Requirements:
// - Large team (10+ developers)
// - Complex domain (financial services)
// - Real-time data
// - High performance requirements
// - 5+ year maintenance horizon

interface ArchitectureDecision {
  // Our decision:
  approach: "Progressive modernization",
  
  what: {
    standalone: "YES - 100% standalone components",
    reason: [
      "Starting fresh - no migration cost",
      "Clearer dependencies",
      "Better code splitting",
      "Future-proof (Angular direction)"
    ]
  },
  
  signals: "YES - For state management",
  reason: [
    "Fine-grained reactivity",
    "No subscription leaks",
    "Better performance",
    "Simpler mental model for team"
  ],
  
  observables: "YES - For async operations",
  reason: [
    "HTTP requests (retry, error handling)",
    "WebSockets (reconnection logic)",
    "Complex async flows (debounce, switchMap)",
    "Team already knows RxJS"
  ],
  
  zoneless: "NO - Not yet",
  reason: [
    "Still experimental",
    "Team learning curve too steep",
    "Third-party library compatibility unclear",
    "Can migrate later with Signals already in place"
  ],
  
  hydration: "YES - SSR with hydration",
  reason: [
    "SEO important for marketing pages",
    "Better initial load performance",
    "Progressive enhancement",
    "Mature feature (Angular 16+)"
  ],
  
  deferredLoading: "YES - Aggressively",
  reason: [
    "Many below-fold components",
    "Large bundle size concern",
    "Better initial load",
    "Easy wins with @defer"
  ]
}

// Technical justification:

// 1. Standalone Components
{
  decision: "100% standalone from day 1",
  
  architecture: {
    noModules: "No NgModules except AppModule (required for bootstrap)",
    imports: "All dependencies explicit at component level",
    routing: "Component-based lazy loading",
    testing: "Simpler component tests"
  },
  
  guidelines: {
    newComponent: "Always standalone: true",
    sharedCode: "Utility functions, not shared modules",
    thirdParty: "Import directly in components that need them"
  },
  
  migration: "N/A (new app)",
  
  teamOnboarding: {
    challenge: "Team used to NgModules",
    solution: [
      "1 week training on standalone patterns",
      "Code review guidelines",
      "Templates and generators",
      "Pair programming for first features"
    ],
    timeline: "2-3 sprints to full proficiency"
  }
}

// 2. Signals + Observables Hybrid
{
  decision: "Signals for state, Observables for async",
  
  patterns: {
    componentState: "Signals",
    example: "form state, UI toggles, counters",
    
    derivedState: "Computed Signals",
    example: "totals, filters, transformations",
    
    http: "Observables → Signals",
    example: "toSignal(http.get())",
    
    websocket: "Observables → Signals",
    example: "subscribe and update signal",
    
    userInput: "Observables → Signals",
    example: "debounce then convert to signal",
    
    sideEffects: "Effects",
    example: "logging, analytics, syncing"
  },
  
  guidelines: {
    default: "Use Signals unless you need Observable operators",
    async: "Use Observables, convert final result to Signal",
    neverMix: "Don't create Signal in subscribe callback unnecessarily",
    bridge: "Use toSignal() and toObservable() at boundaries"
  },
  
  codeReview: {
    redFlags: [
      "BehaviorSubject for simple state (use Signal)",
      "combineLatest for derived state (use computed)",
      "Manual subscribe in component (use toSignal or async)",
      "Observable for synchronous state (use Signal)"
    ]
  }
}

// 3. Zone.js (Keep for now)
{
  decision: "Keep Zone.js, reevaluate in 12 months",
  
  reasoning: {
    team: "Learning Signals already big change",
    risk: "Zoneless still experimental",
    compatibility: "Third-party libraries untested",
    timeline: "Can't afford debugging zoneless issues"
  },
  
  future: {
    when: "Angular 19-20 (zoneless stable)",
    trigger: [
      "Zoneless becomes default",
      "Team comfortable with Signals",
      "Third-party libraries compatible",
      "Clear migration path documented"
    ],
    approach: "Gradual opt-in per feature"
  },
  
  optimization: {
    now: "Use runOutsideAngular() for heavy operations",
    example: "Animations, polling, canvas rendering"
  }
}

// 4. SSR with Hydration
{
  decision: "Enable for all routes",
  
  benefits: {
    seo: "Critical for marketing pages",
    performance: "Better FCP and LCP",
    userExperience: "Instant content visibility"
  },
  
  implementation: {
    server: "Node.js with Express + Angular Universal",
    caching: "Redis cache for rendered pages",
    revalidation: "5 minute TTL for dynamic pages",
    staticPages: "Pre-render at build time"
  },
  
  challenges: {
    browserApis: "Use isPlatformBrowser checks",
    thirdParty: "Load client-side only (defer)",
    testing: "Test both server and client paths"
  }
}

// 5. Deferred Loading
{
  decision: "Aggressive use of @defer",
  
  strategy: {
    aboveFold: "Load immediately",
    belowFold: "defer (on viewport)",
    modals: "defer (on interaction)",
    analytics: "defer (on idle)",
    ads: "defer (on timer(3s))"
  },
  
  metrics: {
    target: {
      mainBundle: "< 200 KB",
      fcp: "< 1.5s",
      lcp: "< 2.5s",
      tti: "< 3.5s"
    }
  }
}

// Team Guidelines Document:
{
  title: "Angular 18 Architecture Guidelines",
  
  principles: [
    "1. All components standalone",
    "2. Signals for state, Observables for async",
    "3. Explicit dependencies (no magic)",
    "4. Fine-grained reactivity where possible",
    "5. @defer aggressively for below-fold content"
  ],
  
  patterns: {
    newFeature: [
      "Create standalone component",
      "Use Signals for local state",
      "Use Observables for HTTP/WebSocket",
      "Convert final async result to Signal",
      "Use @defer for heavy components",
      "Add SSR-safe checks for browser APIs"
    ]
  },
  
  codeReview: {
    mustHave: [
      "✅ Component is standalone",
      "✅ No NgModules (except AppModule)",
      "✅ Signals used for state",
      "✅ No observable subscription leaks",
      "✅ isPlatformBrowser checks for browser APIs",
      "✅ @defer used where appropriate"
    ]
  },
  
  testing: {
    unit: "Test components with TestBed (standalone)",
    integration: "Test signal reactivity",
    e2e: "Test hydration doesn't flicker"
  }
}

// Rollout Plan:
{
  phase1: {
    duration: "Sprint 1-2",
    tasks: [
      "Set up Angular 18 project",
      "Configure SSR with hydration",
      "Create component templates",
      "Team training (Signals, Standalone)",
      "Initial architecture docs"
    ]
  },
  
  phase2: {
    duration: "Sprint 3-6",
    tasks: [
      "Build first features",
      "Establish patterns",
      "Code review process",
      "Update guidelines based on learnings"
    ]
  },
  
  phase3: {
    duration: "Sprint 7+",
    tasks: [
      "Scale team",
      "Onboard new developers",
      "Optimize performance",
      "Consider zoneless migration"
    ]
  }
}

// Why NOT full bleeding edge?
{
  notZoneless: {
    reason: "Too risky for enterprise",
    risk: [
      "Experimental feature",
      "Team learning curve",
      "Unknown unknowns",
      "Third-party compatibility"
    ],
    whenReconsider: "Angular 19+ when stable"
  },
  
  pragmatic: {
    approach: "Progressive modernization",
    philosophy: [
      "Use stable modern features (Signals, Standalone)",
      "Avoid experimental features (Zoneless)",
      "Can adopt later when mature",
      "Balance innovation with stability"
    ]
  }
}

// Final verdict:
{
  architecture: "Standalone + Signals + Observables + Zone.js + SSR + Defer",
  modernization: "High (80%)",
  risk: "Medium (well-tested features)",
  teamReady: "Yes (with 2-3 sprint onboarding)",
  futureProof: "Yes (aligned with Angular direction)",
  production: "Ready to deploy"
}
```

---

## Summary

**Modern Angular is fundamentally different:**

1. **Signals** - Pull-based reactivity with automatic dependency tracking
2. **Standalone** - No NgModules, explicit dependencies, clearer architecture
3. **Zoneless** - Optional Zone.js removal for better performance (but not yet recommended for enterprise)
4. **Deferred** - Component-level code splitting with declarative triggers
5. **Hydration** - Non-destructive SSR hydration for better UX

**Production Decision for New App:**
- ✅ Standalone components (100%)
- ✅ Signals for state management
- ✅ Observables for async operations
- ❌ Zoneless (not yet - too risky)
- ✅ SSR with hydration
- ✅ Aggressive @defer usage

**Rationale:** Balance modern features with enterprise stability. Use proven features (Signals, Standalone) while deferring experimental ones (Zoneless) until mature.

This isn't about using every new feature - it's about making pragmatic architectural decisions for long-term success.

---

[← Back to Fundamentals](./fundamentals.md) | [Table of Contents](#table-of-contents)

