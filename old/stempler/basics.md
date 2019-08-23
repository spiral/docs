# Stempler Engine
The Spiral Stempler engine is mounted in your application by default and provides very simple ways to manage your website templates using html tags. Engine supports a very flexible mechanism to add multiple teams to projects and gives more power into the hands of your frontend/html developers.

> Stempler isn't intended to create or replace basic control structures in your templates. If you want to replace PHP as you primary syntax - you can create a custom view processor and define your own language (for example using Twig, Smarty, Blade, Latte etc). In addition it is better to use Twig for client facing templates (emails and etc) due it has safe mode.

This section of the documentation focuses mainly on the template inheritance in your projects. If you want to read about custom tags/widgets and the ability to include one template into another, please read [Extended Stempler usage](/old/stemplerpler/expert.md). Make sure to read about [using views](/old/viewsiews/overview.md) first.

> Stempler by default associated with files with extension ".dark.php".

## Template Inheritance
Before we start using the Stempler engine, let's imagine the common situation where we have two very similar pages on your website. For example an *about-us* page and a *contact-us* page with form. 

About Us:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
         Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

Contact form:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        Contact Us
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
        <form action="/contact/submit" method="POST">
            <input type="text" name="name">
            ...
        </form>
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

As you see, both templates use a very similar layout that are only different in the title and content on page. If we want to simplify the page creation, we can try to move the header and footer sections of our layouts into separate views and modify our code like this:

```php
<?= $this->container->views->render('header', ['title' => 'About Us']) ?>
    Something about us.
<?= $this->container->views->render('footer') ?>
```

However, this technique has some disadvantages. For example, you can leave a tag unclosed or your application could easily spend extra resources rendering both the footer and header every time your page loads. To prevent these problems and optimize performance, we can create one universal template (layout) for our pages and extend it in our views. Let's try that and place the view file into "app/views/layouts/basic.dark.php":

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
        <yield:content/>
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

You might notice that we kept all of our universal HTML code in a single file and defined two placeholders for the title and content blocks. Now, we can edit our about us template and utilize the abilities of the Stempler engine:

```html
<extends:layouts.basic title="About Us"/>

<block:content>
    Something about us. <a href="/some-url/">Some Address</a>.
</block:content>
```

> One of the nicer side effects of using Stempler is that any IDE or editor will treat its' tags as normal html/xml tags. As a result, your content will be nicely indented and any mistakes will be highlighted. 

The resulting HTML will look exactly the same as before. However, our view is now much smaller in size. Let's examine our template to understand what we just did. First of all, we declared that our view file must extend the parent template/layout "layouts.basic". Because we are not allowed to use the path separators in html tags the - "." will be replaced with "/", which allows us to extend our desired "layouts/basic" view (the current view namespace [default] will be used).

> Read about the [Dark Syntax](/old/stemplerpler/dark.md) which specifies how extend, use and other construction must be declared and what aliases you can use.

Next, in the extends tag, we declared a tag attribute "title". This definition tells the parent layout to replace the "title" block with the given value. These block definitions can be used for simple html strings and php code.

Then, we created an html (xml) tag named "block:content" (alias "default:content") for the same purpose as our title. Because we want to specify the content with inner html tags (in our case a) we can not use short, attribute like, definitions.

> Stempler tokenizes html code before processing it. This means that you can not use `<` and `>` symbols in any of your tag attribute as it create invalid html. We can always use longer blocks definition for example like "<block:content>". We can demonstrate that both definitions are identical by rewriting our template:

```html
<extends:layouts.basic/>

<block:title>About Us</block:title>

<block:content>
   Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>
```

> Attention, stempler does not know about changes in parent layouts (in current version), you should either recompile your views using the `views:compile` command or simply disable the cache in the development enviroment (see [configuring enviroment](/old/installation.md)).

## Parent block content
In our previous example, we demonstrate the ability to inject the desired values into our parent layout. However in many cases, we may want to rewrite specific parts of the parent layout without losing the original content. We can do this by defining a few new blocks in our parent layout:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

If we try to render our about us page now (do not forget to re-set your cache) you will notice there is no difference since there isn't any replacement for newly created blocks. Let's try to overwrite the value of layout footer:

```html
<extends:layouts.basic title="About Us"/>

<block:content>
     Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

This method works using some simple blocks. However, in more complex cases, it might appear useless. Let's try to imagine that we have to use an additional CSS style for the about us page. We can do that, by redefining block resources:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
   <link rel="stylesheet" href="/resources/styles/website.css"/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

Now we encounter a small problem. Every time we'd like to add new content into the parent block, we have to carry the content with us. This can lead to some problems when new resources are mounted to the parent layout. We can solve this issue by injecting the value of the parent block into it's re-definition:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

We can inject the parent block content into any place of re-definition before, after or even in a middle of the new content:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
   <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

> You can inject parent content only once!

This technique lets us manage the set of layouts on a global level with the ability to add/replace the needed assets on the page/template level.

> Attention, stempler complies view code before any of PHP code will be executed, as result you can use almost any control syntax you want, but you are not able to manage Stempler constructions like blocks, extends and etc (however, see below how to switch between layouts).

## Even shorter block/yield definition
There are many scenarios where you might want to inject a child value inside an HTML tag or even php code. Let's say that we want to be able to define a custom wrapper class for different views.

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper <block:wrapper-class>page</block:wrapper-class>">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

Unfortunately, this definition is **not valid** as it can not be easility tokenized. In this case, we can use a fallback Stempler construction that can be injected into almost any part of your template: `${name|default value}`. Let's use it:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper ${wrapper-class|page}">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

Now we have the ability to set a custom wrapper class using our templates:

```html
<extends:layouts.basic title="About Us" wrapper-class="about-us"/>

<block:resources>
   <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

As in examples with normal block definitions, we can always use parent block values. Let's try to add our class without removing the parent one using `${name|default}` like syntax:

```html
<extends:layouts.basic title="About Us" wrapper-class="${wrapper-class} about-us"/>

<block:resources>
   <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

## Multiple inheritance
The Stempler does not have any limitations (except memory and time at moment of compilation :)) on how many times the template can be extended and how deep you can go. As a result, we are able to create multiple website layouts. For example, we can create a new layout with navigation. To start, let's modify our basic layout to see what that does:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
<div class="header">
    <block:header>This is our header.</block:header>
</div>
<block:wrapper>
    <div class="wrapper ${wrapper-class|page}">
        <yield:content/>
    </div>
</block:wrapper>
<div class="footer">
    <block:footer>This is our footer.</block:footer>
</div>
</body>
</html>
```

Now we can create  new layout with navigation related to the html. We can call it "layouts/navigation.dark.php":

```html
<extends:layouts.basic/>

<block:wrapper>
    <div class="navigation">
        Some navigation.
    </div>
    <block:wrapper/>
</block:wrapper>
```

To apply the new layout to our about-us page, let's simply change it's extends tag:

```html
<extends:layouts.navigation title="About Us" wrapper-class="${wrapper-class} about-us"/>

<block:resources>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
    <block:resources/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is the about us specific footer.
</block:footer>
```

As a result, we managed to change the page layout using only one line of code.

> The Stempler fully supports view namespaces. Thus, we can define our extends tag in a form: `<extends:default:layouts.navigation/>` or `<dark:extends path="namespace:layouts/layout"/>` (see Dark Syntax for more details).

## Switching layouts
Even if you are not allowed to control the Stempler structures and inheritance using PHP code - you can achieve your goal of switching between different templates by using the view modifiers which are included in default spiral application. 

One of such modifiers provides us ability to have multiple cache versions based on values set in view environment (see views config):

```php
 'enviroment' => [
        'language' => ['translator', 'getLocale'],
        'basePath' => ['http', 'basePath'],
        'layout'   => [App::class, 'websiteLayout']
    ],
```

Inside our application we have to create the method `websiteLayout`:

```php
/**
 * Website layout to be used.
 * 
 * @return string
 */
public function websiteLayout()
{
    return 'desktop';
}
```

Now, we can use the alternative syntax to define the extends tag which give us the ability to provide the parent layout name in the attribute form:

```html
<dark:extends path="layouts/desktop"/>
```

If you have already read about the ViewManager, it's cache dependencies (evironment values), modifiers and processors, you might remember that you can inject the cache value dependency into html using `@{name}` syntax. This injection will happen before the Stempler engine will start processing our views and as result, we can rewrite our extends tag into such form:

```html
<dark:extends path="layouts/@{layout}"/>
```

The rendered template will look exactly the same as before. However we can now try to create a different layout with new markup and resources. Let's put this into "layouts/mobile.dark.php":

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/mobile.css"/>
    </block:resources>
</head>
<body>
<div class="mobile">
    <yield:content/>
</div>
</body>
</html>
```

Now, the only thing we have left to do to change the website/page layout is to provide a different value from our `Application->websiteLayout` method.

```php
/**
 * Website layout to be used.
 *
 * @return string
 */
public function websiteLayout()
{
    return 'mobile';
}
```

> An interesting side effect of using cache dependencies (view environment values) to switch layouts is that this switch will happen during the compilation stage. As a result, you will get a different template cache versions for different values of the `websiteLayout` method. We can always go into the "app/runtime/cache/views" folder and locate the two different files related to our page.

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</head>
<body>
<div class="header">
    This is our header.
</div>
    <div class="wrapper page about-us">
    Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
    </div>
<div class="footer">
    This is the about us specific footer.
</div>
</body>
</html>
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
        <link rel="stylesheet" href="/resources/styles/mobile.css"/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</head>
<body>
<div class="mobile">
    Something about us, <?= $someVariable ?>. <a href="/some-url/">Some Address</a>.
</div>
</body>
</html>
```

> You can move the `websiteLayout` method to any of the desired class including services and controllers.
