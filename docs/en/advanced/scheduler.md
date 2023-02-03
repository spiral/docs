# Advanced — Task Scheduling

The [spiral-packages/scheduler](https://github.com/spiral-packages/scheduler) package provides a simple API for running
cron job tasks by a schedule. It can be easily integrated with a project based on Spiral.

## Installation

To install the package, you can use the following command:

```terminal
composer require spiral-packages/scheduler
```

Once the package is installed, you need to add the bootloader from the package to your application:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Scheduler\Bootloader\SchedulerBootloader::class,
];
```

To run the schedule, you have two options:

1. Add a cron configuration entry to your server that runs the `schedule:run` command every minute.

```bash
* * * * * cd /path-to-your-project && php app.php schedule:run >> /dev/null 2>&1
```

2. Run the schedule via RoadRunner by using the `schedule:work` command. This command will run in the foreground and
   invoke the scheduler every minute until you terminate the command:

```terminal
php app.php schedule:work
```

You can use any supervisor to keep the process running in the background mode, such as supervisord.

## Usage

To use the Scheduler package in your project, you can create a new `SchedulerBootloader` bootloader.

> **Warning**
> Don't forget to register `SchedulerBootloader` in your application!

This bootloader will be responsible for managing cron job tasks:

```php app/src/Application/Bootloader/SchedulerBootloader.php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scheduler\Schedule;
use Psr\Log\LoggerInterface;
use Spiral\Boot\DirectoriesInterface;

final class SchedulerBootloader extends Bootloader
{
    public function boot(Schedule $schedule, DirectoriesInterface $dirs): void
    {
        // Run console command by name
        $schedule->command('ping', ['https://google.com'])
            ->everyFiveMinutes()
            ->withoutOverlapping()
            ->appendOutputTo($dirs->get('runtime').'logs/cron.log');
            
        // Run console command by class name
        $schedule->command(PingCommand::class, ['https://google.com'])
            ->everyFiveMinutes()
            ->withoutOverlapping()
            ->appendOutputTo($dirs->get('runtime').'logs/cron.log');
            
        // Run callable task
        $schedule->call('Ping url', static function (LoggerInterface $logger, string $url) {
            $headers = @get_headers($url);
            $status = $headers && \strpos($headers[0], '200');

            $logger->info(\sprintf('URL: %s %s', $url, $status ? 'Exists' : 'Does not exist'));

            return $status;
        }, ['url' => 'https://google.com'])
           ->everyFiveMinutes()
           ->withoutOverlapping();
    }
}
```

This example demonstrates how to use the Scheduler package to schedule different types of commands to run at specific
intervals and how to configure the schedule to prevent overlapping and log output.

Now you can check that cron job tasks are registered:

```terminal
php app.php schedule:list
```

```output
+--------------------------------------------------------+-------------+-------------+----------------------------+
|[32m Command                                                [39m|[32m Interval    [39m|[32m Description [39m|[32m Next Due                   [39m|
+--------------------------------------------------------+-------------+-------------+----------------------------+
| /usr/bin/php8.1 app.php ping 'https://google.com'      | */5 * * * * |             | 2023-01-12 12:55:00 +00:00 |
| /usr/bin/php8.1 app.php ping:site 'https://google.com' | */5 * * * * | Ping site   | 2023-01-12 12:55:00 +00:00 |
| callback: 'url'                                        | */5 * * * * | Ping url    | 2023-01-12 12:55:00 +00:00 |
+--------------------------------------------------------+-------------+-------------+----------------------------+
```

### Callbacks

You can use the `before`, `then`, `when`, and `skip` methods to register callbacks.

#### Before callback

The `before` method can be used to register a callback that will be called before the scheduled task is executed. This
callback can be used to perform any necessary preparation or setup before the task runs.

```php
$schedule->command('backup:run')
   ->everyFiveMinutes()
   ->before(static fn(Notifier $notifier) => $notifier->send('Starting backup...'));
   -> ...;
```

#### Then callback

The `then` method can be used to register a callback that will be called after the scheduled task is executed. This
callback can be used to perform any necessary cleanup or additional processing after the task runs.

```php
$schedule->command('backup:run')
   ->everyFiveMinutes()
   ->then(static fn(Notifier $notifier) => $notifier->send('Backup completed'));
   -> ...;
```

#### When callback

The `when` method can be used to register a callback that will be called to determine whether the scheduled task should
be
executed or not. This callback can be used to check for any constraining conditions that may prevent the task from
running. If the when callback returns `false`, the scheduled task will not be executed.

```php
$schedule->command('email:digest')
   ->everyFiveMinutes()
   ->when(static fn(HolidaysCalendar $calendar) => !$calendar->isHoliday());
   -> ...;
```

#### Skip callback

The `skip` method can be used to register a callback that will be called to determine whether the scheduled task should
be
skipped or not. This callback can be used to check for any conditions that would prevent the task from running. If the
skip callback returns `true`, the scheduled task will not be executed.

```php
$schedule->command('email:digest')
   ->everyFiveMinutes()
   ->skip(static fn(HolidaysCalendar $calendar) => $calendar->isHoliday());
   -> ...;
```

### Background tasks

You can run a task in the background by using the `runInBackground()` method:

```php
$schedule->command('ping', ['https://google.com'])
   ->everyFiveMinutes()
   ->runInBackground()
   -> ...;
```

> **Warning**
> Don't run callable tasks in the background. This will cause an error.

### Prevent tasks from overlapping

When `withoutOverlapping()` method is used, the package will check if the command or function is already running before
scheduling a new instance of it. If an instance of the command or function is already running, the new instance will
not be scheduled and will be skipped. This helps to prevent multiple instances of the same command or function from
running simultaneously and potentially causing issues.

```php
$schedule->command('ping', ['https://google.com'])
   ->everyMinute()
   ->withoutOverlapping()
   -> ...;
```

In addition to the default behavior of the method, you can also specify a custom lock expiration time in minutes. This
can be done by passing an integer value as an argument to the method.

```php
$schedule->command('ping', ['https://google.com'])
   ->everyMinute()
   ->withoutOverlapping(60)
   -> ...;
```

> **Note**
> By default, the lock expiration time is 24 hours.

### Cron expressions

You can use the expression argument to assign a custom cron expression to a task.

For example, the following code assigns a cron expression of `* * * * *` (which corresponds to running every minute) to
the email:send command:

```php
$schedule->command('email:send', expression: new \Cron\CronExpression('* * * * *'));
```

Additionally, you can use helper methods to define your schedule in a more human-readable way. These methods include:

#### Minutes

| Method                  | Description                           |
|-------------------------|---------------------------------------|
| `everyMinute()`         | Every minute: `* * * * *`             |
| `everyEvenMinute()`     | Every even minute: `*/2 * * * *`      |
| `everyFiveMinutes()`    | Every five minutes: `*/5 * * * *`     |
| `everyTenMinutes()`     | Every ten minutes: `*/10 * * * *`     |
| `everyFifteenMinutes()` | Every fifteen minutes: `*/15 * * * *` |
| `everyThirtyMinutes()`  | Every thirty minutes: `0,30 * * * *`  |

> **Note**
> More examples of how to schedule a task to run at specific minutes you can
> find [here](https://github.com/butschster/CronExpressionGenerator#manipulate-hours).

#### Hours

| Method                 | Description                                          |
|------------------------|------------------------------------------------------|
| `hourly()`             | Every hour at 00 minutes: `0 * * * *`                |
| `hourlyAt(15)`         | Every hour at the specified minute: `15 * * * *`     |
| `hourlyAt(15, 30, 45)` | Every hour at 15, 30, 45 minutes: `15,30,45 * * * *` |
| `everyTwoHours()`      | Every day at 00:00: `0 0 * * *`                      |

> **Note**
> More examples of how to schedule a task to run on at hourly basis you can
> find [here](https://github.com/butschster/CronExpressionGenerator#manipulate-hours).

#### Days

| Method                       | Description                                              |
|------------------------------|----------------------------------------------------------|
| `daily()`                    | Every day at 00:00: `0 0 * * *`                          |
| `daily(13)`                  | Every day at the specified time: `0 13 * * *`            |
| `daily(3, 15, 23)`           | Every day at 03:00, 15:00, 23:00: `0 3,15,23 * * *`      |
| `twiceDaily(1, 13)`          | Every day at 01:00, 13:00: `0 1,13 * * *`                |
| `lastDayOfMonth()`           | Every month on the last day at 00:00: `0 0 L * *`        |
| `lastDayOfMonth(12)`         | Every month on the last day at 12:00: `0 12 L * *`       |
| `lastDayOfMonth(12, 30)`     | Every month on the last day at 12:30: `30 12 L * *`      |
| `lastWeekdayOfMonth()`       | Every month on the last weekday at 00:00: `0 0 LW * *`   |
| `lastWeekdayOfMonth(12)`     | Every month on the last weekday at 12:00: `0 12 LW * *`  |
| `lastWeekdayOfMonth(12, 30)` | Every month on the last weekday at 12:30: `30 12 LW * *` |

> **Note**
> More examples of how to schedule a task to run on an daily basis you can
> find [here](https://github.com/butschster/CronExpressionGenerator#manipulate-days).

#### Days of week

| Method     | Description                       |
|------------|-----------------------------------|
| `weekly()` | Every week on monday: `0 0 * * 0` |

> **Note**
> More examples of how to schedule a task to run on at weekly basis you can
> find [here](https://github.com/butschster/CronExpressionGenerator#manipulate-days-of-week).

### Months

| Method                  | Description                                               |
|-------------------------|-----------------------------------------------------------|
| `monthly()`             | Every month on 1-st day at 00:00: `0 0 1 * *`             |
| `monthly(12)`           | Every month on 1-st day at 12:00: `0 12 1 * *`            |
| `monthly(12, 30)`       | Every month on 1-st day at 12:30: `30 12 1 * *`           |
| `monthlyOn(15, 12)`     | Every month on 15-st day at 12:00: `0 12 15 * *`          |
| `monthlyOn(15, 12, 30)` | Every month on 15-st day at 12:30: `30 12 15 * *`         |
| `quarterly()`           | Every quarter yyyy-01,03,06,09-01 00:00: `0 0 1 1-12/3 *` |
| `yearly()`              | Every year yyyy-01-01 00:00: `0 0 1 1 *`                  |

> **Note**
> More examples of how to schedule a task to run on at monthly basis you can
> find [here](https://github.com/butschster/CronExpressionGenerator#manipulate-months).

## Console commands

#### List the scheduled jobs

Use the `schedule:list` command to list all of the scheduled jobs:

```terminal
php app.php schedule:list
```

#### Start the schedule worker

Use the `schedule:work` command to start the schedule worker:

```terminal
php app.php schedule:work
```