# Forms

UI component for Forms

They are located in toolkit bundle, that can be included like so: 

```xhtml
<use:bundle path="toolkit:bundle"/>
```
This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/forms.dark.php) in demo repository

## Usage

Forms are mostly regular HTML forms that have ajax submission functionality on top

```xhtml
<form:wrapper action="/" method="PUT">
    <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
    <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
    <form:input name="email" label="Email" value="" required="true"/>

    <form:input type="password" name="password" label="New Password" size="6" required="true"/>
    <form:input type="password" name="confirmPassword" label="Confirm Password" size="6"/>

    <form:label label="User Roles" name="roles" required="true">
        @foreach(['admin'=>'Admin', 'super-admin'=>'Super Admin'] as $role => $label)
        <form:checkbox id="role-{{ $role }}" name="roles[]" value="{{ $role }}" label="{{$label}}"/>
        @endforeach
    </form:label>

    <form:select
        label="Select Something"
        values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
        value="2"
        placeholder="Select Value"
    />

    <form:radio-group
        name="radios"
        values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
        value="2"
    />

    <form:button label="Create"/>
</form:wrapper>
```

![Form image](/keeper/components/form.png)

### form:wrapper

Parameter|Required|Default|Description
--- | --- | --- |---
action|yes|-|action URL of form
method|no|POST|Http method to use, GET or POST
id|no|-|id of form if you want to hardcode it
class|no|-|class to add to wrapper
immediate|no|-|debounce value in ms. if specified any change event will trigger form submission
submit-on-reset|no|false|submit form on "reset" event, i.e. reset button. Useful for filters for datagrids
data-before-submit|no|-|name of callback function in global JS scope that will be called before form is submitted. If that callback returns false, form will not be submitted. 
data-after-submit|no|-|name of callback function in global JS scope that will be called after form is submitted. Note you are expected to check form submission result yourself.

If you need more fine-grain control over form and it's callbacks, refer to [source code](https://github.com/spiral/toolkit/blob/master/packages/form/src/Form.ts)

### form:*

Most form inputs share common properties listed here

Parameter|Required|Default|Description
--- | --- | --- |---
label|no|-|Label to render before input
required|no|false|If true, renders red `*` near label
wrapper-id|no|-|Id to use for wrapper div
wrapper-class|no|-|Class to add to wrapper
size|no|12|Column size for grid system
error|no|-|Pre-rendered error feedback text
success|no|-|Pre-rendered success feedback text
help|no|-|Description text

**Important**: Most input fields will proxy all unrecognized attributes to corresponding input inside.

### form:input

Simple form input

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Field name
value|no|-|Field value
placeholder|no|-|Field placeholder
class|no|-|Additional class to render for field
type|no|-|Field type attribute
disabled|no|-|If field should be disabled

```xhtml
<form:input name="firstName" label="First Name" value="" size="6" required="true"/>
```

![input](/keeper/components/form.input.png)

### form:textarea

Simple form textarea

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Field name
value|no|-|Field value
placeholder|no|-|Field placeholder
class|no|-|Additional class to render for field
type|no|-|Field type attribute
disabled|no|-|If field should be disabled

### form:select

Simple form select dropdown

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Field name
value|no|-|Field value, note when used with 'multiple', should contain an array
values|no|-|Select options, mapping values to labels
placeholder|no|-|Field placeholder. Will be rendered as a separate option with empty value.
class|no|-|Additional class to render for field
type|no|-|Field type attribute
disabled|no|-|If field should be disabled


```xhtml
    <form:select
        name="single"
        label="Select Something"
        values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
        value="2"
        placeholder="Select Value"
    />

    <form:select
        multiple
        name="multi[]"
        label="Select Something"
        values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
        value="{{[1,2]}}"
        placeholder="Select Value"
    />
```

![input](/keeper/components/form.select.png)

