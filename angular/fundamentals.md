# Angular Fundamentals

## Table of Contents
- [What is Angular](#what-is-angular)
- [Angular Architecture](#angular-architecture)
- [Components](#components)
- [Modules](#modules)
- [Dependency Injection](#dependency-injection)
- [Data Binding](#data-binding)

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

### Question: Explain Angular component lifecycle hooks and when to use each one.

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

### Question: Explain content projection and how Angular internally handles it.

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

### Question: What are the real technical causes of Angular performance degradation and how do you fix them?

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

### Question: Explain how NgRx integrates with Angular - walk through the complete data flow from dispatch to UI update.

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

### Question: How do you design and structure production-grade tests for Angular applications?

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
