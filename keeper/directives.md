#Stempler Directives

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
This directive applies the full route name (it's expected to have logout endpoint outside of keeper namespaces).
## Keeper logout directive
Wraps given logout url params and add current auth token:
```html
<a href="@keeperLogout('admin', 'auth:logout')">[[Log out]]</a>
``` 
This directive applies the full route name (it's expected to have logout endpoint outside of keeper namespaces).

## Keeper directive
There's a convenient directive for generating uri:
```html
<a href="@keeper('admin', 'createUser')">[[+ User]]</a>
```
