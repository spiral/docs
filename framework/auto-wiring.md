# Framework - Auto Wiring
Spiral Framework attempts to hide the container from your domain layer by providing rich auto-wiring functionality. Though, 
auto-wiring rules are very simple it's important to learn them to avoid framework misbehaviour.

## Automatic Dependency Resolution
Framework container is able to automatically resolve the constructor or method dependencies by providing instances
of concrete classes.

```php
class MyController
{}





``` 