# Stempler Directives
There are several stempler directives available in the keeper package.

## Auth directive
Provides a permission check with context, uses `GuardInterface` call under the hood.
```html
@auth('permision', ['some' => 'context'])
    <p>[[Allowed]]</p>
@endauth
```

## Logout directive
Wraps given logout url params and add current auth token:
```html
<a href="@logout(admin['auth:logout'])">[[Log out]]</a>
``` 
For now this directive accepts the full route name.

## Route directive
There's a convenient directive for generating uri in the given namespace:
```html
<a href="@keeper('admin', 'createUser')">[[+ User]]</a>
```
