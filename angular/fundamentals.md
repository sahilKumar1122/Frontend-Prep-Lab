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

**Lifecycle Hook Order:**

```typescript
import { 
  Component, OnInit, OnChanges, DoCheck, AfterContentInit,
  AfterContentChecked, AfterViewInit, AfterViewChecked, OnDestroy,
  SimpleChanges, Input, ViewChild, ContentChild
} from '@angular/core';

@Component({
  selector: 'app-lifecycle-demo',
  template: `
    <div>
      <h2>{{ title }}</h2>
      <ng-content></ng-content>
      <div #outputDiv></div>
    </div>
  `
})
export class LifecycleDemoComponent implements 
  OnChanges, OnInit, DoCheck, AfterContentInit, AfterContentChecked,
  AfterViewInit, AfterViewChecked, OnDestroy {
  
  @Input() title: string = '';
  @Input() data: any;
  @ViewChild('outputDiv') outputDiv!: ElementRef;
  @ContentChild('projectedContent') projectedContent!: ElementRef;
  
  // 1. Called before ngOnInit when input properties change
  ngOnChanges(changes: SimpleChanges): void {
    console.log('1. ngOnChanges', changes);
    
    // Use case: React to input changes
    if (changes['data'] && !changes['data'].firstChange) {
      this.processDataChange(changes['data'].currentValue);
    }
  }
  
  // 2. Called once after first ngOnChanges
  ngOnInit(): void {
    console.log('2. ngOnInit');
    
    // ✅ BEST PRACTICE: Initialize component here
    // - Fetch data from services
    // - Set up subscriptions
    // - Initialize complex properties
    this.loadInitialData();
  }
  
  // 3. Called during every change detection run
  ngDoCheck(): void {
    console.log('3. ngDoCheck');
    
    // ⚠️ WARNING: Called very frequently - keep logic minimal
    // Use case: Detect changes Angular doesn't catch (nested objects)
  }
  
  // 4. Called after content (ng-content) is projected
  ngAfterContentInit(): void {
    console.log('4. ngAfterContentInit');
    
    // Use case: Access @ContentChild elements
    if (this.projectedContent) {
      console.log('Projected content ready');
    }
  }
  
  // 5. Called after every check of projected content
  ngAfterContentChecked(): void {
    console.log('5. ngAfterContentChecked');
    // ⚠️ Avoid heavy operations here
  }
  
  // 6. Called after component's view is initialized
  ngAfterViewInit(): void {
    console.log('6. ngAfterViewInit');
    
    // ✅ BEST PRACTICE: Access @ViewChild elements here
    if (this.outputDiv) {
      this.outputDiv.nativeElement.textContent = 'View ready!';
    }
    
    // Initialize third-party libraries (charts, maps, etc.)
    this.initializeChart();
  }
  
  // 7. Called after every check of component's view
  ngAfterViewChecked(): void {
    console.log('7. ngAfterViewChecked');
    // ⚠️ Avoid state changes here (causes ExpressionChangedAfterItHasBeenCheckedError)
  }
  
  // 8. Called just before component is destroyed
  ngOnDestroy(): void {
    console.log('8. ngOnDestroy');
    
    // ✅ CRITICAL: Cleanup here
    // - Unsubscribe from observables
    // - Clear timers
    // - Detach event listeners
    // - Cancel pending HTTP requests
    this.cleanup();
  }
  
  private loadInitialData(): void {
    // Fetch data logic
  }
  
  private processDataChange(newData: any): void {
    // Handle data changes
  }
  
  private initializeChart(): void {
    // Third-party library initialization
  }
  
  private cleanup(): void {
    // Cleanup logic
  }
}
```

**Common Real-World Use Cases:**

```typescript
// Use Case 1: Data Fetching Component
@Component({
  selector: 'app-user-profile',
  template: `...`
})
export class UserProfileComponent implements OnInit, OnDestroy {
  @Input() userId: string = '';
  user$ = new Subject<User>();
  private destroy$ = new Subject<void>();
  
  constructor(private userService: UserService) {}
  
  ngOnInit(): void {
    // ✅ Best place to fetch data
    this.userService.getUser(this.userId)
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => this.user$.next(user));
  }
  
  ngOnDestroy(): void {
    // ✅ Prevent memory leaks
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Use Case 2: Integration with Third-Party Libraries
@Component({
  selector: 'app-chart',
  template: `<div #chartContainer></div>`
})
export class ChartComponent implements AfterViewInit, OnDestroy {
  @ViewChild('chartContainer') chartContainer!: ElementRef;
  private chart: any;
  
  ngAfterViewInit(): void {
    // ✅ DOM is ready, safe to initialize chart
    this.chart = new ThirdPartyChart(this.chartContainer.nativeElement, {
      data: this.data,
      options: this.options
    });
  }
  
  ngOnDestroy(): void {
    // ✅ Destroy chart instance
    if (this.chart) {
      this.chart.destroy();
    }
  }
}

// Use Case 3: Performance Optimization
@Component({
  selector: 'app-optimized-list',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OptimizedListComponent implements OnChanges {
  @Input() items: Item[] = [];
  processedItems: ProcessedItem[] = [];
  
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['items']) {
      // ✅ Only reprocess when input actually changes
      this.processedItems = this.processItems(changes['items'].currentValue);
    }
  }
  
  private processItems(items: Item[]): ProcessedItem[] {
    // Expensive computation only when needed
    return items.map(item => this.transform(item));
  }
}
```

**Performance Optimization Tips:**

1. **Avoid ngDoCheck and ngAfterViewChecked** - Called on every change detection
2. **Use OnPush change detection** - Reduces checks to input changes only
3. **Unsubscribe in ngOnDestroy** - Critical for preventing memory leaks
4. **Don't change state in ngAfterViewChecked** - Causes infinite loops

**Pro Tip:** The most common mistake is forgetting `ngOnDestroy` cleanup. In large SPAs, this causes memory leaks that accumulate over time. Always use the **takeUntil pattern** with a Subject for automatic observable cleanup, or use the **async pipe** which handles subscription lifecycle automatically. Senior engineers look for this in code reviews.

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
