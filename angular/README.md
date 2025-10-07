# Angular Interview Questions

Senior-level Angular interview questions with production-grade answers, following industry best practices.

## 📚 Topics Covered

### 1. [Fundamentals](./fundamentals.md)
- What is Angular and its architecture
- Components and lifecycle hooks
- Modules (Core, Shared, Feature)
- Dependency Injection
- Data Binding (One-way, Two-way)
- Change Detection strategies

**Key Questions:**
- Explain Angular's component lifecycle
- What's the difference between OnPush and Default change detection?
- How does Angular's Dependency Injection work?

### 2. [Reactive Forms](./reactive-forms.md)
- Reactive vs Template-driven forms
- Form Validation (Built-in & Custom)
- Dynamic Forms
- Form Arrays
- Cross-field validation
- Async validators

**Key Questions:**
- How do you implement complex form validation?
- What's the difference between setValue and patchValue?
- How do you create reusable custom validators?

### 3. [RxJS & State Management](./rxjs-state.md)
- Observable fundamentals
- Hot vs Cold observables
- Common operators (map, filter, switchMap, etc.)
- Subject types (BehaviorSubject, ReplaySubject)
- Memory management and unsubscribing
- State management patterns

**Key Questions:**
- What's the difference between switchMap, mergeMap, concatMap, and exhaustMap?
- How do you prevent memory leaks with Observables?
- Explain hot vs cold observables

### 4. [Routing & Navigation](./routing.md)
- Router configuration
- Route parameters and query params
- Route Guards (CanActivate, CanDeactivate, CanLoad)
- Lazy loading modules
- Route Resolvers
- Named outlets

**Key Questions:**
- How do you implement authentication guards?
- What's the difference between CanActivate and CanLoad?
- How does lazy loading improve performance?

## 🎯 Study Approach

### For Each Topic:
1. **Read the question** - Understand what's being asked
2. **Study the answer** - Focus on the explanation and reasoning
3. **Review the code** - Understand the implementation
4. **Note key points** - Remember the "Pro Tips"
5. **Practice** - Implement the patterns in your own projects

### Difficulty Levels:
- 🟢 **Junior** - Basic concepts and usage
- 🟡 **Mid** - Practical patterns and best practices
- 🔴 **Senior** - Advanced patterns, performance, and architecture

## 💡 Interview Tips

### What Interviewers Look For:

**1. Deep Understanding**
- Don't just memorize - understand *why*
- Explain trade-offs and alternatives
- Discuss real-world implications

**2. Production Experience**
- Reference actual project challenges
- Discuss performance considerations
- Mention testing strategies

**3. Best Practices**
- Code organization and structure
- Error handling
- Memory management
- Accessibility

**4. Modern Patterns**
- Standalone components (Angular 14+)
- Signals (Angular 16+)
- Reactive programming with RxJS
- Performance optimization

### Common Mistakes to Avoid:

❌ **Not unsubscribing from Observables**
```typescript
// Bad
this.userService.getUsers().subscribe(users => ...);

// Good
this.userService.getUsers()
  .pipe(takeUntil(this.destroy$))
  .subscribe(users => ...);
```

❌ **Using method calls in templates**
```typescript
// Bad
<div>{{ calculateTotal() }}</div>

// Good
<div>{{ total }}</div>
```

❌ **Not using OnPush change detection**
```typescript
// Good for performance
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
```

❌ **Importing SharedModule in CoreModule**
```typescript
// CoreModule should only be imported once in AppModule
// SharedModule can be imported in feature modules
```

## 🚀 Quick Reference

### Essential Concepts

**Component Communication:**
- `@Input()` - Parent to Child
- `@Output()` - Child to Parent
- Service with Subject - Sibling to Sibling
- `@ViewChild()` - Parent accesses Child
- `@ContentChild()` - Access projected content

**Lifecycle Order:**
1. `ngOnChanges` - Input changes
2. `ngOnInit` - Component initialization
3. `ngDoCheck` - Custom change detection
4. `ngAfterContentInit` - Content projection
5. `ngAfterContentChecked` - After content check
6. `ngAfterViewInit` - View initialization
7. `ngAfterViewChecked` - After view check
8. `ngOnDestroy` - Cleanup

**RxJS Operators:**
- `switchMap` - Cancel previous, use latest
- `mergeMap` - Run all in parallel
- `concatMap` - Queue sequentially
- `exhaustMap` - Ignore while active
- `debounceTime` - Wait for pause
- `distinctUntilChanged` - Only if different
- `takeUntil` - Auto-unsubscribe

**Change Detection:**
- Default - Check entire tree
- OnPush - Check only when:
  - Input reference changes
  - Event from component
  - Observable emits (with async pipe)
  - Manual `markForCheck()`

## 📖 Additional Resources

### Official Documentation
- [Angular.io](https://angular.io/)
- [RxJS](https://rxjs.dev/)
- [Angular CLI](https://cli.angular.io/)

### Advanced Topics
- NgRx for state management
- Angular Universal for SSR
- Testing with Jasmine/Karma
- Performance optimization
- Micro-frontends with Module Federation

## 🎓 Practice Projects

Build these to master Angular:

1. **Todo App** - Basic CRUD, forms, routing
2. **E-commerce** - Complex state, guards, lazy loading
3. **Dashboard** - Charts, real-time data, WebSockets
4. **Admin Panel** - Role-based access, complex forms
5. **Social Media Clone** - Real-time updates, infinite scroll

## 📝 Sample Interview Flow

**Question: "Build a user search with autocomplete"**

**Expected Approach:**
1. Explain reactive forms setup
2. Discuss debouncing and API calls
3. Show switchMap usage
4. Mention error handling
5. Discuss performance (OnPush, trackBy)
6. Consider accessibility
7. Add loading states

```typescript
@Component({
  selector: 'app-user-search',
  template: `
    <input [formControl]="search" placeholder="Search users..." />
    <div *ngIf="loading$ | async">Loading...</div>
    <ul>
      <li *ngFor="let user of users$ | async; trackBy: trackByUserId">
        {{ user.name }}
      </li>
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserSearchComponent implements OnInit {
  search = new FormControl('');
  users$!: Observable<User[]>;
  loading$ = new BehaviorSubject(false);

  ngOnInit(): void {
    this.users$ = this.search.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => this.loading$.next(true)),
      switchMap(term => this.userService.search(term)),
      tap(() => this.loading$.next(false)),
      catchError(() => {
        this.loading$.next(false);
        return of([]);
      })
    );
  }

  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}
```

---

**Good luck with your Angular interviews! 🚀**

Remember: The goal isn't to memorize answers, but to understand the underlying concepts and be able to apply them to real-world scenarios.
