# Stempler - Inheritance and Stacks
As your templates become bigger it's crucial to properly separate page and layout specific content between templates.
Stempler provides a number of control statements to achieve such goal.

## Extend Layout
To start let's create standard HTML template for our page (`app/views/home.dark.php`):

```html
<!DOCTYPE html>
<html>
<head>
  <title>This is homepage.</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body>
  Page content
</body>
</html>
```

Most likely, your application will contain more than one page template. In order to avoid code duplication Stempler
provides the ability to inherit parent layout.

> Stempler will compile tamplate and parent layout into optimized PHP code, you can exclude as many layouts as you want
> without performance penalty.

Create layour in `app/views/layout/base.dark.php`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>This is homepage.</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body>
  Page content
</body>
</html>
```

Now, we can extend this layout using the `home.dark.php` via `extends:path` tag:

```html
<extends:layout.base/>
```

> Use `.` separator to include directory name into your template.

Alternatively use the syntax:

```html
<extends path="layout/base"/>
```

> You can use view namespaces in such declaration, for example: `<extends path="default:layout/base"/>`.

### Replace Blocks
The extending of the parent layout does not make much sense unless we can redefine some of it's content. To define
replaceable block use tag `<block:name>`. Change the `layout/base.dark.php`
accordingly:

```html
<!DOCTYPE html>
<html>
<head>
  <title><block:title/></title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body>
  <block:content>
    Default content body.
  </block:content>
</body>
</html>
```

> You can include the default block content inside `<block:name></block:name>` tag pair.

To redefine the block values use `block:name` or similar tags in `home.dark.php` template:

```html
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:content>
  This is homepage content.
</block:content>
```

### Short Syntax
In cases when your block define short string or operates as tag argument use alternative syntaxt `${name|defalt}`. Change the
layout to:

```html
<!DOCTYPE html>
<html>
<head>
  <title>${title|Default title}</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="${body-class|default}">
  <block:content>
    Default content.
  </block:content>
</body>
</html>
```

Short syntax values can be supplied to the parent layout via `<block:name>value</block:name>` tags.

```html
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:body-class>homepage</block:body-class>

<block:content>
  This is homepage content.
</block:content>
```

You can pass some block values using `extends` tag attributes to avoid large child templates, change `app/views/home.dark.php` accordingly:

```html
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
  This is homepage content.
</block:content>
```

In both cases the produced HTML will look like:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
  This is homepage content.
</body>
</html>
```

### Invoke Parent Content
To leave the parent block content use `<block:parent/>` in any place of redefined block:

```html
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
  This is homepage content.
  <block:parent/>
</block:content>
```

The produced HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
  This is homepage content.
  Default content.
</body>
</html>
```

Use `${parent}` to achieve the same goal in short block definitions:

```html
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:content>
  This is homepage content.
  <block:parent/>
</block:content>
```

The output:

```html

<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage default">
  This is homepage content.
  Default content.
</body>
</html>
```