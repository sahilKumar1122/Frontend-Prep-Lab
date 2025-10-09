# Visual Diagrams Library

> **All diagrams use Mermaid syntax for GitHub native rendering**

## 📚 Available Diagrams

### Angular Architecture
1. [Angular Architecture Overview](#1-angular-architecture-overview)
2. [Change Detection Flow](#2-change-detection-flow)
3. [DI Hierarchy](#3-dependency-injection-hierarchy)
4. [Component Lifecycle](#4-component-lifecycle)
5. [OnPush vs Default](#5-onpush-vs-default-strategy)
6. [NgRx Data Flow](#6-ngrx-data-flow)
7. [RxJS Operator Decision Tree](#7-rxjs-operator-decision-tree)
8. [Memory Leak Patterns](#8-memory-leak-patterns)
9. [Routing Flow](#9-routing-flow)
10. [Testing Strategy](#10-testing-strategy)

---

## 1. Angular Architecture Overview

```mermaid
graph TB
    subgraph "Angular Application"
        Root[Root Module<br/>AppModule]
        
        subgraph "Core Layer"
            Services[Services<br/>Business Logic]
            Guards[Route Guards]
            Interceptors[HTTP Interceptors]
        end
        
        subgraph "Feature Modules"
            Feature1[Feature Module 1]
            Feature2[Feature Module 2]
            Feature3[Feature Module 3]
        end
        
        subgraph "Shared"
            Components[Reusable Components]
            Directives[Custom Directives]
            Pipes[Custom Pipes]
        end
        
        subgraph "Component Tree"
            RootComp[Root Component]
            Parent1[Parent Component]
            Child1[Child Component 1]
            Child2[Child Component 2]
        end
    end
    
    Root --> Core Layer
    Root --> Feature Modules
    Root --> Shared
    Root --> RootComp
    
    RootComp --> Parent1
    Parent1 --> Child1
    Parent1 --> Child2
    
    Feature1 -.-> Services
    Feature2 -.-> Services
    Feature3 -.-> Services
    
    style Root fill:#f9f,stroke:#333,stroke-width:4px
    style Services fill:#bbf,stroke:#333,stroke-width:2px
    style RootComp fill:#bfb,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/fundamentals.md`

---

## 2. Change Detection Flow

```mermaid
graph TD
    Start[User Action<br/>Click, Input, etc.] --> Zone[Zone.js Intercepts Event]
    Zone --> Async{Async Operation?}
    
    Async -->|Yes| Schedule[Schedule Change Detection]
    Async -->|No| Trigger[Trigger CD Immediately]
    
    Schedule --> Trigger
    Trigger --> RootCheck[Start from Root Component]
    
    RootCheck --> Strategy{Component Strategy?}
    
    Strategy -->|Default| CheckAll[Check ALL Components]
    Strategy -->|OnPush| CheckOnPush{Input Changed?<br/>Event Fired?<br/>Async Emitted?}
    
    CheckOnPush -->|Yes| CheckComp[Check Component]
    CheckOnPush -->|No| SkipComp[Skip Component]
    
    CheckAll --> UpdateDOM[Update DOM if Needed]
    CheckComp --> UpdateDOM
    SkipComp --> Done[Done]
    UpdateDOM --> Done
    
    style Start fill:#f96,stroke:#333,stroke-width:2px
    style Zone fill:#ff9,stroke:#333,stroke-width:2px
    style CheckAll fill:#f99,stroke:#333,stroke-width:2px
    style CheckOnPush fill:#9f9,stroke:#333,stroke-width:2px
    style Done fill:#99f,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/change-detection.md`

---

## 3. Dependency Injection Hierarchy

```mermaid
graph TD
    Platform[Platform Injector<br/>providedIn: 'platform'<br/>Shared across apps]
    Root[Root Injector<br/>providedIn: 'root'<br/>App singleton]
    Module[Module Injector<br/>Lazy-loaded modules]
    Component[Component Injector<br/>Component providers]
    Directive[Directive Injector<br/>Directive providers]
    
    Platform --> Root
    Root --> Module
    Module --> Component
    Component --> Directive
    
    Search{Dependency<br/>Resolution}
    Search -->|1. Check| Directive
    Search -->|2. Not found| Component
    Search -->|3. Not found| Module
    Search -->|4. Not found| Root
    Search -->|5. Not found| Platform
    Search -->|6. Not found| Error[Throw Error:<br/>NullInjectorError]
    
    style Platform fill:#f9f,stroke:#333,stroke-width:2px
    style Root fill:#ff9,stroke:#333,stroke-width:2px
    style Component fill:#9ff,stroke:#333,stroke-width:2px
    style Error fill:#f66,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/dependency-injection.md`

---

## 4. Component Lifecycle

```mermaid
sequenceDiagram
    participant Angular
    participant Component
    participant View
    participant Content
    
    Angular->>Component: constructor()
    Note over Component: DI only<br/>@Input undefined
    
    Angular->>Component: ngOnChanges(changes)
    Note over Component: @Input available<br/>Fires on every input change
    
    Angular->>Component: ngOnInit()
    Note over Component: Initialize component<br/>API calls, subscriptions
    
    loop Every Change Detection
        Angular->>Component: ngDoCheck()
        Note over Component: Custom change detection<br/>Use sparingly!
        
        Angular->>Content: ngAfterContentInit()
        Note over Content: @ContentChild available<br/>Once only
        
        Angular->>Content: ngAfterContentChecked()
        Note over Content: After content checked<br/>Every CD cycle
        
        Angular->>View: ngAfterViewInit()
        Note over View: @ViewChild available<br/>DOM ready, Once only
        
        Angular->>View: ngAfterViewChecked()
        Note over View: After view checked<br/>Every CD cycle
    end
    
    Angular->>Component: ngOnDestroy()
    Note over Component: CLEANUP!<br/>Unsubscribe, clear timers
```

**Usage in:** `angular/lifecycle-hooks.md`

---

## 5. OnPush vs Default Strategy

```mermaid
graph LR
    subgraph "Default Strategy"
        Event1[Any Event] --> CD1[Check Entire Tree]
        CD1 --> All1[Check ALL Components]
        All1 --> Update1[Update DOM]
    end
    
    subgraph "OnPush Strategy"
        Event2[Event] --> Check2{Conditions}
        Check2 -->|Input Ref Changed| CD2[Check Component]
        Check2 -->|Event in Component| CD2
        Check2 -->|Async Pipe Emitted| CD2
        Check2 -->|markForCheck Called| CD2
        Check2 -->|None| Skip2[Skip Component]
        CD2 --> Update2[Update DOM]
    end
    
    Comparison[Performance Impact]
    Default Strategy --> Comparison
    OnPush Strategy --> Comparison
    
    Comparison --> Result1[Default: 100% CD calls]
    Comparison --> Result2[OnPush: ~10% CD calls]
    Comparison --> Result3[Result: 90% faster!]
    
    style CD1 fill:#f99,stroke:#333,stroke-width:2px
    style CD2 fill:#9f9,stroke:#333,stroke-width:2px
    style Result3 fill:#9f9,stroke:#333,stroke-width:3px
```

**Usage in:** `angular/change-detection.md`, `angular/performance-optimization.md`

---

## 6. NgRx Data Flow

```mermaid
graph TD
    Component[Component] -->|1. Dispatch| Action[Action<br/>loadUsers]
    
    Action -->|2. Flows to| Effects[Effects]
    Action -->|3. Flows to| Reducer[Reducer]
    
    Effects -->|4. Side Effect| API[HTTP API Call]
    API -->|5. Success| Action2[Action<br/>loadUsersSuccess]
    API -->|5. Error| Action3[Action<br/>loadUsersFailure]
    
    Action2 --> Reducer
    Action3 --> Reducer
    
    Reducer -->|6. Pure Function| NewState[New Immutable State]
    NewState -->|7. Store Update| Store[(Store)]
    
    Store -->|8. Notify| Selector[Selector<br/>Memoized]
    Selector -->|9. If Changed| Component
    Component -->|10. Async Pipe| Template[Template Updates]
    
    style Action fill:#f9f,stroke:#333,stroke-width:2px
    style Effects fill:#ff9,stroke:#333,stroke-width:2px
    style Reducer fill:#9ff,stroke:#333,stroke-width:2px
    style Store fill:#9f9,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/ngrx-state-management.md`

---

## 7. RxJS Operator Decision Tree

```mermaid
graph TD
    Start{What do you need?}
    
    Start -->|Flatten inner observables| Flatten{Concurrency?}
    Start -->|Transform values| Transform[map, pluck, mapTo]
    Start -->|Filter values| Filter[filter, take, skip]
    Start -->|Combine streams| Combine{How?}
    
    Flatten -->|Cancel previous| SwitchMap[switchMap<br/>Use for: Search, Navigation]
    Flatten -->|Run all parallel| MergeMap[mergeMap<br/>Use for: Multiple HTTP calls]
    Flatten -->|Queue sequential| ConcatMap[concatMap<br/>Use for: Order matters]
    Flatten -->|Ignore while busy| ExhaustMap[exhaustMap<br/>Use for: Login, Save buttons]
    
    Combine -->|Latest from all| CombineLatest[combineLatest<br/>Use for: Form validation]
    Combine -->|Wait for all| ForkJoin[forkJoin<br/>Use for: Parallel requests]
    Combine -->|Pair by index| Zip[zip<br/>Use for: Sync streams]
    
    style SwitchMap fill:#9f9,stroke:#333,stroke-width:2px
    style MergeMap fill:#ff9,stroke:#333,stroke-width:2px
    style ConcatMap fill:#9ff,stroke:#333,stroke-width:2px
    style ExhaustMap fill:#f9f,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/rxjs-operators.md`

---

## 8. Memory Leak Patterns

```mermaid
graph TD
    Component[Component Created]
    
    Component --> Sub1[Create Subscription]
    Component --> Timer1[Create Timer]
    Component --> Event1[Add Event Listener]
    Component --> WS1[Open WebSocket]
    Component --> Lib1[Initialize Library]
    
    Sub1 --> Use[Component Used]
    Timer1 --> Use
    Event1 --> Use
    WS1 --> Use
    Lib1 --> Use
    
    Use --> Destroy{Component Destroyed}
    
    Destroy -->|❌ No Cleanup| Leak1[Subscription Still Active]
    Destroy -->|❌ No Cleanup| Leak2[Timer Still Running]
    Destroy -->|❌ No Cleanup| Leak3[Listener Still Attached]
    Destroy -->|❌ No Cleanup| Leak4[WebSocket Still Open]
    Destroy -->|❌ No Cleanup| Leak5[Library Not Destroyed]
    
    Leak1 --> Memory[Memory Leak!<br/>Component Can't be GC'd]
    Leak2 --> Memory
    Leak3 --> Memory
    Leak4 --> Memory
    Leak5 --> Memory
    
    Destroy -->|✅ Cleanup| Clean1[Unsubscribe]
    Destroy -->|✅ Cleanup| Clean2[Clear Timer]
    Destroy -->|✅ Cleanup| Clean3[Remove Listener]
    Destroy -->|✅ Cleanup| Clean4[Close WebSocket]
    Destroy -->|✅ Cleanup| Clean5[Destroy Library]
    
    Clean1 --> GC[Garbage Collected ✅]
    Clean2 --> GC
    Clean3 --> GC
    Clean4 --> GC
    Clean5 --> GC
    
    style Memory fill:#f66,stroke:#333,stroke-width:3px
    style GC fill:#9f9,stroke:#333,stroke-width:3px
```

**Usage in:** `angular/debugging-memory-leaks.md`

---

## 9. Routing Flow

```mermaid
graph TD
    Nav[User Navigation<br/>Click Link / URL Change]
    
    Nav --> Router[Angular Router]
    Router --> Match{Match Route?}
    
    Match -->|No| NotFound[404 Page]
    Match -->|Yes| Guards{Route Guards}
    
    Guards --> CanActivate{CanActivate?}
    CanActivate -->|False| Block[Block Navigation]
    CanActivate -->|True| CanLoad{CanLoad?}
    
    CanLoad -->|False| Block
    CanLoad -->|True| Resolve[Resolvers<br/>Pre-fetch Data]
    
    Resolve --> Load{Lazy Load?}
    Load -->|Yes| LoadModule[Download Module]
    Load -->|No| Activate[Activate Component]
    
    LoadModule --> Activate
    Activate --> Render[Render Component]
    
    Render --> Leave{User Leaves?}
    Leave -->|Yes| CanDeactivate{CanDeactivate?}
    CanDeactivate -->|True| Destroy[Destroy Component]
    CanDeactivate -->|False| Stay[Stay on Page]
    
    style Block fill:#f66,stroke:#333,stroke-width:2px
    style Render fill:#9f9,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/routing.md`

---

## 10. Testing Strategy

```mermaid
graph TD
    Start[Testing Strategy]
    
    Start --> Unit[Unit Tests<br/>60%]
    Start --> Integration[Integration Tests<br/>30%]
    Start --> E2E[E2E Tests<br/>10%]
    
    Unit --> TestBed[TestBed Setup]
    TestBed --> Components[Test Components]
    TestBed --> Services[Test Services]
    TestBed --> Pipes[Test Pipes]
    
    Components --> Async{Async Operations?}
    Async -->|Yes| FakeAsync[fakeAsync + tick]
    Async -->|No| Sync[Synchronous Tests]
    
    FakeAsync --> Mock[Mock Dependencies]
    Sync --> Mock
    
    Mock --> HTTP[HttpTestingController]
    Mock --> Spies[Jasmine Spies]
    
    Integration --> Feature[Feature Tests<br/>Multiple Components]
    Feature --> UserFlow[User Workflows]
    
    E2E --> Critical[Critical Paths Only]
    Critical --> Login[Login Flow]
    Critical --> Purchase[Purchase Flow]
    Critical --> Submit[Data Submission]
    
    style Unit fill:#9f9,stroke:#333,stroke-width:2px
    style Integration fill:#ff9,stroke:#333,stroke-width:2px
    style E2E fill:#9ff,stroke:#333,stroke-width:2px
```

**Usage in:** `angular/testing-strategy.md`

---

## 🎨 Diagram Style Guide

### Color Coding
- 🟢 **Green (#9f9):** Success states, optimal paths
- 🟡 **Yellow (#ff9):** Warning, medium priority
- 🔵 **Blue (#9ff):** Info, alternative paths
- 🔴 **Red (#f66):** Errors, problems, anti-patterns
- 🟣 **Purple (#f9f):** Entry points, important nodes

### When to Use Each Diagram Type

**Flowcharts (`graph TD/LR`):**
- Decision trees
- Process flows
- Conditional logic

**Sequence Diagrams (`sequenceDiagram`):**
- Component interactions
- Lifecycle sequences
- API call flows

**Subgraphs:**
- Grouped concepts
- Layered architecture
- Module organization

---

## 📝 Adding Diagrams to Documentation

### Template for Markdown Files

```markdown
## Visual Explanation

### [Topic] Flow

```mermaid
[diagram code here]
```

**Key Points:**
- Point 1
- Point 2
- Point 3

**Related Topics:**
- [Link to related topic]
```

---

## 🔧 Tools & Resources

### Creating Diagrams
- **[Mermaid Live Editor](https://mermaid.live/)** - Test and preview diagrams
- **[Mermaid Documentation](https://mermaid-js.github.io/mermaid/)** - Full syntax reference
- **[GitHub Mermaid Support](https://github.blog/2022-02-14-include-diagrams-markdown-files-mermaid/)** - Native GitHub rendering

### Exporting
- PNG/SVG export from Mermaid Live
- GitHub renders natively (no export needed)
- For presentations: Use Mermaid CLI

---

**Last Updated:** January 9, 2025  
**Total Diagrams:** 10  
**Format:** Mermaid (GitHub Native)

