# Forms: Autocomplete

UI component for autocomplete to use in forms

They are located in toolkit bundle, that can be included like so:

```xhtml
<use:bundle path="toolkit:bundle"/>
```

This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/autocomplete.dark.php) in
demo repository

## Usage

```xhtml
<form:autocomplete
        name="userId2"
        label="Simple Autocomplete"
        value="1"
>
    <script role="sf-options" type="application/json">
        {
            "url": "/keeper/users/list",
            "searchKey": "firstName",
            "valueKey": "id"
        }
    
    </script>
</form:autocomplete>
```

![Simple Autocomplete](https://user-images.githubusercontent.com/16134699/103222721-b7c27180-4935-11eb-8406-8bdf21952c1e.png)

Autocomplete uses same API as DataGrids. Autocomplete by default only operates values, not labels, so in pre-population
phase it will make request to server to fetch current labels. I.e. sample above will make a request
with `filter[id][]=1` body to `keeper/users/list` URL on load.

To handle comma separated values on backend, implement a [value accessor](/component/data-grid#value-accessors)

## form:autocomplete

There are 2 blocks of parameters you can specify.
Most used ones can be specified directly inside xhtml tag, however more advanced parameters are defined in JSON block

### Basic params

```xhtml
<form:autocomplete
        name="userId2"
        label="Simple Autocomplete"
        value="1"
        url="/keeper/users/list"
></form:autocomplete>
```

| Parameter   | Required | Default | Description                                                                                                         |
|-------------|----------|---------|---------------------------------------------------------------------------------------------------------------------|
| url         | yes      | -       | Url that implements DataGrid API                                                                                    |
| name        | yes      | -       | Field name to use                                                                                                   |
| placeholder | no       | -       | Placeholder for input                                                                                               |
| disabled    | no       | -       | Render input as disabled                                                                                            |
| value       | no       | -       | Provide pre-populated value of autocomplete                                                                         |
| labelValue  | no       | -       | Provide pre-populated label of autocomplete. If specified autocomplete will not try to resolve "value" from server. |
| preserve-id | no       | -       | Don't erase selected id when user types after autocomplete select happens, only for single choice autocompletes     |

### Extended params

Extended params allow to specify templates and customize mapping for API

```xhtml
<form:autocomplete
        name="userId2"
        label="Simple Autocomplete"
        value="1"
>
    <script role="sf-options" type="application/json">
        {
            "url": "/keeper/users/list",
            "searchKey": "firstName",
            "valueKey": "id"
        }
    
    </script>
</form:autocomplete>
```

| Parameter             | Required | Default | Description                                                                                                                                                                                                                                                                        |
|-----------------------|----------|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| id                    | no       | -       | Id to use for input                                                                                                                                                                                                                                                                |
| url                   | yes      | -       | Url that implements DataGrid API                                                                                                                                                                                                                                                   |
| method                | no       | POST    | POST or GET                                                                                                                                                                                                                                                                        |
| headers               | no       | {}      | Headers to send                                                                                                                                                                                                                                                                    |
| name                  | yes      | -       | Field name to use                                                                                                                                                                                                                                                                  |
| isMultiple            | no       | false   | Allow multiple selections                                                                                                                                                                                                                                                          |
| preserveId            | no       | -       | Don't erase selected id when user types after autocomplete select happens, only for single choice autocompletes                                                                                                                                                                    |
| data                  | no       | -       | If specified, can operate as server-less autocomplete with data pre-defined. Accepts array of strings or [IAutocompleteStaticDataItem](https://github.com/spiral/toolkit/blob/master/packages/autocomplete/src/types.ts#L3) array                                                  |
| inputTemplate         | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will display in input. Accepts row data item as variables available in template.                                                                                                                                |
| suggestTemplate       | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will display in suggestions. Accepts row data item as variables available in template.                                                                                                                          |
| loadingTemplate       | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will display during loading.                                                                                                                                                                                    |
| valueKey              | no       | 'id'    | Field that will be used as value                                                                                                                                                                                                                                                   |
| searchKey             | no       | 'name'  | Field that will be used as label                                                                                                                                                                                                                                                   |
| dataField             | no       | 'data'  | Field that will be used as data list field from response                                                                                                                                                                                                                           |
| separator             | no       | ','     | Separator that will be used for separating values in multi-select                                                                                                                                                                                                                  |
| debounce              | no       | 0       | Debounce value to throttle search events                                                                                                                                                                                                                                           |
| exposeLabelAs         | no       | -       | If specified, label will have a separate input that also will be submittted with a form                                                                                                                                                                                            |
| exposeLabelAsRequired | no       | false   | If specified, exposed label will be rendered with "required" attribute                                                                                                                                                                                                             |
| initialDataItems      | no       | -       | If specified will be used as replacement for value + valueLabel. Can contain multiple values, so can be used for pre-populating multiselect inputs. Should be a [IAutocompleteDataItem](https://github.com/spiral/toolkit/blob/master/packages/autocomplete/src/types.ts#L9) array |

For demo of usages refer
to [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/autocomplete.dark.php) 
