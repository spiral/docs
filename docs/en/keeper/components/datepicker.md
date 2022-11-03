# Forms: Datepicker

UI component for autocomplete in forms

They are located in the toolkit bundle, that can be included like so:

```xhtml
<use:bundle path="toolkit:bundle"/>
```

This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/datepicker.dark.php) in
the demo repository

## Native date picker form:date

```xhtml
<form:date
        name="date"
        label="Native Date Picker"
        value=""
        size="6"
/>
```

It renders a native `type=date` element with no additional functionality or extended browser support

## JavaScript date picker form:date-js

```xhtml
<form:date-js
        name="date1"
        label="Date"
        value="2020-10-29T14:14:14+0300"
        size="6"
/>
```

It renders a JS powered datepicker with extended
functionality. [Format tokens](https://moment.github.io/luxon/docs/manual/formatting.html#table-of-tokens) are inherited
from Luxon

| Parameter            | Required | Default                  | Description                                                                                                                  |
|----------------------|----------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| name                 | yes      | -                        | A field name to use                                                                                                            |
| placeholder          | no       | -                        | A placeholder for the input                                                                                                        |
| disabled             | no       | -                        | Renders the input as disabled                                                                                                     |
| value                | no       | -                        | It provides a pre-populated value. It should match the`format` used for server-client communication.                                      |
| format               | no       | yyyy-MM-dd'T'HH:mm:ssZZZ | The format expected from the server and the format that is used to send data to the server. It does not affect how the date looks like in the input    |
| display-format       | no       | yyyy LLL dd              | The format used to display in the datepicker input                                                                                  |
| enable-time          | no       | false                    | Enables time input. It's developer's responsibility to change the display format to have hours and minutes displayed.              |
| force-confirm-button | no       | false                    | Forces the display of the  Apply button. It's recommended for calendars with time pickers                                                 |
| no-calendar          | no       | false                    | Removes calendar and uses only the time picker. You may want to change the format options to send time only, as the default sends the full date |

![Date](https://user-images.githubusercontent.com/16134699/103222720-b7c27180-4935-11eb-9cc4-81ee3a2a0275.png)

## JavaScript date range picker form:date-range-double

```xhtml
<form:date-range-double
        startName="date1"
        endName="date1"
        label="Date"
        startValue="2020-10-29T14:14:14+0300"
        endValue="2020-10-29T14:14:14+0300"
        size="6"
/>
```

It renders a JS powered date range picker with extended
functionality. [Format tokens](https://moment.github.io/luxon/docs/manual/formatting.html#table-of-tokens) are inherited
from Luxon

| Parameter            | Required | Default                  | Description                                                                                                                  |
|----------------------|----------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| startName            | yes      | -                        | A field name to use for the first input                                                                                            |
| startValue           | no       | -                        | Provides a pre-populated value for the first input. It should match the `format` use for server-client communication.                      |
| endName              | yes      | -                        | A field name to use for the second input                                                                                           |
| endValue             | no       | -                        | Provides a pre-populated value for the second input. It should match the `format` use for server-client communication.                     |
| disabled             | no       | -                        | Renders the input as disabled                                                                                                     |
| format               | no       | yyyy-MM-dd'T'HH:mm:ssZZZ | The format expected from the server and the format that is used to send data to the server. It does not affect how the date looks like in the input.    |
| display-format       | no       | yyyy LLL dd              | The Format used to display in the datepicker input.                                                                                  |
| enable-time          | no       | false                    | Enables the time input. It's developer's responsibility to change the display format to have hours and minutes displayed.              |
| force-confirm-button | no       | false                    | Forces the display of the  Apply button. It's recommended for calendars with time pickers                                                 |
| no-calendar          | no       | false                    | Removes the calendar and uses only the time picker. You may want to change the format options to send  time only, as the default sends the full date |

![Date Range](https://user-images.githubusercontent.com/16134699/103222719-b729db00-4935-11eb-876f-496a14a80cc3.png)
