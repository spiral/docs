# Framework - Config Objects
Spiral framework exposes all the underlying configuration of it's components using config objects. The core purpose of config
objects is to separate the bootloading phase and runtime phase and provide easily accessible source of configuration.

![Application Control Phases](https://user-images.githubusercontent.com/796136/64906478-e213ff80-d6ef-11e9-839e-95bac78ef147.png)

While configuration can change during the bootload process, at runtime all the values are frozen and forbidden for change. In 
addition to that you are also able to request any config as dependency injection for the application container.

## Create the Config

