# The Desing
Spiral framework provides specific design approach used to develop it's components and share them to your application.

## Container and Components
One of the primary spiral part is it's container, such module used as glue between various components and provides ability to communicate between modules without linking source code to specific implementation.

In general case this means that no component is allowed to request and instance or service of other component bypassing container.

Usually such approach is implemented using component interfaces which can represent generic component functionality, in some cases, especially when one component is purelly based on functionality of other component (for example ORM and DBAL) commucation is allowed using high level abstractictions or specific classes since mapping every possible functionality into interfaces will create implementation constraints and take long time.

Generally speaking, functionality of other component must be requested using methods `get` and `create` of container.

> Check classes located in namespace `Spiral\Core` to find spiral components foundation.

## Memory


## Application Design
