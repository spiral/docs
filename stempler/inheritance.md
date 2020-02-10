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

### Nested Layouts
It is possible to create layouts based on other layouts, create `app/views/layout/page.dark.php`:

```html
<extends:layout.base body-class="page ${parent}"/>

<block:content>
  <div class="page-wrapper">
    <block:page/>
  </div>
</block:content>
```

> Extend tags always require full path specification, make sure to include `layout` directory.

You can extend this layout instead of `base` in `app/views/home.dark.php`:

```html
<extends:layout.page title="Homepage" body-class="homepage ${parent}"/>

<block:page>
  Page content.
</block:page>
```

The produced HTML:

```html

<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage page default">
  <div class="page-wrapper"> 
    Page content.
  </div>
</body>
</html>
```

> You can nest as many templates as you need, only the compilation speed will be affected.

## Stacks
Stempler includes the ability to aggregate multiple blocks defined within the template. 

### Classic Approach
Often you would need to add custom JS or CSS resource to your layout. To achieve it using `block` directives
wrap needed resources into block and append content to it in your child template.

Modify `app/views/layout/base.dark.php` as:

```html
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <block:styles>
      <link rel="stylesheet" href="/styles/welcome.css"/>
    </block:styles>
</head>
<body class="${body-class|default}">
    <block:content>
      Default content.
    </block:content>
</body>
</html>
``` 

To add custom style resource in your page template:

```html
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:styles>
  <block:parent/>
  <link rel="stylesheet" href="/styles/homepage.css"/>
</block:styles>

<block:page>
  Page content.
</block:page>
```

The produced HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
  <link rel="stylesheet" href="/styles/welcome.css"/>
  <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
  Default content.
</body>
</html>
```

### Create Stack
To demonstrate how the following can be achieved using stacks we should start with simple example in `app/home.dark.php`.
Create stack placeholder using `<stack:collect name="name"/>`: 

```html
<stack:collect name="my-stack">
  default content
</stack:collect>

```

To append value to stack:

```html
<stack:collect name="my-stack">
  default content
</stack:collect>

<stack:push name="my-stack">
  my value
</stack:push>
```

The resulted HTML:

```html
  default content
  my value
```

To prepend value to stack:

```html
<stack:collect name="my-stack">
  default content
</stack:collect>

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```

The output:

```html
  my value  
  default content
```

You can locate stack definition before or after `push` and `prepend` tags:

```html
<stack:prepend name="my-stack">
  my value
</stack:prepend>

<stack:collect name="my-stack">
  default content
</stack:collect>
```

### Deep Stacks
The stack tag will only aggregate `push` and `prepend` values while located in a same tag tree. 


For example given example will work:

```html
<stack:collect name="my-stack">
  default content
</stack:collect>

// stack my-stack is active here
<div>
  // and here
  <stack:prepend name="my-stack">
    my value
  </stack:prepend>
</div>
```

While this example won't work:

```html
<div>
  // stack my-stack is active here
  <stack:collect name="my-stack">
    default content
  </stack:collect>
  // and here
</div>

// stack my-stack is no active at this level

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```

> This limitation is caused by AST nature of stack collectors.

To bypass such limitation without moving the placeholder level above use `stack:collect` attribute `level`:

```html
<div>
  <stack:collect name="my-stack" level="1">
    default content
  </stack:collect>
</div>

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```

The attribute `level` configures stack to be active multiple levels above. For example given example **won't work**:

```html
<div>
  // stack my-stack is active here
  <div>
    // stack my-stack is active here
    <stack:collect name="my-stack" level="1">
      default content
    </stack:collect>
  </div>
</div>

// stack my-stack is no active at this level

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```

But example will work:

```html
<div>
  // stack my-stack is active here
  <div>
    // stack my-stack is active here
    <stack:collect name="my-stack" level="2">
      default content
    </stack:collect>
  </div>
</div>

// stack my-stack is active here

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```


### Stacks in Layouts
You can push values to stacks defined in parent layouts. Modify `app/views/layout/base.dark.php` accordingly:

```html
<!DOCTYPE html>
<html>
<head>
  <title>${title|Default title}</title>
  <stack:collect name="styles" level="2">
    <link rel="stylesheet" href="/styles/welcome.css"/>
  </stack:collect>
  <block:resources/>
</head>
<body class="${body-class|default}">
    <block:content>
      Default content.
    </block:content>
</body>
</html>
```

Now you can push the value from `app/views/home.dark.php`:

```html
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:resources>
  <stack:push name="styles">
    <link rel="stylesheet" href="/styles/homepage.css"/>
  </stack:push>
</block:resources>

<block:page>
  Page content.
</block:page>
```

> You have to make sure that `stack:push` is located in one of extended blocks. See below how to bypass it.

## Context and Hidden content
As you can see in previous example it's is not convenient to use both stack and blocks as the same time. This is caused
by the fact that stack collection happens after the extending of parent layout. Keeping stack outside of any `block` will
leave it out of the template.

All stempler blocks defined in child template outside of `block` tag will appear in system block `context`. We can modify
the parent layout `app/views/layout/base.dark.html` as the following:

```html
<!DOCTYPE html>
<html>
<head>
  <title>${title|Default title}</title>
  <stack:collect name="styles" level="2">
    <link rel="stylesheet" href="/styles/welcome.css"/>
  </stack:collect>
</head>
<body class="${body-class|default}">
    <block:content>
      Default content.
    </block:content>
</body>
<block:context/>
</html>
```

No we can define the stack in `app/views/home.dark.php` as following:

```html
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<stack:push name="styles">
  <link rel="stylesheet" href="/styles/homepage.css"/>
</stack:push>

some random string

<block:page>
  Page content.
</block:page>
```

To understand how context works take a look at generated HTML:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
  Default content.
</body>
some random string
</html>
```

Notice the `some random string` added at the place of `block:context`, this content has been declared by `app/views/home.dark.php`.
Most likely you will use the areas between block definitions of your templates for comments and other control directives,
to hide such content from end use use `<hidden></hidden>` tag in `app/views/layout/base.dark.php`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>${title|Default title}</title>
  <stack:collect name="styles" level="2">
    <link rel="stylesheet" href="/styles/welcome.css"/>
  </stack:collect>
</head>
<body class="${body-class|default}">
    <block:content>
      Default content.
    </block:content>
</body>
<hidden>
  <block:context/>
</hidden>
</html>
```

Now, stacking will work as before however the `some random string` won't appear on a page.

> Combine stacks with inheritance and [components](/stempler/components.md) to create domain specific rendering DSL.
