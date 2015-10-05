# The Desing
Spiral framework use well known design approach used to develop it's components and share them to your application in a modular way.

## Container and Components
One of the primary spiral parts is it's container, such module used as glue between various modules and provides ability to communicate between components without linking source code to a specific implementation.

In general case this means that no component is allowed to request and instance or functionality of other component bypassing container (if you see such violation inside spiral components please open an issue).

Usually such approach is implemented using component interfaces which can represent generic component functionality, in some cases, especially when one component is purelly based on functionality of other component (for example ORM and DBAL) commucation is allowed using lower level abstractitions or even specific classes since mapping every possible functionality into interfaces will create implementation constraints and take long time.

Generally speaking, functionality of other component must be requested using methods `get` and `create` of container.

> Check classes located in namespace `Spiral\Core` to find spiral components foundation.

## Memory
[Application memory](memory.md) component or `HippocampusInterface` does not dictas internal component design but provides ability to split heavy analysis/compiation code from it's runtime part. Such component used to cache configs, support ORM and ODM schema behavious and etc.

Technically, application memory provides support for [Metaprogramming](https://en.wikipedia.org/wiki/Metaprogramming).

## Application Design
Application design is not nessesary must follow framework design and can create it's own communication techniques between it's pieces (for example Singleton class which shares information between application business logic pieces), however default application bundle provides following ideas:
  * Separation of view and controller logic using ViewManager and Templater component 
  * Separation of database logic and business logic using Services and DataEntities
  * Light controllers and hight testability using Services to keep abstract application logic
  * Segregation between application core and flow dispatcher (i.e. HttpDispatcher and ConsoleDispatcher)
