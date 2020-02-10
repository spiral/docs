# Stempler - Components and Props
Stempler provides the ability to create developer driven template components in a form of virtual tags.

## Simple Component
In many cases your templates will reuse not only parent layout but also template partials, for example:

```html
<extends:layout.base title="Homepage"/>

<block:content>
  This is homepage.

  <div class="article">
    <div class="title">Article title</div>
    <div class="preview">article preview</div>
  </div>

  <div class="article">
    <div class="title">Article title 2</div>
    <div class="preview">article preview 2</div>
  </div>

  <div class="article">
    <div class="title">Article title 3</div>
    <div class="preview">article preview 3</div>
  </div>
</block:content>
```

We can move the article div into separate template `app/views/partial/article.dark.php`:

```html
<div class="article">
  <div class="title">Article title</div>
  <div class="preview">article preview</div>
</div>
```

To use this partial on your page make sure to import it first using `<use:element path=""/>` control tag:

```html
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
  This is homepage.

  <article/>
  <article/>
  <article/>
</block:content>
```   

> Read more about mass-importing partials below.

### Props
It's is not very useful to create partials without the ability to configure their content. Use `block:name` or `${name|default}`
syntax (similar as described [here](/stempler/inheritance.md)) to define replaceable parts:


In our partial `app/views/partial/article.dark.php`:

```html
<div class="article">
  <div class="title">${title}</div>
  <div class="preview">
    <block:preview>
      default preview
    </block:preview>
  </div>
</div>
```

You can pass values similar way as in `extend` control tag:

```html
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
  This is homepage.

  <article>
    <block:title>Article 1 title</block:title>
    <block:preview>
      This is article 1 preview.
    </block:preview>
  </article>

  <article title="Article 2">
    <block:preview>
      <block:parent/>
      This is article 1 preview.
    </block:preview>
  </article>

  <article title="Article 3" preview="This is article 3 preview."/>
</block:content>
```

> You can include original block content using `block:parent` tag. Extending components is also allowed.

The resulted HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="default">
  This is homepage.
  <div class="article">
    <div class="title">Article 1 title</div>
    <div class="preview">
      This is article 1 preview.
    </div>
  </div>

  <div class="article">
    <div class="title">Article 2</div>
    <div class="preview">
      Default content.
      This is article 1 preview.
    </div>
  </div>

  <div class="article">
    <div class="title">Article 3</div>
    <div class="preview">
      This is article 3 preview.
    </div>
  </div>
</body>
</html>
```

> Components does not cause any performance penalty, use as many components as you need.

## Import Components
Stempler provides multiple options to import components into your template.

### Import Element
To import single component use `<use:element path=""/>` before component invocation.

```html
<extends:layout.base title="Homepage"/>
<use:element path="partial/article"/>

<block:content>
  <article title="Article" preview="This is article preview."/>
</block:content>
```

The component will be available using the filename, in this case it's `article`. To define custom import alias use
`as` tag attribute:

```html
<extends:layout.base title="Homepage"/>
<use:element path="partial/article" as="custom-article"/>

<block:content>
  <custom-article title="Article" preview="This is article preview."/>
</block:content>
```

### Import Directory
To import all partials from a given directory use `<use:dir dir="" ns=""/>`. You must specify the
namespace prefix to avoid collisions with other components and default HTML tags:

```html
<extends:layout.base title="Homepage"/>
<use:dir dir="partial" ns="partials"/>

<block:content>
  <partials:article title="Article" preview="This is article preview."/>
</block:content>
```

### Inline Import
To define component specific to the given template without the creation of physical view file use `<use:inline name=""></use:inline>`
control tag. In `app/views/home.dark.php`:

```html
<extends:layout.base title="Homepage"/>

<use:inline name="article">
  <div class="article">
    <div class="title">${title}</div>
    <div class="preview">${preview}</div>
  </div>
</use:inline>

<block:content>
  <article title="Article" preview="This is article preview."/>
</block:content>
```

### Bundle Import
Import multiple directories, components and/or inline components using bundled import via `<use:bundle path="">`.

Create view file `app/views/my-bundle.dark.php` to define your bundle:

```html
<use:element path="partial/article" as="article"/>

<use:inline name="article-alt">
  <div class="article">
    <div class="title">${title}</div>
    <div class="preview">${preview}</div>
  </div>
</use:inline>
```

You can use any of defined components in your `app/views/home.dark.php` template:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="my-bundle"/>

<block:content>
  <article title="Article" preview="This is article preview."/>
  <article-alt title="Article" preview="This is article preview."/>
</block:content>
```

To isolate imported bundle via prefix use `ns` attribute of `use:bundle` tag:

```html
<extends:layout.base title="Homepage"/>
<use:bundle path="my-bundle" ns="my"/>

<block:content>
  <my:article title="Article" preview="This is article preview."/>
  <my:article-alt title="Article" preview="This is article preview."/>
</block:content>
```