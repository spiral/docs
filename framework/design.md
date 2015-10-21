# The Design
The Spiral framework uses well-known design techniques to develop it's components and share them to your application in a modular way.

Conceptually spiral is following KISS principle to simplify learning curve and provide more flexibility for project growing.

## Container and Components
One of the primary Spiral features is it's container. This module is the glue between the different modules and provides the ability to communicate between components without linking the source code to a specific implementation.

Generally speaking, no component can request an instance or functionality from another component via bypassing a container (if you see a violation like this inside any spirals components, please open an issue).

Usually this approach requires using component interfaces which represents generic component functionality. In some cases, especially when one component is purely based on the functionality of another component (for example ORM and DBAL), this communication is allowed using lower level abstractions or even specific classes. This is because mapping every possible functionality into the interfaces could create implementation constraints (aka overcomplexity) or take a long time.

Practically speaking, functionality of other component must be requested using the methods `get` and `create` of container.

> Check the classes located in the namespace `Spiral\Core` to find the Spiral components foundation.

One of the side effects of this approach, forbids you from using non implementation related component aliases as it will create hidden dependencies and require configuration that is not obvious.

For example:

```php
//Such approach is allowed
public function __construct(ContainerInterface $container, HttpDispatcher $http)
{
   //...
   $this->component = $container->get(SomeInterface::class);
}
```

```php
//Such appoach is forbidden in core components
public function __construct(ContainerInterface $container)
{
    $this->http = $container->get('http');
}
```

> You can use the component aliases and short binding in your application design to speed up development or switch to the same model as the core components to get more unification.

Default component implementations are allowed to talk to it's classes avoiding usage of container for performance reasons.

## Memory
[Application memory](memory.md) component or `HippocampusInterface` does not dictate internal component design but provides the ability to split heavy analysis/compilation code from it's runtime part. This component is used to cache configs, support ORM and ODM schema behavious, etc.

Technically, the application memory provides support for [Metaprogramming](https://en.wikipedia.org/wiki/Metaprogramming) like features.


## Static/Global Container
Spiral is trying to avoid using static code and global shared instances except only one - `ContainerInterface`. Such instance can be requested using `Component` method `container()` or requested statically via `staticContainer()` method. Global container does not required for core components (they can behave OK without it), hovewer it allows us to bring set of development sugar to your applications:

```php
//Without global container
$post = new Post([], null, $odm);

//With global container
$post = new Post();
```

Since this is only one global instance you are still able (theorecially) to isolate multiple applications in one shared memory by swapping global container before executing application enterpoint (attention, `staticContainer()` method is protected for `Component` members only so you will need something like `ContainerCapsule`):

```php
self::staticContainer($applicationA);
$applicationA->start();
```

> The only one core functionality directly depends on staticContainer - TranslatorTrait.

## Application Design
The application design doesn't need to necessarily follow the framework design and can create it's own communication techniques between it's business logic (for example, the Singleton class, which shares some information, user session, etc.). However, the default application bundle provides the following well-known concepts:
  * Separation of view and controller logic using [ViewManager](/components/views.md) and [Templater](/templater/basics.md) component 
  * Separation of database logic and business logic using [Services](/application/services.md) and [DataEntities](/components/entity.md)
  * Light controllers and high testability using [Services](/application/services.md) to keep generic application logic
  * Segregation between [application core and flow dispatcher](/application/startup.md) (i.e. [HttpDispatcher](/http/flow.md) and [ConsoleDispatcher](/console/commands.md))
