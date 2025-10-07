# Angular Routing & Navigation

## Table of Contents
- [Router Basics](#router-basics)
- [Route Guards](#route-guards)
- [Lazy Loading](#lazy-loading)
- [Route Resolvers](#route-resolvers)
- [Advanced Routing](#advanced-routing)

---

## Router Basics

### Question: Explain Angular Router and how to implement complex routing scenarios.

**Answer:**

The **Angular Router** enables navigation between views, lazy loading, route guards, and complex routing patterns. It's a core part of building SPAs.

**Complete Routing Setup:**

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { AuthGuard } from './core/guards/auth.guard';
import { AdminGuard } from './core/guards/admin.guard';
import { UserResolver } from './core/resolvers/user.resolver';

const routes: Routes = [
  // Default route
  {
    path: '',
    redirectTo: '/home',
    pathMatch: 'full'
  },
  
  // Simple route
  {
    path: 'home',
    component: HomeComponent
  },
  
  // Route with parameters
  {
    path: 'users/:id',
    component: UserDetailComponent
  },
  
  // Route with query parameters and fragment
  {
    path: 'search',
    component: SearchComponent
    // Accessed via: /search?q=angular&filter=all#results
  },
  
  // Route with resolver (pre-fetch data)
  {
    path: 'profile',
    component: ProfileComponent,
    resolve: {
      user: UserResolver
    }
  },
  
  // Route with guard (protected route)
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [AuthGuard],
    canDeactivate: [UnsavedChangesGuard]
  },
  
  // Nested routes (child routes)
  {
    path: 'products',
    component: ProductsComponent,
    children: [
      {
        path: '',
        component: ProductListComponent
      },
      {
        path: ':id',
        component: ProductDetailComponent
      },
      {
        path: ':id/edit',
        component: ProductEditComponent,
        canDeactivate: [UnsavedChangesGuard]
      }
    ]
  },
  
  // Lazy loaded module
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.module')
      .then(m => m.AdminModule),
    canLoad: [AdminGuard],  // Prevent loading if not admin
    canActivate: [AuthGuard]
  },
  
  // Multiple outlets (named outlets)
  {
    path: 'inbox',
    component: InboxComponent,
    children: [
      {
        path: ':id',
        component: MessageDetailComponent,
        outlet: 'detail'
      }
    ]
  },
  
  // Wildcard route (404)
  {
    path: '**',
    component: NotFoundComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes, {
    enableTracing: false,  // Set true for debugging
    useHash: false,  // Use HTML5 history API
    scrollPositionRestoration: 'top',  // Scroll to top on navigation
    anchorScrolling: 'enabled',  // Enable fragment scrolling
    onSameUrlNavigation: 'reload',  // Reload on same URL
    preloadingStrategy: PreloadAllModules  // Preload lazy modules
  })],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

**Navigation in Components:**

```typescript
import { Component } from '@angular/core';
import { Router, ActivatedRoute, NavigationExtras } from '@angular/router';

@Component({
  selector: 'app-navigation-demo',
  template: `
    <div class="nav-demo">
      <!-- 1. RouterLink (declarative) -->
      <a routerLink="/home">Home</a>
      <a [routerLink]="['/users', userId]">User Profile</a>
      <a [routerLink]="['/search']" [queryParams]="{q: 'angular', page: 1}">
        Search
      </a>
      
      <!-- RouterLinkActive for active state -->
      <a routerLink="/dashboard" 
         routerLinkActive="active"
         [routerLinkActiveOptions]="{exact: true}">
        Dashboard
      </a>
      
      <!-- 2. Programmatic navigation -->
      <button (click)="navigateToUser(123)">Go to User</button>
      <button (click)="navigateWithExtras()">Navigate with Options</button>
      <button (click)="goBack()">Go Back</button>
      
      <!-- Router outlet -->
      <router-outlet></router-outlet>
      
      <!-- Named outlet -->
      <router-outlet name="detail"></router-outlet>
    </div>
  `
})
export class NavigationDemoComponent {
  userId = 123;
  
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}
  
  // Basic navigation
  navigateToUser(id: number): void {
    this.router.navigate(['/users', id]);
  }
  
  // Navigation with extras
  navigateWithExtras(): void {
    const navigationExtras: NavigationExtras = {
      queryParams: { search: 'angular', page: 1 },
      fragment: 'results',
      state: { customData: 'passed via state' },
      preserveQueryParams: false,
      skipLocationChange: false,  // Don't add to browser history
      replaceUrl: false  // Replace current history entry
    };
    
    this.router.navigate(['/search'], navigationExtras);
  }
  
  // Relative navigation
  navigateRelative(): void {
    // Navigate relative to current route
    this.router.navigate(['../sibling'], { relativeTo: this.route });
    this.router.navigate(['./child'], { relativeTo: this.route });
  }
  
  // Navigate to named outlet
  navigateToOutlet(): void {
    this.router.navigate([
      {
        outlets: {
          primary: ['inbox'],
          detail: ['message', 123]
        }
      }
    ]);
  }
  
  // Go back
  goBack(): void {
    window.history.back();
    // or use Location service
    // this.location.back();
  }
  
  // Get current route
  getCurrentRoute(): void {
    console.log('Current URL:', this.router.url);
    console.log('Current state:', this.router.routerState);
  }
}
```

**Reading Route Parameters:**

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ActivatedRoute, ParamMap } from '@angular/router';
import { Subject } from 'rxjs';
import { takeUntil, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-user-detail',
  template: `
    <div *ngIf="user$ | async as user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserDetailComponent implements OnInit, OnDestroy {
  user$!: Observable<User>;
  private destroy$ = new Subject<void>();
  
  constructor(
    private route: ActivatedRoute,
    private userService: UserService
  ) {}
  
  ngOnInit(): void {
    // METHOD 1: Snapshot (for one-time read)
    const userId = this.route.snapshot.paramMap.get('id');
    const queryParam = this.route.snapshot.queryParamMap.get('filter');
    const fragment = this.route.snapshot.fragment;
    
    // METHOD 2: Observable (for dynamic updates)
    this.user$ = this.route.paramMap.pipe(
      switchMap((params: ParamMap) => {
        const id = params.get('id')!;
        return this.userService.getUser(id);
      }),
      takeUntil(this.destroy$)
    );
    
    // Query parameters (observable)
    this.route.queryParamMap.pipe(
      takeUntil(this.destroy$)
    ).subscribe(queryParams => {
      const search = queryParams.get('search');
      const page = queryParams.get('page');
      console.log('Query params:', { search, page });
    });
    
    // Data from resolver
    this.route.data.pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log('Resolved data:', data['user']);
    });
    
    // State passed during navigation
    const navigation = this.router.getCurrentNavigation();
    const state = navigation?.extras.state;
    console.log('Navigation state:', state);
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Pro Tip:** Always use **Observable route parameters** (`paramMap`) instead of snapshot when the same component might be reused with different parameters. For example, navigating from `/users/1` to `/users/2` won't recreate the component, so snapshot values won't update. Most candidates make this mistake. Also, use **route resolvers** to pre-fetch data before displaying the component, providing a better UX than showing loading spinners.

---

## Route Guards

### Question: Explain Angular route guards and implement a complete authentication guard with role-based access.

**Answer:**

**Route Guards** are interfaces that control navigation. Angular provides 5 types:
- `CanActivate` - Control if route can be activated
- `CanActivateChild` - Control if child routes can be activated
- `CanDeactivate` - Control if can leave current route
- `CanLoad` - Control if lazy module can be loaded
- `Resolve` - Pre-fetch data before activation

**Complete Guard Implementation:**

```typescript
// auth.guard.ts
import { Injectable } from '@angular/core';
import {
  CanActivate, CanActivateChild, CanLoad,
  ActivatedRouteSnapshot, RouterStateSnapshot,
  UrlTree, Router, Route, UrlSegment
} from '@angular/router';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';
import { AuthService } from '../services/auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate, CanActivateChild, CanLoad {
  
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  // Can activate single route
  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    
    return this.authService.isAuthenticated$.pipe(
      take(1),
      map(isAuthenticated => {
        if (isAuthenticated) {
          // Check role-based access
          const requiredRoles = route.data['roles'] as string[];
          if (requiredRoles) {
            const hasRole = this.authService.hasAnyRole(requiredRoles);
            if (!hasRole) {
              console.warn('Access denied: insufficient permissions');
              return this.router.createUrlTree(['/forbidden']);
            }
          }
          return true;
        }
        
        // Store attempted URL for redirect after login
        this.authService.redirectUrl = state.url;
        console.warn('Access denied: not authenticated');
        return this.router.createUrlTree(['/login'], {
          queryParams: { returnUrl: state.url }
        });
      })
    );
  }
  
  // Can activate child routes
  canActivateChild(
    childRoute: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    return this.canActivate(childRoute, state);
  }
  
  // Can load lazy module
  canLoad(
    route: Route,
    segments: UrlSegment[]
  ): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    return this.authService.isAuthenticated$.pipe(
      take(1),
      map(isAuthenticated => {
        if (!isAuthenticated) {
          console.warn('Cannot load module: not authenticated');
          return this.router.createUrlTree(['/login']);
        }
        return true;
      })
    );
  }
}

// unsaved-changes.guard.ts
export interface CanComponentDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable({ providedIn: 'root' })
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  
  canDeactivate(
    component: CanComponentDeactivate,
    currentRoute: ActivatedRouteSnapshot,
    currentState: RouterStateSnapshot,
    nextState?: RouterStateSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    
    // Allow navigation if component says it's OK
    return component.canDeactivate ? component.canDeactivate() : true;
  }
}

// admin.guard.ts
@Injectable({ providedIn: 'root' })
export class AdminGuard implements CanActivate {
  
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canActivate(): Observable<boolean | UrlTree> {
    return this.authService.currentUser$.pipe(
      take(1),
      map(user => {
        if (user && user.role === 'admin') {
          return true;
        }
        console.warn('Admin access required');
        return this.router.createUrlTree(['/forbidden']);
      })
    );
  }
}
```

**Using Guards in Routes:**

```typescript
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [AuthGuard],  // Must be authenticated
    data: { roles: ['user', 'admin'] }  // Passed to guard
  },
  {
    path: 'admin',
    component: AdminLayoutComponent,
    canActivate: [AuthGuard, AdminGuard],  // Multiple guards (AND logic)
    canActivateChild: [AuthGuard],
    children: [
      { path: 'users', component: UserManagementComponent },
      { path: 'settings', component: SettingsComponent }
    ]
  },
  {
    path: 'editor',
    component: EditorComponent,
    canDeactivate: [UnsavedChangesGuard]  // Prevent leaving with unsaved changes
  },
  {
    path: 'premium',
    loadChildren: () => import('./premium/premium.module').then(m => m.PremiumModule),
    canLoad: [AuthGuard]  // Don't even load module if not authenticated
  }
];
```

**Component with Deactivation Guard:**

```typescript
@Component({
  selector: 'app-editor',
  template: `
    <div class="editor">
      <textarea [(ngModel)]="content" (input)="markAsDirty()"></textarea>
      <button (click)="save()">Save</button>
    </div>
  `
})
export class EditorComponent implements CanComponentDeactivate {
  content = '';
  isDirty = false;
  
  markAsDirty(): void {
    this.isDirty = true;
  }
  
  save(): void {
    // Save logic
    this.isDirty = false;
  }
  
  canDeactivate(): Observable<boolean> | boolean {
    if (!this.isDirty) {
      return true;
    }
    
    // Show confirmation dialog
    return this.dialogService.confirm({
      title: 'Unsaved Changes',
      message: 'You have unsaved changes. Do you want to leave?',
      confirmText: 'Leave',
      cancelText: 'Stay'
    });
  }
  
  // Alternative: Use browser's built-in confirmation
  @HostListener('window:beforeunload', ['$event'])
  unloadNotification($event: any): void {
    if (this.isDirty) {
      $event.returnValue = true;
    }
  }
}
```

**Advanced: Custom Guard with Permissions:**

```typescript
@Injectable({ providedIn: 'root' })
export class PermissionGuard implements CanActivate {
  
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canActivate(route: ActivatedRouteSnapshot): Observable<boolean | UrlTree> {
    const requiredPermission = route.data['permission'] as string;
    
    return this.authService.currentUser$.pipe(
      take(1),
      map(user => {
        if (!user) {
          return this.router.createUrlTree(['/login']);
        }
        
        if (user.permissions.includes(requiredPermission)) {
          return true;
        }
        
        // Log unauthorized access attempt
        this.logService.warn('Unauthorized access attempt', {
          user: user.id,
          permission: requiredPermission,
          route: route.url
        });
        
        return this.router.createUrlTree(['/forbidden']);
      })
    );
  }
}

// Usage
{
  path: 'delete-user',
  component: DeleteUserComponent,
  canActivate: [PermissionGuard],
  data: { permission: 'user:delete' }
}
```

**Pro Tip:** Use `UrlTree` return type instead of calling `router.navigate()` and returning false. This is cleaner and prevents navigation timing issues. Also, implement `CanLoad` guard for lazy-loaded modules to prevent unauthorized users from even downloading the module code, saving bandwidth and improving security. Most candidates only implement `CanActivate` and miss this optimization.

---

*Continue reading: [Lazy Loading](#lazy-loading), [Route Resolvers](#route-resolvers)*
