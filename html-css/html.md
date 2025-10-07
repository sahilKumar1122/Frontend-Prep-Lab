# HTML Fundamentals

## Table of Contents
- [Semantic HTML](#semantic-html)
- [Forms](#forms)
- [Accessibility](#accessibility)
- [SEO](#seo)
- [Best Practices](#best-practices)

---

## Semantic HTML

### What is semantic HTML and why is it important?

**Answer:**

Semantic HTML uses meaningful tags that describe the content's purpose, not just its appearance.

**Code Example:**
```html
<!-- Non-semantic (bad) -->
<div class="header">
  <div class="nav">
    <div class="nav-item">Home</div>
    <div class="nav-item">About</div>
  </div>
</div>
<div class="content">
  <div class="post">
    <div class="title">My Post</div>
    <div class="text">Post content...</div>
  </div>
</div>
<div class="footer">Footer content</div>

<!-- Semantic (good) -->
<header>
  <nav>
    <a href="/">Home</a>
    <a href="/about">About</a>
  </nav>
</header>
<main>
  <article>
    <h1>My Post</h1>
    <p>Post content...</p>
  </article>
</main>
<footer>Footer content</footer>
```

**Common Semantic Elements:**
- `<header>` - Header section
- `<nav>` - Navigation links
- `<main>` - Main content
- `<article>` - Self-contained content
- `<section>` - Thematic grouping
- `<aside>` - Sidebar/tangential content
- `<footer>` - Footer section
- `<figure>` and `<figcaption>` - Images with captions
- `<time>` - Dates and times
- `<mark>` - Highlighted text

**Benefits:**
- Better SEO
- Improved accessibility
- Easier to maintain
- Better for screen readers
- More meaningful code

---

## Forms

### How do you create accessible and user-friendly forms?

**Answer:**

**Code Example:**
```html
<form action="/submit" method="POST" novalidate>
  <!-- Text input with label -->
  <div class="form-group">
    <label for="username">Username:</label>
    <input 
      type="text" 
      id="username" 
      name="username"
      required
      minlength="3"
      maxlength="20"
      placeholder="Enter username"
      aria-describedby="username-help"
    />
    <small id="username-help">Must be 3-20 characters</small>
  </div>

  <!-- Email input -->
  <div class="form-group">
    <label for="email">Email:</label>
    <input 
      type="email" 
      id="email" 
      name="email"
      required
      pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$"
    />
  </div>

  <!-- Password input -->
  <div class="form-group">
    <label for="password">Password:</label>
    <input 
      type="password" 
      id="password" 
      name="password"
      required
      minlength="8"
    />
  </div>

  <!-- Select dropdown -->
  <div class="form-group">
    <label for="country">Country:</label>
    <select id="country" name="country" required>
      <option value="">Select a country</option>
      <option value="us">United States</option>
      <option value="uk">United Kingdom</option>
      <option value="ca">Canada</option>
    </select>
  </div>

  <!-- Radio buttons -->
  <fieldset>
    <legend>Gender:</legend>
    <label>
      <input type="radio" name="gender" value="male" required />
      Male
    </label>
    <label>
      <input type="radio" name="gender" value="female" />
      Female
    </label>
    <label>
      <input type="radio" name="gender" value="other" />
      Other
    </label>
  </fieldset>

  <!-- Checkboxes -->
  <div class="form-group">
    <label>
      <input type="checkbox" name="terms" required />
      I agree to the terms and conditions
    </label>
  </div>

  <!-- Textarea -->
  <div class="form-group">
    <label for="message">Message:</label>
    <textarea 
      id="message" 
      name="message"
      rows="4"
      maxlength="500"
    ></textarea>
  </div>

  <!-- Submit button -->
  <button type="submit">Submit</button>
  <button type="reset">Reset</button>
</form>
```

**HTML5 Input Types:**
- `email`, `url`, `tel`, `number`
- `date`, `time`, `datetime-local`
- `color`, `range`, `search`
- `file` (with `accept` attribute)

**Validation Attributes:**
- `required` - Field must be filled
- `minlength` / `maxlength` - Length constraints
- `min` / `max` - Number constraints
- `pattern` - Regex validation
- `type` - Input type validation

**Key Points:**
- Always use `<label>` with `for` attribute
- Use `<fieldset>` and `<legend>` for groups
- Add ARIA attributes for accessibility
- Use semantic input types
- Provide helpful error messages

---

## Accessibility

### How do you make HTML accessible?

**Answer:**

**Code Example:**
```html
<!-- Proper heading hierarchy -->
<h1>Main Page Title</h1>
  <h2>Section 1</h2>
    <h3>Subsection 1.1</h3>
  <h2>Section 2</h2>

<!-- Alternative text for images -->
<img src="logo.png" alt="Company Logo" />
<img src="decorative.png" alt="" role="presentation" />

<!-- ARIA roles and attributes -->
<nav role="navigation" aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- Skip to main content link -->
<a href="#main-content" class="skip-link">Skip to main content</a>
<main id="main-content">
  <!-- Content -->
</main>

<!-- Accessible buttons -->
<button 
  type="button"
  aria-label="Close modal"
  aria-pressed="false"
>
  ×
</button>

<!-- Loading states -->
<div role="status" aria-live="polite" aria-atomic="true">
  Loading...
</div>

<!-- Custom controls -->
<div 
  role="button" 
  tabindex="0"
  aria-pressed="false"
  onkeypress="handleKeyPress(event)"
>
  Custom Button
</div>

<!-- Tables -->
<table>
  <caption>Employee Data</caption>
  <thead>
    <tr>
      <th scope="col">Name</th>
      <th scope="col">Role</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>John Doe</td>
      <td>Developer</td>
    </tr>
  </tbody>
</table>

<!-- Landmark regions -->
<header role="banner">Header</header>
<nav role="navigation">Navigation</nav>
<main role="main">Main content</main>
<aside role="complementary">Sidebar</aside>
<footer role="contentinfo">Footer</footer>
```

**ARIA Attributes:**
- `aria-label` - Accessible name
- `aria-labelledby` - Reference to label
- `aria-describedby` - Additional description
- `aria-hidden` - Hide from screen readers
- `aria-live` - Live regions
- `aria-expanded` - Expandable elements
- `role` - Element role

**Key Points:**
- Use semantic HTML
- Provide text alternatives
- Ensure keyboard navigation
- Use ARIA when necessary
- Test with screen readers
- Maintain proper heading hierarchy

---

## SEO

### How do you optimize HTML for SEO?

**Answer:**

**Code Example:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- Essential meta tags -->
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- Title (50-60 characters) -->
  <title>Frontend Interview Prep | Complete Guide</title>
  
  <!-- Meta description (150-160 characters) -->
  <meta name="description" content="Master frontend interviews with our comprehensive guide covering JavaScript, React, and more.">
  
  <!-- Keywords (less important now) -->
  <meta name="keywords" content="frontend, interview, javascript, react">
  
  <!-- Canonical URL -->
  <link rel="canonical" href="https://example.com/page">
  
  <!-- Open Graph (Facebook) -->
  <meta property="og:title" content="Frontend Interview Prep">
  <meta property="og:description" content="Complete interview guide">
  <meta property="og:image" content="https://example.com/image.jpg">
  <meta property="og:url" content="https://example.com/page">
  <meta property="og:type" content="website">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:title" content="Frontend Interview Prep">
  <meta name="twitter:description" content="Complete interview guide">
  <meta name="twitter:image" content="https://example.com/image.jpg">
  
  <!-- Favicon -->
  <link rel="icon" type="image/png" href="/favicon.png">
  
  <!-- Structured data (JSON-LD) -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "Frontend Interview Prep",
    "author": {
      "@type": "Person",
      "name": "John Doe"
    },
    "datePublished": "2025-01-01",
    "image": "https://example.com/image.jpg"
  }
  </script>
</head>
<body>
  <!-- Proper heading hierarchy -->
  <h1>Main Title (Only One H1)</h1>
  <h2>Section Title</h2>
  <h3>Subsection Title</h3>
  
  <!-- Descriptive links -->
  <a href="/guide" title="Complete Frontend Guide">
    Read our complete guide
  </a>
  
  <!-- Image optimization -->
  <img 
    src="image.jpg" 
    alt="Descriptive alt text"
    width="800"
    height="600"
    loading="lazy"
  />
</body>
</html>
```

**SEO Best Practices:**
- Unique, descriptive titles
- Compelling meta descriptions
- Proper heading hierarchy (one H1)
- Semantic HTML structure
- Fast page load times
- Mobile-friendly design
- HTTPS enabled
- Clean, descriptive URLs
- Internal linking
- Structured data (Schema.org)

---

## Best Practices

### What are HTML best practices?

**Answer:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  
  <!-- CSS in head -->
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <!-- Content here -->
  
  <!-- JavaScript at end of body -->
  <script src="script.js"></script>
</body>
</html>
```

**Best Practices:**
1. **Always declare DOCTYPE**
2. **Use semantic HTML**
3. **Proper document structure**
4. **Close all tags**
5. **Use lowercase for tags and attributes**
6. **Quote attribute values**
7. **Add alt text to images**
8. **Validate HTML**
9. **Minimize nesting depth**
10. **Use comments wisely**

**Code Example:**
```html
<!-- Good -->
<img src="logo.png" alt="Company Logo" />
<a href="/page" title="Page Title">Link</a>

<!-- Bad -->
<IMG SRC=logo.png>
<a href=/page>Link</a>

<!-- Good structure -->
<article>
  <header>
    <h2>Article Title</h2>
    <time datetime="2025-01-01">January 1, 2025</time>
  </header>
  <p>Article content...</p>
  <footer>Author info</footer>
</article>
```

---

## Interview Practice Questions

1. What is the difference between div and span?
2. What are semantic HTML elements?
3. How do you make a website accessible?
4. What is the purpose of the DOCTYPE?
5. Explain the difference between block and inline elements
6. What are data attributes?
7. How do you optimize images for web?
8. What is the purpose of meta tags?
