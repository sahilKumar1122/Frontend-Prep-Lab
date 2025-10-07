# CSS Fundamentals

## Table of Contents
- [Selectors](#selectors)
- [Box Model](#box-model)
- [Positioning](#positioning)
- [Flexbox](#flexbox)
- [Grid](#grid)
- [Responsive Design](#responsive-design)

---

## Selectors

### What are the different types of CSS selectors?

**Answer:**

```css
/* Element selector */
p { color: blue; }

/* Class selector */
.button { padding: 10px; }

/* ID selector */
#header { background: white; }

/* Attribute selector */
input[type="text"] { border: 1px solid gray; }

/* Pseudo-class */
a:hover { color: red; }
li:first-child { font-weight: bold; }
input:focus { outline: 2px solid blue; }

/* Pseudo-element */
p::first-line { font-weight: bold; }
p::before { content: "→ "; }

/* Descendant selector */
div p { margin: 10px; }

/* Child selector */
ul > li { list-style: none; }

/* Adjacent sibling */
h1 + p { font-size: 18px; }

/* General sibling */
h1 ~ p { color: gray; }

/* Multiple selectors */
h1, h2, h3 { font-family: Arial; }

/* Combining selectors */
div.container > p.intro { font-size: 20px; }
```

**Selector Specificity (from low to high):**
1. Universal selector (*) - 0
2. Element selectors (div) - 1
3. Class selectors (.class) - 10
4. ID selectors (#id) - 100
5. Inline styles - 1000
6. !important - overrides all

---

## Box Model

### Explain the CSS Box Model

**Answer:**

Every element is a rectangular box with:
- **Content** - actual content
- **Padding** - space inside border
- **Border** - surrounds padding
- **Margin** - space outside border

```css
.box {
  /* Content */
  width: 200px;
  height: 100px;
  
  /* Padding */
  padding: 20px;
  /* or */
  padding: 10px 20px; /* vertical horizontal */
  padding: 10px 20px 10px 20px; /* top right bottom left */
  
  /* Border */
  border: 2px solid black;
  border-radius: 8px;
  
  /* Margin */
  margin: 20px;
  margin: 10px auto; /* center horizontally */
  
  /* Box sizing */
  box-sizing: border-box; /* includes padding and border in width */
}

/* Default box-sizing */
.default-box {
  width: 200px;
  padding: 20px;
  border: 2px solid black;
  /* Total width = 200 + (20*2) + (2*2) = 244px */
}

/* Border-box */
.border-box {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 2px solid black;
  /* Total width = 200px (padding and border included) */
}
```

---

## Positioning

### What are the different position values in CSS?

**Answer:**

```css
/* Static (default) */
.static {
  position: static;
  /* Follows normal document flow */
}

/* Relative */
.relative {
  position: relative;
  top: 10px;
  left: 20px;
  /* Offset from normal position */
}

/* Absolute */
.absolute {
  position: absolute;
  top: 0;
  right: 0;
  /* Positioned relative to nearest positioned ancestor */
}

/* Fixed */
.fixed {
  position: fixed;
  bottom: 20px;
  right: 20px;
  /* Fixed to viewport */
}

/* Sticky */
.sticky {
  position: sticky;
  top: 0;
  /* Sticks when scrolling past threshold */
}

/* Z-index */
.layer1 { z-index: 1; }
.layer2 { z-index: 10; }
.layer3 { z-index: 100; }
```

---

## Flexbox

### How does Flexbox work?

**Answer:**

```css
/* Container */
.flex-container {
  display: flex;
  
  /* Direction */
  flex-direction: row; /* row, column, row-reverse, column-reverse */
  
  /* Wrap */
  flex-wrap: wrap; /* nowrap, wrap, wrap-reverse */
  
  /* Justify content (main axis) */
  justify-content: center; /* flex-start, flex-end, center, space-between, space-around, space-evenly */
  
  /* Align items (cross axis) */
  align-items: center; /* flex-start, flex-end, center, stretch, baseline */
  
  /* Align content (multiple lines) */
  align-content: space-between;
  
  /* Gap */
  gap: 20px;
  column-gap: 10px;
  row-gap: 15px;
}

/* Items */
.flex-item {
  /* Flex grow */
  flex-grow: 1; /* take up available space */
  
  /* Flex shrink */
  flex-shrink: 0; /* don't shrink */
  
  /* Flex basis */
  flex-basis: 200px; /* initial size */
  
  /* Shorthand */
  flex: 1 0 200px; /* grow shrink basis */
  
  /* Align self */
  align-self: flex-end;
  
  /* Order */
  order: 2;
}
```

---

## Grid

### How does CSS Grid work?

**Answer:**

```css
/* Container */
.grid-container {
  display: grid;
  
  /* Columns */
  grid-template-columns: 200px 200px 200px;
  grid-template-columns: repeat(3, 1fr);
  grid-template-columns: 1fr 2fr 1fr;
  
  /* Rows */
  grid-template-rows: 100px auto 100px;
  
  /* Gap */
  gap: 20px;
  column-gap: 10px;
  row-gap: 15px;
  
  /* Areas */
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";
}

/* Items */
.grid-item {
  /* Span columns */
  grid-column: 1 / 3; /* start / end */
  grid-column: span 2;
  
  /* Span rows */
  grid-row: 1 / 3;
  grid-row: span 2;
  
  /* Area */
  grid-area: header;
}

/* Responsive grid */
.responsive-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
}
```

---

## Responsive Design

### How do you create responsive designs?

**Answer:**

```css
/* Mobile-first approach */
.container {
  width: 100%;
  padding: 10px;
}

/* Tablet */
@media (min-width: 768px) {
  .container {
    max-width: 720px;
    margin: 0 auto;
    padding: 20px;
  }
}

/* Desktop */
@media (min-width: 1024px) {
  .container {
    max-width: 960px;
  }
}

/* Large desktop */
@media (min-width: 1280px) {
  .container {
    max-width: 1200px;
  }
}

/* Responsive units */
.responsive-text {
  /* Viewport units */
  font-size: 4vw; /* 4% of viewport width */
  padding: 2vh; /* 2% of viewport height */
  
  /* Relative units */
  font-size: 1.5rem; /* relative to root */
  padding: 2em; /* relative to element font-size */
  
  /* Clamp (min, preferred, max) */
  font-size: clamp(16px, 4vw, 32px);
}

/* Responsive images */
img {
  max-width: 100%;
  height: auto;
}

/* Container queries (modern) */
@container (min-width: 400px) {
  .card {
    display: flex;
  }
}
```
