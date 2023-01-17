# Keeper

Keeper is a portal engine and UI framework (based on Stempler). The framework includes various elements such as dynamic forms, 
data grids, tabs and other layout partials. You can use Keeper to create complex Admin panels, User panels and other 
applications.

## Installation:

1. Install the dependency:

```terminal
composer require spiral/keeper
```

2. Register a keeper bootloader (see [bootloaders](../keeper/bootloaders.md) section)

## Namespace

The basic idea of Keeper is that all submodules are isolated code inside the given `namespace`. For example, there 
could be `admin` and `profile` namespaces.

All the Keeper functionality is integrated with `spiral/security` and follows RBAC rules. 

![Keeper Demo](https://user-images.githubusercontent.com/796136/81418518-79353800-9155-11ea-8266-e19fb2cce45a.png)

Configuration can be performed via code or annotations.

## Components

Keeper comes with a set of frontend components for [Stempler](../stempler/basics.md) engine, backed 
by [JavaScript Toolkit](https://github.com/spiral/toolkit)

A demo project featuring a usage of all the components is available [here](https://github.com/spiral/app-keeper).

Most notable components are:

- [Forms](../keeper/components/forms.md)
- [Autocomplete](../keeper/components/autocomplete.md)
- [DatePicker](../keeper/components/datepicker.md)
- [DataGrids](../keeper/components/datagrid.md)


