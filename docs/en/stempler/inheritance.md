# Stempler - Inheritance and Stacks
As your views get more complex, it's crucial to separate pages and layout specific content between templates properly.
Stempler provides several control statements to achieve this.

## Extend Layout
Firts, let's create a standard HTML template for our page (`app/views/home.dark.php`):

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

Most likely, your application will contain more than a one-page template. To avoid code duplication, Stempler
provides an ability to inherit the parent layout.

> Stempler will compile the template and parent layout into an optimized PHP code. You can exclude as many layouts as you want
> without a performance penalty.

Create a layot in `app/views/layout/base.dark.php`:

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

> Use the separator `.` to include the directory name into your template.

Alternatively, use the syntax:

```html
<extends path="layout/base"/>
```

> You can use view namespaces in such a declaration, for example: `<extends path="default:layout/base"/>`.

### Replace Blocks
Extending the parent layout does not make much sense unless we can redefine some of its content. To define a
replaceable block, use the tag `<block:name>`. Change the `layout/base.dark.php`
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

> You can include the default block content inside the `<block:name></block:name>` tag pair.

To redefine the block values, use `block:name` or similar tags in the `home.dark.php` template:

```html
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:content>
  This is homepage content.
</block:content>
```

### Short Syntax
In cases when your block define a short string or operates as a tag argument, use the alternative syntaxt `${name|default}`. Change the layout to:

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

You can pass some block values using the `extends` tag attributes to avoid large child templates, change `app/views/home.dark.php` accordingly:

```html
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
  This is homepage content.
</block:content>
```

In both cases, the produced HTML will look like this:

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
To leave the parent block content, use `<block:parent/>` in any place of the redefined block:

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

Use `${parent}`, to achieve the same goal in short block definitions:

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

> Extend tags always require full path specification, make sure to include the `layout` directory.

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

> You can nest as many templates as you need, it will only affect the compilation speed.

## Stacks
Stempler includes the ability to aggregate multiple blocks defined within the template. 

### Classic Approach
You would often need to add a custom JS or CSS resource to your layout. To achieve it, use the `block` directives,
wrap the necessary resources in a block and append content to it in your child template.

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

To add a custom style resource in your page template:

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
To demonstrate how the following can be achieved using stacks, we should start with a simple example in `app/home.dark.php`.
Create a stack placeholder using `<stack:collect name="name"/>`: 

```html
<stack:collect name="my-stack">
  default content
</stack:collect>

```

To append a value to stack:

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

To prepend a value to stack:

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

You can locate stack definition before or after the `push` and `prepend` tags:

```html
<stack:prepend name="my-stack">
  my value
</stack:prepend>

<stack:collect name="my-stack">
  default content
</stack:collect>
```

### Deep Stacks
The stack tag will only aggregate `push` and `prepend` values if it's located on the same tag tree level. 


For example, this will work:

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

// stack my-stack is not active at this level

<stack:prepend name="my-stack">
  my value
</stack:prepend>
```

> This limitation is caused by the AST nature of stack collectors.

To bypass this limitation without moving the placeholder level higher, use the`stack:collect` attribute `level`:

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

The attribute `level` configures the stack to be multiple active levels higher. For example, this example **won't work**:

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

But this one will:

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

> You have to make sure that `stack:push` is located in one of the extended blocks. See  how to bypass it below.

## Context and Hidden content
As you can see in the previous example, it's not convenient to use both the stack and blocks at the same time. This is because that stack collection happens after the extension of the parent layout. Keeping the stack outside of any `block` will
leave it out of the template.

All the stempler blocks that are defined in the child template outside of the `block` tag will appear in the system block `context`. We can modify the parent layout `app/views/layout/base.dark.html` like this:

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

Now we can define the stack in `app/views/home.dark.php` like this:

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

To understand how context works, take a look at the generated HTML:

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

Notice that `some random string` is added instead of `block:context`, this content was declared by `app/views/home.dark.php`.
You will most likely use the areas between block definitions of your templates for comments and other control directives.
To hide such content from end use, use the `<hidden></hidden>` tag in `app/views/layout/base.dark.php`:

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

Now, stacking will work as before. However, `some random string` won't appear on a page.

> Combine stacks with inheritance and [components](/stempler/components.md) to create domain specific rendering DSL.
