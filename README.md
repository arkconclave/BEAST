# Beast

![Beast](https://github.com/kovchiy/beast/blob/master/images/cover.png)

Beast is a tool for creating web interfaces. It covers the entire development cycle from templating to component interactions. It is derived from several principles and methodologies including:

* [BEM methodology](https://en.bem.info/method/)
* [i-bem.js](https://en.bem.info/technology/i-bem/v2/i-bem-js/)
* [bh](https://github.com/bem/bh)
* [React](http://facebook.github.io/react/)

## Table of Contents

* [Beast](#beast)
* [BML-markup](#bmlmarkup)
* [Component](#component)
* [Declaration](#declaration)
  * [Expansion](#declaration-expand)
  * [Behavior](#declaration-dominit)
  * [Declaration inheritance](#declaration-inheritance)
  * [User methods](#declaration-usermethods)
* [Summary](#declaration-final)
* [Reference](#ref)
  * [BemNode](#ref-bemnode)
  * [Beast.decl()](#ref-decl)
  * [Beast methods](#ref-beast)
* [Hello, world](#helloworld)
* [Further development](#more)

---

## Core Concept

The main idea behind Beast is that an interface is divided into components, each characterized by a unique BEM selector, input data, rules for transforming them into a representation, and a description of behavior.

The foundation consists of three parts:
* **BML markup** of components (Beast Markup Language, XML subset)
* **Component methods**
* **Declaration of component expansion and behavior**

---

## BML-MARKUP

Any interface is a semantic hierarchy. Some components incorporate others, some depend on context, some don't care. For independent components, the term _block_ is introduced. Blocks can have auxiliary components called _elements_. Both can have changing attributes called _modifiers_ (states, behavior types, appearance themes).

XML is a classic way to describe hierarchies. Its convenience lies in allowing visual separation of entity content from its attributes. Content can be text, other entities, or a mix of both. Example of browser interface hierarchy:

```xml
<Browser>
    <head>
        <Tabs>
            <tab State="active">ARK</tab>
            <tab State="release">21e8</tab>
            <newtab/>
        </Tabs>
        <Addressbar>https://ark.studio</Addressbar>
    </head>
    <content>
        <Webview State="active" url="https://ark.studio"/>
        <Webview State="release" url="http://21e8.com"/>
    </content>
</Browser>
```

Different people may describe the same things differently, and each will be right in their own way. The same goes for identifying dependent and independent interface components. In this example, the independent, self-sufficient _blocks_ are:
* Browser
* Tabs
* Addressbar
* Webview

Names of independent components (blocks) start with uppercase letters to highlight the beginning of a new semantic chunk. Names of subordinate _elements_, which have no value without their parent, start with lowercase letters.

Elements are typically components that need to communicate frequently with similar components and depend on some common parameter (parent block modifier, for example). It's convenient for them to be nearby and form an isolated system of frequent message exchange.

## HTML Generation

To display the BML tree in a browser, a corresponding HTML tree is built:

```html
<div class="browser">
    <div class="browser__head">
        <div class="tabs">
            <div class="tabs__tab tabs__tab_state_active">ARK</div>
            <div class="tabs__tab tabs__tab_state_release">21e8</div>
            <div class="tabs__newtab"></div>
        </div>
        <div class="addressbar">https://ark.studio</div>
    </div>
    <div class="browser__content">
        <div class="webview webview_state_active"></div>
        <div class="webview webview_state_release"></div>
    </div>
</div>
```

All semantics from node names goes into HTML classes. This notation:
1. **Speeds up** browser style rendering by reducing selectors needed to identify components
2. **Prevents** selector collisions when nesting components
3. **Enables** each block or element with modifier to be described with a single CSS selector

```css
.webview {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
}
.webview_state_release {
    display: none;
}
.webview_state_active {
    display: block;
}
```

## JavaScript Compilation

BML markup is precompiled into JavaScript equivalent:

```js
Beast.node(
    'Browser',
    {'context':this},
    Beast.node(
        'head',
        null,
        Beast.node(
            'Tabs',
            null,
            Beast.node('tab', {'State':'active'}, 'ARK'),
            Beast.node('tab', {'State':'release'}, '21e8'),
            Beast.node('newtab', null)
        ),
        Beast.node('Addressbar', null, 'https://ark.studio')
    ),
    Beast.node(
        'content',
        null,
        Beast.node('Webview', {'State':'active', 'url':'https://ark.studio'}),
        Beast.node('Webview', {'State':'release', 'url':'http://21e8.com'})
    )
)
```

Since the final format is JavaScript, nothing prevents using its elements in BML:

```xml
<Button id="{Math.random()}">Save</Button>
```

## Including BML in HTML

BML markup is included in HTML files in two ways:
1. `link` tag with `type="bml"` and file address
2. `script` tag with `type="bml"` containing BML tree

```html
<html>
    <head>
        <script src="beast.js"></script>
        <link type="bml" href="browser.bml"/>
        <script type="bml">
            <Browser>...</Browser>
        </script>
    </head>
</html>
```

For brevity, `html` and `head` tags can be omitted. The recommended minimal structure:

```html
<meta charset="utf8">
<script src="beast.js"></script>
<link type="bml" href="browser.bml"/>

<script type="bml">
    <Browser>...</Browser>
</script>
```

---

## Component (BemNode)

In JavaScript, BML markup compiles into nested components — instances of the BemNode class.

```js
var button = <Button>Find</Button>
↓
var button = Beast.node('Button', null, 'Find')
↓
var button = new Beast.BemNode('Button', null, ['Find'])
```

BemNode methods can modify their instance: define content, behavior, parameters, etc.

```js
var button = <Button>Find</Button>
var text = button.text()

button
    .tag('button')
    .empty()
    .append(<label tag="span">{text}</label>)
    .render(document.body)
    .on('click', function () {
        console.log('clicked')
    })
```

Component work is done within declarations. The correspondence between component and declaration is established through a selector. Declarations are executed when the `render()` method is called.

---

## DECLARATION

Describes component expansion (expand) and behavior (domInit). It's a `Beast.decl()` method with two parameters: component selector (CSS class) and set of descriptions.

```js
Beast.decl('my-block', {
    expand: function () {},
    domInit: function () {}
})
```

### Expansion

Input data of a block shouldn't describe the resulting structure in maximum detail — only changeable parts should be specified. For example, in the `tabs` block:

```xml
<Tabs>
    <tab State="active">ARK</tab>
    <tab State="release">21e8</tab>
    <newtab/>
</Tabs>
```

In reality, each tab should have a close button, and since its presence is mandatory, there's no point specifying it every time. However, for the close button to appear in the resulting HTML tree, the final BML tree should be:

```xml
<Tabs>
    <tab State="active">
        <label>ARK</label>
        <close/>
    </tab>
    <tab State="release">
        <label>21e8</label>
        <close/>
    </tab>
    <newtab/>
</Tabs>
```

BML tree transformation is called _expansion_. Expansion rules are described in the declaration's `expand` field:

```js
Beast.decl('tabs__tab', {
    expand: function () {
        this.append(
            <label>{this.text()}</label>,
            <close/>
        )
    }
})
```

### Behavior

After expansion, DOM tree generation and initialization begins — most often attaching event handlers.

```js
Beast.decl('tabs__tab', {
    domInit: function () {
        this.on('click', function () {
            this.mod('state', 'active')
        })
    }
})
```

Event handling methods have their own field in declarations:

```js
Beast.decl('tabs__tab', {
    on: {
        click: function () {
            this.mod('state', 'active')
        }
    }
})
```

### Declaration Inheritance

Components can extend or refine others. For example, the general principle of tab operation — switching between each other — can be separated from appearance and other specifics into a separate `abstract-tabs` block:

```js
Beast
.decl('abstract-tabs__tab', {
    on: {
        click: function () {
            this.parentBlock().get('tab').forEach(function (tab) {
                tab.mod('state', 'release')
            })
            this.mod('state', 'active')
        }
    }
})

.decl('browser-tabs__tab', {
    inherits: 'abstract-tabs__tab',
    expand: function () {
        this.append(this.text(), <close/>)
    },
    domInit: function () {
        this.get('close').on('click', function () {
            this.remove()
        }.bind(this))
    }
})
```

### User Methods

In addition to the standard set of methods, declarations can specify additional ones:

```js
.decl('browser-tabs__tab', {
    expand: function () {
        this.append(this.text(), <close/>)
    },
    domInit: function () {
        this.get('close').on('click', function () {
            this.close()
        }.bind(this))
    },
    close: function () {
        jQuery(this.domNode()).fadeOut(100, function () {
            this.remove()
        }.bind(this))
    }
})
```

---

## Summary

* Interface is represented as a hierarchy of two types of components: independent blocks and dependent elements; the hierarchy is described by BML markup
* Markup compiles into a chain of nested `Beast.node()` methods, which forms the component tree
* Markup must have a single root component
* Component structure is transformed and supplemented with behavior through declarations
* Component structure repeats the corresponding DOM tree, where each component gets a DOM node
* DOM node stores a reference to its component in the `bemNode` property
* Components are instances of the BemNode class
* Declaration functions execute in the context of the corresponding component
* Component structure modification is done only through component methods

---

## API Reference

### BemNode Methods

#### Basic Properties
- `isBlock()` - Returns true if component is a block
- `isElem()` - Returns true if component is an element  
- `selector()` - Get component selector (`block` or `block__elem`)
- `parentBlock()` - Get or set parent block
- `parentNode()` - Get or set parent component
- `domNode()` - Get corresponding DOM element

#### Modifiers and Parameters
- `defineMod(defaults)` - Declare modifiers and default values
- `defineParam(defaults)` - Declare parameters and default values
- `mod(name, [value], [data])` - Get or set modifier
- `param(name, [value])` - Get or set parameter
- `domAttr(name, [value])` - Get or set DOM attribute
- `css(property, [value])` - Get or set CSS rule

#### Events
- `on(eventName, handler)` - Set DOM event handler
- `onWin(eventName, handler)` - React to window events
- `onMod(modName, modValue, handler)` - React to modifier changes
- `trigger(eventName, data)` - Trigger DOM event
- `triggerWin(eventName, data)` - Trigger window event

#### Content Manipulation
- `text()` - Get text content
- `empty()` - Remove all content
- `append(child...)` - Add content to end
- `appendTo(parent)` - Append to another component
- `get(path...)` - Get child components by path
- `has(path...)` - Check if child components exist
- `remove()` - Remove component
- `replaceWith(newNode)` - Replace with new content
- `implementWith(newNode)` - Replace with inheritance

#### Lifecycle
- `afterDomInit(callback)` - Execute after DOM initialization
- `render(domNode)` - Render component to DOM
- `renderHTML()` - Generate HTML string

### Beast.decl() Options

#### Declaration Fields
- `inherits` - Inherit from another declaration
- `expand` - Component expansion function
- `domInit` - DOM initialization function
- `mod` - Default modifiers
- `param` - Default parameters
- `tag` - DOM tag name
- `mix` - Additional CSS classes
- `domAttr` - Default DOM attributes
- `on` - Event handlers
- `onWin` - Window event handlers
- `onMod` - Modifier change handlers

### Beast Utility Methods

- `Beast.node(name, attrs, children...)` - Create component
- `Beast.findNodes(selector...)` - Find components by selector
- `Beast.findNodeById(id)` - Find component by ID

---

<a name="helloworld"/>

## Hello, World

To run a basic project, include the library file, JS component declarations, CSS styles, and create a semantic interface tree:

```xml
<html>
    <head>
        <meta charset="utf8">

        <!-- Tool -->
        <script src="beast-min.js"></script>

        <!-- Declarations -->
        <link type="bml" href="browser.bml"/>

        <!-- Styles -->
        <link type="text/css" rel="stylesheet" href="browser.css">

        <!-- Interface tree -->
        <script type="bml">
            <Browser>
                <head>
                    <Tabs>
                        <tab State="active">ARK</tab>
                        <tab State="release">21e8</tab>
                        <newtab/>
                    </Tabs>
                    <Addressbar>https://ark.studio</Addressbar>
                </head>
                <content>
                    <Webview State="active" url="https://ark.studio"/>
                    <Webview State="release" url="http://21e8.com"/>
                </content>
            </Browser>
        </script>
    </head>
</html>
```

The `html` and `head` tags are optional. Remember the `type="bml"` attribute for `script` and `link` tags; in the first case it tells Beast that code pre-compilation is needed, and in the second case it also loads the file itself. Browser security policies don't allow access to content loaded via `<script src="...">` — so the universal `<link>` tag is used.

<a name="more"/>

## Further Development

For practical advice, mechanisms for organizing complex component relationships, accepted code style, and project development from scratch, see [Beast Practice](docs/beast-practice/README.md).

Welcome home, good hunter.
