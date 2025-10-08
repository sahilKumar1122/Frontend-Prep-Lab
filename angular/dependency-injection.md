# Angular Dependency Injection - Deep Dive

## Table of Contents
- [How Angular Resolves Dependencies](#how-angular-resolves-dependencies)
- [Hierarchical Injector Tree](#hierarchical-injector-tree)
- [Provider Scopes](#provider-scopes)
- [Injection Tokens](#injection-tokens)
- [Multi-Providers](#multi-providers)
- [Tree-Shaking and DI](#tree-shaking-and-di)
- [Real-World Case Study](#real-world-case-study)

---

## How Angular Resolves Dependencies

### Question: Explain Angular's Dependency Injection system in detail. Walk me through exactly how Angular resolves a dependency — from provider registration to instance creation. Cover hierarchical injectors, provider scopes, injection tokens, multi-providers, tree-shaking, and collision handling.

**Answer:**

Angular's Dependency Injection (DI) is a sophisticated system that goes far beyond "it injects services." Let's trace the complete journey from provider registration to instance creation.

---

## **The Complete Dependency Resolution Flow**

### **Step 1: Provider Registration**

When you register a provider, Angular creates an entry in an injector's provider registry:

```typescript
// Three ways to register providers:

// Method 1: providedIn (Modern, Tree-shakeable)
@Injectable({
  providedIn: 'root'  // Registers in root injector
})
export class UserService {
  constructor(private http: HttpClient) {}
}

// Method 2: Module providers
@NgModule({
  providers: [
    UserService,  // Shorthand for { provide: UserService, useClass: UserService }
    { provide: LoggerService, useClass: AdvancedLogger },
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
})
export class AppModule {}

// Method 3: Component providers
@Component({
  selector: 'app-user-profile',
  providers: [UserProfileService]  // New instance per component
})
export class UserProfileComponent {}
```

**What Actually Happens Internally:**

```typescript
// Simplified internal representation
class InjectorRegistry {
  private providers = new Map<any, Provider>();
  
  register(token: any, provider: Provider): void {
    // Angular stores the provider configuration
    this.providers.set(token, {
      token: token,                    // What to inject
      factory: provider.useClass || provider.useFactory,  // How to create it
      dependencies: this.resolveDependencies(provider),   // What it needs
      scope: this.determineScope(provider)                // Where it lives
    });
  }
}
```

---

### **Step 2: Dependency Request**

When a component/service requests a dependency:

```typescript
@Component({
  selector: 'app-dashboard',
  template: `...`
})
export class DashboardComponent {
  // When Angular creates this component, it sees the constructor parameters
  constructor(
    private userService: UserService,      // Token 1
    private logger: LoggerService,         // Token 2
    @Inject(API_URL) private apiUrl: string  // Token 3 with explicit @Inject
  ) {
    // Angular needs to resolve all these before creating the component
  }
}
```

**Angular's Internal Process:**

```typescript
// Simplified internal resolution
class Injector {
  private instances = new Map<any, any>();  // Cache for singleton instances
  private parent: Injector | null;
  
  get<T>(token: Type<T> | InjectionToken<T>): T {
    // Step 1: Check if already instantiated (singleton check)
    if (this.instances.has(token)) {
      return this.instances.get(token);
    }
    
    // Step 2: Look for provider in current injector
    const provider = this.providers.get(token);
    
    if (provider) {
      // Step 3: Resolve dependencies recursively
      const dependencies = provider.dependencies.map(dep => this.get(dep));
      
      // Step 4: Create instance using factory
      const instance = provider.factory(...dependencies);
      
      // Step 5: Cache if singleton
      if (provider.scope === 'singleton') {
        this.instances.set(token, instance);
      }
      
      return instance;
    }
    
    // Step 6: Not found here, check parent injector
    if (this.parent) {
      return this.parent.get(token);
    }
    
    // Step 7: Not found anywhere, throw error
    throw new Error(`No provider for ${token}!`);
  }
}
```

---

### **Step 3: Walking Up the Injector Tree**

Angular searches for providers in a specific order:

```typescript
// Component Tree:
//       AppComponent (Root Injector)
//            |
//       HeaderComponent (Element Injector)
//            |
//       UserMenuComponent (Element Injector)
//            |
//       UserProfileComponent (Element Injector with its own provider)

@Component({
  selector: 'app-user-profile',
  providers: [UserProfileService],  // Creates new ElementInjector here
  template: `...`
})
export class UserProfileComponent {
  constructor(
    private profileService: UserProfileService,  // Found HERE
    private authService: AuthService,            // Found in ROOT injector
    private parentService: ParentService         // Found in PARENT element injector
  ) {}
}
```

**Search Order Visualization:**

```
1. UserProfileComponent's ElementInjector   ← Check here first
   ↓ (not found)
2. UserMenuComponent's ElementInjector      ← Check parent
   ↓ (not found)
3. HeaderComponent's ElementInjector        ← Check parent
   ↓ (not found)
4. AppComponent's ElementInjector           ← Check parent
   ↓ (not found)
5. Module Injector                          ← Check module
   ↓ (not found)
6. Root Injector (providedIn: 'root')       ← Check root
   ↓ (not found)
7. Platform Injector                        ← Check platform
   ↓ (not found)
8. NullInjector                             ← Throw error!
```

---

## Hierarchical Injector Tree

### **The Complete Injector Hierarchy**

Angular has TWO injector hierarchies that work together:

```typescript
/**
 * MODULE INJECTOR HIERARCHY
 * 
 * Platform Injector (Shared across all Angular apps on the page)
 *    ↓
 * Root Injector (providedIn: 'root')
 *    ↓
 * Lazy Module Injector (One per lazy-loaded module)
 */

/**
 * ELEMENT INJECTOR HIERARCHY
 * 
 * Mirrors the component tree
 * Created only when a component/directive has providers
 */
```

**Detailed Hierarchy Example:**

```typescript
// app.module.ts
@NgModule({
  providers: [
    { provide: APP_CONFIG, useValue: { apiUrl: 'https://api.example.com' } }
  ]
})
export class AppModule {}

// user.service.ts
@Injectable({
  providedIn: 'root'  // ROOT INJECTOR
})
export class UserService {
  constructor(private http: HttpClient) {}
}

// app.component.ts
@Component({
  selector: 'app-root',
  providers: [
    ThemeService  // ELEMENT INJECTOR at root component level
  ]
})
export class AppComponent {}

// header.component.ts
@Component({
  selector: 'app-header',
  providers: [
    NotificationService  // ELEMENT INJECTOR at header level
  ]
})
export class HeaderComponent {
  constructor(
    private userService: UserService,           // From ROOT injector
    private themeService: ThemeService,         // From AppComponent ElementInjector
    private notifications: NotificationService  // From THIS ElementInjector
  ) {}
}
```

**Visual Representation:**

```
┌─────────────────────────────────────────┐
│       Platform Injector                  │
│  (BrowserModule, PlatformRef, etc.)     │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│       Root Injector                      │
│  (UserService via providedIn: 'root')   │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│       Module Injector                    │
│  (APP_CONFIG from AppModule providers)  │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│  AppComponent ElementInjector            │
│  (ThemeService)                          │
└───────────────┬─────────────────────────┘
                │
┌───────────────▼─────────────────────────┐
│  HeaderComponent ElementInjector         │
│  (NotificationService)                   │
└──────────────────────────────────────────┘
```

---

## Provider Scopes

### **providedIn: 'root' vs Module Providers vs Component Providers**

This is where most developers get confused. Let's break it down:

### **1. providedIn: 'root' (Recommended)**

```typescript
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private currentUser: User | null = null;
  
  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials)
      .pipe(tap(user => this.currentUser = user));
  }
  
  getCurrentUser(): User | null {
    return this.currentUser;
  }
}
```

**What Happens:**
- Service is registered in **Root Injector**
- **ONE instance** for entire application
- **Tree-shakeable**: If never injected anywhere, won't be included in bundle
- Available to all components, services, directives
- Instance persists for entire app lifetime

**When to Use:**
- Stateful services (auth, user state, global config)
- API services
- Services that need to be singletons
- 99% of services should use this

---

### **2. Module Providers**

```typescript
// user.module.ts
@NgModule({
  providers: [
    UserStateService  // Provided at module level
  ]
})
export class UserModule {}
```

**What Happens:**
- Service is registered in **Module Injector**
- **ONE instance per module**
- **NOT tree-shakeable**: Always included if module is imported
- For lazy-loaded modules: new instance created when module loads
- For eagerly-loaded modules: behaves like root-level

**When to Use:**
- Services scoped to a feature module
- Lazy-loaded module services that need isolated state
- Legacy code (prefer providedIn: 'root' instead)

**Critical Difference with Lazy Loading:**

```typescript
// products.module.ts (Lazy-loaded)
@NgModule({
  providers: [
    ProductStateService  // New instance EVERY TIME module loads
  ]
})
export class ProductsModule {}

// routes
const routes: Routes = [
  {
    path: 'products',
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule)
  }
];
```

**What Actually Happens:**

```typescript
// First navigation to /products
const injector1 = new ModuleInjector(ProductsModule);
const productState1 = new ProductStateService(); // Instance 1

// User navigates away, then back to /products
// Module is loaded AGAIN (unless preloaded)
const injector2 = new ModuleInjector(ProductsModule);
const productState2 = new ProductStateService(); // Instance 2 (different!)

// ❌ PROBLEM: State is lost!
// productState1 !== productState2
```

**Solution: Use providedIn: 'root' for state that should persist:**

```typescript
@Injectable({
  providedIn: 'root'  // ✅ Single instance across all modules
})
export class ProductStateService {
  private products$ = new BehaviorSubject<Product[]>([]);
  
  // State persists even when navigating away and back
}
```

---

### **3. Component Providers**

```typescript
@Component({
  selector: 'app-user-form',
  providers: [
    FormStateService  // New instance per component instance
  ],
  template: `...`
})
export class UserFormComponent {
  constructor(private formState: FormStateService) {}
}
```

**What Happens:**
- Service is registered in **Element Injector** for this component
- **NEW instance** for every component instance
- Instance destroyed when component is destroyed
- NOT available to sibling components
- Available to child components (unless they provide their own)

**When to Use:**
- Component-specific state that shouldn't be shared
- Form state services
- Temporary data that should be garbage collected with component
- Services that need to be reset for each component instance

**Real Example - Multiple Form Instances:**

```typescript
@Injectable()  // No providedIn - must be provided explicitly
export class WizardStateService {
  private currentStep = 0;
  private formData = {};
  
  nextStep(): void {
    this.currentStep++;
  }
  
  saveStepData(step: number, data: any): void {
    this.formData[step] = data;
  }
  
  reset(): void {
    this.currentStep = 0;
    this.formData = {};
  }
}

@Component({
  selector: 'app-wizard',
  providers: [WizardStateService],  // New instance per wizard
  template: `
    <app-step-1 *ngIf="currentStep === 0"></app-step-1>
    <app-step-2 *ngIf="currentStep === 1"></app-step-2>
    <app-step-3 *ngIf="currentStep === 2"></app-step-3>
  `
})
export class WizardComponent {
  constructor(public wizardState: WizardStateService) {}
  
  get currentStep(): number {
    return this.wizardState.currentStep;
  }
}

// Usage: Multiple wizards on same page, each with isolated state
// <app-wizard></app-wizard>  ← Instance 1 of WizardStateService
// <app-wizard></app-wizard>  ← Instance 2 of WizardStateService
// <app-wizard></app-wizard>  ← Instance 3 of WizardStateService
```

---

### **Comparison Table:**

| Aspect | providedIn: 'root' | Module Providers | Component Providers |
|--------|-------------------|------------------|---------------------|
| **Instances** | 1 per app | 1 per module load | 1 per component instance |
| **Lifetime** | App lifetime | Module lifetime | Component lifetime |
| **Tree-shakeable** | ✅ Yes | ❌ No | ❌ No |
| **State Persistence** | ✅ Always | ⚠️ Lost on reload (lazy) | ❌ Lost on destroy |
| **Child Access** | ✅ All children | ✅ Module children | ✅ Component children |
| **Bundle Size** | Smallest | Larger | Larger |
| **Use Case** | 99% of services | Feature-scoped state | Component-scoped state |

---

## Injection Tokens

### **What Are Injection Tokens?**

Injection Tokens are unique identifiers for non-class dependencies. You can't use primitive types or interfaces as DI tokens directly because TypeScript interfaces don't exist at runtime.

```typescript
// ❌ WRONG: Can't inject string type
constructor(private apiUrl: string) {}  // Error: No provider for String!

// ❌ WRONG: Interfaces don't exist at runtime
interface AppConfig {
  apiUrl: string;
  timeout: number;
}
constructor(private config: AppConfig) {}  // Error: Can't find AppConfig!

// ✅ CORRECT: Use InjectionToken
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

constructor(@Inject(APP_CONFIG) private config: AppConfig) {}  // Works!
```

### **Creating and Using Injection Tokens**

```typescript
// config.ts
export interface AppConfig {
  apiUrl: string;
  timeout: number;
  retryAttempts: number;
  enableDebug: boolean;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config', {
  providedIn: 'root',
  factory: () => ({
    apiUrl: 'https://api.example.com',
    timeout: 30000,
    retryAttempts: 3,
    enableDebug: false
  })
});

// Alternative: Provide in module
@NgModule({
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: environment.apiUrl,
        timeout: 30000,
        retryAttempts: 3,
        enableDebug: !environment.production
      }
    }
  ]
})
export class AppModule {}

// Usage in service
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  constructor(
    private http: HttpClient,
    @Inject(APP_CONFIG) private config: AppConfig
  ) {}
  
  getData(): Observable<any> {
    return this.http.get(`${this.config.apiUrl}/data`, {
      timeout: this.config.timeout
    }).pipe(
      retry(this.config.retryAttempts)
    );
  }
}
```

### **When Do You Need Injection Tokens?**

**1. Injecting Configuration Objects:**

```typescript
export const API_URL = new InjectionToken<string>('api.url');
export const FEATURE_FLAGS = new InjectionToken<FeatureFlags>('feature.flags');
export const LOCALE = new InjectionToken<string>('locale');

@NgModule({
  providers: [
    { provide: API_URL, useValue: 'https://api.example.com' },
    { provide: FEATURE_FLAGS, useValue: { newUI: true, betaFeatures: false } },
    { provide: LOCALE, useValue: 'en-US' }
  ]
})
export class AppModule {}
```

**2. Injecting Window, Document, LocalStorage:**

```typescript
// window.token.ts
export const WINDOW = new InjectionToken<Window>('window', {
  providedIn: 'root',
  factory: () => window
});

// document.token.ts
export const DOCUMENT = new InjectionToken<Document>('document', {
  providedIn: 'root',
  factory: () => document
});

// local-storage.token.ts
export const LOCAL_STORAGE = new InjectionToken<Storage>('localStorage', {
  providedIn: 'root',
  factory: () => localStorage
});

// Usage
@Injectable({
  providedIn: 'root'
})
export class StorageService {
  constructor(@Inject(LOCAL_STORAGE) private localStorage: Storage) {}
  
  setItem(key: string, value: any): void {
    this.localStorage.setItem(key, JSON.stringify(value));
  }
  
  getItem<T>(key: string): T | null {
    const item = this.localStorage.getItem(key);
    return item ? JSON.parse(item) : null;
  }
}
```

**Why Use Tokens for Browser APIs?**
- **Testability**: Easy to mock in tests
- **SSR Compatibility**: Can provide different implementations for server-side rendering
- **Dependency Injection**: Follows Angular's DI patterns

**3. Injecting Factory Functions:**

```typescript
export type LoggerFactory = (name: string) => Logger;

export const LOGGER_FACTORY = new InjectionToken<LoggerFactory>('logger.factory', {
  providedIn: 'root',
  factory: () => (name: string) => new Logger(name)
});

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private logger: Logger;
  
  constructor(@Inject(LOGGER_FACTORY) loggerFactory: LoggerFactory) {
    this.logger = loggerFactory('UserService');
  }
  
  getUsers(): Observable<User[]> {
    this.logger.log('Fetching users...');
    return this.http.get<User[]>('/api/users');
  }
}
```

**4. Providing Abstract Classes:**

```typescript
// Abstract base class
export abstract class StorageService {
  abstract setItem(key: string, value: any): void;
  abstract getItem(key: string): any;
  abstract removeItem(key: string): void;
}

// Concrete implementation
@Injectable()
export class LocalStorageService extends StorageService {
  setItem(key: string, value: any): void {
    localStorage.setItem(key, JSON.stringify(value));
  }
  
  getItem(key: string): any {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : null;
  }
  
  removeItem(key: string): void {
    localStorage.removeItem(key);
  }
}

// Provide using class as token
@NgModule({
  providers: [
    { provide: StorageService, useClass: LocalStorageService }
  ]
})
export class AppModule {}

// Or use InjectionToken for more flexibility
export const STORAGE = new InjectionToken<StorageService>('storage');

@NgModule({
  providers: [
    {
      provide: STORAGE,
      useClass: environment.production ? LocalStorageService : MockStorageService
    }
  ]
})
export class AppModule {}
```

---

## Multi-Providers

### **What Are Multi-Providers?**

Multi-providers allow multiple values to be injected for the same token. Angular collects all providers and injects them as an array.

```typescript
// http-interceptor.token.ts
export const HTTP_INTERCEPTOR = new InjectionToken<HttpInterceptor>('http.interceptor');

// Registering multiple interceptors
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTOR, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTOR, useClass: LoggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTOR, useClass: ErrorInterceptor, multi: true },
    { provide: HTTP_INTERCEPTOR, useClass: CacheInterceptor, multi: true }
  ]
})
export class AppModule {}

// Angular injects ALL of them as an array
@Injectable()
export class HttpService {
  constructor(@Inject(HTTP_INTERCEPTOR) private interceptors: HttpInterceptor[]) {
    console.log(interceptors.length); // 4
    // interceptors = [AuthInterceptor, LoggingInterceptor, ErrorInterceptor, CacheInterceptor]
  }
}
```

### **Real-World Use Cases for Multi-Providers**

**1. HTTP Interceptors:**

```typescript
// auth.interceptor.ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}
  
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    
    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
    }
    
    return next.handle(req);
  }
}

// logging.interceptor.ts
@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    console.log('HTTP Request:', req.method, req.url);
    const started = Date.now();
    
    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          const elapsed = Date.now() - started;
          console.log(`Response received in ${elapsed}ms`);
        }
      })
    );
  }
}

// Provide both
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true }
  ]
})
export class CoreModule {}
```

**2. Route Guards:**

```typescript
// admin.guard.ts
export const ADMIN_GUARDS = new InjectionToken<CanActivate[]>('admin.guards');

@Injectable({
  providedIn: 'root'
})
export class AdminRoleGuard implements CanActivate {
  constructor(private authService: AuthService) {}
  
  canActivate(): boolean {
    return this.authService.hasRole('admin');
  }
}

@Injectable({
  providedIn: 'root'
})
export class SubscriptionActiveGuard implements CanActivate {
  constructor(private subscriptionService: SubscriptionService) {}
  
  canActivate(): Observable<boolean> {
    return this.subscriptionService.isActive();
  }
}

// Provide multiple guards
@NgModule({
  providers: [
    { provide: ADMIN_GUARDS, useClass: AdminRoleGuard, multi: true },
    { provide: ADMIN_GUARDS, useClass: SubscriptionActiveGuard, multi: true }
  ]
})
export class AdminModule {}

// Use in router
const routes: Routes = [
  {
    path: 'admin',
    canActivate: [AdminRoleGuard, SubscriptionActiveGuard],
    component: AdminComponent
  }
];
```

**3. Form Validators:**

```typescript
// validators.token.ts
export const FORM_VALIDATORS = new InjectionToken<Validator[]>('form.validators');

@Injectable()
export class EmailValidator implements Validator {
  validate(control: AbstractControl): ValidationErrors | null {
    const email = control.value;
    const valid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    return valid ? null : { invalidEmail: true };
  }
}

@Injectable()
export class PasswordStrengthValidator implements Validator {
  validate(control: AbstractControl): ValidationErrors | null {
    const password = control.value;
    const hasUpperCase = /[A-Z]/.test(password);
    const hasLowerCase = /[a-z]/.test(password);
    const hasNumber = /\d/.test(password);
    const hasSpecial = /[!@#$%^&*]/.test(password);
    
    const valid = hasUpperCase && hasLowerCase && hasNumber && hasSpecial;
    return valid ? null : { weakPassword: true };
  }
}

// Provide validators
@Component({
  selector: 'app-signup',
  providers: [
    { provide: FORM_VALIDATORS, useClass: EmailValidator, multi: true },
    { provide: FORM_VALIDATORS, useClass: PasswordStrengthValidator, multi: true }
  ]
})
export class SignupComponent {
  constructor(@Inject(FORM_VALIDATORS) private validators: Validator[]) {
    // validators = [EmailValidator, PasswordStrengthValidator]
    this.setupForm();
  }
}
```

**4. Plugin System:**

```typescript
// plugin.interface.ts
export interface Plugin {
  name: string;
  initialize(): void;
  execute(data: any): any;
}

export const PLUGINS = new InjectionToken<Plugin[]>('plugins');

// analytics.plugin.ts
@Injectable()
export class AnalyticsPlugin implements Plugin {
  name = 'Analytics';
  
  initialize(): void {
    console.log('Analytics plugin initialized');
  }
  
  execute(data: any): any {
    // Track analytics
    console.log('Analytics:', data);
    return data;
  }
}

// logging.plugin.ts
@Injectable()
export class LoggingPlugin implements Plugin {
  name = 'Logging';
  
  initialize(): void {
    console.log('Logging plugin initialized');
  }
  
  execute(data: any): any {
    // Log data
    console.log('Log:', data);
    return data;
  }
}

// Provide plugins
@NgModule({
  providers: [
    { provide: PLUGINS, useClass: AnalyticsPlugin, multi: true },
    { provide: PLUGINS, useClass: LoggingPlugin, multi: true }
  ]
})
export class AppModule {}

// Plugin manager
@Injectable({
  providedIn: 'root'
})
export class PluginManager {
  constructor(@Inject(PLUGINS) private plugins: Plugin[]) {
    this.plugins.forEach(plugin => plugin.initialize());
  }
  
  executeAll(data: any): any {
    let result = data;
    this.plugins.forEach(plugin => {
      result = plugin.execute(result);
    });
    return result;
  }
}
```

---

## Tree-Shaking and DI

### **How Tree-Shaking Works with Dependency Injection**

Tree-shaking is the process of removing unused code from the final bundle. Angular's DI system is designed to support this.

**Without providedIn (Old Way):**

```typescript
// user.service.ts
@Injectable()  // No providedIn
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// app.module.ts
@NgModule({
  providers: [UserService]  // ❌ Always included, even if never used!
})
export class AppModule {}

// Result: UserService included in bundle even if no component uses it
// Bundle size: Larger
```

**With providedIn (Modern Way):**

```typescript
// user.service.ts
@Injectable({
  providedIn: 'root'  // ✅ Tree-shakeable!
})
export class UserService {
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// app.module.ts
@NgModule({
  providers: []  // No need to list services!
})
export class AppModule {}

// Scenario 1: Service is injected somewhere
@Component({
  selector: 'app-users',
  template: `...`
})
export class UsersComponent {
  constructor(private userService: UserService) {}  // Used!
}
// Result: UserService included in bundle

// Scenario 2: Service is never injected
// No component or service injects UserService
// Result: UserService completely removed from bundle!
// Bundle size: Smaller
```

**How Angular Determines What to Keep:**

```typescript
// During build process (simplified):

1. Angular compiler scans all components, directives, pipes
2. Identifies all constructor parameters and @Inject decorators
3. Marks those services as "used"
4. For services with providedIn: 'root', checks if they're used
5. If never injected → Remove from bundle
6. If injected somewhere → Keep in bundle

// Example:
@Injectable({ providedIn: 'root' })
export class UnusedService {}  // Never injected

@Injectable({ providedIn: 'root' })
export class UsedService {}    // Injected in AppComponent

// Build result:
// - UnusedService: Removed (tree-shaken)
// - UsedService: Included
```

**Tree-Shaking with providedIn: 'any':**

```typescript
@Injectable({
  providedIn: 'any'  // One instance per lazy-loaded module
})
export class FeatureService {
  // Service code
}

// Behavior:
// - Each lazy-loaded module gets its own instance
// - Tree-shakeable: Removed if module never loaded
// - Different from 'root': Multiple instances possible
```

**Measuring Tree-Shaking Impact:**

```bash
# Build with source maps
ng build --source-map

# Analyze bundle
npm install -g source-map-explorer
source-map-explorer dist/**/*.js

# You'll see:
# - Services with providedIn: 'root' only if used
# - Services in module providers always included
# - Potential savings from switching to providedIn
```

---

## What Happens When Two Injectors Provide the Same Token?

### **Collision Handling Rules**

When multiple injectors in the hierarchy provide the same token, **the closest one wins**.

```typescript
// root.service.ts
@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  log(message: string): void {
    console.log('[ROOT]', message);
  }
}

// app.component.ts
@Injectable()
export class AppLoggerService {
  log(message: string): void {
    console.log('[APP]', message);
  }
}

@Component({
  selector: 'app-root',
  providers: [
    { provide: LoggerService, useClass: AppLoggerService }  // Override root
  ]
})
export class AppComponent {
  constructor(private logger: LoggerService) {
    this.logger.log('Hello');  // Output: [APP] Hello
  }
}

// child.component.ts
@Injectable()
export class ChildLoggerService {
  log(message: string): void {
    console.log('[CHILD]', message);
  }
}

@Component({
  selector: 'app-child',
  providers: [
    { provide: LoggerService, useClass: ChildLoggerService }  // Override parent
  ]
})
export class ChildComponent {
  constructor(private logger: LoggerService) {
    this.logger.log('Hello');  // Output: [CHILD] Hello
  }
}
```

**Resolution Order Example:**

```typescript
// Setup:
// Root: LoggerService (from providedIn: 'root')
// AppComponent: LoggerService override (AppLoggerService)
// HeaderComponent: LoggerService override (HeaderLoggerService)
// UserMenuComponent: No override

@Component({
  selector: 'app-user-menu',
  template: `...`
})
export class UserMenuComponent {
  constructor(private logger: LoggerService) {
    // Which logger is injected?
    // 1. Check UserMenuComponent providers → Not found
    // 2. Check HeaderComponent providers → Found! HeaderLoggerService
    this.logger.log('User menu');  // Output: [HEADER] User menu
  }
}
```

**Common Collision Scenarios:**

**Scenario 1: Testing with Mock Services**

```typescript
// Production service
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  getData(): Observable<Data> {
    return this.http.get<Data>('/api/data');
  }
}

// Test
describe('MyComponent', () => {
  let mockApiService: jasmine.SpyObj<ApiService>;
  
  beforeEach(() => {
    mockApiService = jasmine.createSpyObj('ApiService', ['getData']);
    mockApiService.getData.and.returnValue(of({ id: 1, name: 'Test' }));
    
    TestBed.configureTestingModule({
      declarations: [MyComponent],
      providers: [
        { provide: ApiService, useValue: mockApiService }  // Override root service
      ]
    });
  });
  
  it('should use mock service', () => {
    const component = TestBed.createComponent(MyComponent).componentInstance;
    // component uses mockApiService, not the real one
  });
});
```

**Scenario 2: Feature-Specific Implementation**

```typescript
// Global logger
@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

// Admin module needs different logging
@Injectable()
export class AdminLoggerService extends LoggerService {
  override log(message: string): void {
    // Send to admin monitoring system
    console.log('[ADMIN]', message);
    this.sendToMonitoring(message);
  }
  
  private sendToMonitoring(message: string): void {
    // Send to external monitoring service
  }
}

@NgModule({
  declarations: [AdminComponent],
  providers: [
    { provide: LoggerService, useClass: AdminLoggerService }  // Override for admin
  ]
})
export class AdminModule {}
```

**Scenario 3: Environment-Specific Services**

```typescript
// Development API service
@Injectable()
export class DevApiService {
  getData(): Observable<any> {
    return of({ mockData: true });  // Return mock data
  }
}

// Production API service
@Injectable()
export class ProdApiService {
  constructor(private http: HttpClient) {}
  
  getData(): Observable<any> {
    return this.http.get('/api/data');  // Real API call
  }
}

// app.module.ts
@NgModule({
  providers: [
    {
      provide: ApiService,
      useClass: environment.production ? ProdApiService : DevApiService
    }
  ]
})
export class AppModule {}
```

---

## Real-World Case Study

### **The Multi-Instance Service Scope Bug**

**The Problem:**

I was working on an e-commerce application with a shopping cart feature. Users reported that their cart items would randomly disappear when navigating between pages. Sometimes the cart would be empty, sometimes it would show old items.

**Initial Implementation (Broken):**

```typescript
// cart.service.ts
@Injectable()  // ❌ No providedIn!
export class CartService {
  private items: CartItem[] = [];
  
  addItem(item: CartItem): void {
    const existing = this.items.find(i => i.id === item.id);
    if (existing) {
      existing.quantity++;
    } else {
      this.items.push(item);
    }
  }
  
  getItems(): CartItem[] {
    return this.items;
  }
  
  getTotal(): number {
    return this.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
  
  clear(): void {
    this.items = [];
  }
}

// cart.module.ts (Lazy-loaded)
@NgModule({
  declarations: [CartComponent, CartItemComponent],
  providers: [
    CartService  // ❌ Provided at module level!
  ]
})
export class CartModule {}

// product-list.component.ts (Different module)
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products">
      <button (click)="addToCart(product)">Add to Cart</button>
    </div>
  `
})
export class ProductListComponent {
  constructor(private cartService: CartService) {}  // ❌ Different instance!
  
  addToCart(product: Product): void {
    this.cartService.addItem({
      id: product.id,
      name: product.name,
      price: product.price,
      quantity: 1
    });
  }
}
```

**What Was Happening:**

```
User Flow:
1. User browses products (ProductListModule - eagerly loaded)
   → CartService instance #1 created
   → Adds items to cart (stored in instance #1)

2. User clicks "View Cart" (navigates to /cart)
   → CartModule loads (lazy-loaded)
   → CartService instance #2 created  ← PROBLEM!
   → instance #2 has empty items array
   → User sees empty cart!

3. User navigates back to products
   → Still using instance #1
   → Cart items reappear!

4. User navigates to cart again
   → Module already loaded, uses instance #2
   → Empty cart again!
```

**The Root Cause:**

```typescript
// The problem:
// 1. CartService provided in CartModule (lazy-loaded)
// 2. ProductListComponent imports CartService
// 3. Angular creates CartService in ROOT injector when ProductListComponent loads
// 4. When CartModule loads, it creates ANOTHER instance in Module injector
// 5. Two different instances = two different states!

// Injector hierarchy:
Root Injector
  ├─ CartService (instance #1) ← Used by ProductListComponent
  └─ CartModule Injector
      └─ CartService (instance #2) ← Used by CartComponent
```

**The Fix:**

```typescript
// cart.service.ts
@Injectable({
  providedIn: 'root'  // ✅ Single instance for entire app!
})
export class CartService {
  private items$ = new BehaviorSubject<CartItem[]>([]);
  
  // Expose as observable for reactive updates
  readonly items: Observable<CartItem[]> = this.items$.asObservable();
  
  addItem(item: CartItem): void {
    const currentItems = this.items$.value;
    const existing = currentItems.find(i => i.id === item.id);
    
    if (existing) {
      existing.quantity++;
      this.items$.next([...currentItems]);  // Trigger update
    } else {
      this.items$.next([...currentItems, item]);
    }
    
    // Persist to localStorage
    this.saveToStorage();
  }
  
  getItems(): CartItem[] {
    return this.items$.value;
  }
  
  getTotal(): number {
    return this.items$.value.reduce(
      (sum, item) => sum + (item.price * item.quantity),
      0
    );
  }
  
  clear(): void {
    this.items$.next([]);
    this.saveToStorage();
  }
  
  private saveToStorage(): void {
    localStorage.setItem('cart', JSON.stringify(this.items$.value));
  }
  
  private loadFromStorage(): void {
    const stored = localStorage.getItem('cart');
    if (stored) {
      this.items$.next(JSON.parse(stored));
    }
  }
  
  constructor() {
    // Load cart from localStorage on initialization
    this.loadFromStorage();
  }
}

// cart.module.ts
@NgModule({
  declarations: [CartComponent, CartItemComponent],
  providers: []  // ✅ No providers! Service uses providedIn: 'root'
})
export class CartModule {}
```

**Additional Improvements:**

```typescript
// cart-state.service.ts - More robust version
@Injectable({
  providedIn: 'root'
})
export class CartStateService {
  private readonly STORAGE_KEY = 'shopping-cart';
  private items$ = new BehaviorSubject<CartItem[]>(this.loadFromStorage());
  
  // Public observables
  readonly items: Observable<CartItem[]> = this.items$.asObservable();
  readonly itemCount: Observable<number> = this.items.pipe(
    map(items => items.reduce((sum, item) => sum + item.quantity, 0))
  );
  readonly total: Observable<number> = this.items.pipe(
    map(items => items.reduce((sum, item) => sum + (item.price * item.quantity), 0))
  );
  
  constructor(
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService
  ) {
    // Auto-save to localStorage whenever items change
    this.items.pipe(
      debounceTime(500),  // Don't save too frequently
      distinctUntilChanged()
    ).subscribe(items => {
      this.saveToStorage(items);
    });
  }
  
  addItem(product: Product): void {
    const currentItems = this.items$.value;
    const existing = currentItems.find(i => i.id === product.id);
    
    if (existing) {
      const updated = currentItems.map(item =>
        item.id === product.id
          ? { ...item, quantity: item.quantity + 1 }
          : item
      );
      this.items$.next(updated);
      this.notificationService.show(`Increased ${product.name} quantity`);
    } else {
      this.items$.next([...currentItems, {
        id: product.id,
        name: product.name,
        price: product.price,
        quantity: 1,
        image: product.image
      }]);
      this.notificationService.show(`Added ${product.name} to cart`);
    }
    
    // Track analytics
    this.analyticsService.trackEvent('add_to_cart', {
      product_id: product.id,
      product_name: product.name,
      price: product.price
    });
  }
  
  removeItem(productId: number): void {
    const currentItems = this.items$.value;
    this.items$.next(currentItems.filter(item => item.id !== productId));
    this.notificationService.show('Item removed from cart');
  }
  
  updateQuantity(productId: number, quantity: number): void {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }
    
    const currentItems = this.items$.value;
    const updated = currentItems.map(item =>
      item.id === productId ? { ...item, quantity } : item
    );
    this.items$.next(updated);
  }
  
  clear(): void {
    this.items$.next([]);
    this.notificationService.show('Cart cleared');
  }
  
  private loadFromStorage(): CartItem[] {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      return stored ? JSON.parse(stored) : [];
    } catch (error) {
      console.error('Failed to load cart from storage:', error);
      return [];
    }
  }
  
  private saveToStorage(items: CartItem[]): void {
    try {
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(items));
    } catch (error) {
      console.error('Failed to save cart to storage:', error);
    }
  }
}
```

**What I Learned:**

1. **Always use `providedIn: 'root'` for stateful services** - Unless you explicitly need multiple instances, always provide at root level.

2. **Lazy-loaded modules create new injectors** - Services provided in lazy-loaded module providers get new instances every time the module loads.

3. **Test service scope** - I added integration tests to verify service instance behavior:

```typescript
describe('CartService Scope', () => {
  it('should be singleton across modules', () => {
    const cartService1 = TestBed.inject(CartService);
    const cartService2 = TestBed.inject(CartService);
    
    expect(cartService1).toBe(cartService2);  // Same instance
    
    cartService1.addItem({ id: 1, name: 'Product', price: 10, quantity: 1 });
    expect(cartService2.getItems().length).toBe(1);  // Share state
  });
});
```

4. **Use observables for reactive state** - BehaviorSubject + observables make state changes propagate automatically to all subscribers.

5. **Persist important state** - Cart state should survive page refreshes. Use localStorage or session storage for client-side persistence.

6. **Monitor service instances in dev mode** - I added a counter to track instances:

```typescript
@Injectable({
  providedIn: 'root'
})
export class CartService {
  private static instanceCount = 0;
  private instanceId: number;
  
  constructor() {
    CartService.instanceCount++;
    this.instanceId = CartService.instanceCount;
    
    if (!environment.production) {
      console.warn(`CartService instance #${this.instanceId} created`);
      if (CartService.instanceCount > 1) {
        console.error('MULTIPLE CART SERVICE INSTANCES DETECTED!');
      }
    }
  }
}
```

This bug taught me to **always be explicit about service scope** and to **prefer `providedIn: 'root'` as the default**. Module-level providers should be the exception, not the rule.

---

## Key Takeaways

1. **Angular resolves dependencies** by walking up the injector tree from the requesting component to the root injector.

2. **Hierarchical injectors** follow the component tree, with each level potentially providing its own instances.

3. **`providedIn: 'root'`** is almost always the right choice - it's tree-shakeable, creates singletons, and avoids scope bugs.

4. **Injection tokens** are required for non-class dependencies (primitives, interfaces, configuration objects).

5. **Multi-providers** allow multiple values for the same token, useful for plugins, interceptors, and guards.

6. **Tree-shaking** only works with `providedIn` - module providers are always included in the bundle.

7. **Closest injector wins** - when multiple injectors provide the same token, the nearest one in the hierarchy is used.

8. **Test your service scope** - Multi-instance bugs are subtle and hard to debug without proper testing.

---

## Pro Tips

- **Default to `providedIn: 'root'`** for 99% of services
- **Use component providers** only for truly component-scoped state
- **Avoid module providers** in lazy-loaded modules unless you need isolated state
- **Create injection tokens** for configuration and browser APIs
- **Use multi-providers** for extensible systems (plugins, interceptors)
- **Monitor bundle size** - track impact of DI choices on tree-shaking
- **Test in production mode** - some DI issues only appear in optimized builds
- **Add instance counters** in dev mode to detect multi-instance bugs early

Angular's DI system is powerful but complex. Understanding these internals helps you make better architectural decisions and avoid subtle bugs that plague production applications.

