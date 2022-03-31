# Queue and Jobs - Console Commands
The application server provides multiple commands used to control the `spiral/jobs` extension.

## Start and Stop
You can start the extension and init connection to brokers using the default `serve` command, `-v` and `-d` are supported:

```bash
$ ./spiral serve -v -d
```

The application server can be stopped via CTR+C sequence or by calling `./spiral stop`.

To reset jobs worker pool without restarting the server:

```bash
$ ./spiral jobs:reset
```

## Statistics
To view the list of active job workers:

```bash
$ ./spiral jobs:workers
```

To view the stats in interactive mode use flag `-i`:

```
+---------+-----------+---------+---------+--------------------+
|   PID   |  STATUS   |  EXECS  | MEMORY  |      CREATED       |
+---------+-----------+---------+---------+--------------------+
|    8444 | ready     |       0 | 21 MB   | 1 minute ago       |
|    3672 | ready     |       0 | 21 MB   | 1 minute ago       |
+---------+-----------+---------+---------+--------------------+
```

To view active pipelines and jobs:

```bash
$ ./spiral jobs:stat
```

Use flag `-i` to run the stats in an interactive mode:

```
+----------+-----------+----------+-------+---------+--------+
| PIPELINE |  BROKER   |   NAME   | QUEUE | DELAYED | ACTIVE |
+----------+-----------+----------+-------+---------+--------+
| local    | ephemeral | :memory: | 0     | 0       | 0      |
+----------+-----------+----------+-------+---------+--------+
```

## Pipelines
You to start or pause consuming any of the pipeline using commands:

```bash
$ ./spiral jobs:resume {pipeline-1} {pipeline-2}
$ ./spiral jobs:pause {pipeline-1} {pipeline-2}
```

> You can omit name(s) to resume/pause all pipelines. 
