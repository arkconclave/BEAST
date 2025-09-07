# Component Inheritance

Inheritance in Beast is the extension of one declaration by others. This allows both building new components based on existing ones and extracting common code into abstract components.

## New Components Based on Existing Ones

Let's consider a `Snippet` block as an example.

```xml
<Snippet>
    <title>...</title>
    <url>...</url>
    <text>...</text>
</Snippet>
```

```js
Beast.decl({
    Snippet: {
        mod: {
            ShowFavicon: false,
        },
        expand: function () {
            this.append(
                this.get('title', 'url', 'text')
            )
        }
    },
})
```

Over time, this block will need to become more complex: add an "advertisement" badge, sitelinks, metadata, rating, image carousel, table, etc. Later, it may be discovered that in one case it's convenient to have a table above the image carousel, and in another case — below it. The block will become complex both from a user perspective (due to overloaded input data logic) and from a development perspective (more and more code, complexity of making changes grows exponentially). Therefore, such constructions need to be simplified.

The snippet block should be limited to the most general fields, and then start creating descendants. For example, an image widget becomes an independent block extending the snippet block.

```xml
<SnippetImages>
    <title>...</title>
    <url>...</url>
    <images>
        <image>...</image>
        <image>...</image>
        ...
    </images>
</SnippetImages>
```

```js
Beast.decl({
    SnippetImages: {
        inherit: 'Snippet',
        expand: function () {
            this.append(
                this.get('title', 'url'),
                <images>
                    {this.get('images').children()}
                </images>
            )
        }
    },
})
```

The `inherit` property indicates that the `SnippetImages` declaration extends the `Snippet` declaration. All properties of the parent declaration are inherited: modifiers, methods, event handlers, etc.

## Method Override

When inheriting, you can override parent methods:

```js
Beast.decl({
    Snippet: {
        expand: function () {
            this.append(
                this.get('title', 'url', 'text')
            )
        },
        
        updateContent: function () {
            console.log('Updating basic snippet')
        }
    }
})

Beast.decl({
    SnippetWithRating: {
        inherit: 'Snippet',
        
        expand: function () {
            this.append(
                this.get('title', 'url', 'text'),
                <rating>{this.param('rating')}</rating>
            )
        },
        
        updateContent: function () {
            console.log('Updating snippet with rating')
            this.elem('rating').text(this.param('rating'))
        }
    }
})
```

## Calling Parent Methods

You can call parent methods using `this.__base()`:

```js
Beast.decl({
    SnippetExtended: {
        inherit: 'Snippet',
        
        expand: function () {
            // Call parent expand method
            this.__base()
            
            // Add additional content
            this.append(<extra>Additional content</extra>)
        },
        
        updateContent: function () {
            // Call parent update first
            this.__base()
            
            // Then do additional updates
            this.elem('extra').text('Updated extra content')
        }
    }
})
```

## Multiple Inheritance

Beast supports multiple inheritance by listing multiple parent declarations:

```js
Beast.decl({
    Clickable: {
        on: {
            click: function () {
                this.trigger('itemClick')
            }
        }
    },
    
    Hoverable: {
        on: {
            mouseenter: function () {
                this.mod('Hover', true)
            },
            mouseleave: function () {
                this.mod('Hover', false)
            }
        }
    },
    
    InteractiveSnippet: {
        inherit: ['Snippet', 'Clickable', 'Hoverable'],
        
        expand: function () {
            this.__base() // Calls Snippet.expand
            this.mod('Interactive', true)
        }
    }
})
```

## Abstract Components

Abstract components contain common functionality but are not meant to be used directly:

```js
Beast.decl({
    // Abstract base component for all form controls
    FormControl: {
        expand: function () {
            this.mod('Required', this.param('required', false))
            this.mod('Disabled', this.param('disabled', false))
        },
        
        validate: function () {
            const value = this.getValue()
            const isValid = this.isValidValue(value)
            this.mod('Valid', isValid)
            return isValid
        },
        
        // Abstract method - must be implemented by children
        getValue: function () {
            throw new Error('getValue must be implemented')
        },
        
        // Abstract method - must be implemented by children
        isValidValue: function (value) {
            throw new Error('isValidValue must be implemented')
        }
    }
})

// Concrete implementations
Beast.decl({
    TextInput: {
        inherit: 'FormControl',
        
        getValue: function () {
            return this.domNode().value
        },
        
        isValidValue: function (value) {
            return value.length > 0
        }
    },
    
    EmailInput: {
        inherit: 'FormControl',
        
        getValue: function () {
            return this.domNode().value
        },
        
        isValidValue: function (value) {
            const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
            return emailRegex.test(value)
        }
    }
})
```

## Mixin Pattern

For shared functionality that doesn't form a clear hierarchy, use mixins:

```js
Beast.decl({
    // Mixin for draggable functionality
    Draggable: {
        makeDraggable: function () {
            let isDragging = false
            let startX, startY
            
            this.on({
                mousedown: function (e) {
                    isDragging = true
                    startX = e.clientX - this.domNode().offsetLeft
                    startY = e.clientY - this.domNode().offsetTop
                    this.mod('Dragging', true)
                },
                
                mousemove: function (e) {
                    if (!isDragging) return
                    
                    const x = e.clientX - startX
                    const y = e.clientY - startY
                    
                    this.domNode().style.left = x + 'px'
                    this.domNode().style.top = y + 'px'
                },
                
                mouseup: function () {
                    isDragging = false
                    this.mod('Dragging', false)
                }
            })
        }
    },
    
    // Mixin for resizable functionality
    Resizable: {
        makeResizable: function () {
            this.append(<resizeHandle/>)
            
            this.elem('resizeHandle').on({
                mousedown: function (e) {
                    e.preventDefault()
                    // Resize logic here
                }
            })
        }
    }
})

// Component using multiple mixins
Beast.decl({
    DialogWindow: {
        inherit: ['Dialog', 'Draggable', 'Resizable'],
        
        domInit: function () {
            this.makeDraggable()
            this.makeResizable()
        }
    }
})
```

## Inheritance Chain Resolution

When multiple inheritance chains exist, Beast resolves method calls in a specific order:

1. Current component's methods
2. First parent's methods (left to right in inherit array)
3. Second parent's methods
4. And so on...

```js
Beast.decl({
    A: {
        method: function () { return 'A' }
    },
    
    B: {
        method: function () { return 'B' }
    },
    
    C: {
        inherit: ['A', 'B'],
        // Will use A.method since A is listed first
    }
})
```

## Best Practices for Inheritance

### 1. Prefer Composition Over Deep Inheritance

```js
// Good: Shallow inheritance with composition
Beast.decl({
    UserCard: {
        expand: function () {
            this.append(
                <Avatar/>,
                <UserInfo/>,
                <ActionButtons/>
            )
        }
    }
})

// Less preferred: Deep inheritance
Beast.decl({ Card: {} })
Beast.decl({ PersonCard: { inherit: 'Card' } })
Beast.decl({ UserCard: { inherit: 'PersonCard' } })
Beast.decl({ AdminUserCard: { inherit: 'UserCard' } })
```

### 2. Use Abstract Components for Common Patterns

```js
// Good: Abstract base for similar components
Beast.decl({
    MediaItem: {
        expand: function () {
            this.append(
                <thumbnail/>,
                <title/>,
                <metadata/>
            )
        },
        
        play: function () {
            // Common play logic
        }
    }
})

Beast.decl({
    VideoItem: { inherit: 'MediaItem' },
    AudioItem: { inherit: 'MediaItem' },
    ImageItem: { inherit: 'MediaItem' }
})
```

### 3. Always Call Parent Methods When Extending

```js
Beast.decl({
    EnhancedButton: {
        inherit: 'Button',
        
        expand: function () {
            // Always call parent first
            this.__base()
            
            // Then add enhancements
            this.append(<analytics/>)
        }
    }
})
```

### 4. Use Meaningful Inheritance Hierarchies

```js
// Good: Clear hierarchy
Beast.decl({ Control: {} })
Beast.decl({ Input: { inherit: 'Control' } })
Beast.decl({ TextInput: { inherit: 'Input' } })

// Bad: Unclear relationships
Beast.decl({ Thing: {} })
Beast.decl({ Stuff: { inherit: 'Thing' } })
Beast.decl({ Widget: { inherit: 'Stuff' } })
```

### 5. Document Abstract Methods

```js
Beast.decl({
    DataProvider: {
        /**
         * Abstract method: Must be implemented by subclasses
         * @returns {Promise} Promise that resolves with data
         */
        fetchData: function () {
            throw new Error('fetchData must be implemented by subclass')
        },
        
        loadData: function () {
            return this.fetchData().then(data => {
                this.render(data)
            })
        }
    }
})
```

This inheritance system allows for powerful code reuse while maintaining clear component hierarchies and responsibilities.