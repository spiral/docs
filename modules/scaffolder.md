./composer.json has been updated
Loading composer repositories with package information
Updating dependencies (including require-dev)
  - Installing spiral/scaffolder (v0.9.0)
    Loading from cache

Writing lock file
Generating autoload files

wolfy@WOLFY-PC D:\WebServer\domains\application.dev
> spiral register spiral/scaffolder
Following configs are being altered:
+-----------+-------------+-------------------------------------------------------+
| Config    | Section     | Added Lines                                           |
+-----------+-------------+-------------------------------------------------------+
| tokenizer | directories | directory('libraries') . 'spiral/scaffolder/source/', |
+-----------+-------------+-------------------------------------------------------+

Confirm module registration (y/n) y

Module 'Spiral\ScaffolderModule' has been successfully registered.
Module 'Spiral\ScaffolderModule' has been successfully published.

> spiral console:reload
Console commands re-indexed, 35 commands found