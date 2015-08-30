# Directory structure
Spiral appliction does not require any specific stucture or namespace for it's files until every application class can be loaded by composer, hovewer spiral
scaffoling and cache directories comes pre-configured. Let's review default directory structure:

| Directory                         | Description                    
| ---                               | ---       
| /                                 | Root project directory, aliases under "root" directory and accessinble via `Core->directory()` or `directory()` functions.
| **/application/**                 | This is your based application directory with templates, classes, migrations and memory (runtime) directory, alias "application".
| /application/classes/             | Directory to locate your application classes, by default marked as PSR4 root (no namespace).                            
| /application/classes/Commands/    | You can create your console commands here (manually or via scaffoling) given them namespace "Commands";                            
| /application/classes/Controllers/ | Application frontend controllers, default namespace "Controllers".                          
| /application/classes/Database/    | Database entities (ORM and ODM) or data models, default namespace "Database".                          
| /application/classes/Middlewares/ | Application specific HttpDispatcher middlewares, default namespace "middlewares". Do not forget to register middleware in http config.
| /application/classes/Requests/    | RequestFilters used to read request data, validate it and map it into appropriate data entity field. Default namespace "Requests".    
| /application/classes/Services/    | Service layer classes, used to manage you data entities creation, saving and fetching and much more, namespace "Services".       
| **/application/config/**          | Configuration files used by components.                           
| /application/migrations/          | Default location for migrations                            
| /application/runtime/             | Application data directory, can be created automatically via `configure` command and store dynamic data, aliases as "runtime".
| /application/runtime/cache/       | Directory to store application memory files view cache and etc. Aliases under "cache".         
| /application/runtime/i18n/        | Translator will store string bundle files here.                            
| /application/runtime/logging/     | Default directory to store application and error logs and exception shapshots.                       
| /application/views/               | View templates used by ViewManager to generate View instances. Just views.                           
| /vendor/                          | Composer vendor directory, aliased under "libraries" directory.              
| **/webroot/**                     | Application public directory, application enterpoint (index.php) located here as any other public asset file. Aliased as "public" directory.
