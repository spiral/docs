# Dark Syntax
Spiral Stempler engine ships with XML/HTML syntax used to defined imports, extends and use tags. Following sections describe what
constructions are available for you in your dark views.

## Extends declaration
You can declare that your view extends specific parent using following constructions:

```html
<extends:path.to.layout block="value"/>
```

To include namespace use additional ":" separator:

```html
<extends:namespace:path.to.layout block="value"/>
```

If you want to define layout as string use `path` or `dark:path` attributes:

```html
<extends path="namespace:path/to/layout" block="value"/>
<extends dark:path="namespace:path/to/layout" block="value"/>
<extends layout="namespace:path/to/layout" block="value"/>
<extends dark:path="namespace:path/to/layout" block="value"/>
```

Extends have multiple aliases for people who love freedom:

```html
<dark:extends path="namespace:path/to/layout" block="value"/>
<layout:extends path="namespace:path/to/layout" block="value"/>
```

> Attention, attributes "path" and "layout" are reserved for extends declaration and can not be used for block definition.

## Use declaration
Stempler and dark syntax provides you ability to declare use of external view a form of virtual tag/widget, there is multiple ways to declare such use:

### Alias based
If you want to import external view under specific name use:

```html
<dark:use path="path/to/view" as="tag-name"/>
```

Available alternatives:

```html
<dark:use path="path/to/view" as="tag-name"/>
<use path="path/to/view" as="tag-name"/>
<node:use path="path/to/view" as="tag-name"/>
<stempler:use path="path/to/view" as="tag-name"/>
```

### Importing directory of widgets
To import all views from a directory you must define prefix all imported views:

```html
<dark:use path="path/to/directory" prefix="prefix."/>
```

Use view from this directory via prefix:

```html
<prefix.relative.name/>
```

> You can include namespace into path.

Following declarations are shortcuts to define directory with xml like namespace:

```html
<dark:use path="path/to/directory" prefix="prefix:"/>
<dark:use path="path/to/directory" namespace="prefix"/>
```

```html
<prefix:relative.name/>
```

### Bundle import
To import file which contain other "use" declarations replace path with "bundle" attribute:

```html
<dark:use bundle="namespace:path/to/bundle-file"/>
```

## Block declaration
Dark Syntax provides multiple alternatives used to make your templates more expressive, every definition is identical to others:

```html
<block:name>block</block:name>
<define:name>block</define:name>
<section:name>block</section:name>
```

You can also use block definitions as placeholders when default content is empty:

```html
<block:name/>
<yield:name/>
```

> "yield" is preferred keyword when you use a block as placeholder.

## Short Block declaration
Some blocks can be defined using regexp like fashion "${name|default value}":

```html
<div class="${div-class|default-class}"></div>
```
