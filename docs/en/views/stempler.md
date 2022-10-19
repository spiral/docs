# Stempler Templates
Spiral ships with [Stempler HTML markup processor](https://github.com/spiral/stempler). By default
this engine is associated with '.dark.php' files and compatible with [NativeView](/views/native.md).
 
```html
<extends:layouts.html5 title="[[Welcome To Spiral]]"/>

<define:body>
    <spiral:form action="/index/u">
        <form:input type="file" name="images[][image]"/>
        <form:input type="file" name="images[][image]"/>
        <form:input type="file" name="images[][image]"/>
        <form:input type="file" name="images[][image]"/>

        <input type="submit"/>
    </spiral:form>
</define:body>
```
 
You can read more of how to work with stempler [here](/stempler/basics.md).