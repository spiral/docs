# Forms: Rich Text Input

UI component for HTML

They are located in toolkit bundle, that can be included like so: 

```xhtml
<use:bundle path="toolkit:bundle"/>
```
This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/tinymce.dark.php) in demo repository

## Usage

Declare `TINY_MCE_KEY` in `.env` file. Get free key [here](https://www.tiny.cloud/auth/signup/)

After that include on page

```xhtml
<form:tinymce
  name="date"
  label="Custom Options"
  value="">
      <script role="sf-options" type="application/json">
          {
              "options": {
              "menubar": false,
              "height": 300
          }
    </script>
</form:tinymce>
```

`script` tag is optional, if specified, JSON inside will be used for [TinyMCE options](https://www.tiny.cloud/docs/configure/)

Parameter|Required|Default|Description
--- | --- | --- |---
value|no|-|HTML to render
name|yes|-|Field name

![Rich Text](https://user-images.githubusercontent.com/16134699/103222716-b729db00-4935-11eb-8609-38785cca2d58.png)
