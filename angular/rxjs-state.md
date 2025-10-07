# RxJS & State Management in Angular

## Table of Contents
- [RxJS Fundamentals](#rxjs-fundamentals)
- [Common Operators](#common-operators)
- [State Management Patterns](#state-management-patterns)
- [NgRx Basics](#ngrx-basics)
- [Memory Management](#memory-management)

---

## RxJS Fundamentals

### Question: Explain RxJS and how it's used in Angular. What are Observables and why are they important?

**Answer:**

**RxJS (Reactive Extensions for JavaScript)** is a library for reactive programming using Observables. Angular is built on RxJS, making it essential for handling async operations, state management, and event streams.

**Observables vs Promises:**

| Feature | Observable | Promise |
|---------|-----------|---------|
| Values | Multiple values over time | Single value |
| Lazy | Yes (cold by default) | No (eager) |
| Cancellable | Yes (unsubscribe) | No |
| Operators | Rich set (map, filter, etc.) | Limited (then, catch) |
| Multicast | Can be shared | Always multicast |

**Complete RxJS Examples:**

```typescript
import { 
  Observable, Subject, BehaviorSubject, ReplaySubject, AsyncSubject,
  from, of, interval, fromEvent, combineLatest, forkJoin, merge, concat
} from 'rxjs';
import {
  map, filter, debounceTime, distinctUntilChanged, switchMap,
  catchError, retry, tap, take, takeUntil, shareReplay,
  mergeMap, concatMap, exhaustMap, scan, reduce
} from 'rxjs/operators';

// 1. CREATING OBSERVABLES

// From a value
const simple$ = of(1, 2, 3, 4, 5);

// From an array
const fromArray$ = from([1, 2, 3, 4, 5]);

// From a promise
const fromPromise$ = from(fetch('/api/data'));

// From an event
const clicks$ = fromEvent(document, 'click');

// Custom observable
const custom$ = new Observable<number>(subscriber => {
  let count = 0;
  const interval = setInterval(() => {
    subscriber.next(count++);
    
    if (count > 5) {
      subscriber.complete();
      clearInterval(interval);
    }
  }, 1000);
  
  // Cleanup function
  return () => {
    clearInterval(interval);
  };
});

// 2. SUBJECTS (Hot Observables)

// Basic Subject - no initial value
const subject = new Subject<number>();
subject.next(1);
subject.subscribe(val => console.log('Sub 1:', val));
subject.next(2);  // Sub 1 receives this

// BehaviorSubject - requires initial value, replays last value
const behaviorSubject = new BehaviorSubject<number>(0);
behaviorSubject.subscribe(val => console.log('Sub A:', val));  // Immediately logs 0
behaviorSubject.next(1);
behaviorSubject.next(2);
behaviorSubject.subscribe(val => console.log('Sub B:', val));  // Immediately logs 2

// ReplaySubject - replays N values to new subscribers
const replaySubject = new ReplaySubject<number>(2);  // Buffer size 2
replaySubject.next(1);
replaySubject.next(2);
replaySubject.next(3);
replaySubject.subscribe(val => console.log('Replay:', val));  // Logs 2, 3

// AsyncSubject - emits only last value on complete
const asyncSubject = new AsyncSubject<number>();
asyncSubject.next(1);
asyncSubject.next(2);
asyncSubject.complete();
asyncSubject.subscribe(val => console.log('Async:', val));  // Logs 2

// 3. REAL-WORLD ANGULAR SERVICE WITH RXJS

@Injectable({ providedIn: 'root' })
export class UserService {
  // Private subject for internal state management
  private usersSubject = new BehaviorSubject<User[]>([]);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private errorSubject = new Subject<string>();
  
  // Public observables for components
  public users$ = this.usersSubject.asObservable();
  public loading$ = this.loadingSubject.asObservable();
  public error$ = this.errorSubject.asObservable();
  
  constructor(private http: HttpClient) {}
  
  // Load users with error handling and retry
  loadUsers(): void {
    this.loadingSubject.next(true);
    
    this.http.get<User[]>('/api/users')
      .pipe(
        retry(3),  // Retry failed requests 3 times
        catchError(error => {
          this.errorSubject.next('Failed to load users');
          return of([]);  // Return empty array on error
        }),
        tap(() => this.loadingSubject.next(false))
      )
      .subscribe(users => {
        this.usersSubject.next(users);
      });
  }
  
  // Search with debounce
  searchUsers(searchTerm$: Observable<string>): Observable<User[]> {
    return searchTerm$.pipe(
      debounceTime(300),  // Wait 300ms after user stops typing
      distinctUntilChanged(),  // Only emit if value changed
      switchMap(term => {
        if (!term.trim()) {
          return of([]);
        }
        return this.http.get<User[]>(`/api/users/search?q=${term}`);
      }),
      catchError(error => {
        console.error('Search failed', error);
        return of([]);
      })
    );
  }
  
  // Get user by ID with caching
  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      shareReplay(1),  // Cache the result
      catchError(error => {
        throw new Error(`Failed to load user: ${error.message}`);
      })
    );
  }
}

// 4. COMPONENT USING THE SERVICE

@Component({
  selector: 'app-user-search',
  template: `
    <div class="search-container">
      <input 
        type="text" 
        [formControl]="searchControl"
        placeholder="Search users..."
      />
      
      <div *ngIf="loading$ | async" class="loading">
        Searching...
      </div>
      
      <div *ngIf="error$ | async as error" class="error">
        {{ error }}
      </div>
      
      <ul class="user-list">
        <li *ngFor="let user of users$ | async">
          {{ user.name }}
        </li>
      </ul>
    </div>
  `
})
export class UserSearchComponent implements OnInit, OnDestroy {
  searchControl = new FormControl('');
  users$!: Observable<User[]>;
  loading$!: Observable<boolean>;
  error$!: Observable<string>;
  
  private destroy$ = new Subject<void>();
  
  constructor(private userService: UserService) {}
  
  ngOnInit(): void {
    // Set up search stream
    this.users$ = this.userService.searchUsers(
      this.searchControl.valueChanges.pipe(
        takeUntil(this.destroy$)
      )
    );
    
    this.loading$ = this.userService.loading$;
    this.error$ = this.userService.error$;
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Common RxJS Patterns in Angular:**

```typescript
// 1. Type-ahead search
@Component({
  selector: 'app-search',
  template: `<input [formControl]="search" />`
})
export class SearchComponent {
  search = new FormControl('');
  
  results$ = this.search.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.searchService.search(term)),
    catchError(() => of([]))
  );
}

// 2. Combining multiple streams
@Component({
  selector: 'app-dashboard',
  template: `...`
})
export class DashboardComponent {
  // Wait for all to complete
  data$ = forkJoin({
    users: this.userService.getUsers(),
    products: this.productService.getProducts(),
    stats: this.statsService.getStats()
  });
  
  // Emit whenever any emits
  combined$ = combineLatest([
    this.users$,
    this.filter$,
    this.sort$
  ]).pipe(
    map(([users, filter, sort]) => 
      this.applyFiltersAndSort(users, filter, sort)
    )
  );
}

// 3. Polling
@Component({
  selector: 'app-live-data',
  template: `...`
})
export class LiveDataComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // Poll every 5 seconds
    interval(5000).pipe(
      switchMap(() => this.dataService.getData()),
      takeUntil(this.destroy$)
    ).subscribe(data => {
      console.log('Updated data:', data);
    });
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// 4. Master-detail pattern
@Component({
  selector: 'app-master-detail',
  template: `...`
})
export class MasterDetailComponent {
  selectedId$ = new BehaviorSubject<string | null>(null);
  
  detail$ = this.selectedId$.pipe(
    filter(id => id !== null),
    switchMap(id => this.service.getDetail(id!)),
    catchError(error => {
      console.error('Failed to load detail', error);
      return of(null);
    })
  );
  
  selectItem(id: string): void {
    this.selectedId$.next(id);
  }
}
```

**Pro Tip:** The most common mistake is not understanding the difference between **hot and cold observables**. HTTP requests are cold (each subscription triggers a new request), while Subjects are hot (shared execution). Use `shareReplay()` to convert cold to hot and cache results. Also, **always unsubscribe** from subscriptions in components to prevent memory leaks - use the `takeUntil` pattern with a destroy Subject, or use the `async` pipe which handles subscriptions automatically.

---

## Common Operators

### Question: Explain the difference between switchMap, mergeMap, concatMap, and exhaustMap. When should you use each?

**Answer:**

These are **flattening operators** that handle higher-order observables (observables of observables). Understanding when to use each is crucial for building correct, performant Angular applications.

**Visual Comparison:**

```
SOURCE:    ----A--------B--------C-------|

switchMap: ----a1-a2----b1-b2----c1-c2--|  (cancels previous)
mergeMap:  ----a1-a2----b1-b2----c1-c2--|  (all run in parallel)
                 a3       a3-b3    a4-b4-c3
concatMap: ----a1-a2-a3-b1-b2-b3-c1-c2-c3|  (queued sequentially)
exhaustMap:----a1-a2-a3----------c1-c2-c3|  (ignores while active)
```

**Complete Comparison:**

```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { fromEvent, interval } from 'rxjs';
import { switchMap, mergeMap, concatMap, exhaustMap, take } from 'rxjs/operators';

@Component({
  selector: 'app-operator-demo',
  template: `
    <div class="demo-container">
      <button #searchBtn>Search (switchMap)</button>
      <button #logBtn>Log (mergeMap)</button>
      <button #saveBtn>Save (concatMap)</button>
      <button #submitBtn>Submit (exhaustMap)</button>
    </div>
  `
})
export class OperatorDemoComponent implements OnInit {
  constructor(private http: HttpClient) {}
  
  ngOnInit(): void {
    this.demonstrateSwitchMap();
    this.demonstrateMergeMap();
    this.demonstrateConcatMap();
    this.demonstrateExhaustMap();
  }
  
  // 1. switchMap - CANCELS previous, uses latest
  // ✅ Use for: Search, Navigation, Latest value scenarios
  demonstrateSwitchMap(): void {
    const searchBtn = document.querySelector('#searchBtn');
    
    fromEvent(searchBtn!, 'click').pipe(
      switchMap(() => {
        console.log('Starting search...');
        return this.http.get('/api/search');
      })
    ).subscribe(result => {
      console.log('Search result:', result);
    });
    
    // Real-world example: Type-ahead search
    const searchInput = new FormControl('');
    searchInput.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => {
        // Previous search is cancelled automatically
        return this.http.get(`/api/search?q=${term}`);
      })
    ).subscribe(results => {
      console.log('Search results:', results);
    });
  }
  
  // 2. mergeMap - ALL run in parallel, order NOT maintained
  // ✅ Use for: Independent operations, logging, analytics
  demonstrateMergeMap(): void {
    const logBtn = document.querySelector('#logBtn');
    
    fromEvent(logBtn!, 'click').pipe(
      mergeMap(() => {
        console.log('Sending log...');
        return this.http.post('/api/log', { event: 'click' });
      })
    ).subscribe(result => {
      console.log('Log sent:', result);
    });
    
    // Real-world example: Parallel API calls
    const userIds = [1, 2, 3, 4, 5];
    from(userIds).pipe(
      mergeMap(id => 
        // All 5 requests happen simultaneously
        this.http.get(`/api/users/${id}`)
      )
    ).subscribe(user => {
      console.log('User loaded:', user);
    });
    
    // Limit concurrency
    from(userIds).pipe(
      mergeMap(
        id => this.http.get(`/api/users/${id}`),
        3  // Max 3 concurrent requests
      )
    ).subscribe(user => {
      console.log('User loaded:', user);
    });
  }
  
  // 3. concatMap - QUEUES requests, maintains order
  // ✅ Use for: Order-dependent operations, sequential processing
  demonstrateConcatMap(): void {
    const saveBtn = document.querySelector('#saveBtn');
    
    fromEvent(saveBtn!, 'click').pipe(
      concatMap(() => {
        console.log('Saving...');
        return this.http.post('/api/save', { data: 'example' });
      })
    ).subscribe(result => {
      console.log('Saved:', result);
    });
    
    // Real-world example: Sequential file uploads
    const files = [file1, file2, file3];
    from(files).pipe(
      concatMap(file => {
        // Files upload one at a time, in order
        return this.uploadFile(file);
      })
    ).subscribe(response => {
      console.log('File uploaded:', response);
    });
  }
  
  // 4. exhaustMap - IGNORES new while active
  // ✅ Use for: Form submissions, login, one-at-a-time operations
  demonstrateExhaustMap(): void {
    const submitBtn = document.querySelector('#submitBtn');
    
    fromEvent(submitBtn!, 'click').pipe(
      exhaustMap(() => {
        console.log('Submitting...');
        return this.http.post('/api/submit', { data: 'example' });
      })
    ).subscribe(result => {
      console.log('Submitted:', result);
    });
    
    // Real-world example: Login form
    const loginForm = document.querySelector('#loginForm');
    fromEvent(loginForm!, 'submit').pipe(
      exhaustMap((event: Event) => {
        event.preventDefault();
        // Additional clicks ignored until this completes
        return this.authService.login(username, password);
      })
    ).subscribe(result => {
      console.log('Login successful');
    });
  }
  
  private uploadFile(file: File): Observable<any> {
    const formData = new FormData();
    formData.append('file', file);
    return this.http.post('/api/upload', formData);
  }
}
```

**Decision Matrix:**

```typescript
// Use switchMap when:
// - Only care about the latest result
// - Want to cancel previous operations
// - Examples: Search, autocomplete, navigation

searchInput.valueChanges.pipe(
  switchMap(term => this.search(term))  // ✅ Cancel old searches
)

// Use mergeMap when:
// - All operations are independent
// - Order doesn't matter
// - Want maximum concurrency
// - Examples: Logging, analytics, parallel data fetching

clicks$.pipe(
  mergeMap(() => this.sendAnalytics())  // ✅ All events tracked
)

// Use concatMap when:
// - Order matters
// - Need sequential execution
// - One must complete before next starts
// - Examples: File uploads, sequential updates

files$.pipe(
  concatMap(file => this.uploadFile(file))  // ✅ Upload in order
)

// Use exhaustMap when:
// - Want to ignore new events while processing
// - Prevent duplicate operations
// - Examples: Form submission, login, save button

submitBtn.click$.pipe(
  exhaustMap(() => this.saveData())  // ✅ Prevent double-submit
)
```

**Common Mistakes:**

```typescript
// ❌ WRONG: Using mergeMap for search
searchInput.valueChanges.pipe(
  mergeMap(term => this.search(term))  // Multiple concurrent searches!
)

// ❌ WRONG: Using switchMap for file uploads
files$.pipe(
  switchMap(file => this.uploadFile(file))  // Cancels previous uploads!
)

// ❌ WRONG: Using concatMap for independent operations
clicks$.pipe(
  concatMap(() => this.sendAnalytics())  // Unnecessarily queued
)

// ❌ WRONG: Using exhaustMap for search
searchInput.valueChanges.pipe(
  exhaustMap(term => this.search(term))  // Ignores typing!
)
```

**Advanced Pattern: Error Handling:**

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  constructor(private http: HttpClient) {}
  
  // Retry with exponential backoff
  getData(): Observable<Data> {
    return this.http.get<Data>('/api/data').pipe(
      retry({
        count: 3,
        delay: (error, retryCount) => {
          console.log(`Retry attempt ${retryCount}`);
          return timer(Math.pow(2, retryCount) * 1000);  // 2s, 4s, 8s
        }
      }),
      catchError(error => {
        console.error('All retries failed', error);
        return throwError(() => new Error('Failed to load data'));
      })
    );
  }
  
  // Conditional retry
  saveData(data: Data): Observable<any> {
    return this.http.post('/api/data', data).pipe(
      retryWhen(errors =>
        errors.pipe(
          mergeMap((error, index) => {
            // Retry only on 5xx errors, max 3 times
            if (index >= 3) {
              return throwError(() => error);
            }
            if (error.status >= 500) {
              return timer(1000 * (index + 1));
            }
            return throwError(() => error);
          })
        )
      )
    );
  }
}
```

**Pro Tip:** In interviews, demonstrate you know the difference by asking: **"Will previous operations be cancelled?"** and **"Does order matter?"**. Most candidates confuse `switchMap` and `mergeMap`. Remember: **switchMap switches to new, mergeMap merges all, concatMap concatenates in order, exhaustMap exhausts current before accepting new**. A common production bug is using `mergeMap` for search - it creates race conditions where old search results overwrite new ones.

---

*Continue reading: [State Management Patterns](#state-management-patterns), [NgRx Basics](#ngrx-basics)*
