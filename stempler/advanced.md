# Stempler - Advancing DSL
Combine Stempler features such as props, AST code injections, stacks and inheritance to develop feature rich domain 
language for page definitions. 

> Attention, make sure to learn all other aspects of Stempler before jumping into this section. You will also need good 
> cup of coffee.

The approach described in this article is possible due to the fact that stacks can be defined within imported components.

## Grid
Let's create grid component to describe how to use stacks in combination with stacks. The grid will consist of multiple view files.

### Table
Create root element for the component `app/views/grid/table.dark.php`:

```html
<table class="grid-table">
  <thead>
  <stack:collect name="thead" level="2"/>
  </thead>
  <tbody>
  <stack:collect name="tbody" level="2"/>
  </tbody>
  <hidden>${context}</hidden>
</table>
```

> Note the `<hidden>${context}</hidden>`, it allows the component to handle content declared between open and close tags
> without the need of `block` tags.

### Cell
Create element `app/views/grid/cell.dark.php` to declare single table cell with it's header and value:

```html
<stack:push name="thead">
  <tr>${title}</tr>
</stack:push>

<stack:push name="tbody">
  <td>${value}${context}</td>
</stack:push>
```

> Note the `${context}`.

### Bundle
We can pack our elements into bundle `app/views/grid/bundle.dark.php`:

```html
<use:element path="grid/table" as="grid:table"/>
<use:element path="grid/cell" as="grid:cell"/>
```

### Example
To render the grid in our template:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="grid/bundle"/>

<block:content>
  <grid:table>
    <grid:cell title="First cell" value="my value"/>
    <grid:cell title="Second cell" value="another value"/>
  </grid:table>
</block:content>
```

You can pass values into cell `context` directly:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="grid/bundle"/>

<block:content>
  <grid:table>
    <grid:cell title="First cell">my value</grid:cell>
    <grid:cell title="Second cell">second value</grid:cell>
  </grid:table>
</block:content>
```

In both cases the produced HTML:

```html
<body class="default">
  <table class="grid-table">
    <thead>
      <tr>First cell</tr>
      <tr>Second cell</tr>
    </thead>
    <tbody>
      <td>my value</td>
      <td>second value</td>
    </tbody>
  </table>
</body>
```

## UI Assembly
Another example is related to the ability to assemble complex UI interfaces using custom DSL.

### Base Layout
To demonstrate complex UI assembly we are going to create interface with the ability to easily push CSS, JS resources
and define context using multiple tabs instead of single content block. Create `app/views/tabs/layout.dark.php`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>${title|Default title}</title>
  <stack:collect name="styles" level="2"/>
</head>
<body>
  <div class="tab-headers">
    <stack:collect name="tab-headers" level="2"/>
  </div>
  <div class="tab-body">
    <stack:collect name="tab-body" level="2"/>
  </div>
</body>
<stack:collect name="scripts" level="2"/>
<hidden>${context}</hidden>
</html>
```

### Style and Script elements
To simplify registration of style and script elements create components `app/views/tabs/script.dark.php` and `app/views/tabs/style.dark.php`:

```html
<stack:push name="scripts">
  <script src="${src}"></script>
</stack:push>
```

Style:
```html
<stack:push name="styles">
  <link rel="stylesheet" href="${src}"/>
</stack:push>
```

### Tab Element
Create the tab element similar to `grid:cell` in `app/views/tabs/tab.dark.php`:

```html
<stack:push name="tab-headers">
  <div class="tab-head" data-tab-body="${id}">${title}</div>
</stack:push>

<stack:push name="tab-body">
  <div class="tab-body" id="body-${id}">${context}</div>
</stack:push>
```

### Bundle
Create bundle to represent the DSL for your UI framework `app/views/tabs/bundle.dark.php`:

```html
# resources
<use:element path="tabs/style" as="import:style"/>
<use:element path="tabs/script" as="import:script"/>

# tab
<use:element path="tabs/tab" as="ui:tab"/>
```

### Example
To render complex UI modify the `app/views/home.dark.php`. 

> You can import more than one bundle.

```html
<extends:tabs.layout title="Homepage"/>
<use:bundle path="grid/bundle"/>
<use:bundle path="tabs/bundle"/>

<import:style src="/resources/my-style.css"/>
<import:script src="/resources/my-script.js"/>

<ui:tab id="first" title="Information">
  Hello world!
</ui:tab>

<ui:tab id="second" title="Some Grid">
  <grid:table>
    <grid:cell title="First cell">my value</grid:cell>
    <grid:cell title="Second cell">second value</grid:cell>
  </grid:table>
</ui:tab>
```

The generated HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/resources/my-style.css"/>
</head>
<body>
  <div class="tab-headers">
    <div class="tab-head" data-tab-body="first">Information</div>
    <div class="tab-head" data-tab-body="second">Some Grid</div>
  </div>
    <div class="tab-body">
      <div class="tab-body" id="body-first">
        Hello world!
      </div>
    <div class="tab-body" id="body-second">
      <table class="grid-table">
        <thead>
          <tr>First cell</tr>
          <tr>Second cell</tr>
        </thead>
        <tbody>
          <td>my value</td>
          <td>second value</td>
        </tbody>
      </table>
    </div>
  </div>
</body>
<script src="/resources/my-script.js"></script>
</html>
```