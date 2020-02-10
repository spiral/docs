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

> You can include original block content using `block:parent` tag. Extending partials is also allowed.

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

## Import partials