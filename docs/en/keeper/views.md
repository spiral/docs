# Views

Keeper comes with several stempler directives, views and templates

## Directives

### Auth directive

Provides a permission check with context, uses `GuardInterface` call under the hood.

```html
@auth('permision', ['some' => 'context'])
<p>[[Allowed]]</p>
@endauth
```

### Logout directive

Wraps given logout url params and add current auth token:

```html
<a href="@logout(admin['auth:logout'])">[[Log out]]</a>
``` 

For now this directive accepts the full route name.

### Route directive

There's a convenient directive for generating uri in the given namespace:

```html
<a href="@keeper('admin', 'createUser')">[[+ User]]</a>
```

## Views and Templates

The next views and templates are worth mentioning:

- `keeper:login` view contains sign-up form, can be extended for some customization or built from the scratch (don't
  forget to register the view in the keeper config):

```php
<?php
// app/config/path/to/keeper/config.php

return [
    'loginView' => 'default:path/to/login/view'
];
```

- `keeper:layout/page` and `keeper:layout:tabs` are 2 main layouts, example below:

```html

<extends:keeper:layout.page title="[[Title]]"/>
<use:bundle path="keeper:bundle"/>

<define:content>
    [[Some page content.]]
</define:content>
```

```html

<extends:keeper:layout.tabs title="[[Title]]"/>
<use:bundle path="keeper:bundle"/>

<ui:tab id="information" icon="info" title="[[Information]]" active="true">
    [[Information tab content.]]
</ui:tab>
<ui:tab id="data" icon="cog" title="[[Data]]">
    [[Data tab content.]]
</ui:tab>
```

- grid templates are powerful html elements for tables with filters:

```html
//...
<block:content>
    <ui:grid url="@action('users.list')">
        <grid:filter search="true" immediate="300" buttons="true">
            @auth('keeper.statuses')
            <form:select name="status" label="[[Status]]" placeholder="[[Select Status]]"
                         values="{{ ['active' => '[[Active]]', 'disabled' => '[[Disabled]]'] }}"/>
            @endauth
        </grid:filter>

        <grid:cell.text name="first_name" label="[[First Name]]" sort="true" body="{firstName}" sort-dir="asc"
                        sort-default="true"/>
        <grid:cell.text name="last_name" label="[[Last Name]]" sort="true" body="{lastName}"/>
        <grid:cell.link name="email" label="[[Email]]" href="mailto:{email}" body="{email}" sort="true"
                        condition="showEmail"/>
        <grid:cell.render name="status" label="[[Status]]" renderer="status"/>

        <grid:action.link label="[[Edit]]" icon="edit" url="@action('users.edit', ['user' => '{id}'])"
                          permission="keeper.users.view"/>
    </ui:grid>
</block:content>
<stack:push name="scripts" unique-id="datagrid-account-renderer">
    <script type="text/javascript">
        SFToolkit.tools._datagrid.register('status', function () {
            return function (status, item) {
                let map = {
                    "active": 'badge-primary',
                    "disabled": 'badge-warning',
                }
                let badge = (status.toLowerCase() in map) ? map[status.toLowerCase()] : 'badge-secondary';
                return '<span class="badge ' + badge + ' mr-1">' + status.toUpperCase() + '</span>';
            }
        });
    </script>
</stack:push>
```

#### Some explanations:

`sort="true"` allows sorting grid by that column.
Additional `sort-dir="asc|desc"` and `sort-default="true"` enables default sorting with order.
`name="first_name"` is used as a name of the sort key in the query (`http://example.com?sort[first_name]=asc`).

`condition=""` allows showing/hiding the row if needed. `body="{firstName}"` is the name of the row column.

`<grid:cell.render />` with `renderer` attribute allows using custom render template. Current item (grid row) and cell
value are available inside.

Grid example below:

```json
{
  "status": 200,
  "data": [
    {
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@smith.com",
      "showEmail": true,
      "status": "active"
    }
  ]
}
```
