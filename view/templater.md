# Template Processor
Sometimes (especially if you are bulding complex websites) a view component by itself can't satisfy your needs. In this case you can use embedded view composer.

The Template Processor provides the ability to combine, extend, import and overwrite views and view blocks. Like other processors, the templater executes at the time of view compilation, which means the results are cached and cannot be changed at runtime (however you can use static variables to alter template).

> Template Processor does not provides any *logical* functionality besides composing view files together. Use PHP for that purposes.

## Principals
Spiral Templater build on top of the HTML parser, this means you don't need to learn any new templating syntax and any IDE or editor will highlight and format your view files. At the same time, you are not able to use templater inside your php blocks, let's view some examples.

## Inheritance
The most common task in view creation is building pages with common layout. To do that, let's define layout first:
```html
<html>
<head>
    <title>${title|Untitled}</title>
    <block:head>
        <meta name="author" value="Spiral"/>
    </block:head>
</head>
<body>
    <block:body>
        Default page body!
    </block:body>
</body>
</html>
```
As you can notice you defined multiple tags with `block:` and one `${title|Untitled}`, tags like that can be extended by child view files. The only difference between `${}` and `<block:...>` block definitions is that the first can be validly used *inside* php code or as a tag attribute (`<div class="${class}">`).
We are going to save this file in `application/views/layouts/basic.php` file (which is identical to the view name `layouts/basic`).
> Attention, default value for blocks like `${name|default}` is always simple string which will be trimmed by spaces. You can put text, html, php, js or any other value as default. 

Now, when you would like to create your page view we can extend this layout and replace some of its blocks.
```html
<extends:layout.basic title="My Page <?=mt_rand(0, 100)?>"/>
<block:body>
    This is page body.
</block:body>
```
In this view file we redefined parent blocks two different ways, first - by using
`<block:name>content</block:name>` construction, this is longer path but allows us to create really big replacements. In cases where parent block are simple string (usually titles, classes and etc), we can use simplified syntax by decaring replacement as attribute of `extends` opening tag.
> You can't use `/` in HTML or XML tag names, to separate folder names use `.` instead.

Result of this exetending will look like:
```html
<html>
<head>
    <title>My Page <?=mt_rand(0, 100)?></title>
    <meta name="author" value="Spiral"/>
</head>
<body>
    This is page body.
</body>
</html>
```
As you can see php code was successfully passed to parent block.

In many cases your website layout is not always the same and can vary from page to page. Let's say that you have account with navigation, breadcrumps and content. You can add this pieces to every view of your accounts, however we can define new layout to be extended.
```html
<extends:layouts.basic/>
<block:title>${pagename} - Account</block:title>
<block:body>
    <div class="navigation">
        <ul>
            <li>...</li>
            <li>...</li>
        </ul>
    </div>
    <div class="breadcrumps">Account / ${pagename}</div>
    <div class="content">
        <block:content/>
    </div>
</block:body>
```
Now we can save this view under `application/views/layouts/account.php` and extend it by some of our account pages:
```html
<extends:layout.account pagename="Settings"/>
<block:content>
    Some settings...
</block:content>
```
> There is no real limitiation of how nested your extending can be, however more parents will cause longer view compilation.


## Parent block content
In some cases we may need to keep original block content in some form. To do that use block name inside youre declaration:
```html
<block:head>
    <block:head/>
    <meta name="keywords" value="keyword"/>
</block:head>
```
If we will add this statement to our view file output will look like:
```html
<html>
<head>
    <title>My Page <?=mt_rand(0, 100)?></title>
    <meta name="author" value="Spiral"/>
    <meta name="keywords" value="keyword"/>
</head>
<body>
    This is page body.
</body>
</html>
```
Are you can see parent value `<meta name="author" value="Spiral"/>` are still there.
You have no limitation on where to put parent block, it can be at the end of definition:
```html
<block:head>
    <meta name="keywords" value="keyword"/>
    <block:head/>
</block:head>
```
Or even in a middle:
```html
<block:head>
    <meta name="keywords" value="keyword"/>
    <block:head/>
    <meta name="description" value="Description"/>
</block:head>
```

## Imports and "Virtual Tags"

### Import Directory

### Import Single View

### Inline extending

### Magick Blocks (context and attributes)

## Combining Templater with Evaluator and Runtime PHP

## Spiral Toolkit






