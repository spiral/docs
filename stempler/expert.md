# Extended Templater Usage
As you might expect based on the title, the basic templater features (like template inheritance) are only one part of the templater (honestly, a very small part of it). In additon to generic features covered in [basic templater tutorial] (basics.md), you can create and import so called virtual tags or widgets. These widgets are created as normal views and can be located inside your application or provided by a [module] (/components/modules.md).

## Widget/tag creation 
If you remember, the information provided by the [Basic Templater Usage] (basic.md) templater handles situation with extending the parent layouts and preventing in-view like:

```php
Some content...
<?= $this->container->views->render('another-view', [...]) ?>
```

This simplifies the view definition when we are talking about footers, headers and navigations. However there are many situations where including another view is more beneficial. For example, we can create a unviversal view to render some newsfeed or display a comment among other things. Let's start with a very simple example where we have a view dedicated to render a newsline in many different areas of your website. Let's try to locate this view in "application/views/elements/feed.php".

```html
<div class="news">
    This is news feed...
    <div class="content">
        <div class="element">
            <?= \Spiral\Support\StringHelper::random(32) ?>
        </div>
    </div>
</div>
```

We now can create a demo view and render the newsfeed inside it (we are going to use the layouts created in a previous section).

```html
<extends:layouts.desktop title="Demo View"/>

<block:content>
    This is our content...
    <br/>
    <?= $this->container->views->render('elements/feed') ?>
</block:content>
```

Such view will generate cached output like:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        Demo View
    </title>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
<div class="header">
    This is our header.
</div>
    <div class="wrapper page">
    This is our content...
    <br/>
    <?= $this->container->views->render('elements/feed') ?>
    </div>
<div class="footer">
    This is our footer.
</div>
</body>
</html>
```

As you can see, Spiral will render the newsfeed every time someone wants to dispalay the "demo" view. Fortunately, the Templater provides a way to include the content of an external view file during the compilation stage and avoids unnesessary rendering calls. Let's jump to the next section to see how we can do this.

> Including the view source using the `views->render()` method has it's own pros, such as isolation and the separation of cache.

## Imporing widged/tag into view
If we want to include the desired view into our code, first we have to declare the use tag to link the view location (we might include namespace) and the virtual tag we want to use to represent that view. Our content can be simplified like this:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements/feed" as="elements:feed"/>

<block:content>
    This is our content...
    <br/>
    <elements:feed/>
</block:content>
```

This time the feed source will be included to cache without any additional `render()` method calls. This technique in the spiral Templater is called virtual-tags (since you can represent your import as a tag) or widgets. Let's check another section of this tutorial to find other smart ways to import and define your widgets.

> You can define the "use" tag(s) in the parent view (in our case "layouts.desktop"). All child views will inherit this use (with the ability to redefine it at any moment). 

## Widget attributes
First, we can realize that switching from the render method to templater created a problem linked to the inability to provide variables and options into our importer view. We can solve this problem by defining a set of blocks in our newsfeed exactly the same way we do it in the parent layouts. Below we would like to define the feed class.

```html
<div class="news ${class}">
    This is news feed...
    <div class="content">
        <div class="element">
            <?= \Spiral\Support\StringHelper::random(${previewSize | 32}) ?>
        </div>
    </div>
</div>
```

Now we can import this widget with the additional parameters similar to how we extend the parent layout:

```html
<elements:feed class="my-class" previewSize="16"/>
```

Be careful that you don't pass the php variables into the templater blocks without making any additional manipulations and verifications .See below.
> You can use exacly the same pricinciples of block definition and block value passing that is described in the Basic Templater Usage section.

## Context block
Let's make our example more complex. So far we were able to import the existing view and pass some compilation parameters into it. However, these parameters do not allow us to pass a custom HTML code into our element. Let's say that we want to have a special element (commonly used) to format some block of text. We can call it "elements/formatter":

```html
<div class="${class}" style="color: ${color|red}; text-transform: capitalize">${context}</div>
```

We can use and import this element into our demo view the same way we did for the newsfeed. However, in this case, we are going to use the long tag defintion with opening and closing parts:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements/formatter" as="elements:formatter"/>

<block:content>
    This is our content...
    <br/>
    <elements:formatter color="blue">
        Some text we want formatted...
    </elements:formatter>
</block:content>
```

The templater will pass the source declared between the opening and closing widget tags as **context** block. This feature can be very useful in a few different cases. You can freely use other imports inside this use case.

## Dynamic attributes (hard)
One of most useful part of imports is that it can help us mocking up existing elements. Let's say that our website has a universal form element definition which includes some input, input label and some wrapper classes. If we don't want to write each of these wrappers every time, we can locate our input code inside our widget. Let's try this (done by an non-html expert):

```html
<div class="form-input">
    <label class="input-wrapper">
        <span class="item-label">${label}</span>
        <input type="${type|text}" name="${name}" value="${value}"/>
    </label>
</div>
```

Now we can use this input tag exactly the same way as normal inputs, except we don't need to write all the wrappers:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements/input" as="form:input"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <form:input name="name" label="Enter Name:" value="<?= "some-php-value" ?>"/>
        <form:input name="password" type="password" label="Enter Password:"/>
    </form>
</block:content>
```

In following situation we have a tiny problem. We aren't allowed to add any more input attributes without specifically listing them in our element. For example, if we wish to add an inline style for our input - we have to previously modify our widget. This problem can be solved using attribute exporters:

```html
<div class="form-input">
    <label class="input-wrapper">
        <span class="item-label">${label}</span>
        <input type="${type|text}" name="${name}" value="${value}" node:attributes/>
    </label>
</div>
```

Now every additional attribute provided into our virtual tag will be mounted in place of node:attributes:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements/input" as="form:input"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <form:input name="name" label="Enter Name:" value="<?= "some-php-value" ?>"/>
        <form:input name="password" type="password" label="Enter Password:" id="myid" style="color: red"/>
    </form>
</block:content>
```

We can use "node:attributes" in multiple places of our widget and separate their content using additional parameters.

```html
<div class="form-input">
    <label class="input-wrapper" node:attributes="prefix:label">
        <span class="item-label">${label}</span>
        <input type="${type|text}" name="${name}" value="${value}" node:attributes="exclude:label-*"/>
    </label>
</div>
```

Now we have created the ability to pass attributes to our widget using prefix "label-". These attributes will appear in our label div.

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements/input" as="form:input"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <form:input name="name" label="Enter Name:" value="<?= "some-php-value" ?>"/>
        <form:input name="password" type="password" label="Enter Password:" id="myid" style="color: red" label-style="color: blue"/>
    </form>
</block:content>
```

You can use other parameters for node:attribues. For example `node:attributes="include:id,name;exclude:placeholder"` will fetch only id and name. The placeholder value will not be included into such export even if it was provided (we can add in some other place using `node:attributes="include:placeholder"`).

## Importing multiple virtual tags at once
In many cases, you might want to include multiple widgets under the same name or namespaces rather that doing it for each element. Spiral provides a few alternative syntaxes to do that.

### Create wigdet namespace
We already imported our element under the name "form:input". However, Spiral lets us import every folder element under a desired namespace:

```html
<use path="elements" namespace="form"/>
```

You can also import a set of elements from a desired namespace:

```html
<use path="default:elements" namespace="form"/>
```

### Disable import
If for some reason you would like to disable element import we can declare a "stop" use.

```html
<use stop="form:input"/>
```

Now, every `form:input` tag will be preserved as it is.

### Bundle import
There are a few scenarious where you may want to create many different elements (for example in separate module). If your module declares multiple element namespaces, importing such namespaces one by one isn't very optimal. Spiral provides you the ability to move the desired imports into a separate view file called bundle. Let's create a file in "elements/bundle" view:

```html
<use path="default:elements" namespace="form"/>
<use path="default:elements/input" as="form.input"/>
```

You can now combine multiple use tags in one file. This file won't be included in your template, so you can even leave comments for your imports:

```html
<!--This is default namespace for form elements.-->
<use path="default:elements" namespace="form"/>

<!--For some reason we want input element be available as 'form.input' tag.-->
<use path="default:elements/input" as="form.input"/>
```

To import your bundle into your view or layout, simply write:

```html
<use bundle="elements/bundle"/>
```

Now all the elements will be automatically imported based on the desired namespaces.

> Don't forget that you can declare use tags in parent view layout. They will inherit all parent uses.

## Overwriting default html tags (danger!)
Information in this section is for **educational purposes only**. Do not ovewrite the default HTML tags as this might cause any set of unpredicable scenarios in the future.

As you already seen, we can create your widgets using dynamic attributes and context block to make them behave as normal HTML tags. As a result, combining this knowledge with an alias based import, you can overwrite an existing HTML tag (but you should't). Let's try to create our tag to replace "a" element ("elements/a.php" view).

```html
<a class="custom-class ${class}" node:attributes>${context}</a>
```

Now we can use this tag in our view:

```html
<use path="elements/a" as="a"/>

Some simple text. <a href="/some-url">link</a>.
```

As result, the default link will be replaced with our widget and forced class value.

> The only real use for this is if you are creating views for your emails and you have to embedd styles inline. But remember: with great power comes great responsibility.

## Inheritance
You can freely inherit one widget from another much the same way as you can inherit layouts. Let's try to modify existing input widget and create a new widget for a text area based on it:

```html
<div class="form-input">
    <label class="input-wrapper" node:attributes="prefix:label">
        <span class="item-label">${label}</span>
        <block:input>
            <input type="${type|text}" name="${name}" value="${value}" node:attributes="exclude:label-*"/>
        </block:input>
    </label>
</div>
```

Now we can create a view "elements/textarea" and inherit our input:

```html
<extends:elements.input/>

<block:input>
    <textarea node:attributes="exclude:label-*">${value}${context}</textarea>
</block:input>
```

> Please note I removed the name attribute since it will be added automatically using the `node:attributes`.

Now, the only thing we have to do to is convert our from input from input to textarea by changing it's name:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements" namespace="form"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <form:textarea name="name" label="Enter Name:" value="<?= "some-php-value" ?>"/>
        <form:input name="password" type="password" label="Enter Password:" value="test"/>
    </form>
</block:content>
```

This method can be very convenient when you would like to create a set of similar elements for your project.

## Add intellect to your widget (hard)
In many cases, we want to give the widgets more flexibility. This includes not only the ability to replace it's blocks. We can do this by combining our templater code with simple php. Let's say that we don't want to render  the label span if no label value is specified. The simplified way to do this is:

```html
<div class="form-input">
    <label class="input-wrapper" node:attributes="prefix:label">
        <?php
        if (!empty("${label}")) {
            ?><span class="item-label">${label}</span><?php
        }
        ?>
        <block:input>
            <input type="${type|text}" node:attributes="exclude:label-*"/>
        </block:input>
    </label>
</div>
```

You can see that no span is rendered if no label attribute is provided to your widget. However, this code is not really reliable as it will fail if the label includes " symbols or php code. Fortunately, we can always use buffering:

```html
<div class="form-input">
    <label class="input-wrapper" node:attributes="prefix:label">
        <?php
        ob_start(); ?>${label}<?php
        if (!empty(ob_get_clean())) {
            ?><span class="item-label">${label}</span><?php
        }
        ?>
        <block:input>
            <input type="${type|text}" node:attributes="exclude:label-*"/>
        </block:input>
    </label>
</div>
```

In this case, our code can handle any value provided for the label attribute.

## Using EvalutatorProcessor (hard+)
If you remember from the [views and view processors] (/components/views.md) section, you can use the special view processor - `EvaluateProcessor`. This processor can execute PHP blocks marked with a `#compile` comment during the view compilation. This means that no code is preserved in the view cache (which will speed it up).

Since our label related code does not depend on any user argument, we can try to mark it's code as compilable (we have to include a comment into every php block):

```html
<div class="form-input">
    <label class="input-wrapper" node:attributes="prefix:label">
        <?php #compile
        ob_start(); ?>${label}<?php #compile
        if (!empty(ob_get_clean())) {
            ?><span class="item-label">${label}</span><?php #compile
        }
        ?>
        <block:input>
            <input type="${type|text}" node:attributes="exclude:label-*"/>
        </block:input>
    </label>
</div>
```

Now your widget will decide whether it will render the label span or not during compilation and not during runtime. You can also check your view cache to make sure.

## Create php variables using attributes (hard++)
There are many scenarios where you might want to pass the php variable inside your widget. For example, if we want to create a select which can accept an array of it's values. The most obvious way to this is to create some convention variable to excange with values:

```html
<extends:elements.input/>

<block:input>
    <?php
    if (empty($selectValues)) {
        throw new RuntimeException('Variable "selectValues" not defined.');
    }
    ?>
    <select node:attributes="exclude:label-*,context">
        <?php
        foreach ($selectValues as $name => $value) {
            echo '<option value="' . $value . '">' . $name . '</option>';
        }
        ?>
    </select>
</block:input>
```

If we want to use this element in our form, we have to first declare the selectValues variable:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements" namespace="form"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <?php
        $selectValues = [
            1 => 'A',
            2 => 'B',
            3 => 'C'
        ];
        ?>
        <form:select name="select"/>
    </form>
</block:content>
```

This doesn't appear to be very useful. Fortunatelly, there is one interesting way we can use to pass any variable to our element (non-trivial). First of all, let's try to declare your widget specific variable to store values:

```html
<extends:elements.input/>

<block:input>
    <select node:attributes="exclude:label-*,context">
        <?php
        foreach ($__values__ as $name => $value) {
            echo '<option value="' . $value . '">' . $name . '</option>';
        }
        ?>
    </select>
</block:input>
```

> We would like to name this variable in a non obvious way to prevent any conflicts.

The next step will be to populate these variable values based on the provided data. We can achieve this using special evalutator specific function `createVariable`:

```html
<extends:elements.input/>

<block:input>
    <?php #compile
    createVariable('__values__', '${values}');
    ?>
    <select node:attributes="exclude:label-*,context">
        <?php
        foreach ($__values__ as $name => $value) {
            echo '<option value="' . $value . '">' . $name . '</option>';
        }
        ?>
    </select>
</block:input>
```

This function will parse the value of `${values}` attribute and assign it to the created __values__ array (`echo` and `<?=` code will be removed). As result, we can now use our element in this way:

```html
<extends:layouts.desktop title="Demo View"/>
<use path="elements" namespace="form"/>

<block:content>
    This is our content...
    <br/>

    <form action="/">
        <form:select name="select" values="<?= [1 => 'A', 2 => 'B', 3 => 'C'] ?>"/>

        //Or even like that
        <?php
        $selection = [1 => 'A', 2 => 'B', 3 => 'C'];
        ?>

        <form:select name="select" values="<?= $selection ?>"/>
    </form>
</block:content>
```

To understand what happened, let's try to check the cached (compiled) view source:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        Demo View
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
<div class="header">
    This is our header.
</div>
<div class="wrapper page">
    This is our content...
    <br/>

    <form action="/">
        <div class="form-input">
            <label class="input-wrapper">
                <?php $__values__ = [1 => 'A', 2 => 'B', 3 => 'C']; ?>
                <select name="select">
                    <?php
                    foreach ($__values__ as $name => $value) {
                        echo '<option value="' . $value . '">' . $name . '</option>';
                    }
                    ?>
                </select>
            </label>
        </div>
        //Or even like that
        <?php
        $selection = [1 => 'A', 2 => 'B', 3 => 'C'];
        ?>
        <div class="form-input">
            <label class="input-wrapper">
                <?php $__values__ = $selection; ?>
                <select name="select">
                    <?php
                    foreach ($__values__ as $name => $value) {
                        echo '<option value="' . $value . '">' . $name . '</option>';
                    }
                    ?>
                </select>
            </label>
        </div>
    </form>
</div>
<div class="footer">
    This is our footer.
</div>
</body>
</html>
```

## Namespaces and Modules
You can locate and import your elements from any desired namespace. This can be very useful when you want to organize your widgets into separate [module] (/components/modules.md) and connect them in multiple projects. As a side effect, you will be able to update the visuals and source for your widges using the `composer update` command. 

> You only have to register the view namespace in your module installer.

## Spiral Toolkit
The Spiral framework application, which includes the module ['spiral/toolkit'] (/modules/toolkit.md), already aggregates a set of virtual tags used to simplify loading assets, form definitions and etc. For example, the following code will render a form, automatically connect the required js libraries and style sheets to make the form work over ajax and highight it's errors:

```html
<extends:spiral:layouts.html5 title="Demo View"/>

<block:body>
    <spiral:form action="/controller/doSomething">
        <form:input label="Name:" name="name"/>
        <form:select label="Select:" values="<?= [1 => 'A', 2 => 'B', 3 => 'C'] ?>"/>
    </spiral:form>
</block:body>
```

Compiled view will look like this:

```php
<!DOCTYPE html>
<html>
<head>
    <title>
        Demo View
    </title>
    <?php
    /**
     * @var \Psr\Http\Message\ServerRequestInterface $request
     */
    $request = $this->container->get(\Psr\Http\Message\ServerRequestInterface::class);
    ?>
    <script>
        window.csrfToken = "<?= $request->getAttribute('csrfToken') ?>";
    </script>
    <link rel="stylesheet" href="/resources/styles/spiral/spiral.css?1atgrq3"/>
</head>
<body>
<form action="/controller/doSomething" method="post" enctype="multipart/form-data"
      accept-charset="UTF-8" class="js-sf-form">
    <div class="form-content">
        <label class="item-form">
            <span class="item-label">Name:</span>
            <input type="text" name="name" value="" class="item-input"/>
        </label>
        <label class="item-form">
            <span class="item-label">Select:</span>

            <div class="form-group">
                <select name="" class="item-select" context="">
                    <?php $__values__ = [
                        1 => 'A',
                        2 => 'B',
                        3 => 'C'
                    ]; ?><?php $__selected__ = ''; ?>            <?php
                    if (empty($__values__)) {
                        $__values__ = [];
                    }
                    if (!is_array($__values__)) {
                        throw new \Spiral\Core\Exceptions\RuntimeException(
                            "Select values must be supplied as associated array."
                        );
                    }
                    foreach ($__values__ as $__value__ => $__label__) {
                        if ($__value__ == $__selected__) {
                            ?>
                            <option value="<?= $__value__ ?>"
                                    selected><?= $__label__ ?></option><?php
                        } else {
                            ?>
                            <option value="<?= $__value__ ?>"><?= $__label__ ?></option><?php
                        }
                    }
                    ?>
                </select>
            </div>
        </label>
    </div>
</form>
<script type="text/javascript" src="/resources/scripts/spiral/sf.js?1atj8oe"></script>
</body>
</html>
```

> Attention, JS and CSS libraries will be only be connected to the layout decalared in placeholders for assets. In your application, you can simply exclude the layout `spiral:layouts.html5` with pre-defined placeholders and stucture. It will looks like this:

```html
<dark:use bundle="spiral:bundle"/>

<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <?php
    /**
     * @var \Psr\Http\Message\ServerRequestInterface $request
     */
    $request = $this->container->get(\Psr\Http\Message\ServerRequestInterface::class);
    ?>
    <script>
        window.csrfToken = "<?= $request->getAttribute('csrfToken') ?>";
    </script>
    <block:head>
        <resource:style href="resources/styles/spiral/spiral.css"/>
        <yield:resources/>
        <!--[STYLES]-->
    </block:head>
</head>
<body>
<yield:body/>
<!--[SCRIPTS]-->
</body>
</html>
```

> Element "dark:use" is just an alias for short "use" tag.
