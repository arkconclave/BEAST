# Component Implementation

The flexibility of inheritance sometimes comes with a number of inconveniences. For example, let's say the `TaxiForm__select` element inherits from a `Select` control, which has its own elements `Select__option`, `Select__icon`, and `Select__button`. As a result, each element would need to be inherited additionally. Moreover, if the `select` block updates its structure, this would need to be accounted for in all descendants. The implementation mechanism eliminates such inconveniences.

__Implementation__ — replacing an element with a block while adding the element's behavior.

```xml
<TaxiForm>
    ...
    <select>
        <option>Economy</option>
        <option>Comfort</option>
        <option>Business</option>
    </select>
</TaxiForm>
```

```js
Beast.decl({
    TaxiForm__select: {
        expand: function () {
            this.implementWith(<Select>{this.get('option')}</Select>)
        }
    }
})
```

Resulting HTML:

```xml
<div class="taxiform">
    ...
    <select class="taxiform__select select">
        <option class="select__option">Economy</option>
        <option class="select__option">Comfort</option>
        <option class="select__option">Business</option>
    </select>
</div>
```

It's important to note that the elements of the `select` block didn't receive additional CSS classes — only the block itself received the `taxiForm__select` class, becoming a hybrid component. The hybrid nature allows treating the `select` as an element within the `taxiForm` context, while the component remains a full-fledged block.

But that's not all. Most likely, `taxiForm` would need to assign event handlers to the embedded block. This can be done from the parent block:

```js
Beast.decl({
    TaxiForm: {
        domInit: function () {
            this.elem('select')[0].on('Change', function () {...})
        }
    }
})
```

But at this point, the clear declarative approach to describing component behavior is violated. Therefore, the `implementWith` method, in addition to inserting a block in place of the element, also applies fields from the declaration of that very element to the block, except for `expand` for obvious reasons. This allows writing:

```js
Beast.decl({
    TaxiForm__select: {
        expand: function () {
            this.implementWith(<Select>{this.get('option')}</Select>)
        },
        on: {
            Change: function () {...}
        }
    }
})
```

Now the hybrid component `Select` is a full-fledged element with its own declaration.

__Hence a simple rule:__ if a child block suits everything and no additional changes are required in it, it's better to implement it rather than inherit from it.

## Implementation vs Inheritance

### When to Use Implementation

Implementation is preferred when:
- You need a complete existing block's functionality
- The block structure shouldn't change
- You only need to add element-specific behavior
- You want to avoid complex inheritance chains

```js
// Good: Using implementation for stable blocks
Beast.decl({
    UserProfile__avatar: {
        expand: function () {
            this.implementWith(
                <Image src={this.param('avatarUrl')} alt="User Avatar"/>
            )
        },
        on: {
            click: function () {
                this.parentBlock().trigger('avatarClick')
            }
        }
    }
})
```

### When to Use Inheritance

Inheritance is preferred when:
- You need to modify the block's structure
- You want to override multiple methods
- You're creating a family of related components
- The relationship is truly hierarchical

```js
// Good: Using inheritance for component variations
Beast.decl({
    Button: {
        expand: function () {
            this.append(<text>{this.param('text')}</text>)
        }
    },
    
    IconButton: {
        inherit: 'Button',
        expand: function () {
            this.append(
                <icon>{this.param('icon')}</icon>,
                <text>{this.param('text')}</text>
            )
        }
    }
})
```

## Advanced Implementation Patterns

### Conditional Implementation

```js
Beast.decl({
    Form__field: {
        expand: function () {
            const fieldType = this.param('type', 'text')
            
            switch (fieldType) {
                case 'select':
                    this.implementWith(
                        <Select options={this.param('options')}/>
                    )
                    break
                case 'textarea':
                    this.implementWith(
                        <Textarea rows={this.param('rows', 3)}/>
                    )
                    break
                default:
                    this.implementWith(
                        <Input type={fieldType}/>
                    )
            }
        }
    }
})
```

### Implementation with Data Transformation

```js
Beast.decl({
    ProductList__item: {
        expand: function () {
            const product = this.param('product')
            
            this.implementWith(
                <ProductCard 
                    name={product.name}
                    price={product.price}
                    image={product.thumbnail}
                    inStock={product.quantity > 0}
                />
            )
        },
        
        on: {
            addToCart: function () {
                const productId = this.param('product').id
                this.parentBlock().trigger('addToCart', productId)
            }
        }
    }
})
```

### Multiple Implementation Options

```js
Beast.decl({
    Dashboard__widget: {
        expand: function () {
            const widgetType = this.param('type')
            const widgetConfig = this.param('config', {})
            
            const widgets = {
                chart: () => <ChartWidget config={widgetConfig}/>,
                table: () => <TableWidget config={widgetConfig}/>,
                metric: () => <MetricWidget config={widgetConfig}/>,
                list: () => <ListWidget config={widgetConfig}/>
            }
            
            const createWidget = widgets[widgetType]
            if (createWidget) {
                this.implementWith(createWidget())
            } else {
                this.implementWith(<ErrorWidget message="Unknown widget type"/>)
            }
        }
    }
})
```

## Implementation Best Practices

### 1. Keep Implementation Logic Simple

```js
// Good: Simple, clear implementation
Beast.decl({
    Modal__content: {
        expand: function () {
            this.implementWith(
                <ScrollableContent>{this.children()}</ScrollableContent>
            )
        }
    }
})

// Avoid: Complex logic in implementation
Beast.decl({
    Modal__content: {
        expand: function () {
            // Too much logic here - consider moving to parent or using inheritance
            const content = this.children()
            const processedContent = this.processContent(content)
            const validatedContent = this.validateContent(processedContent)
            this.implementWith(<ScrollableContent>{validatedContent}</ScrollableContent>)
        }
    }
})
```

### 2. Preserve Element Semantics

```js
// Good: Element behavior preserved
Beast.decl({
    Toolbar__button: {
        expand: function () {
            this.implementWith(<Button>{this.children()}</Button>)
        },
        on: {
            click: function () {
                // Element-specific behavior
                this.parentBlock().trigger('toolbarAction', this.param('action'))
            }
        }
    }
})
```

### 3. Use Implementation for Stable APIs

```js
// Good: Implementing stable, well-tested components
Beast.decl({
    SearchForm__input: {
        expand: function () {
            this.implementWith(
                <TextInput 
                    placeholder="Search..."
                    autocomplete="off"
                />
            )
        }
    }
})
```

Implementation provides a powerful way to compose complex interfaces from reusable blocks while maintaining clean separation of concerns and avoiding the pitfalls of deep inheritance hierarchies.