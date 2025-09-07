# Component Interaction

Proper web application architecture is important not only for production versions, but also for long-lived prototypes. Iterative changes are only possible when the complexity of conducting subsequent iterations doesn't grow exponentially.

Below we discuss four of the most rational ways to organize interface component interaction in Beast.

## 1. Through Block-Element Relationship

The BEM methodology offers the most convenient and simple way to connect components — when some (elements) are subordinate to others (blocks). It's generally accepted that all connections in hierarchical structures should be directed from parent to child, and the child shouldn't know anything about the context of its use. However, based on the fact that elements cannot exist without their parent block, this rule can and should be violated, but only when connecting a block and an element.

For example, a close button clears the input content by calling the parent's method:

```js
Beast.decl({
    TextInput: {
        clear: function () {
            this.elem('input').domNode().value = ''
        }
    },
    TextInput__clear: {
        on: {
            click: function () {
                this.parentBlock().clear()
            }
        }
    }
})
```

Of course, this could be done following the "parent to child" rule, but then the description of element behavior would mix with the block's behavior, and the clear declarative picture would be lost:

```js
Beast.decl({
    TextInput: {
        domInit: function () {
            this.elem('clear')[0].on('click', function () {
                this.clear()
            }.bind(this))
        },
        clear: function () {
            this.elem('input').domNode().value = ''
        }
    }
})
```

### Element-to-Block Communication Examples

#### Form Field Validation

```js
Beast.decl({
    FormField: {
        validate: function () {
            const value = this.elem('input').domNode().value
            const isValid = this.validateValue(value)
            this.mod('Valid', isValid)
            return isValid
        },
        
        validateValue: function (value) {
            // Override in specific field types
            return value.length > 0
        }
    },
    
    FormField__input: {
        on: {
            blur: function () {
                this.parentBlock().validate()
            },
            input: function () {
                // Clear error state on input
                this.parentBlock().mod('Valid', true)
            }
        }
    }
})
```

#### Tab System

```js
Beast.decl({
    TabPanel: {
        switchTab: function (index) {
            this.elems('tab').forEach((tab, i) => {
                tab.mod('Active', i === index)
            })
            this.elems('content').forEach((content, i) => {
                content.mod('Visible', i === index)
            })
        }
    },
    
    TabPanel__tab: {
        on: {
            click: function () {
                const index = this.index()
                this.parentBlock().switchTab(index)
            }
        }
    }
})
```

## 2. Through Common Parent

When components need to interact but aren't in a direct parent-child relationship, they can communicate through their common parent. The parent acts as a mediator, handling events from one component and triggering actions in another.

```js
Beast.decl({
    ShoppingCart: {
        on: {
            addItem: function (event, item) {
                this.elem('items').append(<item data={item}/>)
                this.elem('total').updateTotal()
                this.elem('counter').increment()
            },
            removeItem: function (event, itemId) {
                this.elem('items').removeItem(itemId)
                this.elem('total').updateTotal()
                this.elem('counter').decrement()
            }
        }
    },
    
    ShoppingCart__addButton: {
        on: {
            click: function () {
                const item = this.getItemData()
                this.parentBlock().trigger('addItem', item)
            }
        }
    },
    
    ShoppingCart__removeButton: {
        on: {
            click: function () {
                const itemId = this.param('itemId')
                this.parentBlock().trigger('removeItem', itemId)
            }
        }
    }
})
```

### Search and Results Example

```js
Beast.decl({
    SearchPage: {
        on: {
            search: function (event, query) {
                this.elem('results').search(query)
                this.elem('history').addQuery(query)
                this.elem('suggestions').hide()
            },
            
            suggestion: function (event, suggestion) {
                this.elem('input').setValue(suggestion)
                this.trigger('search', suggestion)
            }
        }
    },
    
    SearchPage__input: {
        on: {
            keypress: function (event) {
                if (event.keyCode === 13) {
                    const query = this.domNode().value
                    this.parentBlock().trigger('search', query)
                }
            },
            input: function () {
                const query = this.domNode().value
                if (query.length > 2) {
                    this.parentBlock().elem('suggestions').show(query)
                }
            }
        }
    },
    
    SearchPage__suggestion: {
        on: {
            click: function () {
                const suggestion = this.param('text')
                this.parentBlock().trigger('suggestion', suggestion)
            }
        }
    }
})
```

## 3. Through Common Event Bus

For more complex applications, a global event bus can facilitate communication between distant components without coupling them together.

```js
// Global event bus
window.EventBus = {
    events: {},
    
    on: function (event, callback) {
        if (!this.events[event]) {
            this.events[event] = []
        }
        this.events[event].push(callback)
    },
    
    trigger: function (event, data) {
        if (this.events[event]) {
            this.events[event].forEach(callback => callback(data))
        }
    },
    
    off: function (event, callback) {
        if (this.events[event]) {
            const index = this.events[event].indexOf(callback)
            if (index > -1) {
                this.events[event].splice(index, 1)
            }
        }
    }
}
```

### Using Event Bus

```js
Beast.decl({
    NotificationCenter: {
        domInit: function () {
            EventBus.on('notification', this.showNotification.bind(this))
        },
        
        showNotification: function (notification) {
            this.append(
                <notification type={notification.type}>
                    {notification.message}
                </notification>
            )
        }
    },
    
    UserLogin: {
        on: {
            loginSuccess: function (event, user) {
                EventBus.trigger('notification', {
                    type: 'success',
                    message: `Welcome back, ${user.name}!`
                })
            }
        }
    },
    
    ShoppingCart: {
        on: {
            addItem: function (event, item) {
                // ... add item logic
                EventBus.trigger('notification', {
                    type: 'info',
                    message: 'Item added to cart'
                })
            }
        }
    }
})
```

### Event Bus Best Practices

#### 1. Namespace Events

```js
// Good: Namespaced events
EventBus.trigger('user:login', userData)
EventBus.trigger('cart:addItem', itemData)
EventBus.trigger('ui:notification', notificationData)

// Bad: Generic events
EventBus.trigger('success', data)
EventBus.trigger('update', data)
```

#### 2. Clean Up Event Listeners

```js
Beast.decl({
    Component: {
        domInit: function () {
            this._onNotification = this.handleNotification.bind(this)
            EventBus.on('notification', this._onNotification)
        },
        
        destruct: function () {
            EventBus.off('notification', this._onNotification)
        }
    }
})
```

## 4. Domain-Oriented Abstractions

For complex business logic, create domain-specific abstraction layers that handle component coordination:

```js
// User Session Manager
window.UserSession = {
    currentUser: null,
    
    login: function (credentials) {
        return fetch('/api/login', {
            method: 'POST',
            body: JSON.stringify(credentials)
        })
        .then(response => response.json())
        .then(user => {
            this.currentUser = user
            EventBus.trigger('user:login', user)
            return user
        })
    },
    
    logout: function () {
        this.currentUser = null
        EventBus.trigger('user:logout')
    },
    
    isLoggedIn: function () {
        return !!this.currentUser
    }
}

// Shopping Cart Manager
window.CartManager = {
    items: [],
    
    addItem: function (item) {
        this.items.push(item)
        EventBus.trigger('cart:update', this.items)
        EventBus.trigger('notification', {
            type: 'success',
            message: 'Item added to cart'
        })
    },
    
    removeItem: function (itemId) {
        this.items = this.items.filter(item => item.id !== itemId)
        EventBus.trigger('cart:update', this.items)
    },
    
    getTotal: function () {
        return this.items.reduce((total, item) => total + item.price, 0)
    }
}
```

### Using Domain Abstractions

```js
Beast.decl({
    LoginForm: {
        on: {
            submit: function (event) {
                event.preventDefault()
                const credentials = this.getFormData()
                
                UserSession.login(credentials)
                    .then(user => {
                        this.parentBlock().hide()
                    })
                    .catch(error => {
                        this.showError(error.message)
                    })
            }
        }
    },
    
    ProductCard: {
        on: {
            addToCart: function () {
                const product = {
                    id: this.param('productId'),
                    name: this.param('productName'),
                    price: this.param('productPrice')
                }
                
                CartManager.addItem(product)
            }
        }
    },
    
    Header: {
        domInit: function () {
            EventBus.on('user:login', this.updateUserInfo.bind(this))
            EventBus.on('user:logout', this.showLoginButton.bind(this))
            EventBus.on('cart:update', this.updateCartCount.bind(this))
        }
    }
})
```

## Best Practices for Component Interaction

### 1. Choose the Right Communication Pattern

- **Block-Element**: For tightly coupled parent-child relationships
- **Common Parent**: For sibling components with shared state
- **Event Bus**: For loosely coupled, distant components
- **Domain Abstractions**: For complex business logic coordination

### 2. Keep Interfaces Simple

```js
// Good: Simple, focused interface
Beast.decl({
    Modal: {
        show: function () { /* ... */ },
        hide: function () { /* ... */ },
        setContent: function (content) { /* ... */ }
    }
})

// Bad: Complex, overloaded interface
Beast.decl({
    Modal: {
        showWithAnimation: function (animation, duration) { /* ... */ },
        showWithDelay: function (delay) { /* ... */ },
        showWithCallback: function (callback) { /* ... */ }
        // Too many specific methods
    }
})
```

### 3. Document Component Contracts

```js
Beast.decl({
    /**
     * SearchComponent
     * 
     * Events:
     * - search: Triggered when search is performed
     * - clear: Triggered when search is cleared
     * 
     * Methods:
     * - setQuery(query): Set search query programmatically
     * - clear(): Clear search and results
     * 
     * Parameters:
     * - placeholder: Input placeholder text
     * - autoSearch: Whether to search on typing (default: true)
     */
    SearchComponent: {
        // Implementation...
    }
})
```

### 4. Handle Edge Cases

```js
Beast.decl({
    DataList: {
        updateData: function (data) {
            // Handle edge cases
            if (!data || !Array.isArray(data)) {
                this.showError('Invalid data provided')
                return
            }
            
            if (data.length === 0) {
                this.showEmpty()
                return
            }
            
            this.renderData(data)
        }
    }
})
```

By following these patterns and practices, you can create maintainable, scalable component architectures that grow gracefully with your application's complexity.