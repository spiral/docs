# Queue and Jobs - Installation and Configuration
Web and GRPC bundles of Spiral Framework support background PHP processing and a queue out of the box. You are able
to work with one or multiple message brokers such as Beanstalk, AMQP (RabbitMQ) or Amazon SQS.

In order to install the extensions in alternative bundles:

```bash
$ composer require spiral/jobs
```

Make sure to add `Spiral\Bootloader\Jobs\JobsBootloader` to your application kernel.

## Configuration
Jobs extension does not require configuration on application end, however, you must specify broken connections and
available queue pipelines in `.rr` file. The extension configuration must be located in the `jobs` section:

```yaml
jobs:
  # php endpoint to run
  workers.command: "php app.php"

  # pipeline options
  pipelines:
    local.broker: "ephemeral"

  # pipelines to consume
  consume: ["local"]  

  # job type associated with its dispatch options
  dispatch:
    app-job-*.pipeline: "local"
```

## Pipelines
Every issues job must be placed into a designated queue pipeline. Spiral is able to push and consume tasks from multiple
pipelines, however, you must clearly outline which pipelines are available.

Each pipeline has to be associated with a queue broker:

```yaml
pipelines:
  local:
    broker: "ephemeral"
``` 

You are able to use shorter declaration when only one YAML value is presented:

```yaml
pipelines:
  local.broker: "ephemeral"
```

Some brokers will require you to specify additional pipeline options, specific to the implementation. For example Amazon SQS:

```yaml
pipelines:
  pipeline-name:
    broker: sqs
    queue:  default
    declare:
      MessageRetentionPeriod: 86400
```

> See below.

## Dispatching
Each created job must be assigned to a pipeline. You can either specify pipeline directly in your application or let
application server resolve it automatically. 

Use the `dispatch` option to specify how to assign job or job namespace with specific pipeline and/or additional options:

```yaml
dispatch:
  app-job-ping.pipeline: "local"
```

> Job names are calculated automatically based on handler class name, namespace separator replaced with `-`. 

We can use `*` to dispatch multiple jobs into a single pipeline, for example, to dispatch all jobs from namespace `App\Job`:

```yaml
dispatch:
  app-job-*.pipeline: "local"
```

> You can read more about dispatching jobs in the following sections.

You can use wildcard dispatching in combination with the direct association:

```yaml
dispatch:
  app-job-ping.pipeline: "pings" # send all App\Job\Ping jobs into `pings`
  app-job-*.pipeline:    "local" # default fallback
```

## Consuming
Use option `consume` of jobs service to specify which pipelines much be consumed by the local application server. In some cases,
you might want to disable consuming on a given instance to offload tasks to remove machine.

```yaml
consume: ["local"]  
```

> You can always start and stop pipeline consuming via CLI command.

## Local Pipeline
One of the pipelines extremely useful in application development is pipelines associated with `ephemeral` broker.
This pipelines does not require an external broker and can run directly in application server memory. Use this pipeline
to run non-critical background tasks.

> Note, `ephemeral` broker is **not reliable**, any server failure will erase application server memory and your jobs
> will be lost. Use it in development or for non-critical tasks.

## Brokers
You must specify connection options for all of the brokers except ephemeral. 

### AMQP (RabbitMQ)
To enable [AMQP](https://www.amqp.org/) broker you have to specify AMQP dsn in `jobs` config:

```yaml
jobs:
  amqp.addr: amqp://guest:guest@localhost:5672/

  # ...
```

The pipelines must be assigned to `amqp` broker and must specify the `queue` name:

```php
pipelines:
  my-queue:
    broker: amqp
    queue:  default
```

Additional options are supported:

Option   | Default                      | Comment
---      | ---                          | ---
exchange | `amqp.direct`                | Exchange type
consumer | rr-**pipeline-name**-**pid** | Consumer ID

> Queue will be created automatically if not exists.

### Beanstalk
To enable [Beanstalk](https://beanstalkd.github.io/) broker:

```yaml
jobs:
  beanstalk.addr: tcp://localhost:11300
```

Each Beanstalk pipeline requires a `tube` option:

```yaml
pipelines:
  beanstalk:
    broker: beanstalk
    tube:   default
```

Additional options are supported:

Option   | Default  | Comment
--      | ---      | ---
reserve  | 1        | How long to wait for an incoming job until re-sending request (long-pulling). 

### Amazon SQS
To enable [Amazon SQS](https://aws.amazon.com/en/sqs/) broker:


```yaml
jobs:
  sqs:
    key:      api-key
    secret:   api-secret
    region:   us-west-1
```

Each SQS pipeline require `queue` option pointing to SQS queue name:

```yaml
pipelines:
  sqs:
    broker: sqs
    queue:  default        
```

Broker is able to create queue automatically if `declare` option set:

```yaml
pipelines:
  sqs:
    broker: sqs
    queue:  default
    declare:
      MessageRetentionPeriod: 86400
```

You can find a list of available declare options [here](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html).

> Note, Amazon SQS does not support jobs delayed for longer than 15 minutes.
