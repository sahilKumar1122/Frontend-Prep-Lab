# Angular Fundamentals

## Table of Contents

### Core Concepts
- [What is Angular](#what-is-angular)
- [Angular Architecture](#angular-architecture)

### Components & Lifecycle
- [Components](#components)
- [Component Lifecycle Hooks](#question-explain-the-complete-lifecycle-of-an-angular-component-from-initialization-to-destruction) - [Deep Dive →](./lifecycle-hooks.md)

### Advanced Concepts
- [Change Detection Mechanism](#question-walk-me-through-angulars-change-detection-mechanism) - [Deep Dive →](./change-detection.md)
- [Dependency Injection System](#question-explain-angulars-dependency-injection-system-in-detail) - [Deep Dive →](./dependency-injection.md)
- [RxJS Operators (switchMap, mergeMap, concatMap, exhaustMap)](#question-explain-the-difference-between-switchmap-mergemap-concatmap-and-exhaustmap) - [Deep Dive →](./rxjs-operators.md)
- [Constructor vs ngOnInit](#question-explain-the-difference-between-ngoninit-and-the-constructor) - [Deep Dive →](./constructor-vs-ngoninit.md)

### Architecture & Design
- [Modules](#modules)
- [Content Projection](#question-explain-content-projection-and-how-angular-internally-handles-it) - [Deep Dive →](./content-projection.md)

### Performance & Optimization
- [Performance Optimization (10+ Techniques)](#question-angular-apps-dont-just-get-slow--theyre-made-slow-by-developer-mistakes) - [Deep Dive →](./performance-optimization.md)
- [Data Binding](#data-binding)

### State Management
- [NgRx State Management](#question-explain-how-ngrx-or-any-redux-style-library-integrates-with-angular) - [Deep Dive →](./ngrx-state-management.md)

### Testing & Debugging
- [Testing Strategy (TestBed, Async, Mocking)](#question-youve-written-production-grade-angular-code--now-prove-you-can-test-it-properly) - [Deep Dive →](./testing-strategy.md)
- [Memory Leak Debugging & Forensics](#question-real-world-debugging--architecture-no-shortcuts-no-guesses) - [Deep Dive →](./debugging-memory-leaks.md)

### Modern Angular (17-18+)
- [Modern Angular Features](#question-modern-angular-features--evolving-fundamentals) - [Deep Dive →](./modern-angular-features.md)
  - Signals & Reactivity
  - Standalone Components
  - Zoneless Change Detection
  - Deferred Loading
  - Server-Side Hydration

### Additional Topics
- [Directives](#additional-topics)
- [Pipes](#additional-topics)
- [Forms](#additional-topics)
- [Routing & Navigation](#additional-topics)
- [HTTP Client & Interceptors](#additional-topics)

---

## What is Angular?

### Question: Explain Angular and its core architecture. How does it differ from React?

**Answer:**

Angular is a **full-featured, opinionated TypeScript framework** developed by Google for building scalable web applications. Unlike React (a UI library), Angular is a complete platform that includes:

- **Component-based architecture** with decorators
- **Built-in dependency injection** system
- **RxJS-powered reactive programming**
- **CLI for scaffolding and tooling**
- **Router, HTTP client, Forms, Animations** out of the box
- **Ahead-of-Time (AOT) compilation** for production optimization

**Key Architectural Differences from React:**

| Aspect | Angular | React |
|--------|---------|-------|
| Type | Full Framework | UI Library |
| Language | TypeScript (mandatory) | JavaScript/TypeScript (optional) |
| Data Flow | Two-way binding (default) | One-way data flow |
| State Management | Services + RxJS | External (Redux, Zustand, etc.) |
| Learning Curve | Steeper (more concepts) | Gentler (focused scope) |
| Bundle Size | Larger baseline | Smaller, pay-as-you-go |

**Code Example - Angular Component:**

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { UserService } from './services/user.service';
import { Subject, takeUntil } from 'rxjs';

@Component({
  selector: 'app-user-dashboard',
  template: `
    <div class="dashboard">
      <h1>Welcome, {{ user?.name }}</h1>
      <p>Email: {{ user?.email }}</p>
      
      <!-- Two-way binding -->
      <input [(ngModel)]="searchQuery" placeholder="Search..." />
      
      <!-- Event binding -->
      <button (click)="loadUsers()" [disabled]="loading">
        {{ loading ? 'Loading...' : 'Refresh' }}
      </button>
      
      <!-- Structural directives -->
      <ul *ngIf="users.length > 0; else noUsers">
        <li *ngFor="let user of users | async">
          {{ user.name }}
        </li>
      </ul>
      
      <ng-template #noUsers>
        <p>No users found</p>
      </ng-template>
    </div>
  `,
  styleUrls: ['./user-dashboard.component.scss']
})
export class UserDashboardComponent implements OnInit, OnDestroy {
  user: User | null = null;
  users: User[] = [];
  searchQuery = '';
  loading = false;
  
  // Memory leak prevention
  private destroy$ = new Subject<void>();
  
  constructor(private userService: UserService) {}
  
  ngOnInit(): void {
    // Subscribe with automatic cleanup
    this.userService.getCurrentUser()
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => this.user = user);
  }
  
  loadUsers(): void {
    this.loading = true;
    this.userService.getUsers()
      .pipe(takeUntil(this.destroy$))
      .subscribe({
        next: (users) => {
          this.users = users;
          this.loading = false;
        },
        error: (err) => {
          console.error('Failed to load users', err);
          this.loading = false;
        }
      });
  }
  
  ngOnDestroy(): void {
    // Cleanup all subscriptions
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Performance Considerations:**

1. **Change Detection Strategy:**
```typescript
@Component({
  selector: 'app-optimized',
  changeDetection: ChangeDetectionStrategy.OnPush, // Only check when inputs change
  template: `...`
})
```

2. **TrackBy for *ngFor:**
```typescript
trackByUserId(index: number, user: User): number {
  return user.id; // Prevents unnecessary DOM re-renders
}
```

3. **Lazy Loading Modules:**
```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];
```

**Pro Tip:** Most candidates miss the importance of **unsubscribing from Observables**. Always use `takeUntil`, `async` pipe, or `Subscription.unsubscribe()` to prevent memory leaks. In production apps with many components, forgetting this can degrade performance significantly over time.

---

## Angular Architecture

### Question: Explain Angular's modular architecture and how it promotes scalability.

**Answer:**

Angular uses a **hierarchical, module-based architecture** with distinct layers:

1. **Modules (NgModules)** - Cohesive code blocks
2. **Components** - UI building blocks
3. **Services** - Business logic and state
4. **Directives** - DOM manipulation
5. **Pipes** - Data transformation

**Architecture Layers:**

```typescript
// 1. Feature Module (Lazy-loaded)
@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent,
    ProductFormComponent
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    SharedModule,
    ProductRoutingModule
  ],
  providers: [
    ProductService,
    ProductResolver
  ]
})
export class ProductModule {}

// 2. Service Layer (Singleton at root)
@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl = 'api/products';
  private cache$ = new BehaviorSubject<Product[]>([]);
  
  constructor(private http: HttpClient) {}
  
  getProducts(): Observable<Product[]> {
    // Implement caching strategy
    if (this.cache$.value.length === 0) {
      return this.http.get<Product[]>(this.apiUrl)
        .pipe(
          tap(products => this.cache$.next(products)),
          catchError(this.handleError)
        );
    }
    return this.cache$.asObservable();
  }
  
  private handleError(error: HttpErrorResponse) {
    // Centralized error handling
    console.error('API Error:', error);
    return throwError(() => new Error('Something went wrong'));
  }
}

// 3. Smart Component (Container)
@Component({
  selector: 'app-product-container',
  template: `
    <app-product-list
      [products]="products$ | async"
      [loading]="loading$ | async"
      (productSelected)="onProductSelect($event)"
      (refresh)="loadProducts()"
    ></app-product-list>
  `
})
export class ProductContainerComponent {
  products$ = this.productService.getProducts();
  loading$ = new BehaviorSubject(false);
  
  constructor(private productService: ProductService) {}
  
  onProductSelect(product: Product): void {
    // Handle business logic
  }
}

// 4. Presentational Component (Dumb)
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngIf="loading; else content">
      <app-spinner></app-spinner>
    </div>
    
    <ng-template #content>
      <div class="product-grid">
        <app-product-card
          *ngFor="let product of products; trackBy: trackById"
          [product]="product"
          (click)="productSelected.emit(product)"
        ></app-product-card>
      </div>
    </ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush // Performance optimization
})
export class ProductListComponent {
  @Input() products: Product[] = [];
  @Input() loading = false;
  @Output() productSelected = new EventEmitter<Product>();
  @Output() refresh = new EventEmitter<void>();
  
  trackById(index: number, product: Product): number {
    return product.id;
  }
}
```

**Scalability Patterns:**

1. **Feature Modules:** Separate concerns, enable lazy loading
2. **Core Module:** Singleton services (auth, logging, config)
3. **Shared Module:** Reusable components, directives, pipes
4. **Smart/Dumb Components:** Separate logic from presentation

**Module Structure for Large Apps:**

```
src/
├── app/
│   ├── core/
│   │   ├── services/      (AuthService, LoggerService)
│   │   ├── guards/        (AuthGuard, RoleGuard)
│   │   ├── interceptors/  (AuthInterceptor, ErrorInterceptor)
│   │   └── core.module.ts
│   ├── shared/
│   │   ├── components/    (ButtonComponent, ModalComponent)
│   │   ├── directives/    (HighlightDirective)
│   │   ├── pipes/         (FormatDatePipe)
│   │   └── shared.module.ts
│   ├── features/
│   │   ├── products/
│   │   ├── orders/
│   │   └── users/
│   └── app.module.ts
```

**Pro Tip:** Use **providedIn: 'root'** for services instead of declaring them in module providers. This enables tree-shaking - if a service isn't used, it won't be included in the bundle. Most candidates still use the older pattern of adding services to `providers: []` array, which doesn't optimize as well.

---

## Components

### Question: Explain the complete lifecycle of an Angular component from initialization to destruction. Don't just list the hooks - explain WHEN and WHY each fires, and give me a real scenario where you'd use each one. If you miss any or give shallow answers, we're going to have problems.

**Answer:**

Angular components have a **well-defined lifecycle** managed by the framework. Understanding these hooks is crucial for proper resource management, performance optimization, and avoiding memory leaks.

**For a comprehensive deep dive on all 8 lifecycle hooks with detailed WHEN, WHY, and real-world scenarios, see [Lifecycle Hooks Deep Dive](./lifecycle-hooks.md)**

**Quick Summary of Lifecycle Hooks:**

```typescript
@Component({
  selector: 'app-example',
  template: `...`
})
export class ExampleComponent implements OnInit, OnDestroy {
  @Input() data: any;
  @ViewChild('element') element!: ElementRef;
  
  constructor() { /* DI only */ }
  
  ngOnInit(): void {
    // ✅ Initialize component, fetch data, set up subscriptions
  }
  
  ngAfterViewInit(): void {
    // ✅ Access DOM elements, initialize third-party libraries
  }
  
  ngOnDestroy(): void {
    // ✅ CRITICAL: Clean up subscriptions, timers, event listeners
  }
}
```

**The 8 Lifecycle Hooks (in order):**

1. **ngOnChanges()** - Fires when `@Input()` properties change
2. **ngOnInit()** - Fires once after first `ngOnChanges()` (initialization)
3. **ngDoCheck()** - Fires on every change detection cycle (use sparingly)
4. **ngAfterContentInit()** - Fires after `<ng-content>` is projected
5. **ngAfterContentChecked()** - Fires after content is checked (frequent)
6. **ngAfterViewInit()** - Fires after component view is initialized
7. **ngAfterViewChecked()** - Fires after view is checked (frequent)
8. **ngOnDestroy()** - Fires before component is destroyed (cleanup)

**Most Important Hooks:**
- **ngOnInit** for initialization and data fetching
- **ngAfterViewInit** for DOM access
- **ngOnDestroy** for cleanup (prevent memory leaks!)

**Common Mistakes:**
- ❌ Accessing `@ViewChild` in `ngOnInit` (undefined until `ngAfterViewInit`)
- ❌ Forgetting to unsubscribe in `ngOnDestroy` (memory leaks)
- ❌ Heavy operations in `ngDoCheck` or `ngAfterViewChecked` (performance)

**For detailed explanations with real-world scenarios, see [Lifecycle Hooks Deep Dive](./lifecycle-hooks.md)**

---
```typescript
@Component({
  selector: 'app-live-dashboard',
  template: `
    <div class="dashboard">
      <h1>Live Stock Dashboard</h1>
      <div *ngFor="let stock of stocks$ | async">
        {{ stock.symbol }}: ${{ stock.price }}
      </div>
      <canvas #chartCanvas></canvas>
    </div>
  `
})
export class LiveDashboardComponent implements OnInit, OnDestroy {
  @ViewChild('chartCanvas') chartCanvas!: ElementRef;
  
  stocks$ = new Subject<Stock[]>();
  private destroy$ = new Subject<void>();
  private pollingSubscription?: Subscription;
  private intervalId?: number;
  private websocket?: WebSocket;
  private chart: any;
  private resizeHandler?: () => void;
  
  constructor(
    private stockService: StockService,
    private notificationService: NotificationService,
    private analytics: AnalyticsService
  ) {}
  
  ngOnInit(): void {
    // 1. ✅ Observable subscription
    this.stockService.getLiveStocks()
      .pipe(takeUntil(this.destroy$))
      .subscribe(stocks => {
        this.stocks$.next(stocks);
        this.updateChart(stocks);
      });
    
    // 2. ✅ Interval polling (fallback if WebSocket fails)
    this.pollingSubscription = interval(5000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => this.checkForUpdates());
    
    // 3. ✅ setTimeout/setInterval
    this.intervalId = window.setInterval(() => {
      this.analytics.trackEngagement('dashboard-active');
    }, 30000); // Track every 30 seconds
    
    // 4. ✅ WebSocket connection
    this.websocket = new WebSocket('wss://api.stocks.com/live');
    this.websocket.onmessage = (event) => {
      const update = JSON.parse(event.data);
      this.handleRealtimeUpdate(update);
    };
    this.websocket.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.notificationService.showError('Connection lost');
    };
    
    // 5. ✅ DOM event listener
    this.resizeHandler = () => this.handleResize();
    window.addEventListener('resize', this.resizeHandler);
    
    // 6. ✅ Store subscription (NgRx example)
    // this.store.select(selectUser)
    //   .pipe(takeUntil(this.destroy$))
    //   .subscribe(user => this.currentUser = user);
  }
  
  ngOnDestroy(): void {
    // ✅ CRITICAL CLEANUP - Do ALL of these!
    console.log('Cleaning up dashboard component...');
    
    // 1. Unsubscribe all observables using Subject pattern
    this.destroy$.next();
    this.destroy$.complete();
    
    // 2. Unsubscribe manual subscriptions (if not using takeUntil)
    this.pollingSubscription?.unsubscribe();
    
    // 3. Clear timers
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
    
    // 4. Close WebSocket connection
    if (this.websocket && this.websocket.readyState === WebSocket.OPEN) {
      this.websocket.close();
    }
    
    // 5. Remove event listeners
    if (this.resizeHandler) {
      window.removeEventListener('resize', this.resizeHandler);
    }
    
    // 6. Dispose third-party libraries
    if (this.chart) {
      this.chart.destroy();
    }
    
    // 7. Analytics cleanup
    this.analytics.trackEvent('dashboard-closed', {
      timeSpent: Date.now() - this.startTime
    });
  }
  
  private startTime = Date.now();
  
  private handleRealtimeUpdate(update: any): void {
    console.log('Real-time update:', update);
  }
  
  private checkForUpdates(): void {
    console.log('Polling for updates...');
  }
  
  private handleResize(): void {
    if (this.chart) {
      this.chart.resize();
    }
  }
  
  private updateChart(stocks: Stock[]): void {
    // Update chart with new data
  }
}

interface Stock {
  symbol: string;
  price: number;
  change: number;
}
```

**THE #1 MEMORY LEAK MISTAKE - HTTP Subscriptions:**
```typescript
// ❌ WRONG - Memory leak!
ngOnInit(): void {
  this.userService.getUsers().subscribe(users => {
    this.users = users;
  });
  // When component destroys, this subscription keeps running!
  // Even though HTTP completes, the subscription is still in memory
}

// ✅ CORRECT - Pattern 1: takeUntil
private destroy$ = new Subject<void>();

ngOnInit(): void {
  this.userService.getUsers()
    .pipe(takeUntil(this.destroy$))
    .subscribe(users => this.users = users);
}

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}

// ✅ CORRECT - Pattern 2: async pipe (best for templates)
users$ = this.userService.getUsers();
// Template: <div *ngFor="let user of users$ | async">
// No need to unsubscribe, async pipe handles it!

// ✅ CORRECT - Pattern 3: Manual unsubscribe
private subscription?: Subscription;

ngOnInit(): void {
  this.subscription = this.userService.getUsers()
    .subscribe(users => this.users = users);
}

ngOnDestroy(): void {
  this.subscription?.unsubscribe();
}
```

**Common Memory Leak Sources:**
1. ❌ RxJS subscriptions (Observables, Subjects)
2. ❌ Event listeners (window, document, DOM events)
3. ❌ Timers (setTimeout, setInterval)
4. ❌ WebSocket/SSE connections
5. ❌ Third-party libraries (charts, maps, editors)
6. ❌ Store subscriptions (NgRx, Akita)
7. ❌ Route subscriptions (if not using snapshot)

---

## **The Complete Sequence in Action**

Here's what happens from creation to destruction:

```typescript
@Component({
  selector: 'app-lifecycle-complete',
  template: `
    <h1>{{ title }}</h1>
    <ng-content></ng-content>
    <div #output>Output: {{ data }}</div>
  `
})
export class LifecycleCompleteComponent implements 
  OnChanges, OnInit, DoCheck, AfterContentInit, AfterContentChecked,
  AfterViewInit, AfterViewChecked, OnDestroy {
  
  @Input() title = '';
  @Input() data: any;
  @ViewChild('output') output!: ElementRef;
  @ContentChild(SomeComponent) content!: SomeComponent;
  
  constructor() {
    console.log('0. Constructor - DI only');
    // ❌ this.title is undefined here!
    // ❌ this.output is undefined here!
    // ❌ this.content is undefined here!
  }
  
  ngOnChanges(changes: SimpleChanges): void {
    console.log('1. ngOnChanges - Input properties changed');
    // ✅ this.title is NOW available
    // ❌ this.output is STILL undefined
    // ❌ this.content is STILL undefined
  }
  
  ngOnInit(): void {
    console.log('2. ngOnInit - Component initialized');
    // ✅ this.title is available
    // ✅ this.data is available
    // ❌ this.output is STILL undefined
    // ❌ this.content is STILL undefined
    // ✅✅✅ Make API calls here
    // ✅✅✅ Set up subscriptions here
  }
  
  ngDoCheck(): void {
    console.log('3. ngDoCheck - Change detection ran');
    // ⚠️ Runs VERY frequently - be careful!
    // Use for custom change detection only
  }
  
  ngAfterContentInit(): void {
    console.log('4. ngAfterContentInit - Projected content ready');
    // ✅ this.content is NOW available
    // ❌ this.output is STILL undefined
    // ✅ Access @ContentChild here
  }
  
  ngAfterContentChecked(): void {
    console.log('5. ngAfterContentChecked - Content checked');
    // ⚠️ Runs after every check
    // Avoid expensive operations
  }
  
  ngAfterViewInit(): void {
    console.log('6. ngAfterViewInit - View ready');
    // ✅ this.output is NOW available!
    // ✅✅✅ Initialize third-party libraries here
    // ✅✅✅ Access DOM elements here
    // ✅✅✅ Measure element dimensions here
    console.log('Output element:', this.output.nativeElement.textContent);
  }
  
  ngAfterViewChecked(): void {
    console.log('7. ngAfterViewChecked - View checked');
    // ⚠️ Runs after every check
    // ⚠️⚠️⚠️ DON'T modify state here!
  }
  
  ngOnDestroy(): void {
    console.log('8. ngOnDestroy - Component destroyed');
    // ✅✅✅ Cleanup EVERYTHING here
    // ✅ Unsubscribe from observables
    // ✅ Clear timers
    // ✅ Remove event listeners
    // ✅ Close WebSocket connections
    // ✅ Dispose third-party libraries
  }
}
```

**Execution Flow with Parent-Child:**
```
CREATION PHASE:
Parent: constructor → ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit → ngAfterContentChecked
  ↓
Child: constructor → ngOnChanges → ngOnInit → ngDoCheck → ngAfterContentInit → ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked
  ↓
Parent: ngAfterViewInit → ngAfterViewChecked

CHANGE DETECTION PHASE (repeated many times):
Parent: ngDoCheck → ngAfterContentChecked
  ↓
Child: ngDoCheck → ngAfterContentChecked → ngAfterViewChecked
  ↓
Parent: ngAfterViewChecked

DESTRUCTION PHASE:
Child: ngOnDestroy
  ↓
Parent: ngOnDestroy
```

---

## **Quick Reference: When to Use Each Hook**

| Hook | Frequency | Use For | Access Available |
|------|-----------|---------|------------------|
| **ngOnChanges** | On input change | React to input changes, compare old vs new | `@Input()` properties |
| **ngOnInit** ✅ | Once | API calls, subscriptions, initialization | `@Input()` properties |
| **ngDoCheck** ⚠️ | Every CD cycle | Custom change detection (use sparingly) | `@Input()` properties |
| **ngAfterContentInit** | Once | Access projected content | `@ContentChild/Children` |
| **ngAfterContentChecked** ⚠️ | Every CD cycle | React to content changes (avoid) | `@ContentChild/Children` |
| **ngAfterViewInit** ✅ | Once | DOM access, third-party libs | `@ViewChild/Children` + DOM |
| **ngAfterViewChecked** ⚠️ | Every CD cycle | React to view changes (avoid state changes) | `@ViewChild/Children` + DOM |
| **ngOnDestroy** ✅✅✅ | Once | Cleanup (CRITICAL for memory leaks) | Everything |

---

## **Common Mistakes and Solutions**

### **Mistake 1: Accessing @ViewChild in ngOnInit**
```typescript
// ❌ WRONG
ngOnInit(): void {
  console.log(this.viewChild.nativeElement); // undefined!
}

// ✅ CORRECT
ngAfterViewInit(): void {
  console.log(this.viewChild.nativeElement); // Works!
}
```

### **Mistake 2: Forgetting to Unsubscribe**
```typescript
// ❌ WRONG - Memory leak
ngOnInit(): void {
  this.dataService.getData().subscribe(data => this.data = data);
}

// ✅ CORRECT
private destroy$ = new Subject<void>();

ngOnInit(): void {
  this.dataService.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => this.data = data);
}

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### **Mistake 3: Modifying State in ngAfterViewChecked**
```typescript
// ❌ WRONG - Causes ExpressionChangedAfterItHasBeenCheckedError
ngAfterViewChecked(): void {
  this.someProperty = newValue; // DON'T DO THIS!
}

// ✅ CORRECT - Defer to next cycle
ngAfterViewChecked(): void {
  setTimeout(() => {
    this.someProperty = newValue;
  });
}
```

### **Mistake 4: Heavy Operations in ngDoCheck**
```typescript
// ❌ WRONG - Runs hundreds of times per second
ngDoCheck(): void {
  this.expensiveCalculation(); // Kills performance!
}

// ✅ CORRECT - Only when actually changed
ngDoCheck(): void {
  const currentHash = this.quickHash();
  if (this.previousHash !== currentHash) {
    this.expensiveCalculation();
    this.previousHash = currentHash;
  }
}
```

---

**Pro Tip:** In production Angular applications, the three most critical hooks are:

1. **ngOnInit** - For initialization and data fetching
2. **ngAfterViewInit** - For DOM access and third-party libraries
3. **ngOnDestroy** - For cleanup (NEVER skip this for components with subscriptions)

The hooks that run on every change detection cycle (ngDoCheck, ngAfterContentChecked, ngAfterViewChecked) should be avoided unless absolutely necessary. They can destroy performance in large applications. Always use `OnPush` change detection strategy and immutable data patterns when possible.

**Memory Leak Detection:** In Chrome DevTools, take heap snapshots before and after navigating to/from your component multiple times. If memory keeps growing, you have a leak. Common culprits: unsubscribed Observables, event listeners, and third-party libraries not properly disposed.

---

### Question: Walk me through Angular's change detection mechanism from the moment an event (say a button click) occurs in the browser to when the DOM actually updates. I want you to cover: The role of Zone.js in triggering change detection, How Angular's ChangeDetectorRef interacts with it, The difference between ChangeDetectionStrategy.Default and OnPush, What happens internally when you mark a component for check, How you would manually optimize change detection in a large app. And don't just talk theory — give me a concrete example where you switched a component from Default to OnPush, what problem it solved, and what pitfalls you faced. If you skip Zone.js internals or fail to explain how Angular decides which components to re-render, that's a red flag.

**Answer:**

Angular's change detection isn't just "check for changes" - it's a sophisticated system involving Zone.js monkey-patching, change detector trees, and strategic optimization. Understanding the complete flow from browser event to DOM update is crucial for building performant applications.

**For the complete deep dive covering Zone.js internals, ChangeDetectorRef mechanics, Default vs OnPush strategies, manual optimization techniques, and real-world case studies with performance improvements, see [Change Detection Deep Dive](./change-detection.md)**

---

### Question: Explain Angular's Dependency Injection system in detail. Don't just say "it injects services." I want you to walk me through exactly how Angular resolves a dependency — from provider registration to instance creation. Cover these points specifically: How the hierarchical injector tree works (root, module, component, directive level), The difference between providing a service in a module vs component vs using @Injectable({ providedIn: 'root' }), What are injection tokens and when do you need them? What is a multi-provider and where would you use one? How does tree-shaking interact with DI? What happens if two injectors provide the same token? And then — real-world time: Give me an example from an actual project where you had to fix a service scope bug caused by a wrong provider configuration. What did you do, and what did you learn from it? If you just say "Angular creates services as singletons," we're done. So — how does Angular really resolve dependencies?

**Answer:**

Angular's DI system isn't just "inject and use" - it's a sophisticated hierarchical system with multiple injector levels, provider strategies, and tree-shaking optimizations. Understanding how dependency resolution actually works is essential for avoiding scope bugs and memory leaks.

**For the complete deep dive covering hierarchical injector trees, provider scopes, injection tokens, multi-providers, tree-shaking mechanics, collision handling, and a real-world bug fix case study, see [Dependency Injection Deep Dive](./dependency-injection.md)**

---

### Question: Explain the difference between switchMap, mergeMap, concatMap, and exhaustMap — but don't just parrot definitions. I want to know: Exactly how each operator handles inner subscriptions, What happens if the source observable emits again before the inner one completes, Which one you'd pick for HTTP requests, real-time data streams, and user input debouncing — and why, How each operator affects concurrency and memory usage, How you'd debug an observable chain if a stream unexpectedly cancels mid-flight. Then, I'll push further: Give me a real bug you faced where the wrong RxJS operator caused unexpected behavior. What changed when you fixed it, and why did the new operator work? Finally, what's your unsubscribe strategy — and why do you think it's the safest for complex reactive chains? If you answer this with generic "switchMap cancels previous observables" nonsense, that's textbook fluff. I want mechanics — marble diagram-level clarity.

**Answer:**

RxJS flattening operators aren't just "merge observables" - each has specific concurrency behavior, memory implications, and use cases. Understanding the internal mechanics and when each operator fails is critical for building reliable reactive applications.

**For the exhaustive deep dive covering internal mechanics, marble diagrams, use case selection, concurrency/memory analysis, debugging strategies, a real-world bug fix, and comprehensive unsubscribe strategies, see [RxJS Operators Deep Dive](./rxjs-operators.md)**

---

### Question: Explain the difference between ngOnInit() and the constructor in Angular components. What is each one used for? When exactly does Angular call them in the component lifecycle? Give a scenario where using ngOnInit() is required instead of the constructor, and vice versa. Follow-up traps you should be ready for (if you get this in an interview): What happens if you try to access an @Input() property inside the constructor? Can ngOnInit() be async? Should it be? What's the difference between using dependency injection in the constructor vs in ngOnInit()? Let's start — explain exactly how Angular initializes a component and the role of each of these two methods. (Be precise, not vague — I'll challenge incomplete answers.)

**Answer:**

The constructor vs ngOnInit distinction isn't just "constructor is first, ngOnInit is second" - it's about understanding Angular's initialization pipeline, dependency injection timing, and when component bindings are available. Misusing these can lead to subtle bugs.

**For the complete deep dive covering exact call timing, DI differences, @Input() availability, async implications, and real-world scenarios where one is required over the other, see [Constructor vs ngOnInit Deep Dive](./constructor-vs-ngoninit.md)**

---

## Modules

### Question: Explain Angular modules and the difference between feature modules, core modules, and shared modules.

**Answer:**

Angular **NgModules** organize an application into cohesive functional blocks. They act as compilation contexts for components and provide dependency injection scopes.

**Module Types & Best Practices:**

```typescript
// 1. APP MODULE (Root)
@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,          // Only import in AppModule
    BrowserAnimationsModule,
    CoreModule,             // Import once
    SharedModule,           // Can import in any module
    AppRoutingModule        // Root routing
  ],
  providers: [],            // Keep empty, use providedIn: 'root'
  bootstrap: [AppComponent]
})
export class AppModule {}

// 2. CORE MODULE (Singleton Services)
// Import ONCE in AppModule
@NgModule({
  providers: [
    // Global singleton services
    AuthService,
    LoggerService,
    ErrorHandlerService,
    
    // HTTP Interceptors
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ErrorInterceptor,
      multi: true
    }
  ],
  declarations: [
    // Layout components (used once)
    HeaderComponent,
    FooterComponent,
    SidebarComponent
  ],
  exports: [
    HeaderComponent,
    FooterComponent,
    SidebarComponent
  ]
})
export class CoreModule {
  // Prevent re-import
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule is already loaded. Import it only in AppModule');
    }
  }
}

// 3. SHARED MODULE (Reusable Components)
// Import in any feature module that needs it
@NgModule({
  declarations: [
    // Reusable components
    ButtonComponent,
    CardComponent,
    ModalComponent,
    SpinnerComponent,
    
    // Directives
    HighlightDirective,
    TooltipDirective,
    
    // Pipes
    FormatDatePipe,
    TruncatePipe,
    SafeHtmlPipe
  ],
  imports: [
    CommonModule,
    FormsModule,
    ReactiveFormsModule,
    MaterialModule // Third-party UI library
  ],
  exports: [
    // Re-export everything for convenience
    CommonModule,
    FormsModule,
    ReactiveFormsModule,
    MaterialModule,
    
    // Export declared components/directives/pipes
    ButtonComponent,
    CardComponent,
    ModalComponent,
    SpinnerComponent,
    HighlightDirective,
    TooltipDirective,
    FormatDatePipe,
    TruncatePipe,
    SafeHtmlPipe
  ]
})
export class SharedModule {}

// 4. FEATURE MODULE (Lazy-loaded)
@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent,
    ProductFormComponent
  ],
  imports: [
    SharedModule,           // Reusable components
    ProductRoutingModule    // Feature routing
  ],
  providers: [
    // Feature-specific services (scoped to this module)
    ProductService,
    ProductResolver,
    ProductGuard
  ]
})
export class ProductModule {}

// 5. ROUTING CONFIGURATION FOR LAZY LOADING
const routes: Routes = [
  {
    path: '',
    redirectTo: '/home',
    pathMatch: 'full'
  },
  {
    path: 'products',
    loadChildren: () => import('./features/products/product.module')
      .then(m => m.ProductModule),
    canActivate: [AuthGuard]
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.module')
      .then(m => m.AdminModule),
    canActivate: [AuthGuard, AdminGuard]
  }
];
```

**Advanced Module Patterns:**

```typescript
// 1. Material Module (Third-party wrapper)
@NgModule({
  exports: [
    MatButtonModule,
    MatCardModule,
    MatInputModule,
    MatTableModule,
    MatPaginatorModule,
    MatSortModule,
    MatDialogModule,
    MatSnackBarModule
  ]
})
export class MaterialModule {}

// 2. Providers Module with forRoot pattern
@NgModule({})
export class ConfigModule {
  static forRoot(config: AppConfig): ModuleWithProviders<ConfigModule> {
    return {
      ngModule: ConfigModule,
      providers: [
        {
          provide: APP_CONFIG,
          useValue: config
        }
      ]
    };
  }
}

// Usage in AppModule
@NgModule({
  imports: [
    ConfigModule.forRoot({
      apiUrl: environment.apiUrl,
      enableDebug: !environment.production
    })
  ]
})
export class AppModule {}
```

**Module Architecture Decision Tree:**

```
Is it a singleton service?
├─ YES → Core Module OR providedIn: 'root'
└─ NO → Continue

Is it reusable across features?
├─ YES → Shared Module
└─ NO → Continue

Is it feature-specific?
├─ YES → Feature Module (lazy-load if possible)
└─ NO → Reconsider design
```

**Performance Benefits:**

```typescript
// Lazy loading reduces initial bundle size
// Before: All modules loaded = 500KB initial bundle
// After: Only needed modules = 150KB initial, 350KB lazy-loaded

// Measure bundle size
// ng build --prod --stats-json
// npm install -g webpack-bundle-analyzer
// webpack-bundle-analyzer dist/stats.json
```

**Pro Tip:** Most candidates don't understand that **importing SharedModule in multiple feature modules doesn't duplicate code** - Angular's compiler is smart enough to include it once. However, **never import CoreModule more than once**, as this creates multiple instances of singleton services. Use the constructor guard shown above to prevent this. In production apps, this mistake can cause subtle bugs like duplicate HTTP requests or inconsistent state.

---

## Dependency Injection

### Question: Explain Angular's Dependency Injection system and hierarchical injectors.

**Answer:**

Angular's **Dependency Injection (DI)** is a sophisticated design pattern that provides dependencies to classes instead of classes creating them. Angular's DI system is **hierarchical**, creating a tree of injectors that mirror the component tree.

**Injector Hierarchy:**

```
ModuleInjector (providedIn: 'root')
    ↓
ModuleInjector (providedIn: 'platform')
    ↓
ElementInjector (Component level)
    ↓
ElementInjector (Child Component level)
```

**Complete DI Examples:**

```typescript
// 1. SERVICE WITH PROVIDEDIN ROOT (Recommended)
@Injectable({
  providedIn: 'root'  // ✅ Singleton, tree-shakeable
})
export class UserService {
  private apiUrl = '/api/users';
  
  constructor(
    private http: HttpClient,
    private logger: LoggerService
  ) {}
  
  getUsers(): Observable<User[]> {
    this.logger.log('Fetching users');
    return this.http.get<User[]>(this.apiUrl);
  }
}

// 2. COMPONENT-LEVEL INJECTION
@Component({
  selector: 'app-user-list',
  template: `...`,
  providers: [
    // ⚠️ New instance for each component instance
    UserFilterService
  ]
})
export class UserListComponent {
  // Different injection patterns
  
  // Constructor injection (most common)
  constructor(
    private userService: UserService,
    private filterService: UserFilterService
  ) {}
  
  // Property injection (rare, used with @Host, @Optional, etc.)
  @Inject(WINDOW) private window: Window;
}

// 3. INJECTION TOKENS (For non-class dependencies)
export const API_URL = new InjectionToken<string>('api.url');
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Provide in module
@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' },
    { provide: APP_CONFIG, useValue: { timeout: 5000, retries: 3 } }
  ]
})
export class AppModule {}

// Inject in component
constructor(
  @Inject(API_URL) private apiUrl: string,
  @Inject(APP_CONFIG) private config: AppConfig
) {}

// 4. PROVIDER TYPES
@NgModule({
  providers: [
    // useClass (most common)
    { provide: LoggerService, useClass: LoggerService },
    { provide: LoggerService, useClass: AdvancedLoggerService }, // Replace implementation
    
    // useValue (for constants)
    { provide: API_URL, useValue: 'https://api.example.com' },
    { provide: APP_CONFIG, useValue: { debug: true } },
    
    // useFactory (for dynamic creation)
    {
      provide: HttpClient,
      useFactory: (backend: HttpBackend, config: AppConfig) => {
        return config.enableAuth
          ? new HttpClient(backend)
          : new HttpClient(backend);
      },
      deps: [HttpBackend, APP_CONFIG]
    },
    
    // useExisting (alias)
    { provide: NewLoggerService, useExisting: LoggerService }
  ]
})
```

**Advanced DI Decorators:**

```typescript
// @Optional - Don't throw if dependency not found
@Component({ selector: 'app-child' })
export class ChildComponent {
  constructor(
    @Optional() private logger?: LoggerService
  ) {
    if (this.logger) {
      this.logger.log('Logger available');
    }
  }
}

// @Self - Only look in current injector
@Component({
  selector: 'app-component',
  providers: [LocalService]
})
export class MyComponent {
  constructor(
    @Self() private localService: LocalService  // Must be in this component's providers
  ) {}
}

// @SkipSelf - Skip current injector, look in parent
@Component({ selector: 'app-child' })
export class ChildComponent {
  constructor(
    @SkipSelf() private parentService: ParentService  // Get from parent
  ) {}
}

// @Host - Stop searching at host component
@Directive({ selector: '[appTooltip]' })
export class TooltipDirective {
  constructor(
    @Host() private hostComponent: MyComponent  // Stop at host
  ) {}
}

// Combined decorators
constructor(
  @Optional() @SkipSelf() private parentLogger?: LoggerService
) {}
```

**Real-World Pattern: Multi-Level State Management:**

```typescript
// Global State Service
@Injectable({ providedIn: 'root' })
export class GlobalStateService {
  private userState = new BehaviorSubject<User | null>(null);
  user$ = this.userState.asObservable();
  
  setUser(user: User): void {
    this.userState.next(user);
  }
}

// Feature-Scoped State Service
@Injectable()  // No providedIn - provided at feature module level
export class ProductStateService {
  private productsState = new BehaviorSubject<Product[]>([]);
  products$ = this.productsState.asObservable();
  
  constructor(private http: HttpClient) {}
  
  loadProducts(): void {
    this.http.get<Product[]>('/api/products')
      .subscribe(products => this.productsState.next(products));
  }
}

// Feature Module
@NgModule({
  providers: [
    ProductStateService  // ✅ One instance per lazy-loaded module
  ]
})
export class ProductModule {}

// Component-Scoped Service
@Injectable()
export class FormStateService {
  private formData = new BehaviorSubject<any>({});
  formData$ = this.formData.asObservable();
  
  updateFormData(data: any): void {
    this.formData.next(data);
  }
}

@Component({
  selector: 'app-complex-form',
  providers: [FormStateService]  // ✅ New instance per component
})
export class ComplexFormComponent {}
```

**Testing with DI:**

```typescript
describe('UserComponent', () => {
  let component: UserComponent;
  let fixture: ComponentFixture<UserComponent>;
  let userService: jasmine.SpyObj<UserService>;
  
  beforeEach(() => {
    // Create spy
    const userServiceSpy = jasmine.createSpyObj('UserService', ['getUsers']);
    
    TestBed.configureTestingModule({
      declarations: [UserComponent],
      providers: [
        { provide: UserService, useValue: userServiceSpy }  // ✅ Mock service
      ]
    });
    
    fixture = TestBed.createComponent(UserComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });
  
  it('should load users', () => {
    userService.getUsers.and.returnValue(of([{ id: 1, name: 'Test' }]));
    component.ngOnInit();
    expect(userService.getUsers).toHaveBeenCalled();
  });
});
```

**Pro Tip:** Understanding the **injector hierarchy** is crucial for debugging DI issues. When Angular looks for a dependency, it starts at the component's ElementInjector and walks up the tree until it finds it or throws an error. Use `providedIn: 'root'` for singletons (most services), component `providers` for component-scoped instances, and module `providers` only for feature-specific services in lazy-loaded modules. Most candidates over-complicate DI by providing services at multiple levels unnecessarily.

---

## Data Binding

### Question: Explain the different types of data binding in Angular and their performance implications.

**Answer:**

Angular supports **four types of data binding** that enable communication between the component class and the template. Understanding when to use each type is critical for building performant applications.

**The Four Binding Types:**

```typescript
@Component({
  selector: 'app-binding-demo',
  template: `
    <!-- 1. INTERPOLATION (Component → View) -->
    <h1>{{ title }}</h1>
    <p>User: {{ user.firstName }} {{ user.lastName }}</p>
    <p>Total: {{ calculateTotal() }}</p>
    
    <!-- 2. PROPERTY BINDING (Component → View) -->
    <img [src]="imageUrl" [alt]="imageAlt" />
    <button [disabled]="isLoading">Submit</button>
    <div [class.active]="isActive">Active Item</div>
    <div [style.color]="textColor">Colored Text</div>
    
    <!-- 3. EVENT BINDING (View → Component) -->
    <button (click)="handleClick()">Click Me</button>
    <input (input)="onInputChange($event)" />
    <form (submit)="onSubmit($event)">...</form>
    
    <!-- Custom events -->
    <app-child 
      (customEvent)="handleCustom($event)"
      (valueChange)="onValueChange($event)"
    ></app-child>
    
    <!-- 4. TWO-WAY BINDING (Component ↔ View) -->
    <input [(ngModel)]="username" />
    
    <!-- Equivalent to: -->
    <input 
      [ngModel]="username" 
      (ngModelChange)="username = $event"
    />
    
    <!-- Custom two-way binding -->
    <app-counter [(count)]="currentCount"></app-counter>
  `
})
export class BindingDemoComponent {
  // Properties for binding
  title = 'Angular Binding Demo';
  user = { firstName: 'John', lastName: 'Doe' };
  imageUrl = 'assets/logo.png';
  imageAlt = 'Logo';
  isLoading = false;
  isActive = true;
  textColor = '#ff0000';
  username = '';
  currentCount = 0;
  
  // Methods
  calculateTotal(): number {
    // ⚠️ WARNING: Called on every change detection!
    return 100 * 1.2;
  }
  
  handleClick(): void {
    console.log('Button clicked');
  }
  
  onInputChange(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    console.log('Input changed:', value);
  }
  
  onSubmit(event: Event): void {
    event.preventDefault();
    console.log('Form submitted');
  }
}
```

**Custom Two-Way Binding:**

```typescript
// Child Component with custom two-way binding
@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count }}</span>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  // Input property
  @Input() count: number = 0;
  
  // Output property with 'Change' suffix
  @Output() countChange = new EventEmitter<number>();
  
  increment(): void {
    this.count++;
    this.countChange.emit(this.count);  // Emit new value
  }
  
  decrement(): void {
    this.count--;
    this.countChange.emit(this.count);
  }
}

// Usage: <app-counter [(count)]="currentCount"></app-counter>
```

**Performance Optimization Patterns:**

```typescript
@Component({
  selector: 'app-optimized',
  template: `
    <!-- ❌ BAD: Function called on every change detection -->
    <p>{{ expensiveCalculation() }}</p>
    
    <!-- ✅ GOOD: Use property instead -->
    <p>{{ cachedResult }}</p>
    
    <!-- ❌ BAD: Object creation in template -->
    <app-child [config]="{ setting: true }"></app-child>
    
    <!-- ✅ GOOD: Reference stable object -->
    <app-child [config]="childConfig"></app-child>
    
    <!-- ✅ BEST: Use pure pipes for transformations -->
    <p>{{ data | customTransform }}</p>
    
    <!-- ✅ OnPush optimization -->
    <app-child 
      [data]="immutableData"
      (action)="handleAction($event)"
    ></app-child>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedComponent implements OnInit {
  cachedResult: number = 0;
  childConfig = { setting: true };  // Stable reference
  immutableData: readonly Item[] = [];
  
  ngOnInit(): void {
    // Calculate once
    this.cachedResult = this.expensiveCalculation();
  }
  
  private expensiveCalculation(): number {
    // Heavy computation
    return Array.from({ length: 10000 })
      .reduce((sum, _, i) => sum + i, 0);
  }
  
  handleAction(event: any): void {
    // Create new reference for OnPush to detect change
    this.immutableData = [...this.immutableData, event];
  }
}

// Custom Pure Pipe (only recalculates when input changes)
@Pipe({
  name: 'customTransform',
  pure: true  // ✅ Default, but explicit is good
})
export class CustomTransformPipe implements PipeTransform {
  transform(value: any): any {
    // This only runs when value reference changes
    return this.expensiveTransform(value);
  }
  
  private expensiveTransform(value: any): any {
    // Heavy computation
    return value;
  }
}
```

**Advanced Binding Patterns:**

```typescript
@Component({
  selector: 'app-advanced-binding',
  template: `
    <!-- Attribute binding (for non-standard attributes) -->
    <div [attr.data-id]="userId">User Info</div>
    <button [attr.aria-label]="ariaLabel">Action</button>
    
    <!-- Class binding -->
    <div [class]="cssClasses">Multiple classes</div>
    <div [class.active]="isActive">Toggle class</div>
    <div [ngClass]="{ 'active': isActive, 'disabled': isDisabled }">Complex</div>
    
    <!-- Style binding -->
    <div [style.color]="textColor">Colored</div>
    <div [style.width.px]="widthValue">Sized</div>
    <div [ngStyle]="{ 'color': textColor, 'font-size.px': fontSize }">Complex</div>
    
    <!-- Template reference variables -->
    <input #nameInput type="text" />
    <button (click)="nameInput.focus()">Focus Input</button>
    
    <!-- Safe navigation operator -->
    <p>{{ user?.address?.city }}</p>
    
    <!-- Non-null assertion operator -->
    <p>{{ user!.name }}</p>
  `
})
export class AdvancedBindingComponent {
  userId = '12345';
  ariaLabel = 'Close dialog';
  cssClasses = 'btn btn-primary';
  isActive = true;
  isDisabled = false;
  textColor = '#333';
  widthValue = 200;
  fontSize = 16;
  user?: User;
}
```

**Performance Comparison:**

```typescript
// Change Detection Impact

// ❌ WORST: Method call in template (called every CD cycle)
{{ calculatePrice() }}  // ~1000 calls/second

// ⚠️ MEDIUM: Property with getter (called every CD cycle)
get totalPrice() { return this.calculate(); }
{{ totalPrice }}

// ✅ GOOD: Pure pipe (cached, only recalculates on input change)
{{ price | calculatePrice }}

// ✅ BEST: Memoized property (calculated once or on demand)
totalPrice = 0;
ngOnInit() { this.totalPrice = this.calculate(); }
{{ totalPrice }}
```

**Pro Tip:** The biggest performance mistake candidates make is using **method calls in templates** like `{{ getData() }}`. This executes on every change detection cycle (potentially hundreds of times per second). Instead, use **pure pipes** for transformations or **memoize values** in component properties. For complex UIs, combine with `ChangeDetectionStrategy.OnPush` and immutable data patterns. In production apps with hundreds of components, this optimization can reduce CPU usage by 70%+.

---

### Question: Explain content projection and how Angular internally handles it. I'm not asking you to define <ng-content> — I want the full pipeline.

You'll need to cover:
- How Angular parses and renders <ng-content> during component compilation
- The difference between <ng-content>, <ng-container>, and <ng-template> — structurally, not just "one doesn't render DOM"
- How multi-slot projection works
- What happens if projected content has its own bindings or structural directives
- How the change detection context of projected content differs from that of the host component

Then give me a real-world scenario:
- Suppose you're building a highly reusable modal or table component. How would you design it using content projection to allow custom headers, footers, or body templates?
- What pitfalls have you hit (e.g., projected component not updating, losing context, etc.) and how did you fix them?

And finally:
- Would you ever use content projection and dynamic component loading together? Explain how you'd manage lifecycle and change detection in that hybrid setup.

If your answer stops at "ng-content displays child HTML," that's a red flag. So, tell me — how does Angular really handle content projection under the hood?

**Answer:**

Content projection is Angular's mechanism for creating flexible, reusable components by allowing parent components to inject custom content into child components. But it's not just "passing HTML" - Angular has a sophisticated internal pipeline involving template parsing, view hierarchy management, and careful change detection coordination.

**For the complete internal pipeline covering compilation, structural differences between `<ng-content>`, `<ng-container>`, and `<ng-template>`, multi-slot projection, change detection context, real-world reusable component design patterns, common pitfalls, and hybrid approaches with dynamic components, see [Content Projection Deep Dive](./content-projection.md)**

**Quick Overview:**

```typescript
// Simple projection
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content></ng-content>
    </div>
  `
})
export class CardComponent {}

// Usage
<app-card>
  <h1>Card Title</h1>
  <p>Card content</p>
</app-card>
```

**Key Concepts:**

1. **`<ng-content>`** - Projection slot (compiled into projection instruction, no DOM node)
2. **`<ng-container>`** - Logical grouper (doesn't create DOM, can have directives)
3. **`<ng-template>`** - Template definition (not rendered by default)
4. **Multi-slot projection** - Use CSS selectors: `<ng-content select="[header]">`
5. **Change detection context** - Projected content belongs to parent's CD tree
6. **Bindings** - Projected content can only access parent's properties

**Critical Rule:**
Projected content is compiled and change-detected in the **parent's context**, not the host component's context. This affects OnPush strategies and data flow.

**Real-World Use Cases:**
- Reusable modal components with custom header/body/footer
- Table components with customizable columns and rows
- Layout components with flexible content areas
- Card components with varied content structures

**Common Pitfall:**
```typescript
// ❌ Host with OnPush might not update projected content
@Component({
  template: `<ng-content></ng-content>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class HostComponent {}

// Solution: Ensure parent's CD runs or use Default strategy
```

**For detailed explanations including:**
- Internal compilation and AST generation
- Content distribution algorithms
- Managing lifecycle with hybrid projection + dynamic components
- Real bugs and their fixes
- Performance implications

**See the [Content Projection Deep Dive](./content-projection.md)**

---

### Question: Angular apps don't just "get slow" — they're made slow by developer mistakes. So I want you to list at least 10 real, technical causes of Angular performance degradation and how you'd fix each.

You must cover at least these categories:
- Change detection bloat — how it's triggered, and how to reduce it
- DOM rendering inefficiencies — e.g., ngFor pitfalls and trackBy usage
- Inefficient data flow or RxJS misuse — how bad reactive patterns kill performance
- Memory leaks — how to detect, reproduce, and fix them
- Lazy loading & preloading strategies — when and how to use them
- Bundle size optimization — specific techniques (not "use lazy loading" — that's surface-level)
- Template binding performance — what happens with heavy pipes or nested structures
- CDK Virtual Scroll — when to use it, when not to
- Zone.js performance overhead — what can you do to minimize or bypass it
- Third-party library bloat — how to audit and tree-shake

Then — I want a practical case:
- You have a list of 10,000 records rendering in real time with filter inputs. The UI lags badly. Walk me step by step through your optimization process, including exact tools, metrics, and Angular-specific features you'd use.

Don't give me generic answers like "use OnPush." I want the full thought process — profiling, analyzing, fixing, validating.

Would you deploy your optimized solution to production confidently? Defend that answer.

**Answer:**

Angular apps don't "just get slow" - they're made slow by specific, identifiable developer mistakes. Performance optimization requires understanding Angular's internals and applying systematic, measurable fixes.

**For comprehensive coverage of 10+ performance killers with detailed analysis, profiling techniques, and a complete case study optimizing 10,000 records with real-time filtering, see [Performance Optimization Deep Dive](./performance-optimization.md)**

**Key Performance Issues:**

1. **Change Detection Bloat** - Zone.js triggers CD on every event; fix with OnPush, detach, runOutsideAngular
2. **DOM Rendering Inefficiencies** - Missing trackBy causes full re-renders; fix with proper trackBy functions
3. **Inefficient RxJS Patterns** - Wrong operators, no unsubscribe; fix with switchMap, debounce, takeUntil
4. **Memory Leaks** - Unsubscribed observables, event listeners; fix with proper cleanup in ngOnDestroy
5. **No Lazy Loading** - Everything loaded upfront; fix with route-based code splitting
6. **Bundle Size Bloat** - Heavy libraries, no tree-shaking; fix with analysis and alternatives
7. **Template Binding Performance** - Method calls in templates; fix with memoization and pure pipes
8. **Large Lists** - 10k+ DOM nodes; fix with CDK Virtual Scroll (99.85% DOM reduction)
9. **Zone.js Overhead** - CD on every async operation; fix by running outside Angular zone
10. **Third-Party Library Bloat** - Unnecessary dependencies; fix with auditing and replacement

**Quick Example - Virtual Scroll:**

```typescript
// ❌ BAD: 10,000 DOM nodes
<div *ngFor="let item of items">{{ item.name }}</div>
// Result: 8s initial render, 800MB memory

// ✅ GOOD: ~15 DOM nodes (only visible items)
<cdk-virtual-scroll-viewport [itemSize]="50">
  <div *cdkVirtualFor="let item of items; trackBy: trackById">
    {{ item.name }}
  </div>
</cdk-virtual-scroll-viewport>
// Result: 150ms initial render, 120MB memory
// Improvement: 98% faster, 85% less memory
```

**Profiling Tools:**
- Chrome DevTools Performance tab
- Angular DevTools Profiler
- Lighthouse for Core Web Vitals
- `webpack-bundle-analyzer` for bundle analysis
- Custom performance markers

**Real-World Impact:**
- Change detection: 5000 calls/sec → 50 calls/sec (99% reduction)
- Bundle size: 2.5MB → 800KB (68% reduction)
- Initial render: 8s → 150ms (98% faster)
- Memory usage: 800MB → 120MB (85% reduction)
- Frame rate: 8 FPS → 60 FPS (650% improvement)

**For the complete practical case study including:**
- Step-by-step optimization process for 10,000 records
- Exact profiling techniques and metrics
- Web Worker integration
- IndexedDB for large datasets
- Production deployment decision with monitoring
- Before/after measurements

**See the [Performance Optimization Deep Dive](./performance-optimization.md)**

---

### Question: Explain how NgRx (or any Redux-style library) integrates with Angular — but I don't want marketing talk. I want you to walk me through the entire data flow from a component dispatching an action to the UI re-rendering.

Specifically, cover:
- Action lifecycle — from store.dispatch() to the reducer executing
- How reducers produce new immutable state and why immutability is crucial
- How selectors memoize derived state and prevent unnecessary change detection
- How effects work — particularly their relationship with RxJS streams and async operations
- How Angular knows to update only affected components after the store changes

Then I want you to defend your design choices:
- When is NgRx the wrong choice? Give a concrete example.
- Compare NgRx to Akita and NGXS — pros, cons, and trade-offs.
- If you were building a mid-sized project with moderate complexity, how would you decide whether to introduce a state library at all?
- What's your strategy to keep NgRx boilerplate manageable in a large codebase?

And finally:
- Let's say your effect starts firing multiple duplicate API calls due to race conditions. How do you fix that? Which RxJS operator would you use — and why?

If you give me "Actions go to reducers, reducers change state" as your explanation, that's a red flag. I want architecture-level understanding — the why, not just the what.

**Answer:**

NgRx isn't just "Redux for Angular" - it's a sophisticated reactive state management system deeply integrated with Angular's change detection, RxJS observables, and dependency injection. Understanding the architecture requires tracing the complete internal pipeline.

**For the complete architectural deep dive covering action lifecycle internals, reducer mechanics, selector memoization, effects integration, change detection coordination, design trade-offs, library comparisons, and race condition handling, see [NgRx State Management Deep Dive](./ngrx-state-management.md)**

**Complete Data Flow:**

```typescript
// 1. Component dispatches action
store.dispatch(loadUser({ userId: 123 }))
   ↓
// 2. ActionsSubject.next(action)
   ↓
// 3. Action flows to two parallel pipelines:
   ├─→ Effects (side effects)
   │   └→ HTTP call → dispatch success/failure
   └─→ ScannedActionsSubject (state reduction)
       ↓
// 4. withLatestFrom(reducerManager) - combines [action, reducer]
       ↓
// 5. scan() executes: newState = reducer(oldState, action)
       ↓
// 6. New state emitted to state$ stream
       ↓
// 7. Selectors subscribed to state$ check memoization
       ↓
// 8. If value changed, emit to component
       ↓
// 9. Async pipe calls cdr.markForCheck()
       ↓
// 10. Component updates (OnPush bypassed)
```

**Why Immutability is Critical:**

```typescript
// ✅ Immutable update (O(1) reference check)
const newState = { ...oldState, count: 2 };
if (oldState !== newState) {  // Instant!
  // Trigger change detection
}

// ❌ Mutable update (would need O(n) deep equality)
oldState.count = 2;
if (deepEqual(oldState, oldState)) {  // Expensive!
  // Never triggers - same reference
}
```

**Selector Memoization:**

```typescript
// Memoized selector caches results
export const selectExpensiveData = createSelector(
  selectUsers,
  selectFilters,
  (users, filters) => {
    console.log('Computing...'); // Only when inputs change!
    return expensiveCalculation(users, filters);
  }
);

// Without memoization: runs every CD cycle (50-100ms)
// With memoization: runs only on input change (0ms cached)
```

**Effects and Race Conditions:**

```typescript
// ❌ BAD: mergeMap allows race conditions
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(searchProducts),
    mergeMap(action => this.api.search(action.query))
    // User types 'abc' quickly:
    // Request 'a', 'ab', 'abc' all fire
    // Responses arrive out of order
    // UI shows wrong results!
  )
);

// ✅ GOOD: switchMap cancels previous
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(searchProducts),
    debounceTime(300),      // Wait for typing to stop
    switchMap(action => this.api.search(action.query))
    // Only final request ('abc') completes
  )
);
```

**When NgRx is the WRONG Choice:**

1. **Simple CRUD** - 290 lines of boilerplate vs 50 lines with service
2. **Component-local state** - UI state (accordions, modals) doesn't need global store
3. **Small teams** - Learning curve + boilerplate overhead
4. **No shared state** - If components don't share data, service is simpler

**NgRx vs Alternatives:**

| Feature | NgRx | Akita | NGXS |
|---------|------|-------|------|
| Boilerplate | High | Low | Medium |
| Learning Curve | Steep | Moderate | Moderate |
| Community | Largest | Small | Small |
| Best For | Large apps | Small/medium | Medium apps |

**Managing Boilerplate:**

```typescript
// Facade pattern hides complexity
@Injectable({ providedIn: 'root' })
export class UserFacade {
  user$ = this.store.select(selectUser);
  loading$ = this.store.select(selectLoading);
  
  loadUser(id: number): void {
    this.store.dispatch(loadUser({ id }));
  }
}

// Component (much cleaner!)
export class UserComponent {
  constructor(public facade: UserFacade) {}
  
  ngOnInit(): void {
    this.facade.loadUser(123);
  }
}
```

**For complete coverage including:**
- Internal Store/ActionsSubject/ReducerManager implementation
- Why immutability enables OnPush optimization
- Selector memoization algorithm internals
- Effect error handling patterns
- Decision framework (when to use/not use)
- Complete NgRx vs Akita vs NGXS comparison
- Boilerplate reduction strategies
- Race condition fixes with exact RxJS operators

**See the [NgRx State Management Deep Dive](./ngrx-state-management.md)**

---

### Question: You've written production-grade Angular code — now prove you can test it properly. Walk me through how you design, write, and structure tests for a complex Angular module. I want you to cover unit, integration, and end-to-end levels, but focus deep on Angular TestBed and async testing.

Specifically:
- Explain how the Angular TestBed actually works under the hood — how it creates a testing module and component fixture.
- What's the difference between testing a standalone component vs a module-based component?
- How do you test async operations like HTTP calls or observable streams? Explain fakeAsync, tick, flush, and when each fails.
- How do you mock dependencies — services, HTTP requests, child components — and when should you not mock something?
- Explain how to detect and prevent flaky tests — what usually causes them in Angular environments?
- What's the difference between detectChanges() and auto change detection in Angular testing?
- How do you test components using OnPush change detection?

Then give me a real example:
- You have a component that makes an HTTP request, processes the response through RxJS operators, and updates a view. Walk me through writing a complete test for this — from setup to verification.

Finally:
- Would you trust 100% test coverage as a sign of production readiness? Why or why not?

If your answer starts with "we just use Jasmine and Karma," stop — that's textbook. I want deep insight into testing strategy, isolation, and real-world maintainability.

**Answer:**

Angular testing isn't "just use Jasmine and Karma" - it's a sophisticated framework with TestBed creating isolated testing modules, ComponentFixture managing change detection, and various async testing strategies. Production-grade testing requires understanding internal mechanics, not just writing specs.

**For comprehensive coverage of TestBed internals, standalone component testing, async patterns, mocking strategies, flaky test prevention, OnPush testing, and a complete real-world example with HTTP + RxJS, see [Testing Strategy Deep Dive](./testing-strategy.md)**

**TestBed Internal Flow:**

```typescript
// When you write:
TestBed.configureTestingModule({ declarations: [MyComponent] });
fixture = TestBed.createComponent(MyComponent);

// Internal flow:
1. Store declarations/imports/providers
2. Create dynamic @NgModule
3. Compile module with TestingCompiler
4. Create NgModuleRef instance
5. Get ComponentFactory
6. Create ComponentRef with module's injector
7. Wrap in ComponentFixture
8. fixture.detectChanges() → NgZone.run() → CD
```

**Async Testing Strategies:**

```typescript
// ❌ FAILS: Synchronous assertion on async operation
it('bad', () => {
  component.loadData();  // Async HTTP call
  expect(component.data).toBeDefined();  // FAILS!
});

// ✅ fakeAsync + tick (synchronous control)
it('good', fakeAsync(() => {
  component.loadData();
  tick();  // Advance virtual time
  expect(component.data).toBeDefined();
}));

// ✅ async + whenStable (wait for real async)
it('also good', async(() => {
  component.loadData();
  fixture.whenStable().then(() => {
    expect(component.data).toBeDefined();
  });
}));
```

**When tick/flush/flushMicrotasks Fail:**

```typescript
// tick() - Specific time advancement
tick(1000);  // Advance 1 second

// flush() - Execute ALL pending timers
flush();  // ❌ Fails with infinite intervals!

// flushMicrotasks() - Execute only Promises
flushMicrotasks();  // ❌ Doesn't execute setTimeout!
```

**Mocking Strategy:**

```typescript
// ✅ DO mock: External dependencies
// - HTTP (use HttpTestingController)
// - Complex child components
// - Time-dependent code (fakeAsync)

// ❌ DON'T mock: Simple utilities, pure pipes, business logic

// HttpTestingController (best for HTTP)
it('should handle HTTP', fakeAsync(() => {
  component.loadData();
  
  const req = httpMock.expectOne('/api/data');
  req.flush({ data: 'test' });
  tick();
  
  expect(component.data).toBe('test');
}));
```

**Flaky Test Prevention:**

```typescript
// Common causes:
1. Timing issues → Use fakeAsync + tick
2. Uncontrolled CD → Explicit detectChanges()
3. Shared state → Clean up in afterEach()
4. Random data → Use deterministic fixtures
5. Missing httpMock.verify() → Always verify

// Checklist:
✅ fakeAsync/tick instead of real timers
✅ Explicit change detection
✅ Clean side effects (localStorage, etc.)
✅ Verify HTTP requests in afterEach()
✅ Deterministic test data
✅ Isolate tests (no global state)
```

**Testing OnPush Components:**

```typescript
// OnPush blocks CD unless:
1. @Input reference changes
2. Event fires in component
3. Async pipe emits
4. markForCheck() called

// Solution in tests:
fixture.componentRef.markForCheck();
fixture.detectChanges();
```

**Complete Real-World Example:**

Component with HTTP, RxJS operators (debounceTime, switchMap, catchError), and view updates - complete test suite covering:
- Debouncing (300ms)
- Filtering (<3 chars)
- Duplicate query skipping
- Request cancellation (switchMap)
- Loading states
- Error handling
- Results display
- Memory leak prevention
- Full integration flow

**Test Coverage Reality:**

**Would I trust 100% coverage? NO.**

**Why:**
1. Coverage ≠ Quality (can execute code without verifying anything)
2. Missing edge cases (divide by zero, null, etc.)
3. Integration gaps (units work, but together?)
4. Can't test UX, performance, security, real users

**What gives confidence:**
- Test pyramid (60% unit, 30% integration, 10% E2E)
- Critical path coverage (auth, payment, data submission)
- Mutation testing (detect weak assertions)
- Production monitoring (real user metrics)
- Target: 80-90% coverage with strategic focus

**For complete coverage including:**
- TestBed/ComponentFixture internal implementation
- Standalone vs module-based testing differences
- fakeAsync internals and Zone.js behavior
- Complete mocking strategies with trade-offs
- Flaky test causes and prevention checklist
- detectChanges() vs autoDetectChanges()
- OnPush testing strategies
- Full HTTP + RxJS component test suite
- Critical analysis of test coverage

**See the [Testing Strategy Deep Dive](./testing-strategy.md)**

---

### Question: Real-World Debugging & Architecture (No shortcuts, no guesses) - Let's say this is your scenario: Your Angular app has been running in production for a few months. After ~30 minutes of continuous use, the browser tab starts lagging and eventually crashes due to memory exhaustion. There's no obvious culprit in the code.

Walk me through exactly how you'd tackle this. I want you to cover:
- How you'd detect and reproduce the memory leak locally
- Which browser tools or profiling techniques you'd use (be specific — not just "Chrome DevTools")
- Common Angular patterns that cause leaks (give at least 5)
- How to identify leaked subscriptions, detached DOM elements, or zombie components
- How ChangeDetectionStrategy, RxJS misuse, or incorrect DI scoping could be contributing
- How you'd fix the issue — both immediate mitigation and long-term architectural change
- How to validate that the fix actually worked before redeploying

Then I want a real-world story (if you have one):
- A time when you encountered a tricky leak or performance issue in Angular.
- What caused it, how you found it, and what you changed.

Finally, defend your answer:
- Would you hotfix this in production or schedule a refactor?
- How do you ensure your team never ships this kind of problem again?

If your answer is "unsubscribe in ngOnDestroy," that's surface-level and won't cut it. I want your full forensic process — like a senior engineer would do under pressure.

**Answer:**

This is a production crisis requiring systematic forensic analysis, not guesswork. Memory leaks in production are silent killers that require methodical detection, reproduction, profiling, root cause analysis, and both immediate and long-term fixes.

**For the complete forensic process including detection techniques, specific Chrome DevTools usage (heap snapshots, allocation timeline, detached DOM inspector), 7+ common leak patterns, identification strategies, root cause analysis, hotfix vs refactor decision framework, real-world war story, and comprehensive prevention strategy, see [Debugging Memory Leaks Deep Dive](./debugging-memory-leaks.md)**

**The Forensic Process:**

**1. Detection & Evidence Gathering:**
```typescript
// Gather from production:
- Error tracking (Sentry): "Out of memory" after ~30 min
- RUM data: Correlates with /dashboard route
- Browser patterns: Mostly Chrome users
- User behavior: Tab switching + modal opening pattern
```

**2. Reproduction Locally:**
```typescript
// Automated reproduction script
for (let i = 0; i < 100; i++) {
  cy.get('[data-tab="overview"]').click();
  cy.get('[data-tab="details"]').click();
  cy.get('[data-action="open-modal"]').click();
  cy.get('[data-action="close-modal"]').click();
  
  // Memory should stabilize
  // Actual: 50MB → 500MB → crash
}
```

**3. Chrome DevTools Arsenal (Specific Tools):**
- **Heap Snapshots** - Compare before/after to see growing objects
- **Allocation Timeline** - See continuous allocation without GC
- **Performance Tab** - Identify heavy functions (ngDoCheck called 1500 times!)
- **Memory Timeline** - See sawtooth pattern with rising baseline
- **Detached DOM Inspector** - Find nodes removed from DOM but still in memory
- **Allocation Sampling** - Lightweight profiling of allocations

**4. Common Angular Leak Patterns (7+):**
```typescript
1. ❌ Unsubscribed Observables
   - HTTP, intervals, WebSockets without takeUntil

2. ❌ Event Listeners Never Removed
   - window.addEventListener without cleanup

3. ❌ Timers Never Cleared
   - setInterval, setTimeout chains, requestAnimationFrame

4. ❌ Third-Party Libraries Not Destroyed
   - Chart.js, D3, Leaflet without .destroy()

5. ❌ Incorrect DI Scoping
   - Component-provided service with unbounded cache

6. ❌ Detached DOM References
   - Storing element references that prevent GC

7. ❌ Unbounded Data Structures
   - Arrays growing forever (historical data)
```

**5. Root Causes:**

**CD Strategy Masking Leaks:**
```typescript
// OnPush makes leaks silent
// Default CD = slowdown noticed quickly
// OnPush CD = leak accumulates unnoticed until crash
```

**RxJS Misuse:**
```typescript
// Nested subscriptions, uncompleted Subjects
// shareReplay without refCount
// Infinite retry() loops
```

**DI Scoping:**
```typescript
// Module-scoped service with per-component state
// State accumulates across 100s of component instances
```

**6. The Fix - DO BOTH:**

**Immediate Hotfix (2-4 hours):**
```typescript
- Add memory monitoring and alerts
- Implement circuit breaker
- Auto-refresh on high memory
- Fix worst offenders
```

**Long-Term Refactor (1-2 weeks):**
```typescript
- Implement BaseComponent with cleanup
- Add ESLint rules for leak prevention
- Create SubscriptionManager utility
- Add automated memory tests to CI/CD
- Team training and guidelines
```

**7. Validation:**
```typescript
// Automated tests
it('should not leak after 100 create/destroy cycles', () => {
  expect(memoryGrowth).toBeLessThan(10MB);
});

// Staging E2E
// 30 minutes of use → memory stabilizes < 200MB

// Production monitoring
// p99 memory: 1850MB → 210MB (88% improvement!)
```

**Real-World War Story:**

**The WebSocket Trading App Crisis:**
- Financial trading dashboard crashing after 20-30 minutes
- Cost: $10,000+ per crash in lost trades
- Root cause: Unbounded array storing all price updates
  - 10 updates/sec × 30 min = 18,000 items × 5KB = 1.2GB
- Also: 450 leaked subscriptions from destroyed components
- **Fix:** Circular buffer (keep last 1000) + proper cleanup
- **Result:** 1.8GB → 85MB (95% reduction), crashes eliminated

**Hotfix vs Refactor? DO BOTH**
- Hotfix stops immediate bleeding
- Refactor prevents recurrence
- This is the senior engineer answer

**Prevention Strategy:**
- Code review checklist (subscriptions, events, timers, DOM refs, DI)
- ESLint rules (require-subscription-cleanup, no-unbounded-arrays)
- Automated memory tests in CI/CD
- Team training (3-week curriculum)
- Continuous production monitoring

**For the complete step-by-step forensic process with all tools, techniques, patterns, real code examples, and prevention strategies, see [Debugging Memory Leaks Deep Dive](./debugging-memory-leaks.md)**

---

### Question: Modern Angular Features & Evolving Fundamentals (Let's separate the up-to-date from the outdated) - Angular has evolved massively in the last few versions — signals, standalone components, zoneless change detection, deferred loading, hydration, and more. Let's test whether you've evolved with it.

I want you to explain the following modern Angular concepts clearly — with comparisons, internal behavior, and migration reasoning.

🔹 1. Standalone Components
- What are standalone components, and how do they differ from NgModules?
- How does Angular's compiler handle dependency resolution and scope for standalone components?
- How do you integrate them with routing and lazy loading?
- Would you migrate an existing large NgModule-based app to standalone components? Why or why not?

🔹 2. Angular Signals
- Explain what signals are and why Angular introduced them.
- Compare them to RxJS Observables — both conceptually and in terms of change detection triggering.
- How do signal, computed, and effect work internally?
- What problem do signals solve that Zone.js and ChangeDetectorRef could not?
- Would you mix signals and RxJS in one app? If yes, how would you bridge them effectively?

🔹 3. Zoneless Angular (Zone.js Opt-out)
- What is Zone.js doing today in Angular apps, and how does Angular's new zoneless change detection model replace it?
- How does runOutsideAngular() differ in purpose from removing Zone.js entirely?
- What are the tradeoffs of going zoneless — and how would you manually trigger change detection without it?

🔹 4. Deferred Loading & Hydration
- What is deferred loading (introduced in Angular 17+)? How does it differ from traditional lazy loading?
- Explain server-side hydration — what problem does it solve, and how does Angular handle it internally?
- How would you debug hydration mismatches between server and client?

🔹 5. Signals vs Observables — Real Use Case
- You have a dashboard with multiple widgets updating in real-time: When would you pick signals over observables, and vice versa?
- How would you architect a hybrid system that leverages both efficiently?

And finally, a judgment question —
- You're building a new enterprise Angular 18 app from scratch. Would you go fully standalone, zoneless, and signal-based — or stick to the old stable ecosystem? Defend your decision technically and practically (think: team maturity, migration path, long-term maintainability).

If you haven't used signals or standalone APIs hands-on, I'll know immediately. If your answer is "I've read about them," that's a red flag — I want real, production-level reasoning.

Go — start with signals. What exactly makes them different from observables under the hood?

**Answer:**

Angular has fundamentally evolved. These aren't just "new features" - they represent architectural shifts in how Angular handles reactivity, modularity, and performance.

**For comprehensive coverage of all modern Angular features including internal signal mechanics, standalone component compilation, zoneless architecture, deferred loading, SSR hydration, real-world use cases, and production architecture decisions, see [Modern Angular Features Deep Dive](./modern-angular-features.md)**

**Signals vs Observables - Under the Hood:**

```typescript
// Signal: Pull-based reactive primitive with automatic dependency tracking

class Signal<T> {
  private _value: T;
  private _subscribers = new Set<Computation>();
  
  get(): T {
    // Track dependency automatically
    if (CURRENT_COMPUTATION) {
      this._subscribers.add(CURRENT_COMPUTATION);
    }
    return this._value;
  }
  
  set(newValue: T): void {
    this._value = newValue;
    // Notify dependents to mark dirty
    this._subscribers.forEach(c => c.markDirty());
    scheduleChangeDetection();
  }
}

// Key Architectural Differences:
{
  observables: {
    model: "Push (producer pushes values)",
    execution: "Eager (pushes immediately)",
    subscriptions: "Manual subscribe/unsubscribe",
    memoryLeaks: "Possible if not unsubscribed",
    changeDetection: "Async pipe calls markForCheck()"
  },
  
  signals: {
    model: "Pull (consumer pulls values)",
    execution: "Lazy (pulls when needed)",
    subscriptions: "Automatic dependency tracking",
    memoryLeaks: "Not possible (no subscriptions)",
    changeDetection: "Built-in (no markForCheck needed)"
  }
}
```

**Problem Signals Solve:**

1. **Zone.js Overhead** - No need for global change detection
2. **Manual Subscriptions** - No subscribe/unsubscribe needed
3. **CD Noise** - Only affected DOM nodes update (fine-grained reactivity)

**Standalone Components:**

```typescript
// Traditional: NgModule-based
@NgModule({
  declarations: [MyComponent],
  imports: [CommonModule, FormsModule]
})

// Modern: Standalone
@Component({
  standalone: true,
  imports: [CommonModule, FormsModule]  // Direct imports
})
export class MyComponent {}

// Benefits: Clearer dependencies, better tree-shaking, component-level code splitting
```

**Zoneless Angular:**

```typescript
// What Zone.js does: Monkey-patches ~40 browser APIs
// Overhead: ~200KB + wrapper execution on every event

// Zoneless (Angular 17+):
bootstrapApplication(AppComponent, {
  providers: [provideExperimentalZonelessChangeDetection()]
});

// Trade-offs:
// ✅ No Zone.js overhead, more predictable performance
// ❌ Must use Signals or manual markForCheck()
// Decision: Not yet for enterprise (still experimental)
```

**Deferred Loading:**

```typescript
@Component({
  template: `
    @defer (on viewport) {
      <app-heavy-chart />
    } @placeholder {
      <div>Loading...</div>
    }
  `
})

// vs Traditional lazy loading (route-level)
// Deferred = component-level chunks with declarative triggers
```

**Production Decision for New Angular 18 App:**

```typescript
{
  standalone: "YES - 100%",
  reason: "Future-proof, clearer dependencies, better code splitting",
  
  signals: "YES - For state management",
  reason: "Fine-grained reactivity, no memory leaks, simpler",
  
  observables: "YES - For async operations",
  reason: "HTTP, WebSocket, debounce - natural fit",
  
  zoneless: "NO - Not yet",
  reason: "Still experimental, team learning curve, third-party compatibility",
  
  hydration: "YES - SSR with hydration",
  reason: "SEO + performance, mature feature",
  
  deferredLoading: "YES - Aggressively",
  reason: "Easy wins, better initial load"
}

// Rationale: Balance modern features with enterprise stability
// Use proven features (Signals, Standalone)
// Defer experimental ones (Zoneless) until mature
```

**Hybrid Architecture Pattern:**

```typescript
// Data layer - Observables
http.get('/api/data').pipe(retry(3), catchError(...))

// State layer - Signals
count = signal(0);
double = computed(() => count() * 2);

// Bridge layer - toSignal/toObservable
data = toSignal(http.get(...), { initialValue: null });

// Use Signals for: State, derived values, UI
// Use Observables for: HTTP, WebSocket, complex async
// Bridge at boundaries
```

**For complete internal mechanics including:**
- Signal dependency tracking algorithm
- Standalone component compilation process
- Zone.js monkey-patching details
- Zoneless change detection model
- SSR hydration internals and mismatch debugging
- Real trading dashboard architecture (Signals + Observables)
- Complete production decision framework with team considerations

**See the [Modern Angular Features Deep Dive](./modern-angular-features.md)**

---

## Additional Topics

Continue adding questions for:
- Directives (Structural & Attribute)
- Pipes (Built-in & Custom)
- Forms (Template-driven & Reactive)
- Routing & Navigation
- HTTP Client & Interceptors
- State Management (RxJS, NgRx)
- Testing (Unit & E2E)
- Performance Optimization
