# Cookbook - Annotated Routing
At this moment, Spiral Framework does not provide annotated-routes support out of the box like in Symfony. Combine extensions
`spiral/annotated` and `spiral/router` to build your domain-specific routing logic.

> Route parsing will take some time during the bootload phase of your application, but it won't affect the runtime
> performance. 

## Annotation
Build simple annotation which later can be attached to public controller methods:

```php
namespace App\Annotation;

use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation()
 * @Annotation\Target({"METHOD"})
 * @Annotation\Attributes({
 *      @Annotation\Attribute("action", type="string", required=true),
 *      @Annotation\Attribute("verbs", type="array"),
 * })
 */
class Route
{
    /** @var string */
    public $action;

    /** @var string[]|null */
    public $verbs;
}
```

> Default Web build of application enables annotation support by default.

We can use this annotation in our controllers as follows:

```php
namespace App\Controller;

use App\Annotation\Route;

class DemoController
{
    /**
     * @Route(action="/hello", verbs={"GET"})
     */
    public function hello()
    {
        return 'hello world';
    }
}
```

> You can use route patterns and parameters as described [here](/http/routing.md).

## Bootloader
Use `RoutesBootloader` to convert annotations into routes. Use class `Spiral\Annotations\AnnotationLocator`
to find available annotations across application codebase.

```php
namespace App\Bootloader;

use Spiral\Annotations\AnnotationLocator;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(AnnotationLocator $annotationLocator, RouterInterface $router): void
    {
        $methods = $annotationLocator->findMethods(\App\Annotation\Route::class);

        foreach ($methods as $method) {
            $name = sprintf(
                "%s.%s",
                $method->getClass()->getShortName(),
                $method->getMethod()->getShortName()
            );

            $route = new Route(
                $method->getAnnotation()->action,
                new Action(
                    $method->getClass()->getName(),
                    $method->getMethod()->getName()
                )
            );

            $route = $route->withVerbs(...(array)$method->getAnnotation()->verbs);

            $router->setRoute($name, $route);
        }
    }
}
```

Run CLI command to check the list of available routes:

```bash
$ php app.php route:list
```

> Use additional route parameters to configure middleware, common prefix, etc.
