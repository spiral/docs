# Schemas and Documenters
Both the ORM and ODM components in Spiral framework are built based on the same principle - cached **behaviour schemas**. This schemas concept is based on memory structure which is used to describe the specific aspects of every database entity (model) including validations, fields, fiters etc. 

> This article is related to [Metaprogramming](https://en.wikipedia.org/wiki/Metaprogramming).

## Schemas and Memory
The general idea regarding the use of schemas is the ability to separate out heavy analysis and inheritance operations into light and simple runtime processes. Using `HippocampusInterface`, components can collect and process all possible information about the database models (create model reflection) in the background and later share it with runtime code. It can create automatic data filters, synchronize databases (ORM) or even detect errors and security issues in your code.

Since there aren't really any limitation on time in CLI mode, we can make our analysis as complex and powerful as we want. For example, you can describe the desired model behaviour (columns, relations, compositions etc) using model code itself like in other frameworks without the need for huge configurations as we can create reflection and analyse in every existing model.

Behaviour schemas can significantly increase the amount of supported features without slowing down performance of runtime code (sometimes even increase). [**Memory interface**] (/framework/memory.md) plays a very big role in this case, as it works not only as an optional cache, but instead as a required part of component flow.

> In addition to memory interface, another important piece of analysis process is `TokenizerInterface` which can locate the required classes without user intervention.

You can count the spiral schemas as a **compilation** of your code before using it in runtime. Since compilation is required to make your models work and there isn't an efficient way (yet) to detect changes in model behaviours you must execute console command to update schema every time you change your database model schema (you can still freely add methods without rebuilding schema every time). Fortunately such a command is easily accessible - `spiral update`. If you don't have other commands starting with "u" you can use a shorter alias `spiral up`. 

## Documenters
Since every database entity in spiral can be described using it's schema (model reflection), we are able to help IDE or editor to understand the structure of your project (not framework) and create a set of tooltips related to your application code. Tooltips might include columns names, relations and magic setters/getters methods (no one wants to remember all the possible table columns, right?).

Currently, Spiral can generate a "virtual documentation" for PHPStorm IDE using a set of "cloned" classes. This method isn't the cleanest but it works and can be changed at any moment without affecting your application. If you would like to support other IDEs and editors or improve PHPStorm documenter, your help will be much appreciated.

On a side note, having an easily assessable schema of your database layer might help to create automatic project documention. For example, export structure into [UML] (/odm/uml.md).

> I hope one day we can define universal format (JSON or XML) to describe **project specific** structures and dependencies and adapt IDE for our application.
