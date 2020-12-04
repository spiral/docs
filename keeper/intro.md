# Keeper
Keeper is portal engine and UI framework (based on Stempler). Framework includes various elements such as 
dynamic forms, data grids, tabs and other layout partials.
You can use Keeper to create complex Admin panels, User panels and other applications.

## Installation:
- Install the dependency:
```
$ composer require spiral/keeper
```
- register keeper bootloader (see [bootloaders](/keeper/bootloaders.md) section)

## Namespace
The basic idea of keeper is that all sub-modules are isolated code inside of a given `namespace`.
For example, there could be `admin` and `profile` namespaces.

All the Keeper functionality integrated with `spiral/security` and follows RBAC rules. 

![Keeper Demo](https://user-images.githubusercontent.com/796136/81418518-79353800-9155-11ea-8266-e19fb2cce45a.png)

Configuration can be performed via code or annotations.

Demo project is available [here](https://github.com/spiral/app-keeper).
