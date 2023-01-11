# Forms: Autocomplete

UI component for autocomplete in the forms

They are located in the toolkit bundle, that can be included like so:

```xhtml
<use:bundle path="toolkit:bundle"/>
```

This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/autocomplete.dark.php) in
the demo repository

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

Autocomplete uses the same API as DataGrids. Autocomplete by default only operates values, not labels, so in the pre-population
phase it will make a request to server to fetch the current labels. I.e. The sample above will make a request
with `filter[id][]=1` body to `keeper/users/list` URL on load.

To handle comma separated values on backend, implement a [value accessor](../component/data-grid#value-accessors)

## form:autocomplete

There are 2 blocks of parameters you can specify.
The most commonly used ones can be specified directly inside xhtml tag, however more advanced parameters are defined in JSON block

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
| name        | yes      | -       | A field name to use                                                                                                   |
| placeholder | no       | -       | A placeholder for the input                                                                                               |
| disabled    | no       | -       | Renders the input as disabled                                                                                            |
| value       | no       | -       | Provides a pre-populated value of autocomplete                                                                         |
| labelValue  | no       | -       | Provides a pre-populated label of autocomplete. If it's specified, autocomplete will not try to resolve "value" from the server. |
| preserve-id | no       | -       | Don't erase the selected id when a user types after autocomplete selection. It's only for single choice autocompletes     |

### Extended params

Extended params allow to specify templates and customize the mapping for API

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
| id                    | no       | -       | Id to use for the input                                                                                                                                                                                                                                                                |
| url                   | yes      | -       | Url that implements DataGrid API                                                                                                                                                                                                                                                   |
| method                | no       | POST    | POST or GET                                                                                                                                                                                                                                                                        |
| headers               | no       | {}      | Headers to send                                                                                                                                                                                                                                                                    |
| name                  | yes      | -       | A field name to use                                                                                                                                                                                                                                                                  |
| isMultiple            | no       | false   | Allows multiple selections                                                                                                                                                                                                                                                          |
| preserveId            | no       | -       | Don't erase the selected id when a user types after autocomplete selection, it's only for single choice autocompletes                                                                                                                                                                    |
| data                  | no       | -       | If it's specified, it can operate as server-less autocomplete with the data pre-defined. Accepts an array of strings or [IAutocompleteStaticDataItem](https://github.com/spiral/toolkit/blob/master/packages/autocomplete/src/types.ts#L3) array                                                  |
| inputTemplate         | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will be displayed in the input. Accepts a row data item as variables available in the template.                                                                                                                                |
| suggestTemplate       | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will be displayed in the suggestions. Accepts a row data item as variables available in the template.                                                                                                                          |
| loadingTemplate       | no       | -       | [handlebars](https://handlebarsjs.com/) template to customize what will be displayed during the loading.                                                                                                                                                                                    |
| valueKey              | no       | 'id'    | A field that will be used as a value                                                                                                                                                                                                                                                   |
| searchKey             | no       | 'name'  | A field that will be used as a label                                                                                                                                                                                                                                                   |
| dataField             | no       | 'data'  | A field that will be used as a data list field from the response                                                                                                                                                                                                                           |
| separator             | no       | ','     | A separator that will be used for separating values in multi-select                                                                                                                                                                                                                  |
| debounce              | no       | 0       | Debounce value to throttle search events                                                                                                                                                                                                                                           |
| exposeLabelAs         | no       | -       | If it's specified, label will have a separate input that will also be submittted with a form                                                                                                                                                                                            |
| exposeLabelAsRequired | no       | false   | If it's specified, the exposed label will be rendered with the "required" attribute                                                                                                                                                                                                             |
| initialDataItems      | no       | -       | If it's specified, it will be used as a replacement for value + valueLabel. It can contain multiple values, so it can be used for pre-populating multiselect inputs. Its hould be a [IAutocompleteDataItem](https://github.com/spiral/toolkit/blob/master/packages/autocomplete/src/types.ts#L9) array |

For demo of usages refer
to [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/autocomplete.dark.php) 
