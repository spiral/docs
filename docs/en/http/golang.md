# HTTP - Golang Middleware

You can implement some middleware on Golang end, inside the application server. Such an approach makes it possible to
handle high-throughput and filter requests before their arrival to your PHP application. It is the best spot to
implement rate-limiters, geolocation filters, and other types of middleware.

> **Note**
> Make sure to read how to build [application server](/framework/application-server.md) first.

## Passing values from Golang to PHP

To implement more complex logic, you would need to pass middleware specific values into your PHP application safely.
You can not use headers for this purpose, as it will be possible to hijack values from the client's end. Instead, use
`github.com/spiral/roadrunner/service/http/attributes` package to set PSR-7 attribute on your request:

## RoadRunner Service Middleware

For example, we can detect if the incoming request made by a bot and notify the underlying PHP application about that.
We will use `https://github.com/avct/uasurfer` package. Create a service to register our middleware:

```golang
package bdetect

import (
	"encoding/json"
	"fmt"
	"github.com/avct/uasurfer"
	rhttp "github.com/spiral/roadrunner/service/http"
	"github.com/spiral/roadrunner/service/http/attributes"
	"net/http"
)

const ID = "bdetect"

type Service struct{}

func (s *Service) Init(rhttp *rhttp.Service) (bool, error) {
	rhttp.AddMiddleware(s.middleware)

	return true, nil
}

func (s *Service) middleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ua := uasurfer.Parse(r.Header.Get("User-Agent"))
		data, _ := json.Marshal(struct {
			Browser        string `json:"browser"`
			BrowserVersion string `json:"browserVersion"`
			IsBot          bool   `json:"isBot"`
		}{
			Browser: ua.Browser.Name.StringTrimPrefix(),
			BrowserVersion: fmt.Sprintf(
				"%v.%v.%v",
				ua.Browser.Version.Major,
				ua.Browser.Version.Minor,
				ua.Browser.Version.Patch,
			),
			IsBot: ua.IsBot(),
		})

		attributes.Set(r, "bdetect", string(data))

		f(w, r)
	}
}
```

To activate the service in `main.go`:

```golang
func main() {
    // ...

	rr.Container.Register(bdetect.ID, &bdetect.Service{})

	// you can register additional commands using cmd.CLI
	rr.Execute()
}
```

## PHP Middleware

We can consume these PSR-7 attributes in PHP Middleware. Use `php .\app.php create:middleware botDetect` to quickly
scaffold it:

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class BotDetect implements MiddlewareInterface
{
    use PrototypeTrait;

    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        $ua = json_decode($request->getAttribute('bdetect'));

        if ($ua->isBot) {
            return $this->response->html('Not bots allowed', 401);
        }

        return $handler->handle($request);
    }
}
```

Create and register Bootloader to activate the middleware:

```php
namespace App\Bootloader;

use App\Middleware\BotDetect;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http): void
    {
        // parse json payloads
        $http->addMiddleware(BotDetect::class);
    }
}
```
