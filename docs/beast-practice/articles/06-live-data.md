# Working with Live Data

At this stage, it's recommended to switch the component to reactive mode — start storing input data in `state`, where changes trigger complete component re-rendering. From the DOM perspective, only changed areas are re-rendered — this is useful both for performance and animations (you can attach transitions to CSS properties that change during re-rendering).

However, this doesn't mean you'll need to rewrite the entire declaration. Let's examine an example with a news card:

```xml
<NewsCard>
    <item href="...">
        <thumb>...</thumb>
        <rubric>...</rubric>
        <title>...</title>
        <text>...</text>
        <date>...</date>
    </item>
    ...
</NewsCard>
```

The declaration would look like this until the last moment:

```js
Beast.decl({
    NewsCard: {
        expand: function () {
            this.append(
                this.get('item')
            )
        }
    },
    
    NewsCard__item: {
        tag: 'a',
        domAttr: {
            href: function () {
                return this.param('href')
            }
        },
        expand: function () {
            this.append(
                this.get('thumb', 'rubric', 'title', 'text', 'date')
            )
        }
    }
})
```

## Reactive Data Management

### State-Based Components

```js
Beast.decl({
    TodoList: {
        expand: function () {
            // Initialize state
            this.state = {
                items: this.param('items', []),
                filter: 'all' // all, active, completed
            }
            
            this.renderItems()
        },
        
        renderItems: function () {
            const filteredItems = this.getFilteredItems()
            
            this.empty().append(
                <header>
                    <input/>
                    <filters/>
                </header>,
                <list>
                    {filteredItems.map(item => 
                        <item 
                            id={item.id}
                            text={item.text}
                            completed={item.completed}
                        />
                    )}
                </list>,
                <footer>
                    <count>{this.getActiveCount()} items left</count>
                    <clearCompleted/>
                </footer>
            )
        },
        
        addItem: function (text) {
            const newItem = {
                id: Date.now(),
                text: text,
                completed: false
            }
            
            this.state.items.push(newItem)
            this.renderItems()
        },
        
        toggleItem: function (id) {
            const item = this.state.items.find(item => item.id === id)
            if (item) {
                item.completed = !item.completed
                this.renderItems()
            }
        },
        
        removeItem: function (id) {
            this.state.items = this.state.items.filter(item => item.id !== id)
            this.renderItems()
        },
        
        setFilter: function (filter) {
            this.state.filter = filter
            this.renderItems()
        },
        
        getFilteredItems: function () {
            switch (this.state.filter) {
                case 'active':
                    return this.state.items.filter(item => !item.completed)
                case 'completed':
                    return this.state.items.filter(item => item.completed)
                default:
                    return this.state.items
            }
        },
        
        getActiveCount: function () {
            return this.state.items.filter(item => !item.completed).length
        }
    }
})
```

### Optimized Re-rendering

For better performance, implement selective updates:

```js
Beast.decl({
    DataTable: {
        expand: function () {
            this.state = {
                data: this.param('data', []),
                sortColumn: null,
                sortDirection: 'asc'
            }
            
            this.renderTable()
        },
        
        renderTable: function () {
            // Only re-render if structure changed
            if (!this.elem('table').length) {
                this.append(
                    <table>
                        <thead/>
                        <tbody/>
                    </table>
                )
            }
            
            this.renderHeader()
            this.renderBody()
        },
        
        renderHeader: function () {
            const headers = this.getColumns()
            
            this.elem('thead').empty().append(
                <tr>
                    {headers.map(column => 
                        <th 
                            column={column.key}
                            sortable={column.sortable}
                        >
                            {column.title}
                        </th>
                    )}
                </tr>
            )
        },
        
        renderBody: function () {
            const sortedData = this.getSortedData()
            
            this.elem('tbody').empty().append(
                sortedData.map(row => 
                    <tr key={row.id}>
                        {this.getColumns().map(column => 
                            <td>{row[column.key]}</td>
                        )}
                    </tr>
                )
            )
        },
        
        sort: function (column) {
            if (this.state.sortColumn === column) {
                this.state.sortDirection = this.state.sortDirection === 'asc' ? 'desc' : 'asc'
            } else {
                this.state.sortColumn = column
                this.state.sortDirection = 'asc'
            }
            
            // Only re-render body, header stays the same
            this.renderBody()
        }
    }
})
```

### Data Binding Patterns

#### Two-Way Data Binding

```js
Beast.decl({
    FormField: {
        expand: function () {
            this.state = {
                value: this.param('value', ''),
                errors: []
            }
            
            this.append(
                <label>{this.param('label')}</label>,
                <input value={this.state.value}/>,
                <errors/>
            )
            
            this.bindInput()
        },
        
        bindInput: function () {
            this.elem('input').on('input', () => {
                this.state.value = this.elem('input').domNode().value
                this.validate()
            })
        },
        
        validate: function () {
            const value = this.state.value
            const errors = []
            
            if (this.param('required') && !value.trim()) {
                errors.push('This field is required')
            }
            
            if (this.param('minLength') && value.length < this.param('minLength')) {
                errors.push(`Minimum length is ${this.param('minLength')}`)
            }
            
            this.state.errors = errors
            this.renderErrors()
        },
        
        renderErrors: function () {
            this.elem('errors').empty()
            
            if (this.state.errors.length > 0) {
                this.elem('errors').append(
                    this.state.errors.map(error => 
                        <error>{error}</error>
                    )
                )
                this.mod('Valid', false)
            } else {
                this.mod('Valid', true)
            }
        },
        
        getValue: function () {
            return this.state.value
        },
        
        setValue: function (value) {
            this.state.value = value
            this.elem('input').domNode().value = value
            this.validate()
        }
    }
})
```

#### Observable State

```js
// Simple observable implementation
function createObservable(initialState) {
    const observers = []
    let state = initialState
    
    return {
        get() {
            return state
        },
        
        set(newState) {
            const prevState = state
            state = { ...state, ...newState }
            observers.forEach(observer => observer(state, prevState))
        },
        
        subscribe(observer) {
            observers.push(observer)
            return () => {
                const index = observers.indexOf(observer)
                if (index > -1) observers.splice(index, 1)
            }
        }
    }
}

Beast.decl({
    Counter: {
        expand: function () {
            this.observable = createObservable({
                count: this.param('initialCount', 0)
            })
            
            // Subscribe to state changes
            this.unsubscribe = this.observable.subscribe((state, prevState) => {
                this.render()
            })
            
            this.render()
        },
        
        render: function () {
            const state = this.observable.get()
            
            this.empty().append(
                <display>{state.count}</display>,
                <controls>
                    <button action="increment">+</button>
                    <button action="decrement">-</button>
                    <button action="reset">Reset</button>
                </controls>
            )
        },
        
        increment: function () {
            const state = this.observable.get()
            this.observable.set({ count: state.count + 1 })
        },
        
        decrement: function () {
            const state = this.observable.get()
            this.observable.set({ count: state.count - 1 })
        },
        
        reset: function () {
            this.observable.set({ count: 0 })
        },
        
        on: {
            click: function (event) {
                const action = event.target.getAttribute('action')
                if (action && this[action]) {
                    this[action]()
                }
            }
        },
        
        destruct: function () {
            if (this.unsubscribe) {
                this.unsubscribe()
            }
        }
    }
})
```

### Async Data Loading

```js
Beast.decl({
    UserProfile: {
        expand: function () {
            this.state = {
                loading: true,
                user: null,
                error: null
            }
            
            this.render()
            this.loadUser()
        },
        
        render: function () {
            this.empty()
            
            if (this.state.loading) {
                this.append(<loading>Loading user profile...</loading>)
                return
            }
            
            if (this.state.error) {
                this.append(
                    <error>
                        <message>{this.state.error}</message>
                        <retry>Try Again</retry>
                    </error>
                )
                return
            }
            
            if (this.state.user) {
                this.append(
                    <profile>
                        <avatar src={this.state.user.avatar}/>
                        <name>{this.state.user.name}</name>
                        <email>{this.state.user.email}</email>
                        <actions>
                            <edit>Edit Profile</edit>
                            <logout>Logout</logout>
                        </actions>
                    </profile>
                )
            }
        },
        
        loadUser: async function () {
            try {
                this.state.loading = true
                this.state.error = null
                this.render()
                
                const userId = this.param('userId')
                const response = await fetch(`/api/users/${userId}`)
                
                if (!response.ok) {
                    throw new Error('Failed to load user')
                }
                
                const user = await response.json()
                
                this.state.loading = false
                this.state.user = user
                this.render()
                
            } catch (error) {
                this.state.loading = false
                this.state.error = error.message
                this.render()
            }
        },
        
        on: {
            click: function (event) {
                if (event.target.matches('[retry]')) {
                    this.loadUser()
                }
            }
        }
    }
})
```

## Best Practices for Live Data

### 1. Minimize Re-renders

Only re-render when necessary and target specific parts of the component:

```js
// Good: Selective updates
updateTotal: function () {
    this.elem('total').text(this.calculateTotal())
}

// Avoid: Full re-render for small changes
updateTotal: function () {
    this.render() // Re-renders entire component
}
```

### 2. Batch State Updates

```js
// Good: Batch multiple changes
updateItems: function (newItems) {
    this.state = {
        ...this.state,
        items: newItems,
        loading: false,
        lastUpdated: Date.now()
    }
    this.render()
}

// Avoid: Multiple separate updates
updateItems: function (newItems) {
    this.state.items = newItems
    this.render()
    this.state.loading = false
    this.render()
    this.state.lastUpdated = Date.now()
    this.render()
}
```

### 3. Handle Loading and Error States

Always provide feedback for async operations:

```js
Beast.decl({
    DataComponent: {
        loadData: async function () {
            this.showLoading()
            
            try {
                const data = await this.fetchData()
                this.showData(data)
            } catch (error) {
                this.showError(error)
            }
        }
    }
})
```

This reactive approach ensures your components stay synchronized with changing data while maintaining good performance and user experience.