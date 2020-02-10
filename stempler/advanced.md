# Stempler - Advancing DSL
Combine Stempler features such as props, AST code injections, stacks and inheritance to develop feature rich domain 
language for page definitions. 

> Attention, make sure to learn all other aspects of Stempler before jumping into this section. You will also need good 
> cup of coffee.

## Prerequisites
The approach described in this article is possible due to the fact that stacks can be defined within imported components.

## Grid
Let's create grid component to describe how to use stacks in combination with stacks. The grid will consist of multiple view files.

### Grid Element
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

### Cell Element
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