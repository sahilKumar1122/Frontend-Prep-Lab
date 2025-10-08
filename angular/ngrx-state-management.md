# NgRx State Management - Architecture Deep Dive

## Table of Contents
- [Complete Data Flow](#complete-data-flow)
- [Action Lifecycle Internals](#action-lifecycle-internals)
- [Reducer Mechanics and Immutability](#reducer-mechanics-and-immutability)
- [Selector Memoization](#selector-memoization)
- [Effects and RxJS Integration](#effects-and-rxjs-integration)
- [Change Detection Integration](#change-detection-integration)
- [When NgRx is the Wrong Choice](#when-ngrx-is-the-wrong-choice)
- [NgRx vs Akita vs NGXS](#ngrx-vs-akita-vs-ngxs)
- [Decision Framework](#decision-framework)
- [Managing Boilerplate](#managing-boilerplate)
- [Race Conditions in Effects](#race-conditions-in-effects)

---

## Complete Data Flow

### Question: Explain how NgRx integrates with Angular - walk through the entire data flow from component dispatch to UI re-rendering, covering action lifecycle, reducer mechanics, selector memoization, effects integration, and change detection.

**Answer:**

NgRx isn't just "Redux for Angular" - it's a sophisticated reactive state management system deeply integrated with Angular's change detection, RxJS observables, and dependency injection. Let's trace the complete data flow.

---

## Action Lifecycle Internals

### **From store.dispatch() to Reducer Execution**

```typescript
// Component dispatches action
@Component({
  template: `
    <button (click)="loadUser()">Load User</button>
    <div>{{ user$ | async | json }}</div>
  `
})
export class UserComponent {
  user$ = this.store.select(selectUser);
  
  constructor(private store: Store) {}
  
  loadUser(): void {
    // 1. Component dispatches action
    this.store.dispatch(loadUser({ userId: 123 }));
  }
}
```

**What happens internally when you call `store.dispatch()`:**

```typescript
// Simplified NgRx Store implementation
class Store<T> extends Observable<T> implements Observer<Action> {
  private actionsObserver: ActionsSubject;
  private reducerManager: ReducerManager;
  private state$: StateObservable;
  
  constructor(
    state$: StateObservable,
    actionsObserver: ActionsSubject,
    reducerManager: ReducerManager
  ) {
    super();
    this.state$ = state$;
    this.actionsObserver = actionsObserver;
    this.reducerManager = reducerManager;
  }
  
  // When you call store.dispatch()
  dispatch<V extends Action = Action>(action: V): void {
    // Step 1: Push action to actions stream
    this.actionsObserver.next(action);
    // This triggers the entire pipeline!
  }
  
  select<K>(mapFn: (state: T) => K): Observable<K> {
    return this.state$.pipe(
      map(mapFn),
      distinctUntilChanged()  // Only emit when value actually changes
    );
  }
}
```

**Step-by-step action flow:**

```typescript
// Step 1: ActionsSubject receives action
class ActionsSubject extends BehaviorSubject<Action> {
  next(action: Action): void {
    // Validate action has 'type' property
    if (!action.type) {
      throw new TypeError(`Actions must have a type property`);
    }
    
    // Push to all subscribers (reducers, effects, devtools)
    super.next(action);
  }
}

// Step 2: ReducerManager processes action
class ReducerManager extends BehaviorSubject<ActionReducer<any, any>> {
  private reducers: Map<string, ActionReducer<any, any>>;
  
  constructor() {
    super((state, action) => state);  // Initial no-op reducer
    this.reducers = new Map();
  }
  
  addReducer(key: string, reducer: ActionReducer<any, any>): void {
    this.reducers.set(key, reducer);
    this.updateReducers();
  }
  
  private updateReducers(): void {
    // Combine all feature reducers into single root reducer
    const rootReducer = this.combineReducers();
    this.next(rootReducer);
  }
  
  private combineReducers(): ActionReducer<any, any> {
    return (state: any, action: Action) => {
      // Call each feature reducer
      const nextState = {};
      
      for (const [key, reducer] of this.reducers.entries()) {
        nextState[key] = reducer(state[key], action);
      }
      
      return nextState;
    };
  }
}

// Step 3: ScannedActionsSubject combines state + actions
class ScannedActionsSubject extends Subject<any> {
  constructor(
    private actionsSubject: ActionsSubject,
    private reducerManager: ReducerManager,
    private initialState: any
  ) {
    super();
    this.setupScan();
  }
  
  private setupScan(): void {
    // This is the CORE of NgRx - scan operator accumulates state
    this.actionsSubject.pipe(
      // For each action...
      withLatestFrom(this.reducerManager),
      // Apply current reducer to current state
      scan((state, [action, reducer]) => {
        return reducer(state, action);
      }, this.initialState)
    ).subscribe((state) => {
      // Emit new state to all subscribers
      this.next(state);
    });
  }
}
```

**Complete flow visualization:**

```
1. Component calls store.dispatch(loadUser({ userId: 123 }))
   ↓
2. ActionsSubject.next(action)
   ↓
3. Action pushed to actions$ stream
   ↓
   ├─→ Effects subscribe to actions$ (parallel)
   │   └→ Effect triggers HTTP call
   │       └→ On success, dispatches loadUserSuccess()
   │
   └─→ ScannedActionsSubject (main pipeline)
       ↓
4. withLatestFrom(reducerManager) combines [action, currentReducer]
   ↓
5. scan() applies: newState = reducer(oldState, action)
   ↓
6. New state emitted to state$ stream
   ↓
7. Selectors subscribed to state$ receive new state
   ↓
8. Selectors check if their slice changed (memoization)
   ↓
9. If changed, emit to component
   ↓
10. Component's async pipe triggers change detection
   ↓
11. UI updates
```

---

## Reducer Mechanics and Immutability

### **How Reducers Produce Immutable State**

```typescript
// Action definition
export const loadUserSuccess = createAction(
  '[User API] Load User Success',
  props<{ user: User }>()
);

// State interface
export interface UserState {
  entities: { [id: number]: User };
  selectedUserId: number | null;
  loading: boolean;
  error: string | null;
}

// Initial state
const initialState: UserState = {
  entities: {},
  selectedUserId: null,
  loading: false,
  error: null
};

// Reducer using createReducer
export const userReducer = createReducer(
  initialState,
  
  on(loadUser, (state) => ({
    ...state,
    loading: true,
    error: null
  })),
  
  on(loadUserSuccess, (state, { user }) => ({
    ...state,
    entities: {
      ...state.entities,
      [user.id]: user
    },
    loading: false
  })),
  
  on(loadUserFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error: error
  }))
);
```

**What createReducer does internally:**

```typescript
// Simplified createReducer implementation
function createReducer<S>(
  initialState: S,
  ...ons: ReducerTypes<S, ActionCreator[]>[]
): ActionReducer<S> {
  // Build map of action types to reducer functions
  const map = new Map<string, (state: S, action: Action) => S>();
  
  for (const on of ons) {
    for (const type of on.types) {
      map.set(type, on.reducer);
    }
  }
  
  // Return reducer function
  return function(state: S = initialState, action: Action): S {
    const reducer = map.get(action.type);
    
    if (reducer) {
      // Found matching reducer - execute it
      return reducer(state, action);
    }
    
    // No matching reducer - return same state reference
    return state;
  };
}
```

### **Why Immutability is Crucial**

**Reason 1: Change Detection Optimization**

```typescript
// With immutability
const oldState = { count: 1, items: [1, 2, 3] };
const newState = { ...oldState, count: 2 };

// Reference equality check (O(1) - instant!)
if (oldState !== newState) {
  console.log('State changed!');
  // Trigger change detection
}

// Without immutability
const mutableState = { count: 1, items: [1, 2, 3] };
mutableState.count = 2;  // Mutated in place

// Reference equality check fails!
if (mutableState !== mutableState) {  // Always false
  // Never triggers!
}

// Would need deep equality check (O(n) - slow!)
function deepEqual(obj1: any, obj2: any): boolean {
  // Must traverse entire object tree
  // For large state: extremely expensive
}
```

**Reason 2: OnPush Change Detection**

```typescript
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">
      {{ user.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  @Input() users: User[] = [];
  
  // OnPush only checks when:
  // 1. Input reference changes
  // 2. Event fires in component
  // 3. Async pipe emits
}

// Parent component
@Component({
  template: `<app-user-list [users]="users"></app-user-list>`
})
export class ParentComponent {
  users = [{ id: 1, name: 'John' }];
  
  // ❌ BAD: Mutation - OnPush won't detect
  addUserBad(): void {
    this.users.push({ id: 2, name: 'Jane' });
    // Same array reference - OnPush child won't update!
  }
  
  // ✅ GOOD: New reference - OnPush detects
  addUserGood(): void {
    this.users = [...this.users, { id: 2, name: 'Jane' }];
    // New array reference - OnPush child updates!
  }
}
```

**Reason 3: Time-Travel Debugging**

```typescript
// With immutability, you can store state snapshots
class StateHistory {
  private history: any[] = [];
  private currentIndex = -1;
  
  push(state: any): void {
    // Truncate forward history
    this.history = this.history.slice(0, this.currentIndex + 1);
    
    // Add new state snapshot
    this.history.push(state);
    this.currentIndex++;
  }
  
  undo(): any | null {
    if (this.currentIndex > 0) {
      this.currentIndex--;
      return this.history[this.currentIndex];
    }
    return null;
  }
  
  redo(): any | null {
    if (this.currentIndex < this.history.length - 1) {
      this.currentIndex++;
      return this.history[this.currentIndex];
    }
    return null;
  }
}

// Without immutability:
const state = { count: 1 };
history.push(state);
state.count = 2;  // Mutated!
history.push(state);  // Same reference!
// Can't go back to count: 1 - it's been mutated!
```

**Reason 4: Predictable State Updates**

```typescript
// ❌ BAD: Mutation leads to bugs
function badReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      state.count++;  // MUTATION!
      return state;
      
    case 'ADD_ITEM':
      state.items.push(action.payload);  // MUTATION!
      return state;
  }
  return state;
}

// What happens:
// 1. Multiple reducers might receive same state reference
// 2. One reducer mutates it
// 3. Other reducers see mutated state
// 4. Unpredictable behavior!

// ✅ GOOD: Immutable updates are predictable
function goodReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
      
    case 'ADD_ITEM':
      return {
        ...state,
        items: [...state.items, action.payload]
      };
  }
  return state;
}
```

**Immutability Helpers:**

```typescript
// Native spread (shallow)
const newState = { ...oldState, count: 2 };

// Nested updates
const newState = {
  ...oldState,
  user: {
    ...oldState.user,
    profile: {
      ...oldState.user.profile,
      name: 'New Name'
    }
  }
};

// Using Immer (recommended for deep updates)
import { produce } from 'immer';

const newState = produce(oldState, draft => {
  draft.user.profile.name = 'New Name';
  // Looks like mutation, but Immer creates new immutable state
});

// NgRx Entity adapter (optimized)
import { createEntityAdapter } from '@ngrx/entity';

const adapter = createEntityAdapter<User>();

// Immutable operations built-in
adapter.addOne(user, state);
adapter.updateOne({ id: 1, changes: { name: 'Updated' } }, state);
adapter.removeOne(1, state);
```

---

## Selector Memoization

### **How Selectors Prevent Unnecessary Change Detection**

```typescript
// Basic selector
export const selectUserState = (state: AppState) => state.user;

export const selectUser = createSelector(
  selectUserState,
  (state) => state.entities[state.selectedUserId]
);

export const selectUserFullName = createSelector(
  selectUser,
  (user) => user ? `${user.firstName} ${user.lastName}` : ''
);
```

**What createSelector does internally:**

```typescript
// Simplified createSelector implementation
function createSelector<S, R>(
  ...args: any[]
): MemoizedSelector<S, R> {
  const projector = args.pop() as (...args: any[]) => R;
  const selectors = args as Selector<S, any>[];
  
  // Memoization state
  let lastArgs: any[] | null = null;
  let lastResult: R | null = null;
  
  const memoizedSelector = (state: S): R => {
    // Get input values from parent selectors
    const args = selectors.map(selector => selector(state));
    
    // Check if inputs changed (shallow equality)
    if (lastArgs && argsEqual(args, lastArgs)) {
      // Inputs unchanged - return cached result
      console.log('Selector: using cached result');
      return lastResult!;
    }
    
    // Inputs changed - recalculate
    console.log('Selector: recalculating');
    lastArgs = args;
    lastResult = projector(...args);
    return lastResult;
  };
  
  return memoizedSelector as MemoizedSelector<S, R>;
}

function argsEqual(args1: any[], args2: any[]): boolean {
  if (args1.length !== args2.length) return false;
  
  for (let i = 0; i < args1.length; i++) {
    if (args1[i] !== args2[i]) {  // Reference equality!
      return false;
    }
  }
  
  return true;
}
```

**Why Memoization Matters:**

```typescript
// Component using selectors
@Component({
  template: `
    <div>{{ userName$ | async }}</div>
    <div>{{ userName$ | async }}</div>
    <div>{{ userName$ | async }}</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent {
  // Selector called multiple times per CD cycle
  userName$ = this.store.select(selectUserFullName);
  
  constructor(private store: Store) {}
}

// Without memoization:
// - selectUserFullName() executes 3 times per CD cycle
// - String concatenation happens 3 times
// - If complex calculation: major performance hit

// With memoization:
// - First call: calculates and caches result
// - Second call: returns cached result (no calculation)
// - Third call: returns cached result (no calculation)
```

**Complex Selector Example:**

```typescript
// Expensive derived state
export const selectFilteredAndSortedUsers = createSelector(
  selectUsers,           // Input 1: User[]
  selectSearchQuery,     // Input 2: string
  selectSortField,       // Input 3: string
  selectSortDirection,   // Input 4: 'asc' | 'desc'
  (users, query, field, direction) => {
    console.log('Computing filtered and sorted users...');
    
    // Expensive operations
    let result = users;
    
    // Filter
    if (query) {
      result = result.filter(user =>
        user.name.toLowerCase().includes(query.toLowerCase()) ||
        user.email.toLowerCase().includes(query.toLowerCase())
      );
    }
    
    // Sort
    result = [...result].sort((a, b) => {
      const aVal = a[field];
      const bVal = b[field];
      
      if (direction === 'asc') {
        return aVal > bVal ? 1 : -1;
      } else {
        return aVal < bVal ? 1 : -1;
      }
    });
    
    return result;
  }
);

// Usage
@Component({
  template: `
    <input [formControl]="searchControl" />
    <button (click)="sort('name')">Sort by Name</button>
    
    <div *ngFor="let user of users$ | async">
      {{ user.name }} - {{ user.email }}
    </div>
  `
})
export class UserListComponent {
  users$ = this.store.select(selectFilteredAndSortedUsers);
  searchControl = new FormControl('');
  
  constructor(private store: Store) {
    this.searchControl.valueChanges
      .pipe(debounceTime(300))
      .subscribe(query => {
        this.store.dispatch(setSearchQuery({ query }));
      });
  }
  
  sort(field: string): void {
    this.store.dispatch(setSortField({ field }));
  }
}

// Performance:
// - Without memoization: Filter + sort on EVERY change detection (50-100ms)
// - With memoization: Only when inputs actually change (0ms cached, 50-100ms recalc)
```

**Selector Composition:**

```typescript
// Build complex selectors from simple ones
export const selectUser = createSelector(
  selectUserState,
  (state) => state.entities[state.selectedUserId]
);

export const selectUserOrders = createSelector(
  selectOrdersState,
  selectUser,
  (ordersState, user) => {
    if (!user) return [];
    return Object.values(ordersState.entities)
      .filter(order => order.userId === user.id);
  }
);

export const selectUserOrdersTotal = createSelector(
  selectUserOrders,
  (orders) => {
    return orders.reduce((sum, order) => sum + order.total, 0);
  }
);

// Each selector memoizes independently
// Only recalculates when its inputs change
```

---

## Effects and RxJS Integration

### **How Effects Work with Observables**

```typescript
// Effect definition
@Injectable()
export class UserEffects {
  // Effect that loads user on action
  loadUser$ = createEffect(() =>
    this.actions$.pipe(
      // 1. Listen for specific action type
      ofType(loadUser),
      
      // 2. Extract action payload
      map(action => action.userId),
      
      // 3. Cancel previous request if new one arrives
      switchMap(userId =>
        // 4. Make HTTP call
        this.userService.getUser(userId).pipe(
          // 5. On success, dispatch success action
          map(user => loadUserSuccess({ user })),
          
          // 6. On error, dispatch failure action
          catchError(error => of(loadUserFailure({ error: error.message })))
        )
      )
    )
  );
  
  constructor(
    private actions$: Actions,
    private userService: UserService
  ) {}
}
```

**What happens internally:**

```typescript
// Simplified Actions service
@Injectable()
class Actions extends Observable<Action> {
  constructor(private actionsSubject: ActionsSubject) {
    super();
  }
  
  lift<R>(operator: Operator<Action, R>): Observable<R> {
    const observable = new Observable<R>();
    observable.source = this.actionsSubject;
    observable.operator = operator;
    return observable;
  }
}

// ofType operator
function ofType<T extends Action>(
  ...allowedTypes: string[]
): OperatorFunction<Action, T> {
  return filter((action: Action): action is T =>
    allowedTypes.includes(action.type)
  );
}

// createEffect function
function createEffect<
  C extends EffectConfig,
  DT extends DispatchType<C>,
  R extends Observable<Action> | ((...args: any[]) => Observable<Action>)
>(
  source: () => R,
  config?: C
): R & CreateEffectMetadata {
  const effect = source();
  const effectConfig = {
    dispatch: config?.dispatch !== false,  // Default: dispatch result
    useEffectsErrorHandler: config?.useEffectsErrorHandler !== false
  };
  
  // Mark function as effect (metadata)
  (effect as any)[CREATE_EFFECT_METADATA_KEY] = effectConfig;
  
  return effect as any;
}
```

**Effect Lifecycle:**

```
1. App bootstrap
   ↓
2. EffectsModule.forRoot([UserEffects]) registers effects
   ↓
3. EffectSources subscribes to all effects
   ↓
4. Each effect subscribes to actions$ stream
   ↓
5. Action dispatched: loadUser({ userId: 123 })
   ↓
6. Actions stream emits action
   ↓
7. Effect's ofType(loadUser) filters - match!
   ↓
8. switchMap cancels any previous request
   ↓
9. HTTP call made via userService
   ↓
10. HTTP observable emits response
    ↓
11. map transforms to loadUserSuccess({ user })
    ↓
12. Effect returns action observable
    ↓
13. EffectSources subscribes and dispatches returned action
    ↓
14. New action flows through normal dispatch pipeline
    ↓
15. Reducer processes loadUserSuccess
    ↓
16. State updated
    ↓
17. Selectors emit new values
    ↓
18. Components update
```

**Non-Dispatching Effects:**

```typescript
// Effect that only performs side effects (no action dispatch)
@Injectable()
export class LoggingEffects {
  logActions$ = createEffect(() =>
    this.actions$.pipe(
      tap(action => console.log('Action dispatched:', action))
    ),
    { dispatch: false }  // Don't dispatch result
  );
  
  // Navigate on success
  navigateOnSuccess$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUserSuccess),
      tap(() => this.router.navigate(['/dashboard']))
    ),
    { dispatch: false }
  );
  
  constructor(
    private actions$: Actions,
    private router: Router
  ) {}
}
```

**Effect Error Handling:**

```typescript
@Injectable()
export class RobustEffects {
  // ❌ BAD: Error crashes the effect stream
  badEffect$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUser),
      switchMap(action =>
        this.userService.getUser(action.userId).pipe(
          map(user => loadUserSuccess({ user }))
          // No catchError - error propagates up and kills stream!
        )
      )
    )
  );
  
  // ✅ GOOD: Error handled in inner observable
  goodEffect$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUser),
      switchMap(action =>
        this.userService.getUser(action.userId).pipe(
          map(user => loadUserSuccess({ user })),
          catchError(error => {
            // Error handled here - stream continues
            return of(loadUserFailure({ error: error.message }));
          })
        )
      )
    )
  );
  
  // ✅ BETTER: Centralized error handling
  betterEffect$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUser),
      switchMap(action =>
        this.userService.getUser(action.userId).pipe(
          map(user => loadUserSuccess({ user })),
          catchError(error => of(loadUserFailure({ error: error.message })))
        )
      ),
      // Outer catchError for unexpected errors
      catchError(error => {
        console.error('Unexpected effect error:', error);
        return EMPTY;  // Complete the stream
      })
    )
  );
  
  constructor(private userService: UserService) {}
}
```

---

## Change Detection Integration

### **How Angular Knows Which Components to Update**

```typescript
// The magic: Store is an Observable
class Store<T> extends Observable<T> {
  // When you select...
  select<K>(selector: Selector<T, K>): Observable<K> {
    return this.state$.pipe(
      map(selector),
      distinctUntilChanged()  // Only emit on actual change
    );
  }
}

// Component subscribes via async pipe
@Component({
  template: `<div>{{ user$ | async | json }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent {
  user$ = this.store.select(selectUser);
  
  constructor(private store: Store) {}
}
```

**What async pipe does:**

```typescript
// Simplified AsyncPipe implementation
@Pipe({ name: 'async', pure: false })
export class AsyncPipe implements OnDestroy {
  private subscription: Subscription | null = null;
  private latestValue: any = null;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  transform(obj: Observable<any>): any {
    if (!this.subscription) {
      // First time - subscribe
      this.subscription = obj.subscribe(value => {
        this.latestValue = value;
        
        // ⚡ CRITICAL: Mark component for check
        this.cdr.markForCheck();
      });
    }
    
    return this.latestValue;
  }
  
  ngOnDestroy(): void {
    // Clean up subscription
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
  }
}
```

**Complete flow with OnPush:**

```
1. State changes in store (e.g., user loaded)
   ↓
2. state$ observable emits new state
   ↓
3. Selector receives new state
   ↓
4. Selector checks if its slice changed (reference equality)
   ↓
5. If changed, selector emits new value
   ↓
6. Component's async pipe subscription receives value
   ↓
7. AsyncPipe calls cdr.markForCheck()
   ↓
8. Component marked "dirty" in change detection tree
   ↓
9. On next change detection cycle:
   ↓
10. Angular checks marked component (OnPush bypassed)
    ↓
11. Component re-renders with new value
    ↓
12. Child components checked if inputs changed
```

**Why OnPush + Observables is Powerful:**

```typescript
@Component({
  template: `
    <h1>{{ title }}</h1>
    <p>Count: {{ count$ | async }}</p>
    <p>User: {{ user$ | async | json }}</p>
    <button (click)="increment()">Increment</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SmartComponent {
  title = 'Dashboard';  // Regular property
  count$ = this.store.select(selectCount);  // Observable
  user$ = this.store.select(selectUser);  // Observable
  
  constructor(private store: Store) {}
  
  increment(): void {
    this.store.dispatch(increment());
  }
}

// Behavior:
// 1. Count changes in store
//    → count$ emits
//    → async pipe calls markForCheck()
//    → Component checks
//    → Only count updated in template
//
// 2. User changes in store
//    → user$ emits
//    → async pipe calls markForCheck()
//    → Component checks
//    → Only user updated in template
//
// 3. Random event elsewhere in app
//    → Component NOT checked (OnPush)
//    → No wasted rendering
```

**Component Tree Optimization:**

```typescript
// Parent component
@Component({
  template: `
    <app-header [user]="user$ | async"></app-header>
    <app-sidebar [items]="items$ | async"></app-sidebar>
    <app-content [data]="data$ | async"></app-content>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class LayoutComponent {
  user$ = this.store.select(selectUser);
  items$ = this.store.select(selectSidebarItems);
  data$ = this.store.select(selectContentData);
}

// When user changes:
// 1. user$ emits
// 2. Parent's async pipe marks parent for check
// 3. Parent checks
// 4. app-header input changed (new reference)
// 5. app-header checks (input changed)
// 6. app-sidebar input NOT changed (same reference)
// 7. app-sidebar NOT checked (OnPush optimization!)
// 8. app-content input NOT changed
// 9. app-content NOT checked

// Result: Only parent + header checked
// Sidebar and content skipped = major performance win
```

---

## When NgRx is the Wrong Choice

### **Concrete Examples Where NgRx Adds Unnecessary Complexity**

**Example 1: Simple CRUD App with No Shared State**

```typescript
// Scenario: Basic todo app, no real-time updates, no shared state

// ❌ OVERKILL WITH NGRX:
// - actions/todo.actions.ts (50 lines)
// - reducers/todo.reducer.ts (100 lines)
// - effects/todo.effects.ts (80 lines)
// - selectors/todo.selectors.ts (40 lines)
// - models/todo.model.ts (20 lines)
// Total: ~290 lines of boilerplate

// Actions
export const loadTodos = createAction('[Todo] Load Todos');
export const loadTodosSuccess = createAction(
  '[Todo] Load Todos Success',
  props<{ todos: Todo[] }>()
);
// ... 10 more actions

// Reducer
export const todoReducer = createReducer(
  initialState,
  on(loadTodos, state => ({ ...state, loading: true })),
  on(loadTodosSuccess, (state, { todos }) => ({ ...state, todos, loading: false })),
  // ... 10 more handlers
);

// Effects
loadTodos$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadTodos),
    switchMap(() => this.todoService.getTodos().pipe(
      map(todos => loadTodosSuccess({ todos })),
      catchError(error => of(loadTodosFailure({ error })))
    ))
  )
);

// Component
export class TodoComponent implements OnInit {
  todos$ = this.store.select(selectAllTodos);
  loading$ = this.store.select(selectTodosLoading);
  
  ngOnInit(): void {
    this.store.dispatch(loadTodos());
  }
  
  addTodo(title: string): void {
    this.store.dispatch(addTodo({ title }));
  }
}

// ✅ BETTER: Simple service
@Injectable({ providedIn: 'root' })
export class TodoService {
  private todos$ = new BehaviorSubject<Todo[]>([]);
  
  getTodos(): Observable<Todo[]> {
    return this.http.get<Todo[]>('/api/todos').pipe(
      tap(todos => this.todos$.next(todos)),
      shareReplay(1)
    );
  }
  
  addTodo(title: string): Observable<Todo> {
    return this.http.post<Todo>('/api/todos', { title }).pipe(
      tap(todo => {
        const current = this.todos$.value;
        this.todos$.next([...current, todo]);
      })
    );
  }
  
  watchTodos(): Observable<Todo[]> {
    return this.todos$.asObservable();
  }
}

// Component (much simpler!)
export class TodoComponent implements OnInit {
  todos$ = this.todoService.watchTodos();
  
  ngOnInit(): void {
    this.todoService.getTodos().subscribe();
  }
  
  addTodo(title: string): void {
    this.todoService.addTodo(title).subscribe();
  }
}

// Total: ~50 lines vs 290 lines
// Clarity: Much easier to understand
// Maintainability: No boilerplate
```

**Example 2: Component-Local State**

```typescript
// Scenario: Accordion with expand/collapse state

// ❌ OVERKILL: NgRx for local UI state
export const toggleAccordion = createAction(
  '[Accordion] Toggle',
  props<{ id: string }>()
);

// Storing in global store is WRONG
// This state doesn't need to be shared

// ✅ BETTER: Component state
@Component({
  selector: 'app-accordion',
  template: `
    <div *ngFor="let item of items" class="accordion-item">
      <button (click)="toggle(item.id)">
        {{ item.title }}
      </button>
      <div *ngIf="expandedIds.has(item.id)">
        {{ item.content }}
      </div>
    </div>
  `
})
export class AccordionComponent {
  @Input() items: AccordionItem[] = [];
  expandedIds = new Set<string>();
  
  toggle(id: string): void {
    if (this.expandedIds.has(id)) {
      this.expandedIds.delete(id);
    } else {
      this.expandedIds.add(id);
    }
  }
}

// Simple, local, no store needed
```

**Example 3: Small Team, Limited Time**

```typescript
// Scenario: Startup with 2-3 developers, tight deadline

// NgRx learning curve:
// - 1-2 weeks to understand concepts
// - 2-3 weeks to internalize patterns
// - Ongoing: maintaining boilerplate

// Better: Start simple, add complexity when needed
// - Use services + BehaviorSubject
// - If state gets complex, consider NgRx later
// - Don't over-engineer early
```

**When NgRx IS the Right Choice:**

1. **Multiple components need same data** - Shared state across routes
2. **Complex async flows** - Multiple API calls with interdependencies
3. **Time-travel debugging needed** - Complex business logic
4. **Large team** - Consistent patterns reduce confusion
5. **Real-time updates** - WebSocket state synchronization
6. **Optimistic updates** - Need rollback capability
7. **Caching & offline** - Persistent state management

---

## NgRx vs Akita vs NGXS

### **Detailed Comparison**

**NgRx:**

```typescript
// Pros:
// - Most popular (largest community)
// - Strict Redux patterns (predictable)
// - Excellent DevTools
// - Strong typing
// - Well-documented

// Cons:
// - Most boilerplate (actions, reducers, effects, selectors)
// - Steeper learning curve
// - Verbose (lots of files)
// - Can be overkill for simple apps

// Example:
// actions/user.actions.ts
export const loadUser = createAction('[User] Load', props<{ id: number }>());
export const loadUserSuccess = createAction('[User] Load Success', props<{ user: User }>());
export const loadUserFailure = createAction('[User] Load Failure', props<{ error: string }>());

// reducers/user.reducer.ts
export const userReducer = createReducer(
  initialState,
  on(loadUser, state => ({ ...state, loading: true })),
  on(loadUserSuccess, (state, { user }) => ({ ...state, user, loading: false })),
  on(loadUserFailure, (state, { error }) => ({ ...state, error, loading: false }))
);

// effects/user.effects.ts
loadUser$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUser),
    switchMap(({ id }) =>
      this.userService.getUser(id).pipe(
        map(user => loadUserSuccess({ user })),
        catchError(error => of(loadUserFailure({ error: error.message })))
      )
    )
  )
);

// selectors/user.selectors.ts
export const selectUser = createSelector(
  selectUserState,
  state => state.user
);
```

**Akita:**

```typescript
// Pros:
// - Less boilerplate (no actions/reducers)
// - Object-oriented (stores are classes)
// - Built-in entity management
// - Simpler for small/medium apps
// - Active queries (like selectors)

// Cons:
// - Smaller community
// - Less strict patterns (can be good or bad)
// - Fewer resources/tutorials
// - Not pure Redux (no actions)

// Example:
// stores/user.store.ts
export interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })
@StoreConfig({ name: 'user' })
export class UserStore extends Store<UserState> {
  constructor() {
    super({ user: null, loading: false, error: null });
  }
  
  setLoading(loading: boolean): void {
    this.update({ loading });
  }
  
  setUser(user: User): void {
    this.update({ user, loading: false, error: null });
  }
  
  setError(error: string): void {
    this.update({ error, loading: false });
  }
}

// queries/user.query.ts
@Injectable({ providedIn: 'root' })
export class UserQuery extends Query<UserState> {
  user$ = this.select(state => state.user);
  loading$ = this.select(state => state.loading);
  
  constructor(protected store: UserStore) {
    super(store);
  }
}

// services/user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private userStore: UserStore,
    private http: HttpClient
  ) {}
  
  loadUser(id: number): Observable<User> {
    this.userStore.setLoading(true);
    
    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap(user => this.userStore.setUser(user)),
      catchError(error => {
        this.userStore.setError(error.message);
        return throwError(error);
      })
    );
  }
}

// Usage
export class UserComponent {
  user$ = this.userQuery.user$;
  loading$ = this.userQuery.loading$;
  
  constructor(
    private userService: UserService,
    private userQuery: UserQuery
  ) {}
  
  ngOnInit(): void {
    this.userService.loadUser(123).subscribe();
  }
}
```

**NGXS:**

```typescript
// Pros:
// - Less boilerplate than NgRx
// - Class-based stores (like Akita)
// - Actions + state in same file
// - Simpler mental model
// - Good TypeScript support

// Cons:
// - Smaller community than NgRx
// - Uses decorators heavily (can be verbose)
// - Fewer resources
// - State is mutable (uses Immer internally)

// Example:
// states/user.state.ts
export class LoadUser {
  static readonly type = '[User] Load';
  constructor(public id: number) {}
}

export class LoadUserSuccess {
  static readonly type = '[User] Load Success';
  constructor(public user: User) {}
}

export class LoadUserFailure {
  static readonly type = '[User] Load Failure';
  constructor(public error: string) {}
}

export interface UserStateModel {
  user: User | null;
  loading: boolean;
  error: string | null;
}

@State<UserStateModel>({
  name: 'user',
  defaults: {
    user: null,
    loading: false,
    error: null
  }
})
@Injectable()
export class UserState {
  constructor(private userService: UserService) {}
  
  @Action(LoadUser)
  loadUser(ctx: StateContext<UserStateModel>, action: LoadUser) {
    ctx.patchState({ loading: true });
    
    return this.userService.getUser(action.id).pipe(
      tap(user => ctx.dispatch(new LoadUserSuccess(user))),
      catchError(error => ctx.dispatch(new LoadUserFailure(error.message)))
    );
  }
  
  @Action(LoadUserSuccess)
  loadUserSuccess(ctx: StateContext<UserStateModel>, action: LoadUserSuccess) {
    ctx.patchState({
      user: action.user,
      loading: false,
      error: null
    });
  }
  
  @Action(LoadUserFailure)
  loadUserFailure(ctx: StateContext<UserStateModel>, action: LoadUserFailure) {
    ctx.patchState({
      error: action.error,
      loading: false
    });
  }
  
  @Selector()
  static user(state: UserStateModel): User | null {
    return state.user;
  }
  
  @Selector()
  static loading(state: UserStateModel): boolean {
    return state.loading;
  }
}

// Usage
export class UserComponent {
  @Select(UserState.user) user$!: Observable<User | null>;
  @Select(UserState.loading) loading$!: Observable<boolean>;
  
  constructor(private store: Store) {}
  
  ngOnInit(): void {
    this.store.dispatch(new LoadUser(123));
  }
}
```

**Comparison Table:**

| Feature | NgRx | Akita | NGXS |
|---------|------|-------|------|
| **Boilerplate** | High | Low | Medium |
| **Learning Curve** | Steep | Moderate | Moderate |
| **Community** | Largest | Small | Small |
| **Redux Pattern** | Strict | Loose | Moderate |
| **Actions** | Explicit | None | Explicit |
| **Reducers** | Pure functions | Methods | @Action decorators |
| **Effects** | Separate | In service | In state class |
| **Selectors** | Memoized | Active queries | @Selector decorators |
| **DevTools** | Excellent | Good | Good |
| **Type Safety** | Excellent | Excellent | Excellent |
| **File Count** | High | Medium | Low |
| **Testability** | Excellent | Good | Good |
| **Best For** | Large apps, teams | Small/medium apps | Medium apps |

---

## Decision Framework

### **Should You Use a State Library?**

**Decision Tree:**

```
Is your app component state or global state?
├─ Component state (accordion, tabs, forms)
│  └─ NO STATE LIBRARY ✅
│     Use component state or services
│
└─ Global state
    │
    ├─ How many routes share this data?
    │  ├─ 1-2 routes
    │  │  └─ SIMPLE SERVICE ✅
    │  │     BehaviorSubject + shareReplay
    │  │
    │  └─ 3+ routes
    │     └─ Continue...
    │
    ├─ How complex are the data flows?
    │  ├─ Simple CRUD
    │  │  └─ SIMPLE SERVICE or AKITA ✅
    │  │
    │  └─ Complex (websockets, optimistic updates, rollback)
    │     └─ NGRX ✅
    │
    ├─ How many developers?
    │  ├─ 1-3 developers
    │  │  └─ AKITA or SIMPLE SERVICE ✅
    │  │     Less overhead
    │  │
    │  └─ 4+ developers
    │     └─ NGRX ✅
    │        Consistent patterns
    │
    └─ Do you need time-travel debugging?
       ├─ Yes
       │  └─ NGRX ✅
       │
       └─ No
          └─ AKITA or SIMPLE SERVICE ✅
```

**Mid-Sized Project Example:**

```typescript
// Scenario: E-commerce app
// - 10-15 routes
// - 3-4 developers
// - Product catalog, cart, checkout, orders
// - Real-time inventory updates

// Decision: NgRx

// Reasoning:
// 1. Multiple routes share cart/user data
// 2. Real-time updates (WebSocket) = complex async
// 3. Optimistic updates for cart (need rollback)
// 4. Team needs consistent patterns
// 5. Complex state dependencies (cart → checkout → orders)

// Architecture:
// - User state: NgRx (shared across app)
// - Product state: NgRx (real-time updates)
// - Cart state: NgRx (optimistic updates)
// - Checkout state: NgRx (multi-step flow)
// - UI state (modals, tooltips): Component state (no store)
```

**Alternative: Start Simple, Add Complexity**

```typescript
// Phase 1: MVP (2-4 weeks)
// - Simple services with BehaviorSubject
// - Get to market fast
// - Validate product-market fit

@Injectable({ providedIn: 'root' })
export class CartService {
  private items$ = new BehaviorSubject<CartItem[]>([]);
  
  getItems(): Observable<CartItem[]> {
    return this.items$.asObservable();
  }
  
  addItem(item: CartItem): void {
    const current = this.items$.value;
    this.items$.next([...current, item]);
  }
}

// Phase 2: Growth (3-6 months)
// - App complexity increasing
// - Team growing
// - Need better patterns

// Migrate incrementally:
// 1. Add NgRx for cart (most complex)
// 2. Keep simple services for user
// 3. Monitor complexity

// Phase 3: Scale (6+ months)
// - Full NgRx migration
// - Consistent patterns
// - Better debugging
```

---

## Managing Boilerplate

### **Strategies for Large Codebases**

**Strategy 1: Code Generation**

```bash
# Use @ngrx/schematics
npm install @ngrx/schematics --save-dev

# Set as default collection
ng config cli.defaultCollection @ngrx/schematics

# Generate store
ng generate store State --root --module app.module.ts

# Generate feature state
ng generate feature users/User -m users/users.module.ts

# Generates:
# - actions/user.actions.ts
# - reducers/user.reducer.ts
# - effects/user.effects.ts
# - selectors/user.selectors.ts
# All with boilerplate filled in!
```

**Strategy 2: Facades**

```typescript
// Facade pattern hides complexity
@Injectable({ providedIn: 'root' })
export class UserFacade {
  // Selectors
  user$ = this.store.select(selectUser);
  users$ = this.store.select(selectAllUsers);
  loading$ = this.store.select(selectUsersLoading);
  error$ = this.store.select(selectUsersError);
  
  // Computed
  currentUserName$ = this.user$.pipe(
    map(user => user ? `${user.firstName} ${user.lastName}` : '')
  );
  
  constructor(private store: Store) {}
  
  // Actions (simple API)
  loadUser(id: number): void {
    this.store.dispatch(loadUser({ id }));
  }
  
  loadUsers(): void {
    this.store.dispatch(loadUsers());
  }
  
  updateUser(id: number, changes: Partial<User>): void {
    this.store.dispatch(updateUser({ id, changes }));
  }
  
  deleteUser(id: number): void {
    this.store.dispatch(deleteUser({ id }));
  }
}

// Component (much cleaner!)
@Component({
  template: `
    <div *ngIf="facade.loading$ | async">Loading...</div>
    <div *ngIf="facade.error$ | async as error">{{ error }}</div>
    
    <div *ngFor="let user of facade.users$ | async">
      {{ user.name }}
      <button (click)="facade.deleteUser(user.id)">Delete</button>
    </div>
  `
})
export class UsersComponent implements OnInit {
  constructor(public facade: UserFacade) {}
  
  ngOnInit(): void {
    this.facade.loadUsers();
  }
}

// Benefits:
// - Components don't import 10+ selectors/actions
// - Easier testing (mock facade)
// - Cleaner component code
// - Encapsulation
```

**Strategy 3: Entity Adapter**

```typescript
// Instead of manually managing collections...

// ❌ Manual collection management (100+ lines)
export interface UsersState {
  entities: { [id: number]: User };
  ids: number[];
  selectedId: number | null;
  loading: boolean;
}

export const userReducer = createReducer(
  initialState,
  on(loadUsersSuccess, (state, { users }) => {
    const entities = {};
    const ids = [];
    users.forEach(user => {
      entities[user.id] = user;
      ids.push(user.id);
    });
    return { ...state, entities, ids };
  }),
  on(addUserSuccess, (state, { user }) => ({
    ...state,
    entities: { ...state.entities, [user.id]: user },
    ids: [...state.ids, user.id]
  })),
  // ... many more handlers
);

// ✅ Use Entity Adapter (30 lines)
import { createEntityAdapter, EntityState } from '@ngrx/entity';

export interface UsersState extends EntityState<User> {
  selectedId: number | null;
  loading: boolean;
}

export const adapter = createEntityAdapter<User>();

const initialState: UsersState = adapter.getInitialState({
  selectedId: null,
  loading: false
});

export const userReducer = createReducer(
  initialState,
  on(loadUsersSuccess, (state, { users }) =>
    adapter.setAll(users, { ...state, loading: false })
  ),
  on(addUserSuccess, (state, { user }) =>
    adapter.addOne(user, state)
  ),
  on(updateUserSuccess, (state, { user }) =>
    adapter.updateOne({ id: user.id, changes: user }, state)
  ),
  on(deleteUserSuccess, (state, { id }) =>
    adapter.removeOne(id, state)
  )
);

// Built-in selectors
export const {
  selectIds,
  selectEntities,
  selectAll,
  selectTotal
} = adapter.getSelectors();
```

**Strategy 4: Action Groups**

```typescript
// Instead of individual actions...

// ❌ Verbose
export const loadUsers = createAction('[Users] Load');
export const loadUsersSuccess = createAction('[Users] Load Success', props<{ users: User[] }>());
export const loadUsersFailure = createAction('[Users] Load Failure', props<{ error: string }>());

export const loadUser = createAction('[Users] Load One', props<{ id: number }>());
export const loadUserSuccess = createAction('[Users] Load One Success', props<{ user: User }>());
export const loadUserFailure = createAction('[Users] Load One Failure', props<{ error: string }>());

// ✅ Concise (NgRx 13+)
import { createActionGroup, emptyProps, props } from '@ngrx/store';

export const UsersActions = createActionGroup({
  source: 'Users',
  events: {
    'Load': emptyProps(),
    'Load Success': props<{ users: User[] }>(),
    'Load Failure': props<{ error: string }>(),
    
    'Load One': props<{ id: number }>(),
    'Load One Success': props<{ user: User }>(),
    'Load One Failure': props<{ error: string }>(),
  }
});

// Usage: UsersActions.load()
// Usage: UsersActions.loadSuccess({ users })
```

---

## Race Conditions in Effects

### **Problem: Duplicate API Calls**

```typescript
// Scenario: User types in search box quickly
// "a" → "ab" → "abc"
// Three HTTP requests fired

// ❌ BAD: mergeMap allows all requests
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(searchProducts),
    mergeMap(action =>
      this.productService.search(action.query).pipe(
        map(results => searchProductsSuccess({ results })),
        catchError(error => of(searchProductsFailure({ error })))
      )
    )
  )
);

// What happens:
// t=0ms:   searchProducts({ query: 'a' })    → HTTP request for 'a' starts
// t=50ms:  searchProducts({ query: 'ab' })   → HTTP request for 'ab' starts
// t=100ms: searchProducts({ query: 'abc' })  → HTTP request for 'abc' starts
// t=200ms: Response for 'a' arrives          → UI shows results for 'a'
// t=250ms: Response for 'abc' arrives        → UI shows results for 'abc'
// t=300ms: Response for 'ab' arrives         → UI shows results for 'ab' (WRONG!)
// 
// User typed 'abc' but sees results for 'ab'!
```

### **Solution: switchMap**

```typescript
// ✅ GOOD: switchMap cancels previous request
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(searchProducts),
    switchMap(action =>
      this.productService.search(action.query).pipe(
        map(results => searchProductsSuccess({ results })),
        catchError(error => of(searchProductsFailure({ error })))
      )
    )
  )
);

// What happens:
// t=0ms:   searchProducts({ query: 'a' })    → HTTP request for 'a' starts
// t=50ms:  searchProducts({ query: 'ab' })   → Cancel 'a', start 'ab'
// t=100ms: searchProducts({ query: 'abc' })  → Cancel 'ab', start 'abc'
// t=300ms: Response for 'abc' arrives        → UI shows results for 'abc' ✅
//
// Only final request completes!
```

**Why switchMap?**

```typescript
// switchMap behavior:
// - Subscribes to inner observable
// - If new source emission, UNSUBSCRIBES from previous inner observable
// - Subscribes to new inner observable
// - Result: Only latest observable active

// Internal logic (simplified):
function switchMap<T, R>(
  project: (value: T) => Observable<R>
): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    return new Observable(subscriber => {
      let innerSubscription: Subscription | null = null;
      
      const outerSubscription = source.subscribe({
        next: value => {
          // Cancel previous inner subscription
          if (innerSubscription) {
            innerSubscription.unsubscribe();
          }
          
          // Create new inner subscription
          const inner$ = project(value);
          innerSubscription = inner$.subscribe({
            next: result => subscriber.next(result),
            error: err => subscriber.error(err)
          });
        },
        error: err => subscriber.error(err),
        complete: () => subscriber.complete()
      });
      
      return () => {
        outerSubscription.unsubscribe();
        innerSubscription?.unsubscribe();
      };
    });
  };
}
```

**Advanced: Debounce + SwitchMap**

```typescript
// ✅ BEST: Debounce before switching
searchProducts$ = createEffect(() =>
  this.actions$.pipe(
    ofType(searchProducts),
    debounceTime(300),  // Wait 300ms after typing stops
    distinctUntilChanged((a, b) => a.query === b.query),  // Skip duplicates
    switchMap(action =>
      this.productService.search(action.query).pipe(
        map(results => searchProductsSuccess({ results })),
        catchError(error => of(searchProductsFailure({ error })))
      )
    )
  )
);

// What happens:
// t=0ms:   searchProducts({ query: 'a' })    → Wait...
// t=50ms:  searchProducts({ query: 'ab' })   → Reset timer, wait...
// t=100ms: searchProducts({ query: 'abc' })  → Reset timer, wait...
// t=400ms: 300ms passed, start HTTP request  → Only ONE request!
// t=600ms: Response arrives                  → UI updates ✅
//
// Reduced from 3 requests to 1!
```

**Other Race Condition Scenarios:**

```typescript
// Scenario 1: Save button clicked multiple times
// Solution: exhaustMap (ignore new clicks while saving)
saveUser$ = createEffect(() =>
  this.actions$.pipe(
    ofType(saveUser),
    exhaustMap(action =>
      this.userService.save(action.user).pipe(
        map(user => saveUserSuccess({ user })),
        catchError(error => of(saveUserFailure({ error })))
      )
    )
  )
);

// Scenario 2: Load multiple resources in parallel
// Solution: mergeMap (allow concurrent requests)
loadUserDetails$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUserDetails),
    mergeMap(action =>
      forkJoin({
        profile: this.userService.getProfile(action.userId),
        orders: this.orderService.getOrders(action.userId),
        preferences: this.prefService.getPreferences(action.userId)
      }).pipe(
        map(data => loadUserDetailsSuccess(data)),
        catchError(error => of(loadUserDetailsFailure({ error })))
      )
    )
  )
);

// Scenario 3: Sequential operations (must complete in order)
// Solution: concatMap (queue operations)
processQueue$ = createEffect(() =>
  this.actions$.pipe(
    ofType(processItem),
    concatMap(action =>
      this.processingService.process(action.item).pipe(
        map(result => processItemSuccess({ result })),
        catchError(error => of(processItemFailure({ error })))
      )
    )
  )
);
```

**Summary of Flattening Operators:**

| Operator | Behavior | Use Case |
|----------|----------|----------|
| **switchMap** | Cancels previous | Search, autocomplete, latest-wins scenarios |
| **mergeMap** | All concurrent | Independent parallel requests |
| **concatMap** | Sequential queue | Order-dependent operations |
| **exhaustMap** | Ignore new while active | Prevent duplicate submissions |

---

## Summary

**NgRx Architecture:**

1. **Action dispatched** → ActionsSubject emits
2. **Reducer executes** → scan() accumulates new state
3. **State emitted** → State$ observable broadcasts
4. **Selectors memoize** → Only recalculate on input change
5. **Async pipe subscribes** → Calls markForCheck() on emit
6. **OnPush bypassed** → Component updates
7. **Effects process** → Parallel RxJS streams handle side effects

**Key Principles:**
- Immutability enables reference equality checks
- Memoization prevents expensive recalculations
- Effects isolate side effects
- Selectors decouple state shape from components
- OnPush + Observables = optimal performance

**When to Use:**
- Large apps with shared state
- Complex async flows
- Multiple developers
- Time-travel debugging needed

**When NOT to Use:**
- Simple CRUD apps
- Component-local state
- Small teams with tight deadlines
- No state sharing

**Managing Complexity:**
- Use facades to hide boilerplate
- Entity adapters for collections
- Code generation with schematics
- Action groups for conciseness

**Race Conditions:**
- Use switchMap for cancellable operations
- Add debounce for user input
- exhaustMap for preventing duplicates
- concatMap for sequential operations

NgRx isn't "actions go to reducers" - it's a sophisticated reactive architecture that integrates deeply with Angular's change detection, requiring careful understanding of observables, immutability, and memoization to use effectively.

