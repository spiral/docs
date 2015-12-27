# Dark Syntax
Spiral Stempler engine ships with XML/HTML syntax used to defined imports, extends and use tags. Following sections describes what
constructions available for you in your dark views.

## Extends declaration
You can declare that your view extends specific parent using following constructions:

```html
<extends:path.to.layout block="value"/>
```

In a following example parent path embedded into tag name, you can also use namespaces by simply separating them using ":" symbol:

```html
<extends:namespace:path.to.layout block="value"/>
```

In some cases you might not want to include parent layout path into tag name, dark syntax provides you following alternatives:

```html
<extends path="namespace:path/to/layout" block="value"/>
<extends dark:path="namespace:path/to/layout" block="value"/>
<extends layout="namespace:path/to/layout" block="value"/>
<extends dark:path="namespace:path/to/layout" block="value"/>
```

You can use other extends tags:

```html
<dark:extends path="namespace:path/to/layout" block="value"/>
<layout:extends path="namespace:path/to/layout" block="value"/>
```

> Attention, attributes "path" and "layout" are reserved for extends declaration and can not be used for block definition.

## Use declaration
Stempler and dark syntax provdes you ability to declare use of external view a form of virtual tag or widget, there is multiple ways 
to declare such use:

### Alias based
If you want to import external view under specific name use followin statement:

```html
<dark:use path="path/to/view" as="tag-name"/>
```

Following syntax alternatives available for you:

```html
<dark:use path="path/to/view" as="tag-name"/>
<use path="path/to/view" as="tag-name"/>
<node:use path="path/to/view" as="tag-name"/>
<stempler:use path="path/to/view" as="tag-name"/>
```

Pick one you like more.

### Prefixing directory
In some cases you migh want to include whole directory as set of virtual tags, in this case "as" attribute has to be replaced
with "prefix".

```html
<dark:use path="path/to/directory" prefix="prefix."/>
```

As result you can use views from this directory as:

```html
<prefix.relative.name/>
```

### Namespacing directory
Often your prefix might look like "prefix:", you can use shortcut for such definitions via "namespace" attribute:

```html
<dark:use path="path/to/directory" prefix="prefix:"/>
<dark:use path="path/to/directory" namespace="prefix"/>
```

> Both declaration are identical.

### Bundle import
To import file which contain other "use" declarations replace path with "bundle" attribute:

```html
<dark:use bundle="namespace:path/to/bundle-file"/>
```

## Block declaration
Dark Syntax provides multiple alternatives used to make your templates more expressive, every definition is idential to other:

```html
<block:name>block</block:name>
<define:name>block</define:name>
<section:name>block</section:name>
```

You can also use block definitions as placeholders:

```html
<block:name/>
<yield:name/>
```

> "yield" is preferred keyword when you using block as placeholder.

## Short Block declaration
In some cases where you can't use tag based block declaration you can use short block definitions in a form of "${name|default value}":

```html
<div class="${div-class|default-class}"></div>
```
