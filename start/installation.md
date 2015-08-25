# Installation and Requiments
Spiral Framework uses **Composer** to resolve dependencies and components. Check how to install composer 
[here] (https://getcomposer.org/download/).

## Requiments
Spiral Framework has the following server requirements:
* PHP 5.5+
* OpenSSL Extension
* MbString Extension
* Tokenizer Extension

## Installation
One of fastest ways to install a fresh spiral application is to use the composer command:
`php composer.phar create-project spiral/application`

Right after installation, the Spiral framework will execute the console command `configure` to ensure that all neccesary 
directories have the correct permissions and are available for application (you can register you own
commands in configure sequence, see **Console and CLI Mode**).

## Environment
By default, the Spiral application will define it's enviroment name using data located in "application/runtime/enviroment.php"
file (default application logic will use enviroment value to alter component configuration files to define different 
behaviours). 

When this file isn't set or the application is just installed, spiral will force "development" enviroment. To change enviroment simply
execute `spiral environment {VALUE}` command.
