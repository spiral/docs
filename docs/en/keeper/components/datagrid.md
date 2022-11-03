# DataGrids

UI component for DataGrids provides UI component for [DataGrids](/component/data-grid.md)

They are located in the keeper bundle, that can be included like so:

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/datagrids.dark.php) in the
demo repository

## Using Component

A simple data grid declaration would look like so:

```xhtml

<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:cell.text name="id" label="#"/>
    <grid:cell.text name="firstName" label="First Name"/>
    <grid:cell.text name="lastName" label="Last Name"/>
</ui:grid>
```

DataGrids are described in a declarative way. The code is not iterated in php/stempler phase, but later in the JavaScript phase,
meaning you can't use php conditions for cell renders inside declarations. If you need a conditional renderer for cell,
you are expected to write one in JavaScript.

Columns should be defined prioritizing semantics over column names. Typically most columns would match a sort key. For
example, if you have a data row with the first and the last name and a user, you might want to have a column `name` with `firstName`
and `lastName` joined.

```xhtml

<grid:cell.link name="name" label="Name" url="@action('users.edit', ['user' => '{id}'])" sort="true">
    {firstName}&nbsp;{lastName}
</grid:cell.link>
```

## Wrapper (ui:grid)

```xhtml

<ui:grid
        url="/some/url"
        method="GET"
        id="my-grid"
        namespace="foo"
        capture-forms="['form1','form2']"
        capture-filters="['filter1','filter2']"
        paginate-options="[10,20,30]"

        actions-title=""
        actions-label="Actions"
        actions-kind=""
        actions-icon="cog"
        actions-size="sm"
        actions-class=""
        actions-cell-class="">

</ui:grid>
```

| Parameter        | Required | Default          | Description                                                                                                                                                                                                                                                                                                                                             |
|------------------|----------|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| url              | yes      | -                | Url that implements DataGrid API                                                                                                                                                                                                                                                                                                                        |
| method           | no       | GET              | Http method to use, GET or POST                                                                                                                                                                                                                                                                                                                         |
| id               | no       | [auto generated] | Id of datagrid to use                                                                                                                                                                                                                                                                                                                                   |
| namespace        | no       | [empty]          | A prefix for field names that is used in the datagrid filters serialization. It's used when multiple datagrids are present on page. I.e. if <code>namespace="foo"</code> filter value <code>bar=1</code> will end up as "foo-bar=1" in URL. It's developer's responsibility to use namespaces if multiple datagrids are present on page. Otherwise the behavior is unpredictable. |
| capture-forms    | no       | [empty]          | JSON array of strings. It attaches forms by their ids to datagrid as a filter fields source. It can be used for filters that are visually separated from datagrids, i.e. in the nav panel or sidebar. It's used internally to attach the filter defined with `<ui:filter>`                                                                                                |
| capture-filters  | no       | [empty]          | JSON array of strings. It attaches the instances of [filter toggle buttons](https://github.com/spiral/toolkit/tree/master/packages/datagrid/src/filter-toggle).                                                                                                                                                                                      |
| paginate-options | no       | [empty]          | JSON array of numbers. It options for the default paginator.                                                                                                                                                                                                                                                                                                   |
| action-*         | no       | [as in sample]   | A set of parameters that is used to generate the Actions button and the corresponding column.                                                                                                                                                                                                                                                                             |

**Limitation**: All the filter forms should produce a flat object map without files. Nesting and/or arrays are not supported
at the moment.

## Filters (grid:filter)

Grid:filter component attaches the filter and/or search forms directly to the grid. Internally it works with the 'capture-forms'
parameter that technically allows to attach any number of forms to the datagrid.

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true" immediate="300" buttons="true">
        <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
        <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
        <form:input name="email" label="Email" value="" required="true"/>
    </grid:filter>
</ui:grid>
```

| Parameter   | Required | Default  | Description                                                                                                                                                                                                                                                                                                               |
|-------------|----------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| search      | no       | false    | Renders a search input in the top right corner                                                                                                                                                                                                                                                                                 |
| search-name | no       | "search" | The name of the search field                                                                                                                                                                                                                                                                                                      |
| fields      | no       | -        | JSON array of strings. If you want the filter button to indicate the number of active filters correctly, list all  the form fields names as a JSON array of strings. For the example above that would be `fields="['firstName','lastName','email']"`. If it's not specified, the filter button does not indicate if the filter values are active or not. |
| refresh     | no       | false    | Renders a refresh button to trigger data refresh.                                                                                                                                                                                                                                                                          |
| immediate   | no       | -        | If it's specified,  the search input will have no "Submit" button and the search will be performed as user types with the debounce value equivalent to this param value in milliseconds.                                                                                                                                                    |
| [tag body]  | no       | -        | Insides of tag are used as an additional filter form shown in the modal window if it's specified                                                                                                                                                                                                                                    |
| buttons     | no       | false    | If they're specified as true, the default "Clear" and "Apply" buttons will append to the tag body.                                                                                                                                                                                                                                          |

Note, the filter form is a separate form to the "search input" form and its reset button has effect only on the filter form.

### Only Search

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true"/>
</ui:grid>
```

![Search Only](https://user-images.githubusercontent.com/16134699/103222727-b98c3500-4935-11eb-989a-801eb5649529.png)

### Search And Modal

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true" immediate="300" buttons="true">
        <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
        <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
        <form:input name="email" label="Email" value="" required="true"/>
    </grid:filter>
</ui:grid>
``` 

![Search And Modal](https://user-images.githubusercontent.com/16134699/103222726-b8f39e80-4935-11eb-9c76-ec23e5cecd2c.png)

## Cell Types

Lots of cells allow to use templates using row fields as variables.

The used template system is [handlebars](https://handlebarsjs.com/) templates. Typically field accepting templates have
pre-processors that convert `{` to `{{`, so if you want to output something like `{{firstName}} {{lastName}}`, you pass
it in as `{firstName} {lastName}`

### Text (grid:text)

```xhtml
<grid:cell.text
        name="user"
        label="User Name"
/>
```

Directly outputs a field in a cell

| Parameter    | Required | Default | Description                                                                                    |
|--------------|----------|---------|------------------------------------------------------------------------------------------------|
| name         | yes      | -       | A semantic column name which is matching the field from the server data.                                          |
| label        | yes      | -       | A column label                                                                                   |
| sort         | no       | -       | Enables sorting for a column                                                                   |
| sort-default | no       | -       | Supply "true" to make this column a defaultly sorted column. Only one column should have this. |
| sort-dir     | no       | 'asc'   | A default sort direction, 'asc' or 'desc'                                                        |

### Link (grid:link)

```xhtml
<grid:cell.link
        name="user"
        title="Edit User {firstName}"
        body="{fistName}"
        href="/edit/{id}"
/>
```

Outputs a link

| Parameter    | Required | Default | Description                                                                                                                                                                                            |
|--------------|----------|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name         | yes      | -       | A semantic column name which is matching the field from the server data. It should match the sorter name if the sort is used. Content is the field itself as default, but it can be customized with the 'body' parameter to output another field. |
| label        | yes      | -       | A column label                                                                                                                                                                                           |
| href         | yes      | -       | A template to use for the link url                                                                                                                                                                           |
| title        | no       | -       | A template to use for the link title text                                                                                                                                                                    |
| body         | no       | -       | A template to use for the link body. Instead of specifying the 'body' attribute, it possible to use the tag body                                                                                                   |
| sort         | no       | -       | Enables sorting for a column                                                                                                                                                                           |
| sort-default | no       | -       | Supply "true" to make this column a defaultly sorted column. Only one column should have this.                                                                                                         |
| sort-dir     | no       | 'asc'   | A default sort direction, 'asc' or 'desc'                                                                                                                                                                |

### Date (grid:date)

```xhtml
<grid:cell.link
        name="created"
        label="Created At"
        format="LLL dd, yyyy HH:mm"
        sort="true"
        sort-dir="desc"
        sort-default="true"
/>
```

Outputs a date

| Parameter    | Required | Default            | Description                                                                                                                 |
|--------------|----------|--------------------|-----------------------------------------------------------------------------------------------------------------------------|
| name         | yes      | -                  | A semantic column name which is matching the field from the server data. It accepts ISO dates with a timezone.                                      |
| label        | yes      | -                  | A column label                                                                                                                |
| format       | no       | LLL dd, yyyy HH:mm | A date format. See [Luxon docs for available tokens](https://moment.github.io/luxon/docs/manual/parsing.html#table-of-tokens) |
| sort         | no       | -                  | Enables sorting for a column                                                                                                |
| sort-default | no       | -                  | Supply "true" to make this column a defaultly sorted column. Only one column should have this.                              |
| sort-dir     | no       | 'asc'              | A default sort direction, 'asc' or 'desc'                                                                                     |

### Actions

A separate kind of tags that are provided are action tags. Using any of them adds the 'actions' column with a button with a drwondown list of actions. Each action's tag appends an action to that dropdown.

To customize the look of the button, use `action-*` parameters of `ui:grid`

![Actions](https://user-images.githubusercontent.com/16134699/103222723-b85b0800-4935-11eb-8683-87e3cbcfd28a.png)

#### Actions: Link

```xhtml
<grid:action.link
        href="edit/{id}"
        template="{id}"
        label="Edit"
        title="Edit {firstName}"
        icon="edit"
        target="_blank"
/>
```

Outputs a link

| Parameter | Required | Default | Description                                                                                                                                                               |
|-----------|----------|---------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| href      | yes      | -       | A template to use for link url                                                                                                                                              |
| title     | no       | -       | A template to use for the link title text                                                                                                                                       |
| target    | no       | _self   | Specifies the target of the link                                                                                                                                                       |
| template  | no       | -       | A template to use for the link body. Instead of specifying the 'template' attribute, the tag body can be used.                                                                 |
| label     | no       | -       | Text to use for the link text if  the template param is not used. It supports templates if it's passed as unescaped value, i.e. `label = "{!! '{{variable}}' !!}"`                         |
| icon      | no       | -       | A name to use for the link icon if the template param is not used. It uses Font Awesome icons. It supports templates if it's passed as unescaped value, i.e. `icon = "{!! '{{variable}}' !!}"` |

#### Actions: Action

```xhtml
<grid:action.action
        href="edit/{id}"
        template="{id}"
        label="Edit"
        title="Edit {firstName}"
        icon="edit"
        method="POST"
        data="{ foo: 1}"
        refresh="true"
        confirm="true"
        confirm-title="true"
        confirm-ok="true"
        confirm-cancel="true"
/>
```

Makes a request to server

| Parameter      | Required | Default | Description                                                                                                                                                                                                                                     |
|----------------|----------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| href           | yes      | -       | A template to use for action url                                                                                                                                                                                                                  |
| method         | no       | GET     | A method name to use                                                                                                                                                                                                                              |
| title          | no       | -       | A template to use for the action title text                                                                                                                                                                                                           |
| template       | no       | -       | A template to use for the link body. Instead of specifying the 'template' attribute, it's possible to use the tag body                                                                                                                                       |
| label          | no       | -       | Text to use for the link text if the template param is not used. It supports templates if it's passed as unescaped value, i.e. `label = "{!! '{{variable}}' !!}"`                                                                                               |
| icon           | no       | -       | A name to use for the link icon if the template param is not used. It uses Font Awesome icons. It supports templates if it's passed as unescaped value, i.e. `icon = "{!! '{{variable}}' !!}"`                                                                       |
| body           | no       | -       | JSON String to attach as a request body. It supports templates if it's passed as unescaped value, i.e. `data = "{!! '{ "id": {{variable}} }' !!}"`                                                                                                        |
| refresh        | no       | -       | A boolean value which is indicating if a success response from the server should trigger a datagrid refresh                                                                                                                                                        |
| condition      | no       | -       | A field name that contains boolean variable indicating if that action should be shown. Alternatively there could be a handlebars template that should return '' for a false value or anything else for a positive value in case more complex logic is needed |
| toastError     | no       | -       | A template for the toast message to show on an action error                                                                                                                                                                                              |
| toastSuccess   | no       | -       | A template for the toast message to show on an action success                                                                                                                                                                                            |
| confirm        | no       | -       | If it's specified, specify confirm-title, confirm-ok and confirm-cancel. It will show a confirm dialog before executing an action. <code>confirm</code> specifies handlebars template for confirm dialog body                                           |
| confirm-title  | no       | -       | Specifies the handlebars template for the confirm dialog title                                                                                                                                                                                          |
| confirm-ok     | no       | -       | Specifies the handlebars template for the confirmation dialog of the positive result button text             |
| confirm-cancel | no       | -       | Specifies the handlebars template for the confirmation dialog of the negative result button text                                                                                                                                                                    |

#### Actions: Delete

```xhtml
<grid:action.delete
        href="edit/{id}"
/>
```

It's the same as 'grid:action' but with predefined confirmation texts and a danger class. Typically only `href` is
needed for that action type.

### Template (grid:template)

```xhtml
<grid:cell.template
        name="user"
        label="User Name"
        body="<i class='fa fa-user'></i> {firstName} {lastName}"
/>
```

Outputs custom template

| Parameter    | Required | Default | Description                                                                                                                                                                                             |
|--------------|----------|---------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name         | yes      | -       | A semantic column name which is matching the field from the server data. It should match the sorter name if the sort is used. The content is the field itself as default, but it can be customized with the 'body' parameter to output another field. |
| label        | yes      | -       | A column label                                                                                                                                                                                            |
| body         | no       | -       | A template to use for the body. Instead of specifying the 'body' attribute, it's possible to use the tag body                                                                                                          |
| sort         | no       | -       | Enables sorting for a column                                                                                                                                                                            |
| sort-default | no       | -       | Supply "true" to make this column a defaultly sorted column. Only one column should have this.                                                                                                          |
| sort-dir     | no       | 'asc'   | A default sort direction, 'asc' or 'desc'                                                                                                                                                                 |

### Fully Custom Render (grid:render)

```xhtml
<grid:cell.render
        name="roles"
        label="Roles"
        renderer="roles"
/>
```

It uses a custom function to render the cell

| Parameter    | Required | Default | Description                                                                                                                                                                                                 |
|--------------|----------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name         | yes      | -       | A Semantic column name  matching the field from server the data. It should match the sorter name if the sort is used. The content is the field itself as default, but it can be customized with the 'renderer' parameter to output another field. |
| label        | yes      | -       | A column label                                                                                                                                                                                                |
| renderer     | no       | -       | A Name of a pre-registered function that is able to render a cell                                                                                                                                               |
| sort         | no       | -       | Enables sorting for a column                                                                                                                                                                                |
| sort-default | no       | -       | Supply "true" to make this column a defaultly sorted column. Only one column should have this.                                                                                                              |
| sort-dir     | no       | 'asc'   | A default sort direction, 'asc' or 'desc'                                                                                                                                                                     |

To define a custom function for the datagrid, add a script tag before the toolkit declarations. If you see 'renderer not
found' error, most likely you've outputted the renderer too late in the code.

```xhtml
<stack:push name="scripts" unique-id="datagrid-roles-renderer">
    <script type="text/javascript">
        window.SFToolkit_tools_datagrid = window.SFToolkit_tools_datagrid || {};
        window.SFToolkit_tools_datagrid['roles'] = function () {
            return function (roles) {
                return roles.map(function (role) {
                    return '<span class="badge badge-primary mr-1">' + role.toUpperCase() + '</span>'
                }).join('');
            }
        };
    </script>
</stack:push>
```

The function returned by the render function should
be [CellRenderFunction](https://github.com/spiral/toolkit/blob/master/packages/datagrid/src/types.ts#L79) and it accepts
a lot of useful params allowing to customize the render in a really flexible way.

## Advanced Usage

DataGrid is based on the JavaScript library and can be initialized with JavaScript

The JavaScript declaration allows for more flexibility, i.e. you can have multiple actions columns, batch actions and
selections, custom renderers directly in the code without the need to pre-define them.

[See DataGrid Source Code](https://github.com/spiral/toolkit/tree/master/packages/datagrid/src) for more details.

![Custom](https://user-images.githubusercontent.com/16134699/103222725-b8f39e80-4935-11eb-8b35-464ae0224906.png)
![Custom](https://user-images.githubusercontent.com/16134699/103222724-b85b0800-4935-11eb-9c2f-5c4cd3eaa051.png)

```xhtml
<div class="sf-table">
    <div class="js-sf-datagrid">
        @declare(syntax=off)
        <script type="text/javascript" role="sf-options">
            (function () {
                return {
                    "id": "custom",
                    "url": "/keeper/users/list",
                    "namespace": "custom",
                    "method": "GET",
                    "ui": {
                        "headerCellClassName": {"actions": "text-right"},
                        "cellClassName": {"actions": "text-right py-2", "created": "text-nowrap"}
                    },
                    "paginator": {"limitOptions": [10, 20, 50, 100]},
                    "sort": "created",
                    "columns": [{"id": "name", "title": "Name", "sortDir": "asc"}, {"id": "actions2", "title": " "}, {
                        "id": "email",
                        "title": "Email",
                        "sortDir": "asc"
                    }, {"id": "created", "title": "Created At", "sortDir": "desc"}, {
                        "id": "roles",
                        "title": "Roles",
                        "sortDir": null
                    }, {"id": "id", "title": "ID", "sortDir": null}, {"id": "actions", "title": " "}],
                    "selectable": {
                        "type": "multiple",
                        "id": "id"
                    },
                    "renderers": {
                        "cells": {
                            "name": {
                                "name": "link",
                                "arguments": {
                                    "title": "",
                                    "body": "{{firstName}}&nbsp;{{lastName}}",
                                    "href": "\/keeper\/users\/{{id}}"
                                }
                            },
                            "email": {
                                "name": "link",
                                "arguments": {"title": "", "body": "{{email}}", "href": "mailto:{{email}}"}
                            },
                            "created": {"name": "dateFormat", "arguments": ["LLL dd, yyyy HH:mm"]},
                            "roles": {"name": "roles", "arguments": []},
                            "id": {"name": "template", "arguments": ["{{id}}"]},
                            "actions": {
                                "name": "actions",
                                "arguments": {
                                    "kind": "",
                                    "size": "sm",
                                    "className": "",
                                    "icon": "cog",
                                    "label": "Actions",
                                    "actions": [{
                                        "type": "href",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "label": "Edit",
                                        "target": null,
                                        "icon": "edit",
                                        "template": ""
                                    }, {
                                        "type": "action",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "method": "DELETE",
                                        "label": "Delete",
                                        "icon": "trash",
                                        "template": "<span class=\"text-danger\"><i class=\"fa fw fa-trash\"><\/i>&nbsp;&nbsp; Delete<\/span>",
                                        "condition": null,
                                        "data": [],
                                        "refresh": true,
                                        "confirm": {
                                            "body": "Are you sure to delete this entry?",
                                            "title": "Confirmation Required",
                                            "confirm": "Delete",
                                            "confirmKind": "danger",
                                            "cancel": "Cancel"
                                        },
                                        "toastSuccess": "<i class=\"fa fa-check-circle\"><\/i>&nbsp; {{message}}\n              ",
                                        "toastError": "<i class=\"fa fa-exclamation\"><\/i>&nbsp; {{error}}\n              "
                                    }]
                                }
                            },
                            "actions2": {
                                "name": "actions",
                                "arguments": {
                                    "kind": "",
                                    "size": "sm",
                                    "className": "",
                                    "icon": "cog",
                                    "label": "Actions 2",
                                    "actions": [{
                                        "type": "href",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "label": "Edit",
                                        "target": null,
                                        "icon": "edit",
                                        "template": ""
                                    }, {
                                        "type": "action",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "method": "DELETE",
                                        "label": "Delete",
                                        "icon": "trash",
                                        "template": "<span class=\"text-danger\"><i class=\"fa fw fa-trash\"><\/i>&nbsp;&nbsp; Delete<\/span>",
                                        "condition": null,
                                        "data": [],
                                        "refresh": true,
                                        "confirm": {
                                            "body": "Are you sure to delete this entry?",
                                            "title": "Confirmation Required",
                                            "confirm": "Delete",
                                            "confirmKind": "danger",
                                            "cancel": "Cancel"
                                        },
                                        "toastSuccess": "<i class=\"fa fa-check-circle\"><\/i>&nbsp; {{message}}\n              ",
                                        "toastError": "<i class=\"fa fa-exclamation\"><\/i>&nbsp; {{error}}\n              "
                                    }]
                                }
                            }
                        },
                        "actions": {
                            "delete": {
                                renderAs: "<div class='btn btn-danger'>Delete</div>",
                                onClick: function (state, grid) {
                                    console.log(state, grid);
                                }
                            }
                        }
                    }
                };
            });
        </script>
        @declare(syntax=on)
    </div>
</div>
```
