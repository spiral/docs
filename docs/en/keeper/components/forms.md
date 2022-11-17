# Forms

UI component for Forms

They are located in the toolkit bundle, that can be included like so:

```xhtml
<use:bundle path="toolkit:bundle"/>
```

This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/forms.dark.php) in the demo
repository

## Usage

Forms are mostly regular HTML forms that have the ajax submission functionality on top

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

![Form image](https://user-images.githubusercontent.com/16134699/103222722-b85b0800-4935-11eb-9cb1-45a8c44f1834.png)

### form:wrapper

| Parameter          | Required | Default | Description                                                                                                                                               |
|--------------------|----------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| action             | yes      | -       | action URL of the form                                                                                                                                        |
| method             | no       | POST    | a Http method to use, GET or POST                                                                                                                           |
| id                 | no       | -       | id of form if you want to hardcode it                                                                                                                     |
| class              | no       | -       | a class to add to the wrapper                                                                                                                                   |
| immediate          | no       | -       | debounces value in ms. if it's specified, any change event will trigger a form submission                                                                          |
| submit-on-reset    | no       | false   | submits the form on "reset" event, i.e. the reset button. Useful for filters for datagrids                                                                         |
| data-before-submit | no       | -       | the name of a callback function in the global JS scope that will be called before the form is submitted. If that callback returns false, the form will not be submitted.    |
| data-after-submit  | no       | -       | the name of a callback function in the global JS scope that will be called after the form is submitted. Note, you are expected to check the result of submitting the form yourself. |

If you need more fine-grain control over the form and its callbacks, refer
to [source code](https://github.com/spiral/toolkit/blob/master/packages/form/src/Form.ts)

### form:*

Most form inputs share common properties listed here

| Parameter     | Required | Default | Description                         |
|---------------|----------|---------|-------------------------------------|
| label         | no       | -       | A Label to render before input        |
| required      | no       | false   | If true, renders red `*` near the label |
| wrapper-id    | no       | -       | Id to use for the wrapper div           |
| wrapper-class | no       | -       | A class to add to the wrapper             |
| size          | no       | 12      | Column size for the grid system         |
| error         | no       | -       | Pre-rendered error feedback text    |
| success       | no       | -       | Pre-rendered success feedback text  |
| help          | no       | -       | Description text                    |

**Important**: Most input fields will proxy all unrecognized attributes to the corresponding input inside.

### form:input

Simple form input

| Parameter   | Required | Default | Description                          |
|-------------|----------|---------|--------------------------------------|
| name        | yes      | -       | A field name                           |
| value       | no       | -       | A field value                          |
| placeholder | no       | -       | A field placeholder                    |
| class       | no       | -       | An additional class to render for the field |
| type        | no       | -       | A field type attribute                 |
| disabled    | no       | -       | If the field should be disabled          |

```xhtml
<form:input name="firstName" label="First Name" value="" size="6" required="true"/>
```

![input](https://user-images.githubusercontent.com/16134699/103222711-b6914480-4935-11eb-8bc7-a30dbacc9eaf.png)

### form:textarea

Simple form textarea

| Parameter   | Required | Default | Description                          |
|-------------|----------|---------|--------------------------------------|
| name        | yes      | -       | A field name                           |
| value       | no       | -       | A field value                          |
| placeholder | no       | -       | A field placeholder                    |
| class       | no       | -       | An additional class to render for the field |
| type        | no       | -       | A field type attribute                 |
| disabled    | no       | -       | If the field should be disabled          |

### form:select

Simple form select dropdown

| Parameter   | Required | Default | Description                                                                |
|-------------|----------|---------|----------------------------------------------------------------------------|
| name        | yes      | -       | A field name                                                                 |
| value       | no       | -       | A field value, note that when it's used with 'multiple', it should contain an array       |
| values      | no       | -       | Select options which are mapping values to labels                                   |
| placeholder | no       | -       | A field placeholder. It will be rendered as a separate option with an empty value. |
| class       | no       | -       | An additional class to render for the field                                       |
| type        | no       | -       | A field type attribute                                                       |
| disabled    | no       | -       | If the field should be disabled                                                |

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

![input](https://user-images.githubusercontent.com/16134699/103222709-b5f8ae00-4935-11eb-8966-ec957fca8b6b.png)

