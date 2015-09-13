# Schemas and Documenters
Both ORM and ODM components in Spiral framework work based on same principle - cached **behaviour schemas**. Schemas concept involves memory structure which used to decribe specific aspects of every model including validations, fields, fiters and etc. 

## Schemas as Memory
Generic idea behind using schemas is ability to separate heavy analysis and inheritance operations from light and simple runtime processes. Using `HippocampusInterface`, components are able to collect and process all possible information about database models (create model reflection) in backgroud. It can create automatic data filters, synchronize databases (ORM) or even detect errors and security issues in your code. Since we don't really have limitation on time in CLI mode, our analysis can be as complex and powerful as we want (for example you can decribe model behaviour using model code itself as you would do it in other frameworks).

Behaviour schemas can singficantly increace amount of supported features without slowing down (and sometime even increacing performance) of runtime code. [**Memory interface**] (/framework/memory.md) plays very big role in this case, as it works not as optional cache, but rather required part of component flow.

In addition to memory interface, other important piece of analysis process is `TokenizerInterface` which is able to locate required classes without user intervention.

You can count spiral schemas as **compilation** of your code before using it in runtime. Since compilation is required to make your models works and there is no efficient way (yet) to detect changes in model behaviours you must execute console command to update schema every time you change your database model schema (you can still freely add methods without rebuilding schema every time). Fortunatelly such command can be easy to accessed - `spiral update`, if you don't have other commands starting with "u" you can use shorter alias `spiral up`. 

## Documenters
Since every database model in spiral can be well described using it's schema (model reflection) we are able to help IDE or editor to understand structure of project (not framework) and create set of tooltips related to application code. Tooltips might include columns names, relations and magic setters/getters methods (no one want to remember all possible table columns, right?).

At this moment spiral is able to generate "virtual documentation" for PHPStorm IDE using set of "cloned" classes, such methodic is not very clean but it works and can be changed at any moment without affecting your application. If you would like to support other IDEs and editors or improve PHPStorm documenter, your help will be much appreciated.

On a side note, having easy assible schema of your database level might help to create automatic project documention, check [ODM UML] (/odm/uml.md) for example.

> I hope one day we can define universal format (JSON or XML) to describe **project specific** structures and dependencies and adapt IDE for our application.