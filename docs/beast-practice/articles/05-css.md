# Style Organization

## Dynamic Grid System
- Why screen grids are better than paper grids
- Grid influence zone: entire screen, container
- Discrete grid setups for specific screens

## Unified Typography
- Main text styles and auxiliary styles
- Rules for using px and em

## Working with Other Constants
- Global constants
- Local constants

## CSS Architecture Best Practices

### 1. Component-Based Styles

Following BEM methodology, organize styles around components:

```css
/* Block styles */
.button {
    display: inline-block;
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

/* Element styles */
.button__text {
    font-weight: bold;
}

.button__icon {
    margin-right: 8px;
}

/* Modifier styles */
.button_size_large {
    padding: 12px 24px;
    font-size: 18px;
}

.button_type_primary {
    background-color: #007bff;
    color: white;
}
```

### 2. CSS Custom Properties for Constants

```css
:root {
    /* Colors */
    --color-primary: #007bff;
    --color-secondary: #6c757d;
    --color-success: #28a745;
    --color-danger: #dc3545;
    
    /* Spacing */
    --spacing-xs: 4px;
    --spacing-sm: 8px;
    --spacing-md: 16px;
    --spacing-lg: 24px;
    --spacing-xl: 32px;
    
    /* Typography */
    --font-size-sm: 14px;
    --font-size-md: 16px;
    --font-size-lg: 18px;
    --font-size-xl: 24px;
    
    /* Layout */
    --container-max-width: 1200px;
    --grid-gap: 20px;
}
```

### 3. Responsive Grid System

```css
.grid {
    display: grid;
    gap: var(--grid-gap);
    grid-template-columns: repeat(12, 1fr);
    max-width: var(--container-max-width);
    margin: 0 auto;
    padding: 0 var(--spacing-md);
}

.grid__item {
    grid-column: span 12;
}

/* Responsive breakpoints */
@media (min-width: 768px) {
    .grid__item_md_6 {
        grid-column: span 6;
    }
    
    .grid__item_md_4 {
        grid-column: span 4;
    }
    
    .grid__item_md_3 {
        grid-column: span 3;
    }
}

@media (min-width: 1024px) {
    .grid__item_lg_8 {
        grid-column: span 8;
    }
    
    .grid__item_lg_4 {
        grid-column: span 4;
    }
}
```

### 4. Typography System

```css
/* Base typography */
body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    font-size: var(--font-size-md);
    line-height: 1.5;
    color: #333;
}

/* Heading hierarchy */
.heading {
    margin: 0 0 var(--spacing-md) 0;
    font-weight: bold;
    line-height: 1.2;
}

.heading_level_1 {
    font-size: var(--font-size-xl);
}

.heading_level_2 {
    font-size: var(--font-size-lg);
}

.heading_level_3 {
    font-size: var(--font-size-md);
}

/* Text utilities */
.text_size_small {
    font-size: var(--font-size-sm);
}

.text_weight_bold {
    font-weight: bold;
}

.text_align_center {
    text-align: center;
}
```