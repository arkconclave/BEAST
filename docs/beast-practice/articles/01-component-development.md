# Component Development

## Optimal Naming

Entity names (blocks, elements, modifiers, events, parameters) should use __camelCase__ notation — this eliminates the need for quotes in JSON. Block names are capitalized.

__Incorrect:__
```js
Beast.decl({
    'block-name': {
        ...
    },
    'block-name__element-name'
})
```

__Correct:__
```js
Beast.decl({
    BlockName: {
        ...
    },
    BlockName__elementName: {
        ...
    }
})
```

__Compound names__ are used for homogeneous components (usually sharing a common ancestor). Consider a `Card` block with background and title. From this block, a taxi booking card could inherit — logically named `CardTaxi`; an airline search card — `CardAvia`, and so on.

If blocks differ significantly in appearance and behavior but share a common ancestor — this shouldn't be reflected in their names. For example, `Button` and `Input` may inherit from `Control`, but names like `ControlButton` and `ControlInput` would be redundant.

Compound names are given to elements when they're subordinate to other elements.

```xml
<Head>
    <menu>
        <menuItem>...</menuItem>
        <menuItem>...</menuItem>
    </menu>
    <submenu>
        <submenuItem>...</submenuItem>
        <submenuItem>...</submenuItem>
    </submenu>
<Head>
```

## Working with Modifiers

Modifiers define component appearance and behavior characteristics. Changing a modifier triggers a corresponding event. A modifier corresponds to a CSS class. Modifiers are capitalized.

```js
Beast.decl('Button', {
    expand: function () {
        this.mod('Type', 'normal')
    }
})
```

If the component should have a certain modifier by default, it's set in the `expand` method. When the component is created, modifiers can be passed through attributes:

```xml
<Button Type="submit"/>
```

### Modifier Values

Boolean modifiers have values `true/false`. String modifiers can have any textual value.

```js
node.mod('visible', true)
node.mod('Type', 'submit')
node.mod('Size', 'large')
```

### Modifier Events

Changing a modifier value automatically triggers an event in the format `onMod{ModifierName}`. The event handler receives two arguments: the new value and the previous value.

```js
Beast.decl('Button', {
    onModType: function (newValue, prevValue) {
        console.log(`Type changed from ${prevValue} to ${newValue}`)
    }
})
```

### CSS Classes for Modifiers

Setting a modifier automatically adds a corresponding CSS class to the DOM element. The class format follows BEM methodology: `block_modifier_value` for blocks and `block__element_modifier_value` for elements.

```js
// Setting modifier
button.mod('Type', 'submit')

// Results in CSS class
// .button_type_submit
```

## Event Handling

Beast provides several ways to handle events:

### DOM Events

Standard DOM events can be handled using the `on` object in the declaration:

```js
Beast.decl('Button', {
    on: {
        click: function (event) {
            console.log('Button clicked')
        },
        
        mouseenter: function (event) {
            this.mod('Hover', true)
        },
        
        mouseleave: function (event) {
            this.mod('Hover', false)
        }
    }
})
```

### Custom Events

Custom events can be triggered using the `trigger` method and handled using `on`:

```js
Beast.decl('Modal', {
    show: function () {
        this.mod('Visible', true)
        this.trigger('show')
    },
    
    hide: function () {
        this.mod('Visible', false)
        this.trigger('hide')
    },
    
    on: {
        show: function () {
            console.log('Modal is shown')
        },
        
        hide: function () {
            console.log('Modal is hidden')
        }
    }
})
```

### Window Events

Global window events can be handled using the `onWin` object:

```js
Beast.decl('Header', {
    onWin: {
        scroll: function () {
            const scrollTop = window.pageYOffset
            this.mod('Fixed', scrollTop > 100)
        },
        
        resize: function () {
            this.updateLayout()
        }
    }
})
```

## Component Lifecycle

### expand Method

The `expand` method is called during component initialization. It's used to set up the component structure, default modifiers, and initial state:

```js
Beast.decl('Dialog', {
    expand: function () {
        // Set default modifiers
        this.mod('Visible', false)
        this.mod('Size', 'medium')
        
        // Build component structure
        this.append(
            <header>
                <title>{this.param('title')}</title>
                <close/>
            </header>,
            <content>
                {this.children()}
            </content>
        )
    }
})
```

### domInit Method

The `domInit` method is called after the component is rendered to the DOM. It's used for DOM-related initialization:

```js
Beast.decl('Slider', {
    domInit: function () {
        this.slider = new SomeSliderLibrary(this.domNode())
        this.slider.init()
    }
})
```

### afterDomInit Method

The `afterDomInit` method is called after all child components have been initialized:

```js
Beast.decl('Form', {
    afterDomInit: function () {
        // All form fields are now initialized
        this.validateAllFields()
    }
})
```

## Parameters

Parameters are used to pass data to components. They're accessible via the `param` method:

```js
Beast.decl('User', {
    expand: function () {
        const name = this.param('name')
        const email = this.param('email')
        
        this.append(
            <name>{name}</name>,
            <email>{email}</email>
        )
    }
})
```

Usage:
```xml
<User name="John Doe" email="john@example.com"/>
```

### Default Parameters

Default parameter values can be set in the declaration:

```js
Beast.decl('Button', {
    Size: 'medium',
    Type: 'button',
    
    expand: function () {
        // this.param('Size') will return 'medium' if not specified
        // this.param('Type') will return 'button' if not specified
    }
})
```

## Component State Management

### Internal State

Components can maintain internal state using regular JavaScript properties:

```js
Beast.decl('Counter', {
    expand: function () {
        this._count = 0
        
        this.append(
            <display>{this._count}</display>,
            <button>+</button>,
            <button>-</button>
        )
    },
    
    increment: function () {
        this._count++
        this.elem('display').text(this._count)
    },
    
    decrement: function () {
        this._count--
        this.elem('display').text(this._count)
    },
    
    on: {
        click: function (event) {
            const target = event.target
            if (target.textContent === '+') {
                this.increment()
            } else if (target.textContent === '-') {
                this.decrement()
            }
        }
    }
})
```

### Reactive Updates

For more complex state management, use modifiers which automatically trigger updates:

```js
Beast.decl('TodoItem', {
    expand: function () {
        this.mod('Completed', this.param('completed', false))
        
        this.append(
            <checkbox/>,
            <text>{this.param('text')}</text>
        )
    },
    
    onModCompleted: function (isCompleted) {
        this.elem('checkbox').mod('Checked', isCompleted)
        this.mod('State', isCompleted ? 'done' : 'pending')
    }
})
```

## Component Communication

### Parent-Child Communication

Children can communicate with parents using events:

```js
Beast.decl('TabPanel', {
    expand: function () {
        this.append(
            <tabs>
                <tab>Tab 1</tab>
                <tab>Tab 2</tab>
                <tab>Tab 3</tab>
            </tabs>,
            <content/>
        )
    },
    
    on: {
        tabSelect: function (event, tabIndex) {
            this.showTab(tabIndex)
        }
    }
})

Beast.decl('TabPanel__tab', {
    on: {
        click: function () {
            const index = this.index()
            this.parentBlock().trigger('tabSelect', index)
        }
    }
})
```

### Sibling Communication

Siblings communicate through their common parent:

```js
Beast.decl('SearchForm', {
    on: {
        search: function (event, query) {
            this.elem('results').search(query)
        }
    }
})

Beast.decl('SearchForm__input', {
    on: {
        keypress: function (event) {
            if (event.keyCode === 13) { // Enter
                const query = this.domNode().value
                this.parentBlock().trigger('search', query)
            }
        }
    }
})

Beast.decl('SearchForm__results', {
    search: function (query) {
        // Perform search and display results
        this.empty().append(
            <item>Result 1</item>,
            <item>Result 2</item>
        )
    }
})
```

## Best Practices

### 1. Keep Components Focused

Each component should have a single responsibility:

```js
// Good: Focused responsibility
Beast.decl('SearchInput', {
    // Only handles input behavior
})

Beast.decl('SearchResults', {
    // Only handles results display
})

// Bad: Too many responsibilities
Beast.decl('SearchComponent', {
    // Handles input, results, pagination, filtering, etc.
})
```

### 2. Use Composition Over Inheritance

Prefer composing components rather than deep inheritance:

```js
// Good: Composition
Beast.decl('UserCard', {
    expand: function () {
        this.append(
            <Avatar user={this.param('user')}/>,
            <UserInfo user={this.param('user')}/>,
            <ContactButtons user={this.param('user')}/>
        )
    }
})

// Less preferred: Deep inheritance
Beast.decl('BaseCard', { /* ... */ })
Beast.decl('UserCard', { inherit: 'BaseCard' })
Beast.decl('AdminUserCard', { inherit: 'UserCard' })
```

### 3. Minimize DOM Queries

Cache DOM references when needed multiple times:

```js
Beast.decl('Slider', {
    domInit: function () {
        this._slider = this.domNode().querySelector('.slider-track')
        this._handle = this.domNode().querySelector('.slider-handle')
    },
    
    updatePosition: function (value) {
        // Use cached references
        const position = (value / 100) * this._slider.offsetWidth
        this._handle.style.left = position + 'px'
    }
})
```

### 4. Use Semantic Element Names

Choose meaningful names that describe purpose, not appearance:

```js
// Good: Semantic names
Beast.decl('ProductCard', {
    expand: function () {
        this.append(
            <image/>,
            <title/>,
            <price/>,
            <addToCart/>
        )
    }
})

// Bad: Appearance-based names
Beast.decl('ProductCard', {
    expand: function () {
        this.append(
            <topImage/>,
            <boldText/>,
            <redPrice/>,
            <blueButton/>
        )
    }
})
```

### 5. Handle Edge Cases

Always consider and handle edge cases:

```js
Beast.decl('ImageGallery', {
    expand: function () {
        const images = this.param('images', [])
        
        if (images.length === 0) {
            this.append(<empty>No images available</empty>)
            return
        }
        
        if (images.length === 1) {
            this.mod('SingleImage', true)
        }
        
        this.append(
            images.map(img => <image src={img.url} alt={img.alt}/>)
        )
    }
})
```

This completes the component development guide with comprehensive coverage of naming conventions, modifiers, events, lifecycle methods, state management, and best practices.