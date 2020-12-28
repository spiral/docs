# DataGrids

UI component for DataGrids provide UI component for [DataGrids](/component/data-grid.md)

They are located in keeper bundle, that can be included like so: 

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/datagrids.dark.php) in demo repository

## Using Component

Simple data grid declaration would look like so

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:cell.text name="id" label="#" />
    <grid:cell.text name="firstName" label="First Name" />
    <grid:cell.text name="lastName" label="Last Name" />        
</ui:grid>
```

DataGrids are described in declarative way. Code is not iterated in php/stempler phase, but later in JavaScript phase, meaning you can't use php conditions for cell renders inside declarations. If you need conditional renderer for cell, you are expected to write one in JavaScript. 

Columns should be defined prioritizing semantics over column names. Typically most columns would match a sort key. For example if you have data row with first and last name and user, you might want to have a column `name` with `firstName` and `lastName` joined.

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

Parameter|Required|Default|Description
--- | --- | --- |---
url|yes|-|Url that implements DataGrid API
method|no|GET|Http method to use, GET or POST
id|no|[auto generated]|Id of datagrid to use
namespace|no|[empty]|Prefix for field names that is used in datagrid filters serialization. Used when multiple datagrids present on page. I.e. if <code>namespace="foo"</code> filter value <code>bar=1</code> will end up as "foo-bar=1" in URL. It's developer responsibility to use namespaces if multiple datagrids present on page. Otherwise behavior is unpredictable.
capture-forms|no|[empty]|JSON array of strings. Attaches forms by their ids to datagrid as filter fields source. Can be used for filters that are visually separated from datagrids, i.e. in nav panel or sidebar. Is used internally to attach filter defined with <ui:filter>
capture-filters|no|[empty]|JSON array of strings. Attaches instances of <a href="https://github.com/spiral/toolkit/tree/master/packages/datagrid/src/filter-toggle">filter toggle buttons.
paginate-options|no|[empty]|JSON array of numbers. Options for default paginator.  
action-*|no|[as in sample]|Set of parameters used to generate Actions button and corresponding column.

**Limitation**: All filter forms should produce a flat object map without files. Nesting and/or arrays are not supported at the moment.  

## Filters (grid:filter)

grid:filter component attaches filter and/or search forms directly to grid. Internally it works with 'capture-forms' parameter that technically allows to attach any number of forms to datagrid.

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true" immediate="300" buttons="true">
        <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
        <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
        <form:input name="email" label="Email" value="" required="true"/>
    </grid:filter>
</ui:grid>
```

Parameter|Required|Default|Description
--- | --- | --- |---
search|no|false|Render a search input in top right corner
search-name|no|"search"|Name of search field
fields|no|-|JSON array of strings. If you wish filter button to correctly indicate number of active filters, list all form fields names as a JSON array of strings. For example for example above that would be `fields="['firstName','lastName','email']"`. If not specified filter button not indicate if filter values are active. 
refresh|no|false|Render a refresh button to trigger data refresh.
immediate|no|-|If specified, search input will have no "Submit" button and search will be performed as user types with debounce value equivalent to this param value in milliseconds.
[tag body]|no|-|Insides of tag is used as an additional filter form shown in modal window if specified 
buttons|no|false|If specified as true will append default "Clear" and "Apply" buttons to tag body

Note filter form is a separate form to "search input" form and it's reset button has only effect on filter form.

### Only Search

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true"/>
</ui:grid>
```

![Search Only](/keeper/components/data-grid-filter-search-only.png)


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

![Search Only](/keeper/components/data-grid-filter-search-modal.png)

## Cell Types

Lot of cells allow to use templates using row fields as variables.

Template system used is [handlebars](https://handlebarsjs.com/) templates. Typically field accepting templates have pre-processors that convert `{` to `{{`, so if you want to output something like `{{firstName}} {{lastName}}` you pass it in as `{firstName} {lastName}`

### Text (grid:text)

```xhtml
<grid:cell.text
    name="user"
    label="User Name"
/>
```

Directly outputs field in cell

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Semantic column name matching field from server data.
label|yes|-|Column label
sort|no|-|Enables sorting for a column
sort-default|no|-|Supply "true" to make this column a defaultly sorted column. Only one column should have this.
sort-dir|no|'asc'|Default sort direction, 'asc' or 'desc'

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

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Semantic column name matching field from server data. Should match sorter name if sort is used. Content is field itself as default, but can be customized with 'body' parameter to output other field.
label|yes|-|Column label
href|yes|-|Template to use for link url
title|no|-|Template to use for link title text
body|no|-|Template to use for link body. Instead of specifying 'body' attribute, can also use tag body instead
sort|no|-|Enables sorting for a column
sort-default|no|-|Supply "true" to make this column a defaultly sorted column. Only one column should have this.
sort-dir|no|'asc'|Default sort direction, 'asc' or 'desc'

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

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Semantic column name matching field from server data. Accepts ISO dates with timezone.
label|yes|-|Column label
format|no|LLL dd, yyyy HH:mm|Date format. See [Luxon docs for available tokens](https://moment.github.io/luxon/docs/manual/parsing.html#table-of-tokens)
sort|no|-|Enables sorting for a column
sort-default|no|-|Supply "true" to make this column a defaultly sorted column. Only one column should have this.
sort-dir|no|'asc'|Default sort direction, 'asc' or 'desc'

### Actions

Separate kind of tags provided are action tags. Using any of them adds 'actions' column with button with actions dropdown. Each actions tag appends an action to that dropdown.

To customize look of the button, use `action-*` parameters of `ui:grid`

![Actions](/keeper/components/data-grid-actions.png)

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

Parameter|Required|Default|Description
--- | --- | --- |---
href|yes|-|Template to use for link url
title|no|-|Template to use for link title text
target|no|_self|Specify link target
template|no|-|Template to use for link body. Instead of specifying 'template' attribute, can also use tag body instead.
label|no|-|Text to use for link text if template param is not used. Supports templates if passed as unescaped value, i.e. `label = "{!! '{{variable}}' !!}"` 
icon|no|-|Name to use for link icon if template param is not used. Uses Font Awesome icons. Supports templates if passed as unescaped value, i.e. `icon = "{!! '{{variable}}' !!}"`


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

Parameter|Required|Default|Description
--- | --- | --- |---
href|yes|-|Template to use for action url
method|no|GET|Method name to use
title|no|-|Template to use for action title text
template|no|-|Template to use for link body. Instead of specifying 'template' attribute, can also use tag body instead.
label|no|-|Text to use for link text if template param is not used. Supports templates if passed as unescaped value, i.e. `label = "{!! '{{variable}}' !!}"` 
icon|no|-|Name to use for link icon if template param is not used. Uses Font Awesome icons. Supports templates if passed as unescaped value, i.e. `icon = "{!! '{{variable}}' !!}"`
body|no|-|JSON String to attach as request body. Supports templates if passed as unescaped value, i.e. `data = "{!! '{ "id": {{variable}} }' !!}"`
refresh|no|-|boolean value indicating if success response from server should trigger datagrid refresh
condition|no|-|Field name that contains boolean variable indicating if that action should be shown. Alternatively can be a handlebars template that should return '' for false value or anything else for positive value in case more complex logic is needed.</p>
toastError|no|-|Template for toast message to show on action error
toastSuccess|no|-|Template for toast message to show on action success
confirm|no|-|If specified, specify confirm-title, confirm-ok and confirm-cancel. Will show a confirm dialog before executing an action. <code>confirm</code> specifies handlebars template for confirm dialog body
confirm-title|no|-|specifies handlebars template for confirm dialog title
confirm-ok|no|-|specifies handlebars template for confirm dialog positive result button text
confirm-cancel|no|-|specifies handlebars template for confirm dialog negative result button text

#### Actions: Delete

```xhtml
<grid:action.delete
    href="edit/{id}"
/>
```

Same as 'grid:action' but with confirm texts pre-defined beforehand and having danger class. Typically only `href` is needed for that action type

### Template (grid:template)

```xhtml
<grid:cell.template
    name="user"
    label="User Name"
    body="<i class='fa fa-user'></i> {firstName} {lastName}"
/>
```

Outputs custom template

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Semantic column name matching field from server data.. Should match sorter name if sort is used. Content is field itself as default, but can be customized with 'body' parameter to output other field. 
label|yes|-|Column label
body|no|-|Template to use for body. Instead of specifying 'body' attribute, can also use tag body instead
sort|no|-|Enables sorting for a column
sort-default|no|-|Supply "true" to make this column a defaultly sorted column. Only one column should have this.
sort-dir|no|'asc'|Default sort direction, 'asc' or 'desc'

### Fully Custom Render (grid:render)

```xhtml
<grid:cell.render
    name="roles"
    label="Roles"
    renderer="roles"
/>
```

Uses custom function to render cell

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Semantic column name matching field from server data.. Should match sorter name if sort is used. Content is field itself as default, but can be customized with 'renderer' parameter to output other field.
label|yes|-|Column label
renderer|no|-|Name of pre-registered function that is able to render a cell
sort|no|-|Enables sorting for a column
sort-default|no|-|Supply "true" to make this column a defaultly sorted column. Only one column should have this.
sort-dir|no|'asc'|Default sort direction, 'asc' or 'desc'

To define custom function for datagrid, add script tag before toolkit declarations. If you are seeing 'renderer not found' error, most likely you've outputted renderer too late in code.

```xhtml
<stack:push name="scripts" unique-id="datagrid-roles-renderer">
    <script type="text/javascript">
        window.SFToolkit_tools_datagrid = window.SFToolkit_tools_datagrid || {}; 
        window.SFToolkit_tools_datagrid['roles'] =function () {
            return function (roles) {
                return roles.map(function (role) {
                    return '<span class="badge badge-primary mr-1">' + role.toUpperCase() + '</span>'
                }).join('');
            }
        };
    </script>
</stack:push>
```

Function returned by render function should be [CellRenderFunction](https://github.com/spiral/toolkit/blob/master/packages/datagrid/src/types.ts#L79) and it accepts lot of useful params allowing to customize render in really flexible way.

## Advanced Usage

DataGrid is based on JavaScript library and can be initialized with JavaScript

JavaScript declaration allows to use most of flexibility, i.e. you can have multiple actions columns, batch actions and selections, custom renderers directly in code without a need to pre-define them.

[See DataGrid Source Code](https://github.com/spiral/toolkit/tree/master/packages/datagrid/src) for more details.


![Custom](/keeper/components/data-grid-advanced.png)
![Custom](/keeper/components/data-grid-advanced-footer.png)

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
                            "email": {"name": "link", "arguments": {"title": "", "body": "{{email}}", "href": "mailto:{{email}}"}},
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




 

 


