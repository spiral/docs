# Embedded Commands
Spiral provides big set of commands you might use if your application:

Command            | Description                                                              
---                | ---                                                                      
configure          | Configure file permissions, install modules and render view file             
register           | Register module configs and publish it's resources                           
publish            | Publish specific module resources                                            
server             | Run Spiral Development server on specified host and port                 
update             | Perform application schemas and cache update                             
console:reload     | **Reindex console commands (run after creating command in application)** 
app:key            | Update encryption key for current environment                            
app:reload         | Reload application boot-loading list *(only when cache is enabled)*      
app:extensions     | Get list of available php extensions
app:clean          | Clean application runtime cache                  
db:describe        | Describe table schema of specific database                               
db:list            | Get list of available databases, their tables and records count      
migrate:init       | Init migrations component (create migrations table)
migrate:replay     | Replay (down, up) one or multiple migrations
migrate:rollback   | Rollback one (default) or multiple migrations
migrate:status     | Get list of all available migrations and their statuses
i18n:dump          | Dump given locale using specified dumper and path                        
i18n:reload        | Force Translator to reload locales                                       
i18n:index         | Index all declared translation strings and usages                        
orm:schema         | Update ORM schema.                                                       
views:compile      | Compile every available view file                                        
views:reset        | Clear view cache for all environments                                    

> You can get list of currently available commands by calling `ConsoleDisptacher->getCommands()`.

You can disable automatic command location and define your own set of commands in console config.
