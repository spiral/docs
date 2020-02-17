# Broadcast - WebSockets
The default GRPC and Web bundles include pre-build component to gain access to the broadcast topic via WebSocket client.
To activate the component enable the `ws` section `.rr.yaml` config file with the desired path:

```yaml
ws.path: "/ws"
```

Enable the `Spiral\Bootloader\Broadcast\WebsocketsBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Http\WebsocketsBootloader::class,
    // ...
];
```

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
To consume topic using JavaScript client your must send command message in a form of `{"cmd":"join", "args": ["topic-name"]}`.
Modify your script to automatically consume `topic`:

```html
<script type="text/javascript">
    window.addEventListener("load", function (evt) {
        // ...

        ws.onopen = function (evt) {
            print("open");
            ws.send(`{"cmd":"join", "args":["topic"]}`);
        };

       // ...
    });
</script>
```

> Use command `leave` to leave the topic. You are able to consume multiple topics at the same time.

Currently, your websocket connection will be interrupted by the server, since client does not have the permission to consume
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

> You should observe the confirmation message `{"topic":"@join","payload":["topic"]}` after the connection is made.

Method `authorizeTopic` support method injection and can be used to access container. Use it to read client cookies:

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
You can publish messages to websockets using the default broadcast functionality. For example, we can use the console command:

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

> Run `php app.php post hello` to observe message on a page.
