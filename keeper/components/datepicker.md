# Forms: Datepicker

UI component for autocomplete to use in forms

They are located in toolkit bundle, that can be included like so: 

```xhtml
<use:bundle path="toolkit:bundle"/>
```
This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/datepicker.dark.php) in demo repository

## Native date picker form:date

```xhtml
<form:date
    name="date"
    label="Native Date Picker"
    value=""
    size="6"
/>
```

Renders a native `type=date` element with no additional functionality or extended browser support

## JavaScript date picker form:date-js

```xhtml
<form:date-js
    name="date1"
    label="Date"
    value="2020-10-29T14:14:14+0300"
    size="6"
/>
```

Renders a JS powered datepicker with extended functionality. [Format tokens](https://moment.github.io/luxon/docs/manual/formatting.html#table-of-tokens) are inherited from Luxon

Parameter|Required|Default|Description
--- | --- | --- |---
name|yes|-|Field name to use
placeholder|no|-|Placeholder for input
disabled|no|-|Render input as disabled
value|no|-|Provide pre-populated value. Should match `format` use for server-client communication.
format|no|yyyy-MM-dd'T'HH:mm:ssZZZ|Format expected from server and format that is used to send data to server. Does not affect how date looks like in input.
display-format|no|yyyy LLL dd|Format used to display in datepicker input.
enable-time|no|false|Enables time input. It's developer responsibility to change display format to have hours and minutes displayed.
force-confirm-button|no|false|Force showing Apply button. It's recommended for calendars with time pickers
no-calendar|no|false|Remove calendar and use only time picker. You may want to change format options to send time only as default sends full date

![Date](/keeper/components/datepicker.png)

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

Renders a JS powered date range picker with extended functionality. [Format tokens](https://moment.github.io/luxon/docs/manual/formatting.html#table-of-tokens) are inherited from Luxon

Parameter|Required|Default|Description
--- | --- | --- |---
startName|yes|-|Field name to use for first input
startValue|no|-|Provide pre-populated value for first input. Should match `format` use for server-client communication.
endName|yes|-|Field name to use for second input
endValue|no|-|Provide pre-populated value for second input. Should match `format` use for server-client communication.
disabled|no|-|Render input as disabled
format|no|yyyy-MM-dd'T'HH:mm:ssZZZ|Format expected from server and format that is used to send data to server. Does not affect how date looks like in input.
display-format|no|yyyy LLL dd|Format used to display in datepicker input.
enable-time|no|false|Enables time input. It's developer responsibility to change display format to have hours and minutes displayed.
force-confirm-button|no|false|Force showing Apply button. It's recommended for calendars with time pickers
no-calendar|no|false|Remove calendar and use only time picker. You may want to change format options to send time only as default sends full date

![Date Range](/keeper/components/date-range.png)
