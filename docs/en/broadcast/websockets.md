# Broadcast - WebSockets
The default GRPC and Web bundles include a pre-build component to gain access to the broadcast topic via WebSocket client.
To activate the component, enable the `ws` section `.rr.yaml` config file with the desired path:

```yaml
ws.path: "/ws"
```

Enable the `Spiral\Bootloader\Http\WebsocketsBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Http\WebsocketsBootloader::class,
    // ...
];
```

> The HTTP component is required for WebSockets to function.

## Spiral Framework WebSocket client

Recommended way to consume broadcast events is to use Spiral Framework WebSocket client

### Installation

SFSocket available for installing with npm or yarn

```bash
npm install @spiralscout/websockets -D  
```

```bash
yarn add @spiralscout/websockets 
```

Next use it like so

```js
    import { SFSocket } from '@spiralscout/websockets';
```

Alternatively download [socket.js bundle file](https://github.com/spiral/websocket-client/tree/master/build) and use it directly 

```html
    <script src="/build/socket.js"></script>
    <script type="text/javascript">
        var Socket = SFSocket.SFSocket;
        var connection = new Socket({ host: 'localhost'});
    </script>
```

### Usage sample for broadcast events

```js
import { SFSocket } from '@spiralscout/websockets';

const socketOptions = { host: 'localhost' };

// create an instance of SFSocket
const ws = new SFSocket(socketOptions);

const prepareEvent = event => doSomething(event);

// subscribe to server
ws.subscribe('message', prepareEvent);

// runtime ready for all instances
SFSocket.ready();

// unsubscribe from server 
ws.unsubscribe('message', prepareEvent);

// disconnect from server 
ws.disconnect();
```

### Usage sample to listen a specific channel

```js
import { SFSocket } from '@spiralscout/websockets';

const socketOptions = { host: 'localhost' };

const ws = new SFSocket(socketOptions);

SFSocket.ready();

// create a channel and it is automatically connected to server
const channel1 = ws.joinChannel('channel_1');
const channel2 = ws.joinChannel('channel_2', true); // This one wont auto-join now

// subscribe the channel to server 
channel1.subscribe('message', (event) => doSomething(event));
channel2.subscribe('message', (event) => doSomething(event));

channel2.join(); // Start receiving messages for channel2 

// disconnect the channel from server 
channel1.leave();
channel2.leave();

// disconnect everything
ws.disconnect()
```

### Detailed API

Detailed API description is [available on GitHub](https://github.com/spiral/websocket-client) 

## Native JavaScript Client
Use native JavaScript websockets functionality to connect to the server on `/ws`:


```html
<div id="messages">

</div>
<script type="text/javascript">
    window.addEventListener("load", function (evt) {
        let messages = document.getElementById("messages");

        let print = function (message) {
            messages.innerText += message + "\n";
        };

        let ws = new WebSocket('ws://localhost:8080/ws');

        ws.onopen = function (evt) {
            print("open");
        };

        ws.onclose = function (evt) {
            print("close");
        };

        ws.onmessage = function (evt) {
            print("message: " + evt.data);
        };

        ws.onerror = function (evt) {
            print("error: " + evt.data);
        };
    });
</script>
```

> You should observe `DEBU[0003] [ws] [::1]:51350 connected` if everything is configured properly.

## Consume Topic
To consume topic using JavaScript client your must send command message in a form of `{"topic":"join", "payload": ["topic-name"]}`.
Modify your script to automatically consume `topic`:

```html
<script type="text/javascript">
    window.addEventListener("load", function (evt) {
        // ...

        ws.onopen = function (evt) {
            print("open");
            ws.send(`{"topic":"join", "payload":["topic"]}`);
        };

       // ...
    });
</script>
```

> Use command `leave` to leave the topic. You can consume multiple topics at the same time.

Currently, your WebSocket connection will be interrupted by the server, since the client does not have the permission to consume
the topic.

### Authorize Topic
To authorize the topic, create the bootloader and register topic handler using `WebsocketsBootloader`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\WebsocketsBootloader;

class TopicsBootloader extends Bootloader
{
    public function boot(WebsocketsBootloader $ws)
    {
        $ws->authorizeTopic(
            'topic',
            function () {
                // must return true for authorized topics
                return true;
            }
        );
    }
}
```

> You should observe the confirmation message `{"topic":"@join", "payload":["topic"]}` after the connection is made.

Method `authorizeTopic` support method injection and can be used to access the container. Use it to read client cookies:

```php
$ws->authorizeTopic(
    'topic',
    function (ServerRequestInterface $request) {
        dumprr($request->getCookieParams());
            
        // must return true for authorized topics
        return true;
    }   
);
```

The `authorizeTopic` method support pattern based authorization, use `{}` to embrace dynamic parameters. Request parameters
via method injection:

```php
$ws->authorizeTopic(
    'topic.{id}',
    function ($id, Actor $actor) {           
        return $actor->getID() === $id;
    }   
);
```

## Publish Messages
You can publish messages to WebSockets using the default broadcast functionality. For example, we can use the console command:

```php
namespace App\Command;

use Spiral\Broadcast\BroadcastInterface;
use Spiral\Broadcast\Message;
use Spiral\Console\Command;
use Symfony\Component\Console\Input\InputArgument;

class PostCommand extends Command
{
    protected const NAME      = 'post';
    protected const ARGUMENTS = [
        ['message', InputArgument::REQUIRED]
    ];

    public function perform(BroadcastInterface $broadcast)
    {
        $broadcast->publish(new Message('topic', $this->argument('message')));
    }
}
```

> Run `php app.php post hello` to observe messages on a page.
