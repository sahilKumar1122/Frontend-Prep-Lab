# Constructor vs ngOnInit - Deep Dive

## Table of Contents
- [Component Initialization Flow](#component-initialization-flow)
- [Constructor Purpose and Timing](#constructor-purpose-and-timing)
- [ngOnInit Purpose and Timing](#ngoninit-purpose-and-timing)
- [The @Input Trap](#the-input-trap)
- [Async ngOnInit](#async-ngoninit)
- [Dependency Injection Timing](#dependency-injection-timing)
- [Real-World Scenarios](#real-world-scenarios)
- [Common Mistakes](#common-mistakes)

---

## Component Initialization Flow

### Question: Explain the difference between ngOnInit() and the constructor in Angular components. What is each used for? When exactly does Angular call them? Cover scenarios requiring each, the @Input trap, async ngOnInit, and DI timing.

**Answer:**

Let's trace the EXACT sequence of events when Angular creates a component, from the moment it decides to instantiate it until it's fully initialized.

---

## **The Complete Component Creation Timeline**

### **Step-by-Step: What Angular Does Internally**

```typescript
// Simplified internal Angular component creation flow
class ComponentFactory {
  createComponent(componentType: Type<any>, injector: Injector): ComponentRef {
    // STEP 1: Resolve dependencies from injector
    const dependencies = this.resolveDependencies(componentType, injector);
    // At this point: DI is ready, but component doesn't exist yet
    
    // STEP 2: Call constructor with dependencies
    const componentInstance = new componentType(...dependencies);
    // At this point: 
    // - Component object exists in memory
    // - Properties are initialized to their default values
    // - Constructor has executed
    // - @Input properties are UNDEFINED
    // - @ViewChild/@ContentChild are UNDEFINED
    
    // STEP 3: Set input properties
    this.setInputs(componentInstance, inputValues);
    // At this point: @Input properties are now available
    
    // STEP 4: Call ngOnChanges (if component implements OnChanges)
    if (componentInstance.ngOnChanges) {
      componentInstance.ngOnChanges(changes);
    }
    
    // STEP 5: Call ngOnInit (if component implements OnInit)
    if (componentInstance.ngOnInit) {
      componentInstance.ngOnInit();
    }
    // At this point: Component is considered "initialized"
    
    // STEP 6: Continue with other lifecycle hooks...
    // ngDoCheck, ngAfterContentInit, ngAfterViewInit, etc.
    
    return componentRef;
  }
}
```

### **Visual Timeline**

```
Time →
│
├─ Constructor() called
│   │
│   ├─ DI happens HERE
│   ├─ Class fields initialized
│   ├─ @Input: undefined ❌
│   ├─ @ViewChild: undefined ❌
│   └─ Constructor body executes
│
├─ @Input properties set by Angular
│   └─ @Input: available ✅
│
├─ ngOnChanges() called
│   └─ SimpleChanges object passed
│
├─ ngOnInit() called
│   │
│   ├─ @Input: available ✅
│   ├─ @ViewChild: undefined ❌ (view not ready yet)
│   └─ ngOnInit body executes
│
├─ ngDoCheck() called
│
├─ ngAfterContentInit() called
│   └─ @ContentChild: available ✅
│
├─ ngAfterViewInit() called
│   └─ @ViewChild: available ✅
│
└─ Component fully initialized
```

---

## Constructor Purpose and Timing

### **What Is the Constructor?**

The constructor is a **TypeScript/JavaScript feature**, NOT an Angular feature. It's called by the JavaScript engine when `new Component()` is invoked.

```typescript
@Component({
  selector: 'app-user-profile',
  template: `...`
})
export class UserProfileComponent {
  // These are initialized BEFORE constructor body runs
  name = 'Default Name';  // Class field initialization
  count = 0;
  items: string[] = [];
  
  constructor(
    private userService: UserService,      // DI happens here
    private http: HttpClient,              // DI happens here
    @Inject(API_URL) private apiUrl: string  // DI happens here
  ) {
    // Constructor body runs AFTER DI
    console.log('Constructor executed');
    console.log('UserService:', this.userService);  // ✅ Available
    console.log('Name:', this.name);  // ✅ Available (class field)
    console.log('Count:', this.count);  // ✅ Available (class field)
  }
}
```

### **What Happens in the Constructor**

```typescript
// Detailed execution order
export class DetailedComponent {
  // 1. Class fields initialized (in declaration order)
  public title = 'Default Title';
  private counter = 0;
  
  // 2. These are NOT initialized yet (set by Angular later)
  @Input() userId!: string;
  @ViewChild('element') element!: ElementRef;
  
  // 3. Constructor called with resolved dependencies
  constructor(
    private service: MyService,
    private http: HttpClient
  ) {
    // 4. Constructor body executes
    
    // ✅ AVAILABLE: Class fields
    console.log(this.title);     // "Default Title"
    console.log(this.counter);   // 0
    
    // ✅ AVAILABLE: Injected dependencies
    console.log(this.service);   // MyService instance
    console.log(this.http);      // HttpClient instance
    
    // ❌ NOT AVAILABLE: @Input properties
    console.log(this.userId);    // undefined ❌
    
    // ❌ NOT AVAILABLE: @ViewChild/@ContentChild
    console.log(this.element);   // undefined ❌
    
    // ❌ DON'T DO THIS: API calls based on @Input
    // this.http.get(`/api/users/${this.userId}`);  // userId is undefined!
  }
}
```

### **Constructor: What You SHOULD Do**

```typescript
@Component({
  selector: 'app-example',
  template: `...`
})
export class ExampleComponent {
  // ✅ GOOD: Initialize class properties
  private isInitialized = false;
  private cache = new Map<string, any>();
  
  constructor(
    private userService: UserService,
    private logger: LoggerService,
    @Optional() private config?: AppConfig
  ) {
    // ✅ GOOD: Store injected dependencies (already done by TypeScript)
    // (No action needed - parameters are automatically assigned to class properties)
    
    // ✅ GOOD: Initialize non-Angular services
    this.cache.clear();
    
    // ✅ GOOD: Set up static configuration
    if (this.config) {
      this.applyConfig(this.config);
    }
    
    // ✅ GOOD: Log for debugging
    this.logger.log('Component constructor called');
    
    // ⚠️ ACCEPTABLE: Setup that doesn't depend on inputs
    this.setupStaticEventListeners();
  }
  
  private applyConfig(config: AppConfig): void {
    // Apply configuration that doesn't depend on @Input
  }
  
  private setupStaticEventListeners(): void {
    // Setup that doesn't need component inputs
  }
}
```

### **Constructor: What You SHOULD NOT Do**

```typescript
@Component({
  selector: 'app-bad-example',
  template: `...`
})
export class BadExampleComponent {
  @Input() userId!: string;
  @ViewChild('element') element!: ElementRef;
  
  constructor(
    private http: HttpClient,
    private userService: UserService
  ) {
    // ❌ BAD: Access @Input properties
    console.log(this.userId);  // undefined!
    
    // ❌ BAD: Make HTTP calls
    this.http.get('/api/data').subscribe();
    // Why bad? Can't use @Input values, wastes resources if component destroyed quickly
    
    // ❌ BAD: Access DOM elements
    this.element.nativeElement.focus();  // undefined! Crash!
    
    // ❌ BAD: Access parent/child components
    // (They don't exist yet)
    
    // ❌ BAD: Complex initialization logic
    this.initializeComplexFeature();
    // Why bad? If component is never actually used, you wasted resources
    
    // ❌ BAD: Async operations
    this.loadDataAsync();
    // Why bad? Can't await in constructor, error handling is difficult
    
    // ❌ BAD: Subscribe to observables
    this.userService.currentUser$.subscribe(user => {
      this.processUser(user);
    });
    // Why bad? Memory leak if component destroyed before unsubscribe
  }
}
```

---

## ngOnInit Purpose and Timing

### **What Is ngOnInit?**

`ngOnInit` is an **Angular lifecycle hook** that's called ONCE after:
1. Constructor has executed
2. First change detection has run
3. All `@Input` properties have been set

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div>
      <h1>{{ userName }}</h1>
      <p>ID: {{ userId }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId!: string;
  userName = '';
  
  constructor(private userService: UserService) {
    console.log('1. Constructor');
    console.log('   userId:', this.userId);  // undefined ❌
  }
  
  ngOnInit(): void {
    console.log('2. ngOnInit');
    console.log('   userId:', this.userId);  // "123" ✅
    
    // ✅ NOW we can use @Input properties
    this.loadUserData();
  }
  
  private loadUserData(): void {
    this.userService.getUser(this.userId).subscribe(user => {
      this.userName = user.name;
    });
  }
}

// Usage:
// <app-user-profile [userId]="'123'"></app-user-profile>

// Console output:
// 1. Constructor
//    userId: undefined
// 2. ngOnInit
//    userId: 123
```

### **ngOnInit: What You SHOULD Do**

```typescript
@Component({
  selector: 'app-dashboard',
  template: `...`
})
export class DashboardComponent implements OnInit, OnDestroy {
  @Input() dashboardId!: string;
  @Input() config?: DashboardConfig;
  
  data$!: Observable<Data>;
  private destroy$ = new Subject<void>();
  
  constructor(
    private dataService: DataService,
    private analytics: AnalyticsService,
    private websocketService: WebSocketService
  ) {}
  
  ngOnInit(): void {
    // ✅ GOOD: Initialize observables based on @Input
    this.data$ = this.dataService.getData(this.dashboardId);
    
    // ✅ GOOD: Make HTTP requests using @Input values
    this.loadDashboardData();
    
    // ✅ GOOD: Set up subscriptions
    this.setupRealtimeUpdates();
    
    // ✅ GOOD: Track analytics with @Input data
    this.analytics.trackView('dashboard', {
      dashboardId: this.dashboardId,
      config: this.config
    });
    
    // ✅ GOOD: Initialize complex features
    this.initializeCharts();
    
    // ✅ GOOD: Apply configuration from @Input
    if (this.config) {
      this.applyConfiguration(this.config);
    }
  }
  
  private loadDashboardData(): void {
    this.dataService.getDashboard(this.dashboardId)
      .pipe(takeUntil(this.destroy$))
      .subscribe(dashboard => {
        this.processDashboard(dashboard);
      });
  }
  
  private setupRealtimeUpdates(): void {
    this.websocketService.connect(`dashboard/${this.dashboardId}`)
      .pipe(takeUntil(this.destroy$))
      .subscribe(update => {
        this.handleUpdate(update);
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### **ngOnInit: What You SHOULD NOT Do**

```typescript
@Component({
  selector: 'app-bad-init',
  template: `...`
})
export class BadInitComponent implements OnInit {
  @ViewChild('element') element!: ElementRef;
  
  constructor(private service: MyService) {}
  
  ngOnInit(): void {
    // ❌ BAD: Access @ViewChild/@ContentChild
    this.element.nativeElement.focus();
    // Why bad? DOM not ready yet - use ngAfterViewInit
    
    // ❌ BAD: Do DI (already done in constructor)
    // Can't inject here anyway
    
    // ❌ BAD: Long blocking operations
    for (let i = 0; i < 1000000; i++) {
      this.doExpensiveCalculation(i);
    }
    // Why bad? Blocks UI, delays rendering
    
    // ❌ BAD: Unmanaged subscriptions
    this.service.getData().subscribe();
    // Why bad? Memory leak - no cleanup
    
    // ⚠️ QUESTIONABLE: Change @Input values
    this.someInput = 'modified';
    // Why questionable? Parent controls inputs, confusing data flow
  }
}
```

---

## The @Input Trap

### **The Most Common Mistake**

```typescript
// ❌ WRONG: Accessing @Input in constructor
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserCardComponent {
  @Input() userId!: string;
  user: User | null = null;
  
  constructor(private userService: UserService) {
    // ❌ THIS WILL FAIL!
    console.log('User ID:', this.userId);  // undefined
    
    // ❌ THIS WILL CRASH!
    this.userService.getUser(this.userId).subscribe(user => {
      this.user = user;  // Never executes because userId is undefined
    });
    
    // ❌ THIS ALSO FAILS!
    if (this.userId === 'admin') {  // Always false - userId is undefined
      this.loadAdminData();
    }
  }
}

// Usage:
// <app-user-card [userId]="'123'"></app-user-card>

// What happens:
// 1. Constructor runs → userId is undefined
// 2. API call: GET /api/users/undefined → 404 error
// 3. Component displays nothing
```

### **The Correct Way**

```typescript
// ✅ CORRECT: Using @Input in ngOnInit
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card" *ngIf="user">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
    </div>
    <div class="loading" *ngIf="!user">Loading...</div>
  `
})
export class UserCardComponent implements OnInit, OnDestroy {
  @Input() userId!: string;
  user: User | null = null;
  
  private destroy$ = new Subject<void>();
  
  constructor(private userService: UserService) {
    // ✅ GOOD: Just store the injected service
    console.log('Constructor: Service injected');
  }
  
  ngOnInit(): void {
    // ✅ GOOD: userId is now available
    console.log('ngOnInit: userId =', this.userId);  // "123"
    
    // ✅ GOOD: Make API call with userId
    this.userService.getUser(this.userId)
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        this.user = user;
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### **Why This Trap Exists**

```typescript
// Angular's internal process:

// Step 1: Create component
const component = new UserCardComponent(userService);
// At this point: component.userId = undefined

// Step 2: Set inputs (AFTER constructor)
component.userId = '123';
// Now: component.userId = '123'

// Step 3: Call ngOnInit
component.ngOnInit();
// Now inputs are available!

// Timeline:
// t=0:  new UserCardComponent() → constructor runs → userId = undefined
// t=1:  Angular sets userId = '123'
// t=2:  Angular calls ngOnInit() → userId = '123' ✅
```

### **Advanced: Detecting @Input Changes**

```typescript
@Component({
  selector: 'app-smart-component',
  template: `...`
})
export class SmartComponent implements OnInit, OnChanges {
  @Input() userId!: string;
  @Input() filter?: string;
  
  constructor(private service: MyService) {
    console.log('Constructor: userId =', this.userId);  // undefined
  }
  
  // Called BEFORE ngOnInit when inputs change
  ngOnChanges(changes: SimpleChanges): void {
    console.log('ngOnChanges:', changes);
    
    // First time called: userId changes from undefined → '123'
    if (changes['userId']) {
      console.log('userId changed:',
        changes['userId'].previousValue,  // undefined (first time)
        '→',
        changes['userId'].currentValue    // '123'
      );
      
      // ✅ GOOD: React to input changes
      if (!changes['userId'].firstChange) {
        // Not the first change - user changed the userId
        this.loadNewUserData(changes['userId'].currentValue);
      }
    }
  }
  
  ngOnInit(): void {
    // ✅ GOOD: Initial data load
    this.loadUserData();
  }
  
  private loadUserData(): void {
    console.log('Loading user:', this.userId);  // '123' ✅
  }
  
  private loadNewUserData(newUserId: string): void {
    console.log('User changed to:', newUserId);
  }
}

// Lifecycle order:
// 1. constructor() → userId = undefined
// 2. Angular sets userId = '123'
// 3. ngOnChanges() → userId = '123' (firstChange = true)
// 4. ngOnInit() → userId = '123'
```

---

## Async ngOnInit

### **Can ngOnInit Be Async?**

**Technically yes, but with caveats.**

```typescript
// ✅ TECHNICALLY ALLOWED: async ngOnInit
@Component({
  selector: 'app-async',
  template: `...`
})
export class AsyncComponent implements OnInit {
  data: any;
  
  async ngOnInit(): Promise<void> {
    console.log('ngOnInit start');
    
    // Angular will NOT wait for this
    this.data = await this.loadData();
    
    console.log('ngOnInit end');
  }
  
  private async loadData(): Promise<any> {
    return new Promise(resolve => {
      setTimeout(() => resolve({ name: 'Data' }), 1000);
    });
  }
}

// What happens:
// 1. ngOnInit starts executing
// 2. Hits await → returns Promise immediately
// 3. Angular continues with lifecycle (doesn't wait!)
// 4. ngDoCheck, ngAfterContentInit, etc. all run
// 5. Component renders with data = undefined
// 6. 1 second later: Promise resolves, data is set
// 7. Change detection eventually runs again, data appears
```

### **The Problem with Async ngOnInit**

```typescript
@Component({
  selector: 'app-problematic',
  template: `
    <div>{{ user.name }}</div>  <!-- ERROR: user is undefined initially -->
  `
})
export class ProblematicComponent implements OnInit {
  user!: User;
  
  async ngOnInit(): Promise<void> {
    // Angular doesn't wait - template renders before this completes
    this.user = await this.userService.getUser().toPromise();
  }
}

// Timeline:
// t=0:    ngOnInit starts
// t=0:    Template renders → user is undefined → ERROR
// t=100:  Promise resolves → user is set
// t=101:  Next change detection → template updates
```

### **When to Use Async ngOnInit**

```typescript
// ✅ ACCEPTABLE: When you handle loading states properly
@Component({
  selector: 'app-handled',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="!loading && user">{{ user.name }}</div>
    <div *ngIf="!loading && error">Error: {{ error }}</div>
  `
})
export class HandledComponent implements OnInit {
  user: User | null = null;
  loading = true;
  error: string | null = null;
  
  async ngOnInit(): Promise<void> {
    try {
      this.loading = true;
      
      // Load data
      this.user = await firstValueFrom(this.userService.getUser());
      
      this.loading = false;
    } catch (err) {
      this.error = (err as Error).message;
      this.loading = false;
    }
  }
}
```

### **Better: Use RxJS Instead of Async/Await**

```typescript
// ✅ BETTER: Use observables with async pipe
@Component({
  selector: 'app-reactive',
  template: `
    <div *ngIf="user$ | async as user; else loading">
      {{ user.name }}
    </div>
    <ng-template #loading>Loading...</ng-template>
  `
})
export class ReactiveComponent implements OnInit {
  user$!: Observable<User>;
  
  ngOnInit(): void {
    // ✅ GOOD: No async/await needed
    this.user$ = this.userService.getUser().pipe(
      catchError(err => {
        console.error('Error loading user:', err);
        return of(null);
      })
    );
    // async pipe handles subscription/unsubscription automatically
  }
}
```

### **Should ngOnInit Be Async?**

**Generally NO, prefer RxJS patterns:**

```typescript
// ❌ AVOID: Multiple async/await in ngOnInit
async ngOnInit(): Promise<void> {
  this.user = await firstValueFrom(this.userService.getUser());
  this.posts = await firstValueFrom(this.postService.getPosts(this.user.id));
  this.comments = await firstValueFrom(this.commentService.getComments());
  // Sequential loading - slow!
}

// ✅ BETTER: Use observables
ngOnInit(): void {
  this.data$ = this.userService.getUser().pipe(
    switchMap(user => 
      forkJoin({
        user: of(user),
        posts: this.postService.getPosts(user.id),
        comments: this.commentService.getComments()
      })
    )
  );
  // Parallel loading - fast!
  // Template: <div *ngIf="data$ | async as data">
}

// ✅ BEST: Separate observables
ngOnInit(): void {
  this.user$ = this.userService.getUser().pipe(shareReplay(1));
  this.posts$ = this.user$.pipe(
    switchMap(user => this.postService.getPosts(user.id))
  );
  this.comments$ = this.commentService.getComments();
  
  // Templates use async pipe for each
}
```

---

## Dependency Injection Timing

### **DI in Constructor vs ngOnInit**

**KEY POINT: You can ONLY do DI in the constructor.**

```typescript
@Component({
  selector: 'app-example',
  template: `...`
})
export class ExampleComponent implements OnInit {
  // ✅ CORRECT: DI in constructor
  constructor(
    private userService: UserService,
    private http: HttpClient,
    @Inject(API_URL) private apiUrl: string
  ) {
    // Dependencies are injected and available here
    console.log('UserService:', this.userService);  // ✅ Works
  }
  
  ngOnInit(): void {
    // ❌ WRONG: Can't do DI here
    // There's no syntax to inject dependencies in ngOnInit
    // You can only USE already-injected dependencies
    
    console.log('UserService:', this.userService);  // ✅ Still works (from constructor)
  }
}
```

### **Why DI Only Works in Constructor**

```typescript
// This is how Angular creates components:

// 1. Analyze constructor parameters (at compile time)
class ComponentFactory {
  createComponent(componentType: Type<any>): any {
    // 2. Get constructor metadata
    const ctorParams = Reflect.getMetadata('design:paramtypes', componentType);
    // ctorParams = [UserService, HttpClient, String]
    
    // 3. Resolve each dependency from injector
    const resolvedDeps = ctorParams.map(token => 
      this.injector.get(token)
    );
    // resolvedDeps = [userServiceInstance, httpClientInstance, apiUrlValue]
    
    // 4. Call constructor with resolved dependencies
    const instance = new componentType(...resolvedDeps);
    
    // 5. Constructor parameters are assigned to class properties
    // instance.userService = userServiceInstance
    // instance.http = httpClientInstance
    // instance.apiUrl = apiUrlValue
    
    return instance;
  }
}

// ngOnInit doesn't have parameters, so no DI!
```

### **Working with Services in ngOnInit**

```typescript
@Component({
  selector: 'app-dashboard',
  template: `...`
})
export class DashboardComponent implements OnInit {
  data: any;
  
  // STEP 1: Declare and inject in constructor
  constructor(
    private dataService: DataService,      // Injected here
    private authService: AuthService,      // Injected here
    private router: Router                 // Injected here
  ) {
    // STEP 2: Services are now available as class properties
    console.log('Services injected');
    
    // ⚠️ CAN use services, but SHOULDN'T do complex logic here
    if (!this.authService.isAuthenticated()) {
      // This is acceptable - quick sync check
      this.router.navigate(['/login']);
    }
  }
  
  ngOnInit(): void {
    // STEP 3: Use injected services for initialization
    // Services are accessible via this.serviceName
    
    this.data = this.dataService.getData();  // ✅ Works
    
    const user = this.authService.getCurrentUser();  // ✅ Works
    
    this.loadDashboard();  // ✅ Works
  }
  
  private loadDashboard(): void {
    // Can use services in any method
    this.dataService.loadDashboard().subscribe(dashboard => {
      this.data = dashboard;
    });
  }
}
```

### **Advanced: Manual Service Injection**

```typescript
// Rare case: Getting a service without constructor injection
@Component({
  selector: 'app-manual',
  template: `...`
})
export class ManualComponent implements OnInit {
  private userService: UserService;
  
  // Manual injection using Injector
  constructor(private injector: Injector) {
    // Get service from injector manually
    this.userService = this.injector.get(UserService);
  }
  
  ngOnInit(): void {
    // Can use manually-injected service
    this.userService.getUsers().subscribe();
  }
}

// Why you might do this:
// 1. Conditional injection
// 2. Dynamic service resolution
// 3. Plugin systems
// But 99% of the time, use normal constructor injection
```

---

## Real-World Scenarios

### **Scenario 1: When Constructor Is Required**

```typescript
// Use case: Setting up service configuration before ANY lifecycle hooks
@Component({
  selector: 'app-logger',
  template: `...`
})
export class LoggerComponent {
  constructor(
    private logger: LoggerService,
    @Inject(ENVIRONMENT) private env: Environment
  ) {
    // ✅ REQUIRED IN CONSTRUCTOR: Configure service before use
    this.logger.setEnvironment(this.env.name);
    this.logger.setLogLevel(this.env.production ? 'error' : 'debug');
    
    // This MUST happen before ngOnInit or any other hooks
    // because other components might use logger in their ngOnInit
  }
}
```

### **Scenario 2: When ngOnInit Is Required**

```typescript
// Use case: Loading data based on @Input
@Component({
  selector: 'app-product-details',
  template: `
    <div *ngIf="product">
      <h1>{{ product.name }}</h1>
      <p>{{ product.description }}</p>
      <p>Price: {{ product.price | currency }}</p>
    </div>
  `
})
export class ProductDetailsComponent implements OnInit, OnChanges {
  @Input() productId!: string;
  product: Product | null = null;
  
  constructor(private productService: ProductService) {
    // ❌ CAN'T load product here - productId is undefined
  }
  
  ngOnInit(): void {
    // ✅ MUST be in ngOnInit - productId is now available
    this.loadProduct();
  }
  
  ngOnChanges(changes: SimpleChanges): void {
    // ✅ ALSO handle when productId changes after initialization
    if (changes['productId'] && !changes['productId'].firstChange) {
      this.loadProduct();
    }
  }
  
  private loadProduct(): void {
    this.productService.getProduct(this.productId).subscribe(product => {
      this.product = product;
    });
  }
}
```

### **Scenario 3: Both Are Needed**

```typescript
// Use case: Complex component with configuration and data loading
@Component({
  selector: 'app-chart',
  template: `<canvas #chartCanvas></canvas>`
})
export class ChartComponent implements OnInit, AfterViewInit, OnDestroy {
  @Input() chartType!: 'line' | 'bar' | 'pie';
  @Input() dataUrl!: string;
  @ViewChild('chartCanvas') canvas!: ElementRef;
  
  private chart: any;
  private chartConfig: ChartConfiguration;
  private destroy$ = new Subject<void>();
  
  constructor(
    private http: HttpClient,
    private chartService: ChartService,
    @Inject(CHART_CONFIG) defaultConfig: ChartConfiguration
  ) {
    // CONSTRUCTOR: Setup that doesn't depend on @Input or DOM
    // ✅ Initialize configuration from injected default
    this.chartConfig = { ...defaultConfig };
    
    // ✅ Register component with service
    this.chartService.registerChart(this);
    
    // ✅ Setup static event listeners
    this.setupResizeListener();
  }
  
  ngOnInit(): void {
    // NGONINIT: Setup that depends on @Input but not DOM
    // ✅ Apply chart type from @Input
    this.chartConfig.type = this.chartType;
    
    // ✅ Load data from URL provided via @Input
    this.loadChartData();
  }
  
  ngAfterViewInit(): void {
    // AFTERVIEWINIT: Setup that requires DOM
    // ✅ Canvas is now available
    this.initializeChart();
  }
  
  private loadChartData(): void {
    this.http.get(this.dataUrl)
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {
        this.updateChartData(data);
      });
  }
  
  private initializeChart(): void {
    this.chart = new Chart(this.canvas.nativeElement, this.chartConfig);
  }
  
  private setupResizeListener(): void {
    fromEvent(window, 'resize')
      .pipe(
        debounceTime(300),
        takeUntil(this.destroy$)
      )
      .subscribe(() => {
        this.chart?.resize();
      });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.chart?.destroy();
    this.chartService.unregisterChart(this);
  }
}
```

---

## Common Mistakes

### **Mistake 1: Using @Input in Constructor**

```typescript
// ❌ WRONG
@Component({
  selector: 'app-user',
  template: `...`
})
export class UserComponent {
  @Input() userId!: string;
  
  constructor(private service: UserService) {
    this.service.loadUser(this.userId);  // userId is undefined!
  }
}

// ✅ CORRECT
export class UserComponent implements OnInit {
  @Input() userId!: string;
  
  constructor(private service: UserService) {}
  
  ngOnInit(): void {
    this.service.loadUser(this.userId);  // userId is available
  }
}
```

### **Mistake 2: Doing Complex Logic in Constructor**

```typescript
// ❌ WRONG: Heavy operations in constructor
constructor(private http: HttpClient) {
  // This runs even if component is destroyed immediately
  for (let i = 0; i < 1000000; i++) {
    this.processData(i);
  }
  
  this.http.get('/api/data').subscribe(data => {
    this.data = data;
  });
}

// ✅ CORRECT: Heavy operations in ngOnInit
constructor(private http: HttpClient) {
  // Lightweight initialization only
}

ngOnInit(): void {
  // If component is destroyed before ngOnInit, this never runs
  this.processLargeDataset();
  this.loadData();
}
```

### **Mistake 3: Not Cleaning Up Subscriptions**

```typescript
// ❌ WRONG: Subscription without cleanup
ngOnInit(): void {
  this.dataService.getData().subscribe(data => {
    this.data = data;
  });
  // Memory leak! Subscription continues after component destroyed
}

// ✅ CORRECT: Proper cleanup
private destroy$ = new Subject<void>();

ngOnInit(): void {
  this.dataService.getData()
    .pipe(takeUntil(this.destroy$))
    .subscribe(data => {
      this.data = data;
    });
}

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}
```

### **Mistake 4: Accessing @ViewChild in ngOnInit**

```typescript
// ❌ WRONG
@Component({
  selector: 'app-example',
  template: `<div #myDiv>Content</div>`
})
export class ExampleComponent implements OnInit {
  @ViewChild('myDiv') myDiv!: ElementRef;
  
  ngOnInit(): void {
    this.myDiv.nativeElement.focus();  // undefined! Crash!
  }
}

// ✅ CORRECT
export class ExampleComponent implements AfterViewInit {
  @ViewChild('myDiv') myDiv!: ElementRef;
  
  ngAfterViewInit(): void {
    this.myDiv.nativeElement.focus();  // Available now
  }
}
```

### **Mistake 5: Async Constructor (Not Possible)**

```typescript
// ❌ IMPOSSIBLE: Async constructor
constructor(private service: UserService) {
  // ❌ Can't use await in constructor
  const user = await this.service.getUser().toPromise();  // Syntax error!
}

// ✅ CORRECT: Async in ngOnInit
async ngOnInit(): Promise<void> {
  const user = await firstValueFrom(this.service.getUser());
  this.processUser(user);
}

// ✅ BETTER: Use observables
ngOnInit(): void {
  this.user$ = this.service.getUser();
  // Template: {{ user$ | async }}
}
```

---

## Quick Reference

### **Constructor**
- **Purpose**: Dependency injection, basic initialization
- **Timing**: Before any lifecycle hooks
- **Available**: Injected services, class fields
- **NOT Available**: @Input, @ViewChild, @ContentChild
- **Use for**: DI, simple initialization, service configuration
- **DON'T use for**: HTTP calls, @Input access, DOM manipulation

### **ngOnInit**
- **Purpose**: Component initialization after inputs are set
- **Timing**: After constructor, after first ngOnChanges
- **Available**: Injected services, @Input properties
- **NOT Available**: @ViewChild, @ContentChild (use ngAfterViewInit)
- **Use for**: HTTP calls, subscriptions, @Input-based logic
- **DON'T use for**: DI (not possible), DOM manipulation

### **Decision Tree**

```
Need to do something?
│
├─ Need injected dependency?
│  └─ Constructor (only place for DI)
│
├─ Need @Input value?
│  └─ ngOnInit (or ngOnChanges)
│
├─ Need @ViewChild/@ContentChild?
│  └─ ngAfterViewInit (or ngAfterContentInit)
│
├─ Simple class property initialization?
│  └─ Class field initializer (or constructor)
│
├─ HTTP request / Observable subscription?
│  └─ ngOnInit
│
└─ DOM manipulation?
   └─ ngAfterViewInit
```

---

## Summary

**Constructor is for TypeScript, ngOnInit is for Angular.**

- **Constructor**: Where JavaScript creates the object and Angular injects dependencies
- **ngOnInit**: Where Angular tells you "your component is ready to use"

**The golden rule**: If it depends on `@Input`, it goes in `ngOnInit`. If it's dependency injection or static initialization, it goes in `constructor`.

Understanding this distinction is fundamental to avoiding the most common Angular bugs - trying to use inputs before they're set, or trying to do dependency injection outside the constructor. Get this right, and you'll avoid hours of debugging "undefined is not an object" errors.

