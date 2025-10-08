# Angular Content Projection - Deep Dive

## Table of Contents
- [Internal Compilation Pipeline](#internal-compilation-pipeline)
- [Structural Differences](#structural-differences)
- [Multi-Slot Projection](#multi-slot-projection)
- [Projected Content Bindings](#projected-content-bindings)
- [Change Detection Context](#change-detection-context)
- [Real-World Reusable Components](#real-world-reusable-components)
- [Pitfalls and Solutions](#pitfalls-and-solutions)
- [Hybrid: Projection + Dynamic Components](#hybrid-projection--dynamic-components)

---

## Internal Compilation Pipeline

### Question: Explain how Angular internally handles content projection - the full pipeline from parsing to rendering. Cover compilation, structural differences between projection primitives, multi-slot projection, bindings in projected content, change detection context, real-world design patterns, and hybrid approaches with dynamic components.

**Answer:**

Content projection isn't just "displaying child HTML" - it's a sophisticated mechanism involving template parsing, view hierarchy management, and careful change detection coordination. Let's trace the complete pipeline.

---

## **How Angular Compiles Content Projection**

### **Step 1: Template Parsing and AST Generation**

When Angular encounters a component with `<ng-content>`, it creates an Abstract Syntax Tree (AST) that distinguishes between the component's template and projected content.

```typescript
// Source template
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="header">
        <ng-content select="[header]"></ng-content>
      </div>
      <div class="body">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {}

// Usage
@Component({
  template: `
    <app-card>
      <h1 header>Card Title</h1>
      <p>Card content goes here</p>
    </app-card>
  `
})
export class ParentComponent {}
```

**Angular's Internal AST Representation:**

```typescript
// Simplified AST structure
{
  type: 'Component',
  selector: 'app-card',
  template: {
    nodes: [
      {
        type: 'Element',
        name: 'div',
        attrs: { class: 'card' },
        children: [
          {
            type: 'Element',
            name: 'div',
            attrs: { class: 'header' },
            children: [
              {
                type: 'NgContent',
                selector: '[header]',  // CSS selector for projection
                index: 0  // Projection slot index
              }
            ]
          },
          {
            type: 'Element',
            name: 'div',
            attrs: { class: 'body' },
            children: [
              {
                type: 'NgContent',
                selector: null,  // Default slot (no selector)
                index: 1
              }
            ]
          }
        ]
      }
    ]
  }
}

// Parent's AST (simplified)
{
  type: 'Component',
  template: {
    nodes: [
      {
        type: 'Element',
        name: 'app-card',
        projectedContent: [
          // Slot 0: content with [header] attribute
          {
            type: 'Element',
            name: 'h1',
            attrs: { header: true },
            children: [{ type: 'Text', value: 'Card Title' }]
          },
          // Slot 1: default content
          {
            type: 'Element',
            name: 'p',
            children: [{ type: 'Text', value: 'Card content goes here' }]
          }
        ]
      }
    ]
  }
}
```

### **Step 2: Content Projection Compilation**

Angular compiles content projection into JavaScript code that manages view references.

```typescript
// Simplified compiled output of CardComponent
class CardComponent_Compiled {
  static ngComponentDef = {
    template: function CardComponent_Template(rf, ctx) {
      if (rf & 1) {  // Creation mode
        elementStart(0, 'div', ['class', 'card']);
          elementStart(1, 'div', ['class', 'header']);
            // ↓ This is the key: projection instruction
            projection(2, 0);  // Project content into slot 0
          elementEnd();
          elementStart(3, 'div', ['class', 'body']);
            projection(4, 1);  // Project content into slot 1
          elementEnd();
        elementEnd();
      }
    },
    
    // Define projection slots
    ngContentSelectors: ['[header]', '*'],  // Slot 0 and default slot
  };
}

// Simplified compiled output of ParentComponent
class ParentComponent_Compiled {
  static ngComponentDef = {
    template: function ParentComponent_Template(rf, ctx) {
      if (rf & 1) {
        // Create component
        elementStart(0, 'app-card');
          // ↓ Content to be projected
          elementStart(1, 'h1', ['header', '']);
            text(2, 'Card Title');
          elementEnd();
          elementStart(3, 'p');
            text(4, 'Card content goes here');
          elementEnd();
        elementEnd();
      }
    }
  };
}
```

**Key Compilation Step: Content Distribution**

```typescript
// Angular's internal projection logic (simplified)
class ProjectionEngine {
  distributeProjectableNodes(
    componentView: View,
    projectableNodes: Node[][]
  ): void {
    // Step 1: Group projected content by selector
    const selectors = componentView.def.ngContentSelectors;
    const distributed: Node[][] = [];
    
    selectors.forEach((selector, index) => {
      distributed[index] = [];
    });
    
    // Step 2: Match projected nodes to slots
    projectableNodes.forEach(node => {
      const matchedSlot = this.findMatchingSlot(node, selectors);
      distributed[matchedSlot].push(node);
    });
    
    // Step 3: Store references for projection instructions
    componentView.projectedNodes = distributed;
  }
  
  findMatchingSlot(node: Node, selectors: string[]): number {
    // Match node against each selector
    for (let i = 0; i < selectors.length; i++) {
      if (selectors[i] === '*') continue;  // Default slot, check last
      
      if (this.matchesSelector(node, selectors[i])) {
        return i;
      }
    }
    
    // No match, use default slot (last one)
    return selectors.length - 1;
  }
  
  matchesSelector(node: Node, selector: string): boolean {
    // Use CSS selector matching
    if (node instanceof Element) {
      return node.matches(selector);
    }
    return false;
  }
}
```

### **Step 3: Runtime View Creation**

When Angular creates a component instance, it manages projected content separately from the component's own template.

```typescript
// Angular's internal view hierarchy
class ViewHierarchy {
  createComponentView(
    componentType: Type<any>,
    projectedContent: Node[][]
  ): ComponentView {
    // Create component's own view
    const componentView = {
      def: componentType.ngComponentDef,
      nodes: [],
      parent: null,
      projectedNodes: projectedContent  // Store projected content
    };
    
    // Important: Projected content stays in parent's view context
    // It's NOT moved to component's view
    
    return componentView;
  }
  
  projectContent(view: View, slotIndex: number, nodeIndex: number): void {
    // Get projected nodes for this slot
    const nodesToProject = view.projectedNodes[slotIndex];
    
    // Create projection anchor in component's view
    const projectionAnchor = {
      type: 'projection',
      slotIndex: slotIndex,
      projectedNodes: nodesToProject,
      nodeIndex: nodeIndex
    };
    
    view.nodes[nodeIndex] = projectionAnchor;
    
    // Key: Projected nodes maintain reference to parent view
    // This is crucial for change detection!
  }
}
```

**Visual Representation:**

```
ParentComponent View
├─ app-card element
│  └─ (Projects content, but content stays in parent view)
│
└─ Projected Content (lives here!)
    ├─ <h1 header>Card Title</h1>
    └─ <p>Card content goes here</p>

CardComponent View
├─ div.card
│  ├─ div.header
│  │  └─ [projection slot 0] ← Points to parent's <h1>
│  └─ div.body
│     └─ [projection slot 1] ← Points to parent's <p>
```

---

## Structural Differences

### **<ng-content> - The Projection Slot**

**What It Is:**
- A **marker** in the template where content should be projected
- Compiled into a `projection()` instruction
- Does NOT create a DOM node

```typescript
// Template
<div class="wrapper">
  <ng-content></ng-content>
</div>

// Compiled to (simplified)
elementStart(0, 'div', ['class', 'wrapper']);
  projection(1);  // No DOM node created, just projects content
elementEnd();

// Rendered DOM
<div class="wrapper">
  <!-- Projected content appears here -->
  <h1>Projected heading</h1>
  <p>Projected paragraph</p>
</div>
```

**Key Characteristics:**
- Cannot have attributes (except `select`)
- Cannot have bindings
- Cannot be conditionally rendered with *ngIf
- Always present in the template (compiled away)

```typescript
// ❌ INVALID: Cannot use directives on ng-content
<ng-content *ngIf="showContent"></ng-content>  // Compilation error!

// ❌ INVALID: Cannot have attributes
<ng-content class="content"></ng-content>  // Ignored!

// ✅ VALID: Only select attribute
<ng-content select="[header]"></ng-content>
```

### **<ng-container> - The Logical Grouper**

**What It Is:**
- A **logical grouping element** that doesn't create a DOM node
- Useful for applying structural directives without extra markup
- Can have any directives/attributes

```typescript
// Template
<div class="wrapper">
  <ng-container *ngIf="showContent">
    <h1>Title</h1>
    <p>Content</p>
  </ng-container>
</div>

// Rendered DOM (when showContent = true)
<div class="wrapper">
  <h1>Title</h1>
  <p>Content</p>
  <!-- No ng-container in DOM -->
</div>
```

**Key Characteristics:**
- Can have structural directives (*ngIf, *ngFor, etc.)
- Can have property/event bindings
- Does NOT render to DOM
- Useful for grouping without extra elements

```typescript
// ✅ VALID: Grouping with structural directive
<ng-container *ngFor="let item of items">
  <h2>{{ item.title }}</h2>
  <p>{{ item.description }}</p>
</ng-container>

// ✅ VALID: Multiple structural directives (using ng-container)
<ng-container *ngIf="showItems">
  <ng-container *ngFor="let item of items">
    <div>{{ item.name }}</div>
  </ng-container>
</ng-container>

// ❌ INVALID: Without ng-container, would need wrapper
<div *ngIf="showItems" *ngFor="let item of items">
  <!-- Can't have two structural directives on same element! -->
</div>
```

### **<ng-template> - The Template Definition**

**What It Is:**
- Defines a **template that is NOT rendered by default**
- Must be explicitly instantiated via structural directive or TemplateRef
- Creates a template context for variable binding

```typescript
// Template
<div class="wrapper">
  <ng-template #myTemplate let-name="name" let-age="age">
    <h1>Name: {{ name }}</h1>
    <p>Age: {{ age }}</p>
  </ng-template>
  
  <button (click)="showTemplate()">Show Template</button>
</div>

// Component
@Component({
  selector: 'app-example',
  template: `...`
})
export class ExampleComponent {
  @ViewChild('myTemplate') template!: TemplateRef<any>;
  
  constructor(private vcr: ViewContainerRef) {}
  
  showTemplate(): void {
    // Explicitly create view from template
    this.vcr.createEmbeddedView(this.template, {
      name: 'John',
      age: 30
    });
  }
}
```

**Key Characteristics:**
- NOT rendered by default
- Creates template context (variables with `let-`)
- Used by structural directives internally
- Can be passed to components as @Input

```typescript
// Structural directives desugar to ng-template

// ❌ Sugar syntax:
<div *ngIf="condition">Content</div>

// ✅ Desugared to:
<ng-template [ngIf]="condition">
  <div>Content</div>
</ng-template>

// ❌ Sugar syntax:
<div *ngFor="let item of items">{{ item }}</div>

// ✅ Desugared to:
<ng-template ngFor let-item [ngForOf]="items">
  <div>{{ item }}</div>
</ng-template>
```

### **Comparison Table**

| Feature | ng-content | ng-container | ng-template |
|---------|------------|--------------|-------------|
| **Creates DOM** | No | No | No (until instantiated) |
| **Purpose** | Project content | Group elements | Define template |
| **Structural Directives** | ❌ No | ✅ Yes | N/A (is the directive target) |
| **Property Bindings** | ❌ No (only `select`) | ✅ Yes | ✅ Yes |
| **Default Rendering** | Projects immediately | Renders immediately | Does NOT render |
| **Use Case** | Content projection | Grouping without wrapper | Template definition |

---

## Multi-Slot Projection

### **How It Works Internally**

```typescript
// Component with multiple slots
@Component({
  selector: 'app-layout',
  template: `
    <div class="layout">
      <header class="header">
        <ng-content select="[header]"></ng-content>
      </header>
      
      <aside class="sidebar">
        <ng-content select="[sidebar]"></ng-content>
      </aside>
      
      <main class="main">
        <ng-content></ng-content>  <!-- Default slot -->
      </main>
      
      <footer class="footer">
        <ng-content select="[footer]"></ng-content>
      </footer>
    </div>
  `
})
export class LayoutComponent {}

// Usage
@Component({
  template: `
    <app-layout>
      <div header>
        <h1>My App</h1>
        <nav>Navigation</nav>
      </div>
      
      <div sidebar>
        <ul>
          <li>Menu 1</li>
          <li>Menu 2</li>
        </ul>
      </div>
      
      <div>
        <h2>Main Content</h2>
        <p>This goes to default slot</p>
      </div>
      
      <div footer>
        <p>&copy; 2024 My App</p>
      </div>
    </app-layout>
  `
})
export class AppComponent {}
```

**Internal Compilation:**

```typescript
// LayoutComponent compiled (simplified)
class LayoutComponent_Compiled {
  static ngContentSelectors = [
    '[header]',   // Slot 0
    '[sidebar]',  // Slot 1
    '*',          // Slot 2 (default)
    '[footer]'    // Slot 3
  ];
  
  static template(rf, ctx) {
    if (rf & 1) {
      elementStart(0, 'div', ['class', 'layout']);
        elementStart(1, 'header', ['class', 'header']);
          projection(2, 0);  // Project slot 0 ([header])
        elementEnd();
        
        elementStart(3, 'aside', ['class', 'sidebar']);
          projection(4, 1);  // Project slot 1 ([sidebar])
        elementEnd();
        
        elementStart(5, 'main', ['class', 'main']);
          projection(6, 2);  // Project slot 2 (default)
        elementEnd();
        
        elementStart(7, 'footer', ['class', 'footer']);
          projection(8, 3);  // Project slot 3 ([footer])
        elementEnd();
      elementEnd();
    }
  }
}
```

**Content Distribution Algorithm:**

```typescript
class ContentDistributor {
  distribute(
    content: Node[],
    selectors: string[]
  ): Map<number, Node[]> {
    const distributed = new Map<number, Node[]>();
    
    // Initialize empty arrays for each slot
    selectors.forEach((_, index) => {
      distributed.set(index, []);
    });
    
    // Distribute each content node
    content.forEach(node => {
      const slotIndex = this.findSlot(node, selectors);
      distributed.get(slotIndex)!.push(node);
    });
    
    return distributed;
  }
  
  findSlot(node: Node, selectors: string[]): number {
    // Check each selector except default (last one)
    for (let i = 0; i < selectors.length - 1; i++) {
      if (selectors[i] === '*') continue;
      
      if (node instanceof Element && node.matches(selectors[i])) {
        return i;
      }
    }
    
    // No match, return default slot (last index)
    return selectors.length - 1;
  }
}
```

### **Complex Selectors**

```typescript
@Component({
  selector: 'app-advanced',
  template: `
    <!-- Attribute selector -->
    <ng-content select="[header]"></ng-content>
    
    <!-- Element selector -->
    <ng-content select="h1"></ng-content>
    
    <!-- Class selector -->
    <ng-content select=".special"></ng-content>
    
    <!-- Complex selector -->
    <ng-content select="div.card[data-type='primary']"></ng-content>
    
    <!-- Multiple selectors (CSS OR) -->
    <ng-content select="header, [header], .header"></ng-content>
    
    <!-- Default (everything else) -->
    <ng-content></ng-content>
  `
})
export class AdvancedComponent {}

// Usage
<app-advanced>
  <h1>Matches h1 selector</h1>
  <div header>Matches [header] selector</div>
  <p class="special">Matches .special selector</p>
  <div class="card" data-type="primary">Matches complex selector</div>
  <span>Goes to default slot</span>
</app-advanced>
```

---

## Projected Content Bindings

### **How Bindings Work in Projected Content**

**Key Concept:** Projected content maintains its bindings to the **parent component**, not the host component.

```typescript
// Child component (host)
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content></ng-content>
    </div>
  `
})
export class CardComponent {
  cardTitle = 'Card Title';  // Not accessible to projected content!
}

// Parent component
@Component({
  selector: 'app-parent',
  template: `
    <app-card>
      <!-- ✅ Can access parent's properties -->
      <h1>{{ parentTitle }}</h1>
      
      <!-- ❌ CANNOT access card's properties -->
      <h2>{{ cardTitle }}</h2>  <!-- undefined! -->
      
      <!-- ✅ Parent's methods work -->
      <button (click)="parentMethod()">Click</button>
    </app-card>
  `
})
export class ParentComponent {
  parentTitle = 'Parent Title';  // ✅ Accessible
  
  parentMethod(): void {
    console.log('Parent method called');
  }
}
```

### **Projected Content with Structural Directives**

```typescript
// Component
@Component({
  selector: 'app-list',
  template: `
    <div class="list">
      <ng-content></ng-content>
    </div>
  `
})
export class ListComponent {}

// Usage
@Component({
  template: `
    <app-list>
      <!-- Structural directive in projected content -->
      <div *ngFor="let item of items">
        {{ item.name }}
      </div>
      
      <!-- Multiple directives -->
      <ng-container *ngIf="showMore">
        <div *ngFor="let extra of extraItems">
          {{ extra.name }}
        </div>
      </ng-container>
    </app-list>
  `
})
export class ParentComponent {
  items = [
    { name: 'Item 1' },
    { name: 'Item 2' }
  ];
  
  showMore = true;
  extraItems = [
    { name: 'Extra 1' },
    { name: 'Extra 2' }
  ];
}
```

**What Happens Internally:**

```typescript
// The *ngFor in projected content is compiled in the PARENT's context

// Parent's compiled template (simplified)
function ParentTemplate(rf, ctx) {
  if (rf & 1) {
    elementStart(0, 'app-list');
      // ↓ Projected content compiled here (in parent)
      template(1, projectedForTemplate, ...);  // The *ngFor template
    elementEnd();
  }
  
  if (rf & 2) {
    // ↓ *ngFor evaluated with parent's ctx
    property('ngForOf', ctx.items);  // ctx is ParentComponent!
  }
}

// The projected *ngFor template
function projectedForTemplate(rf, ctx) {
  if (rf & 1) {
    elementStart(0, 'div');
      text(1);
    elementEnd();
  }
  
  if (rf & 2) {
    const item = ctx.$implicit;  // From parent's *ngFor context
    advance(1);
    textInterpolate(item.name);
  }
}
```

### **Passing Data from Host to Projected Content**

**Problem:** Projected content can't directly access host component properties.

**Solution 1: Template Variables**

```typescript
@Component({
  selector: 'app-data-host',
  template: `
    <div>
      <ng-content></ng-content>
    </div>
  `
})
export class DataHostComponent {
  hostData = { value: 42 };
}

// ❌ This doesn't work:
<app-data-host>
  <div>{{ hostData.value }}</div>  <!-- undefined! -->
</app-data-host>

// ✅ Use template variables:
<app-data-host #host>
  <div>{{ host.hostData.value }}</div>  <!-- Works! -->
</app-data-host>
```

**Solution 2: Custom Projection API**

```typescript
@Component({
  selector: 'app-list-container',
  template: `
    <div class="list">
      <ng-container *ngFor="let item of items">
        <ng-content [ngOutlet]="itemTemplate" [ngOutletContext]="{$implicit: item}"></ng-content>
      </ng-container>
    </div>
  `
})
export class ListContainerComponent {
  @Input() items: any[] = [];
  @ContentChild(TemplateRef) itemTemplate!: TemplateRef<any>;
}

// Usage
<app-list-container [items]="myItems">
  <ng-template let-item>
    <div class="item">{{ item.name }}</div>
  </ng-template>
</app-list-container>
```

---

## Change Detection Context

### **Projected Content and Change Detection**

**Critical Rule:** Projected content belongs to the **parent's change detection tree**, not the host component's tree.

```typescript
// Host component
@Component({
  selector: 'app-host',
  template: `
    <div class="host">
      <button (click)="updateHost()">Update Host</button>
      <p>Host: {{ hostValue }}</p>
      <ng-content></ng-content>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush  // OnPush!
})
export class HostComponent {
  hostValue = 0;
  
  updateHost(): void {
    this.hostValue++;
  }
}

// Parent component
@Component({
  template: `
    <div class="parent">
      <button (click)="updateParent()">Update Parent</button>
      <p>Parent: {{ parentValue }}</p>
      
      <app-host>
        <!-- This content is in PARENT's change detection context -->
        <p>From parent: {{ parentValue }}</p>
        <button (click)="updateFromProjected()">Update from Projected</button>
      </app-host>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.Default  // Default
})
export class ParentComponent {
  parentValue = 0;
  
  updateParent(): void {
    this.parentValue++;
    // ✅ Projected content updates immediately (parent's CD)
  }
  
  updateFromProjected(): void {
    this.parentValue++;
    // ✅ Also updates immediately
  }
}
```

**What Happens:**

```
Change Detection Tree:

ParentComponent (Default CD)
├─ Parent's template
│  ├─ Button
│  ├─ <p>Parent: {{ parentValue }}</p>
│  └─ Projected content
│     ├─ <p>From parent: {{ parentValue }}</p>  ← In parent's tree!
│     └─ Button
│
└─ HostComponent (OnPush CD)
    ├─ Button
    ├─ <p>Host: {{ hostValue }}</p>
    └─ [Projection slot] ← Just a reference, content is in parent

When parent's CD runs:
1. Parent's template checked ✅
2. Projected content checked ✅ (part of parent's tree)
3. Host component checked only if:
   - Input changed
   - Event fired in host
   - Manually marked for check

When host's CD runs:
1. Host's template checked ✅
2. Projected content NOT checked (belongs to parent)
```

### **OnPush Strategy with Projected Content**

```typescript
// Scenario: OnPush host, Default parent

@Component({
  selector: 'app-onpush-host',
  template: `
    <div>
      <p>Host counter: {{ hostCounter }}</p>
      <ng-content></ng-content>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushHostComponent {
  hostCounter = 0;
  
  increment(): void {
    this.hostCounter++;
    // Host's template WON'T update (OnPush without markForCheck)
  }
}

@Component({
  template: `
    <app-onpush-host>
      <!-- Projected content uses PARENT's CD strategy -->
      <p>Parent counter: {{ parentCounter }}</p>
      <button (click)="parentCounter++">Increment Parent</button>
      <button (click)="host.increment()">Increment Host</button>
    </app-onpush-host>
  `,
  changeDetection: ChangeDetectionStrategy.Default
})
export class ParentComponent {
  parentCounter = 0;
  
  // When parentCounter changes:
  // ✅ Projected content updates (parent's CD runs)
  // ❌ Host template doesn't update (OnPush, no input change)
}
```

### **Memory Implications**

```typescript
// Each component maintains references to projected content

class ComponentView {
  def: ComponentDef;
  nodes: ViewNode[];
  parent: ComponentView | null;
  projectedNodes: Node[][];  // References to projected content
  
  // Projected nodes are NOT copied!
  // They're references to nodes in parent's view
  // Memory-efficient but creates dependency on parent's lifecycle
}

// When component is destroyed:
// 1. Component's own template is destroyed
// 2. Projected content references are cleared
// 3. BUT projected content itself is NOT destroyed
//    (it's still in parent's view)

// Projected content is only destroyed when:
// - Parent component is destroyed
// - Structural directive removes the content (*ngIf, *ngFor)
```

---

## Real-World Reusable Components

### **Scenario 1: Reusable Modal Component**

```typescript
@Component({
  selector: 'app-modal',
  template: `
    <div class="modal-backdrop" *ngIf="isOpen" (click)="closeOnBackdrop && close()">
      <div class="modal-container" (click)="$event.stopPropagation()">
        <!-- Header slot -->
        <div class="modal-header" *ngIf="hasHeader">
          <ng-content select="[modal-header]"></ng-content>
          <button class="close-btn" (click)="close()" *ngIf="showCloseButton">
            ×
          </button>
        </div>
        
        <!-- Body slot (default) -->
        <div class="modal-body">
          <ng-content></ng-content>
        </div>
        
        <!-- Footer slot -->
        <div class="modal-footer" *ngIf="hasFooter">
          <ng-content select="[modal-footer]"></ng-content>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .modal-backdrop {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 1000;
    }
    
    .modal-container {
      background: white;
      border-radius: 8px;
      max-width: 90%;
      max-height: 90%;
      display: flex;
      flex-direction: column;
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    
    .modal-header {
      padding: 16px;
      border-bottom: 1px solid #ddd;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    
    .modal-body {
      padding: 16px;
      flex: 1;
      overflow-y: auto;
    }
    
    .modal-footer {
      padding: 16px;
      border-top: 1px solid #ddd;
      display: flex;
      justify-content: flex-end;
      gap: 8px;
    }
    
    .close-btn {
      background: none;
      border: none;
      font-size: 24px;
      cursor: pointer;
      padding: 0;
      width: 32px;
      height: 32px;
    }
  `]
})
export class ModalComponent implements AfterContentInit {
  @Input() isOpen = false;
  @Input() closeOnBackdrop = true;
  @Input() showCloseButton = true;
  @Output() closed = new EventEmitter<void>();
  
  @ContentChild(ModalHeaderDirective) header?: ModalHeaderDirective;
  @ContentChild(ModalFooterDirective) footer?: ModalFooterDirective;
  
  hasHeader = false;
  hasFooter = false;
  
  ngAfterContentInit(): void {
    // Detect if header/footer slots have content
    this.hasHeader = !!this.header;
    this.hasFooter = !!this.footer;
  }
  
  close(): void {
    this.isOpen = false;
    this.closed.emit();
  }
}

// Helper directives for better TypeScript support
@Directive({
  selector: '[modal-header]'
})
export class ModalHeaderDirective {}

@Directive({
  selector: '[modal-footer]'
})
export class ModalFooterDirective {}

// Usage
@Component({
  template: `
    <button (click)="modal.isOpen = true">Open Modal</button>
    
    <app-modal #modal [closeOnBackdrop]="true">
      <!-- Header -->
      <div modal-header>
        <h2>{{ modalTitle }}</h2>
        <span class="subtitle">{{ modalSubtitle }}</span>
      </div>
      
      <!-- Body (default slot) -->
      <div class="content">
        <p>{{ modalContent }}</p>
        
        <form [formGroup]="form">
          <input formControlName="name" placeholder="Name" />
          <input formControlName="email" placeholder="Email" />
        </form>
      </div>
      
      <!-- Footer -->
      <div modal-footer>
        <button (click)="modal.close()">Cancel</button>
        <button (click)="save(); modal.close()" [disabled]="!form.valid">
          Save
        </button>
      </div>
    </app-modal>
  `
})
export class ParentComponent {
  modalTitle = 'Edit User';
  modalSubtitle = 'Update user information';
  modalContent = 'Please fill in the form below.';
  
  form = new FormGroup({
    name: new FormControl(''),
    email: new FormControl('')
  });
  
  save(): void {
    console.log('Saving:', this.form.value);
  }
}
```

### **Scenario 2: Reusable Table Component**

```typescript
@Component({
  selector: 'app-table',
  template: `
    <div class="table-container">
      <!-- Custom header template -->
      <div class="table-header" *ngIf="hasCustomHeader">
        <ng-content select="[table-header]"></ng-content>
      </div>
      
      <!-- Default header -->
      <div class="table-header" *ngIf="!hasCustomHeader">
        <h3>{{ title }}</h3>
        <div class="actions">
          <input [(ngModel)]="searchTerm" placeholder="Search..." />
          <button (click)="refresh.emit()">Refresh</button>
        </div>
      </div>
      
      <!-- Table -->
      <table>
        <!-- Column headers -->
        <thead>
          <tr>
            <ng-content select="[table-columns]"></ng-content>
          </tr>
        </thead>
        
        <!-- Rows with template -->
        <tbody>
          <ng-container *ngIf="hasRowTemplate; else defaultRows">
            <ng-container *ngFor="let row of filteredData; trackBy: trackByFn">
              <ng-container
                *ngTemplateOutlet="rowTemplate; context: {$implicit: row, index: i}"
              ></ng-container>
            </ng-container>
          </ng-container>
          
          <ng-template #defaultRows>
            <tr *ngFor="let row of filteredData; let i = index">
              <ng-content select="[table-cells]"></ng-content>
            </tr>
          </ng-template>
        </tbody>
      </table>
      
      <!-- Custom footer -->
      <div class="table-footer" *ngIf="hasCustomFooter">
        <ng-content select="[table-footer]"></ng-content>
      </div>
      
      <!-- Default footer with pagination -->
      <div class="table-footer" *ngIf="!hasCustomFooter && showPagination">
        <span>Showing {{ startIndex + 1 }}-{{ endIndex }} of {{ totalItems }}</span>
        <button (click)="previousPage()" [disabled]="currentPage === 0">Previous</button>
        <span>Page {{ currentPage + 1 }} of {{ totalPages }}</span>
        <button (click)="nextPage()" [disabled]="currentPage >= totalPages - 1">Next</button>
      </div>
    </div>
  `,
  styles: [`
    .table-container { border: 1px solid #ddd; border-radius: 8px; }
    .table-header { padding: 16px; border-bottom: 1px solid #ddd; }
    .actions { display: flex; gap: 8px; margin-top: 8px; }
    table { width: 100%; border-collapse: collapse; }
    th, td { padding: 12px; text-align: left; border-bottom: 1px solid #eee; }
    .table-footer { padding: 16px; border-top: 1px solid #ddd; display: flex; justify-content: space-between; }
  `]
})
export class TableComponent implements AfterContentInit, OnChanges {
  @Input() title = '';
  @Input() data: any[] = [];
  @Input() pageSize = 10;
  @Input() showPagination = true;
  @Input() trackByFn: TrackByFunction<any> = (i) => i;
  
  @Output() refresh = new EventEmitter<void>();
  
  @ContentChild('rowTemplate') rowTemplate?: TemplateRef<any>;
  @ContentChild(TableHeaderDirective) customHeader?: TableHeaderDirective;
  @ContentChild(TableFooterDirective) customFooter?: TableFooterDirective;
  
  searchTerm = '';
  currentPage = 0;
  
  hasRowTemplate = false;
  hasCustomHeader = false;
  hasCustomFooter = false;
  
  filteredData: any[] = [];
  totalItems = 0;
  totalPages = 0;
  startIndex = 0;
  endIndex = 0;
  
  ngAfterContentInit(): void {
    this.hasRowTemplate = !!this.rowTemplate;
    this.hasCustomHeader = !!this.customHeader;
    this.hasCustomFooter = !!this.customFooter;
  }
  
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['data'] || changes['pageSize']) {
      this.updateFilteredData();
    }
  }
  
  private updateFilteredData(): void {
    // Filter by search term
    let filtered = this.data;
    if (this.searchTerm) {
      filtered = this.data.filter(item =>
        JSON.stringify(item).toLowerCase().includes(this.searchTerm.toLowerCase())
      );
    }
    
    this.totalItems = filtered.length;
    this.totalPages = Math.ceil(this.totalItems / this.pageSize);
    
    // Paginate
    this.startIndex = this.currentPage * this.pageSize;
    this.endIndex = Math.min(this.startIndex + this.pageSize, this.totalItems);
    this.filteredData = filtered.slice(this.startIndex, this.endIndex);
  }
  
  previousPage(): void {
    if (this.currentPage > 0) {
      this.currentPage--;
      this.updateFilteredData();
    }
  }
  
  nextPage(): void {
    if (this.currentPage < this.totalPages - 1) {
      this.currentPage++;
      this.updateFilteredData();
    }
  }
}

@Directive({ selector: '[table-header]' })
export class TableHeaderDirective {}

@Directive({ selector: '[table-footer]' })
export class TableFooterDirective {}

// Usage Example 1: Simple table
@Component({
  template: `
    <app-table [data]="users" title="Users">
      <!-- Columns -->
      <th table-columns>Name</th>
      <th table-columns>Email</th>
      <th table-columns>Role</th>
      
      <!-- Cells (repeated for each row) -->
      <td table-cells>{{ user.name }}</td>
      <td table-cells>{{ user.email }}</td>
      <td table-cells>{{ user.role }}</td>
    </app-table>
  `
})
export class SimpleTableComponent {
  users = [
    { name: 'John', email: 'john@example.com', role: 'Admin' },
    { name: 'Jane', email: 'jane@example.com', role: 'User' }
  ];
}

// Usage Example 2: Advanced table with custom templates
@Component({
  template: `
    <app-table [data]="products" [pageSize]="5">
      <!-- Custom header -->
      <div table-header>
        <h2>Product Catalog</h2>
        <button (click)="addProduct()">Add Product</button>
      </div>
      
      <!-- Custom row template -->
      <ng-template #rowTemplate let-product let-i="index">
        <tr [class.highlight]="product.featured">
          <td>{{ i + 1 }}</td>
          <td>
            <img [src]="product.image" alt="{{ product.name }}" width="50" />
          </td>
          <td>{{ product.name }}</td>
          <td>{{ product.price | currency }}</td>
          <td>
            <span class="badge" [class]="product.status">
              {{ product.status }}
            </span>
          </td>
          <td>
            <button (click)="edit(product)">Edit</button>
            <button (click)="delete(product)">Delete</button>
          </td>
        </tr>
      </ng-template>
      
      <!-- Custom footer -->
      <div table-footer>
        <span>Total: {{ products.length }} products</span>
        <button (click)="export()">Export CSV</button>
      </div>
    </app-table>
  `
})
export class AdvancedTableComponent {
  products = [
    { name: 'Product 1', price: 29.99, status: 'active', featured: true, image: '...' },
    { name: 'Product 2', price: 49.99, status: 'inactive', featured: false, image: '...' }
  ];
  
  addProduct(): void {}
  edit(product: any): void {}
  delete(product: any): void {}
  export(): void {}
}
```

---

## Pitfalls and Solutions

### **Pitfall 1: Projected Component Not Updating**

**Problem:**
```typescript
@Component({
  selector: 'app-host',
  template: `<ng-content></ng-content>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class HostComponent {}

@Component({
  template: `
    <app-host>
      <!-- This updates when parent changes -->
      <p>{{ parentValue }}</p>
    </app-host>
  `
})
export class ParentComponent {
  parentValue = 0;
  
  updateValue(): void {
    this.parentValue++;
    // ✅ Projected content updates (part of parent's CD)
  }
}
```

**No problem here! But if you add an input...**

```typescript
@Component({
  selector: 'app-host',
  template: `
    <div>Host: {{ hostValue }}</div>
    <ng-content></ng-content>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class HostComponent {
  @Input() hostValue = 0;
}

@Component({
  template: `
    <button (click)="update()">Update</button>
    <app-host [hostValue]="value">
      <!-- Problem: This doesn't update when value changes! -->
      <p>Parent value: {{ value }}</p>
    </app-host>
  `
})
export class ParentComponent {
  value = 0;
  
  update(): void {
    this.value++;
    // Host updates (input changed)
    // But projected content might not update if parent is OnPush!
  }
}
```

**Solution:**
```typescript
// Make sure parent's CD runs
@Component({
  template: `...`,
  changeDetection: ChangeDetectionStrategy.Default  // Or manually markForCheck
})
export class ParentComponent {
  constructor(private cdr: ChangeDetectorRef) {}
  
  update(): void {
    this.value++;
    this.cdr.markForCheck();  // Force parent CD
  }
}
```

### **Pitfall 2: Losing Template Context**

**Problem:**
```typescript
@Component({
  selector: 'app-iterator',
  template: `
    <div *ngFor="let item of items">
      <ng-content></ng-content>
    </div>
  `
})
export class IteratorComponent {
  @Input() items: any[] = [];
}

// Usage - THIS DOESN'T WORK!
<app-iterator [items]="myItems">
  <p>{{ item.name }}</p>  <!-- item is undefined! -->
</app-iterator>
```

**Solution: Use ng-template with context**
```typescript
@Component({
  selector: 'app-iterator',
  template: `
    <div *ngFor="let item of items">
      <ng-container *ngTemplateOutlet="template; context: {$implicit: item, index: i}">
      </ng-container>
    </div>
  `
})
export class IteratorComponent {
  @Input() items: any[] = [];
  @ContentChild(TemplateRef) template!: TemplateRef<any>;
}

// Usage - NOW IT WORKS!
<app-iterator [items]="myItems">
  <ng-template let-item let-i="index">
    <p>{{ i }}: {{ item.name }}</p>
  </ng-template>
</app-iterator>
```

### **Pitfall 3: Multiple Default Slots**

**Problem:**
```typescript
@Component({
  selector: 'app-multi',
  template: `
    <div class="section1">
      <ng-content></ng-content>  <!-- Default slot -->
    </div>
    <div class="section2">
      <ng-content></ng-content>  <!-- Same content appears twice! -->
    </div>
  `
})
export class MultiComponent {}

<app-multi>
  <p>This appears in BOTH sections!</p>
</app-multi>
```

**Solution: Use selectors**
```typescript
@Component({
  selector: 'app-multi',
  template: `
    <div class="section1">
      <ng-content select="[section1]"></ng-content>
    </div>
    <div class="section2">
      <ng-content select="[section2]"></ng-content>
    </div>
  `
})
export class MultiComponent {}

<app-multi>
  <p section1>In section 1</p>
  <p section2>In section 2</p>
</app-multi>
```

### **Pitfall 4: Projected Content Not Destroyed**

**Problem:**
```typescript
@Component({
  selector: 'app-conditional',
  template: `
    <div *ngIf="show">
      <ng-content></ng-content>
    </div>
  `
})
export class ConditionalComponent {
  @Input() show = true;
}

@Component({
  template: `
    <app-conditional [show]="showContent">
      <app-expensive-component></app-expensive-component>
    </app-conditional>
  `
})
export class ParentComponent {
  showContent = true;
}

// When showContent = false:
// ❌ app-expensive-component is hidden but NOT destroyed!
// It's still in parent's view, just not projected
```

**Solution: Move *ngIf to parent**
```typescript
@Component({
  template: `
    <app-conditional>
      <app-expensive-component *ngIf="showContent"></app-expensive-component>
    </app-conditional>
  `
})
export class ParentComponent {
  showContent = true;
  // ✅ Now component is destroyed when showContent = false
}
```

---

## Hybrid: Projection + Dynamic Components

### **Combining Both Approaches**

```typescript
@Component({
  selector: 'app-tab-container',
  template: `
    <div class="tabs">
      <!-- Tab headers -->
      <div class="tab-header">
        <button
          *ngFor="let tab of tabs; let i = index"
          (click)="selectTab(i)"
          [class.active]="i === activeTabIndex">
          {{ tab.title }}
        </button>
        
        <!-- Dynamic tab button -->
        <button
          *ngFor="let dynamicTab of dynamicTabs; let i = index"
          (click)="selectDynamicTab(i)"
          [class.active]="'dynamic-' + i === activeTabIndex">
          {{ dynamicTab.title }}
        </button>
      </div>
      
      <!-- Projected tab content -->
      <div class="tab-content">
        <ng-container *ngFor="let tab of tabs; let i = index">
          <div *ngIf="i === activeTabIndex">
            <ng-container *ngTemplateOutlet="tab.template"></ng-container>
          </div>
        </ng-container>
        
        <!-- Dynamic component host -->
        <ng-container #dynamicHost></ng-container>
      </div>
    </div>
  `
})
export class TabContainerComponent implements AfterContentInit, OnDestroy {
  @ContentChildren(TabDirective) tabs!: QueryList<TabDirective>;
  @ViewChild('dynamicHost', { read: ViewContainerRef }) dynamicHost!: ViewContainerRef;
  
  dynamicTabs: DynamicTab[] = [];
  activeTabIndex: number | string = 0;
  private componentRefs: ComponentRef<any>[] = [];
  
  ngAfterContentInit(): void {
    // Initialize first projected tab
    if (this.tabs.length > 0) {
      this.activeTabIndex = 0;
    }
  }
  
  selectTab(index: number): void {
    this.activeTabIndex = index;
    this.clearDynamicComponents();
  }
  
  selectDynamicTab(index: number): void {
    this.activeTabIndex = `dynamic-${index}`;
    this.clearDynamicComponents();
    this.loadDynamicComponent(this.dynamicTabs[index]);
  }
  
  addDynamicTab(tab: DynamicTab): void {
    this.dynamicTabs.push(tab);
  }
  
  private loadDynamicComponent(tab: DynamicTab): void {
    const componentRef = this.dynamicHost.createComponent(tab.component);
    
    // Pass inputs to dynamic component
    if (tab.inputs) {
      Object.keys(tab.inputs).forEach(key => {
        (componentRef.instance as any)[key] = tab.inputs![key];
      });
    }
    
    // Subscribe to outputs
    if (tab.outputs) {
      Object.keys(tab.outputs).forEach(key => {
        const emitter = (componentRef.instance as any)[key];
        if (emitter && emitter.subscribe) {
          emitter.subscribe(tab.outputs![key]);
        }
      });
    }
    
    this.componentRefs.push(componentRef);
    
    // Manually trigger change detection
    componentRef.changeDetectorRef.detectChanges();
  }
  
  private clearDynamicComponents(): void {
    this.componentRefs.forEach(ref => ref.destroy());
    this.componentRefs = [];
    this.dynamicHost.clear();
  }
  
  ngOnDestroy(): void {
    this.clearDynamicComponents();
  }
}

@Directive({
  selector: '[appTab]'
})
export class TabDirective {
  @Input('appTab') title = '';
  
  constructor(public template: TemplateRef<any>) {}
}

interface DynamicTab {
  title: string;
  component: Type<any>;
  inputs?: Record<string, any>;
  outputs?: Record<string, (event: any) => void>;
}

// Usage
@Component({
  template: `
    <app-tab-container #container>
      <!-- Projected tabs -->
      <ng-template appTab="Overview">
        <h2>Overview Content</h2>
        <p>{{ overviewData }}</p>
      </ng-template>
      
      <ng-template appTab="Settings">
        <app-settings-form [config]="config"></app-settings-form>
      </ng-template>
    </app-tab-container>
    
    <button (click)="addDynamicChart()">Add Chart Tab</button>
  `
})
export class AppComponent {
  @ViewChild('container') container!: TabContainerComponent;
  
  overviewData = 'This is overview data';
  config = { theme: 'dark' };
  
  addDynamicChart(): void {
    this.container.addDynamicTab({
      title: 'Chart',
      component: ChartComponent,
      inputs: {
        data: [1, 2, 3, 4, 5],
        type: 'line'
      },
      outputs: {
        dataPointClicked: (point: any) => {
          console.log('Point clicked:', point);
        }
      }
    });
  }
}
```

### **Managing Lifecycle and Change Detection**

```typescript
@Component({
  selector: 'app-hybrid-container',
  template: `
    <div class="container">
      <!-- Projected content -->
      <div class="projected-section">
        <ng-content></ng-content>
      </div>
      
      <!-- Dynamic content -->
      <div class="dynamic-section">
        <ng-container #dynamicHost></ng-container>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class HybridContainerComponent implements OnDestroy {
  @ViewChild('dynamicHost', { read: ViewContainerRef }) dynamicHost!: ViewContainerRef;
  
  private componentRefs = new Map<string, ComponentRef<any>>();
  private subscriptions = new Map<string, Subscription>();
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  loadComponent(id: string, component: Type<any>, inputs?: any): void {
    // Clear existing component with same id
    this.unloadComponent(id);
    
    // Create component
    const componentRef = this.dynamicHost.createComponent(component);
    
    // Set inputs
    if (inputs) {
      Object.keys(inputs).forEach(key => {
        (componentRef.instance as any)[key] = inputs[key];
      });
    }
    
    // Store reference
    this.componentRefs.set(id, componentRef);
    
    // Subscribe to component's change events
    if ((componentRef.instance as any).stateChange) {
      const sub = (componentRef.instance as any).stateChange.subscribe(() => {
        // When dynamic component changes, mark host for check
        this.cdr.markForCheck();
      });
      this.subscriptions.set(id, sub);
    }
    
    // Trigger CD for dynamic component
    componentRef.changeDetectorRef.detectChanges();
    
    // Trigger CD for host (affects projected content)
    this.cdr.markForCheck();
  }
  
  unloadComponent(id: string): void {
    const componentRef = this.componentRefs.get(id);
    if (componentRef) {
      componentRef.destroy();
      this.componentRefs.delete(id);
    }
    
    const subscription = this.subscriptions.get(id);
    if (subscription) {
      subscription.unsubscribe();
      this.subscriptions.delete(id);
    }
  }
  
  updateComponentInput(id: string, key: string, value: any): void {
    const componentRef = this.componentRefs.get(id);
    if (componentRef) {
      (componentRef.instance as any)[key] = value;
      componentRef.changeDetectorRef.detectChanges();
    }
  }
  
  ngOnDestroy(): void {
    // Clean up all dynamic components
    this.componentRefs.forEach(ref => ref.destroy());
    this.componentRefs.clear();
    
    // Clean up all subscriptions
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
  }
}
```

---

## Summary

**Content Projection Pipeline:**
1. **Parse** templates and identify `<ng-content>` slots
2. **Compile** into projection instructions with slot indices
3. **Distribute** projected nodes to appropriate slots based on selectors
4. **Create** view hierarchy with projected content in parent's context
5. **Run** change detection respecting view boundaries

**Key Principles:**
- **Projected content belongs to parent** - bindings, change detection, lifecycle
- **ng-content is a marker** - not a DOM element
- **Multi-slot uses CSS selectors** - powerful but can be complex
- **Change detection context matters** - OnPush can cause issues
- **Hybrid approaches work** - but require careful lifecycle management

**Best Practices:**
- Use content projection for flexible, reusable components
- Use selectors for multi-slot projection
- Use ng-template for passing templates with context
- Clean up dynamic components properly
- Consider change detection strategy interactions
- Document expected projected content structure

Content projection is one of Angular's most powerful features for building reusable components, but it requires understanding the internal mechanics to avoid common pitfalls with change detection, lifecycle, and context management.

