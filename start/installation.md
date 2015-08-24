# Installation and Requiments
Spiral Framework uses **Composer** to resolve dependencies and components. Check how to install composer 
[here] (https://getcomposer.org/download/).

## Requiments
Spiral Framework has following server requirements:
* PHP 5.5+
* OpenSSL Extension
* MbString Extension
* Tokenizer Extension

## Installation
One of easiest way to install fresh spiral application is using composer command:
`php composer.phar create-project spiral/application`

Right after installation framework will execute console command `configure` to ensure that all needed 
directories has correction permissions and available for application (you can register you own
commands in configure sequence, see **Console and CLI Mode**).

## Environment
By default spiral application will define it's enviroment name using data located in "application/runtime/enviroment.php"
file (default application logic will use enviroment value to alter component configuration files to define different 
behaviours). 

When this file not set or application just installed spiral will force "development" enviroment. To change enviroment simply
execute `spiral environment {VALUE}` command.