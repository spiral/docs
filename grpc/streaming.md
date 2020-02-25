# GRPC - Streaming and Batching
































An ability to handle an incoming steam of data is essential in many application domains, such as IoC, analytics, social 
networking.

In some cases it is more efficient to handle incoming data in a format of batches instead of processing each item separately.
This article will show how to implement batch streaming on PHP using Spiral framework and GRPC handler using custom build
of RoadRunner.

> Attention, this article will require basic golang knowledge.

## PHP Processor
To distribute the load over all available application instances and ensure persistence use `spiral/jobs` and pass batch
to PHP using Queue Job. Simple job handler might look like:

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Jobs\JobHandler;

class BatchJob extends JobHandler
{
    /**
     * @param array $items
     */
    public function invoke(array $items): void
    {
        foreach ($items as $item) {
            dumprr($item); // do something
        }
    }
}
```

For example you can map data into batch insert into database.

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Jobs\JobHandler;
use Spiral\Prototype\Traits\PrototypeTrait;

class BatchJob extends JobHandler
{
    use PrototypeTrait;

    /**
     * @param array $items
     */
    public function invoke(array $items): void
    {
        $insert = $this->db->insert('items');
        $insert->columns('timestamp', 'payload');

        foreach ($items as $item) {
            $insert->values(
                new \DateTimeImmutable($item['createdAt']),
                $item['payload']
            );
        }

        $insert->run();
    }
}
```

> Make sure to enable queue dispatcher, read more about Jobs [here](/queue/configuration.md).

## Producer
The sample data will be produced using simple Golang application via GRPC stream. We can describe the proto as the following
in `proto/items.proto`:

```json
syntax = "proto3";

package items;

service ItemHandler {
    rpc Stream (stream ItemMessage) returns (JoinMessage) {
    }
}

message ItemMessage {
    string createdAt = 1;
    bytes payload = 2;
}

message JoinMessage {
    string id = 1;
}
```

> Make sure to read [gRPC Basics - Go](https://grpc.io/docs/tutorials/basic/go/). 

The producer code will locate in `producer` directory, server will locate in `server`. Generate the producer client code:

```bash
$ protoc proto/items.proto --go_out=plugins=grpc:.
$ cp proto/items.pb.go producer/items.pb.go

```
