# Stempler - AST Modifications
Stempler template engine fully exposes template AST (DOM) and provides API for the modifications similar to
https://github.com/nikic/PHP-Parser.

You can create magical (in both ways) workflows and helpers by implementing your Node Visitors.

> You can read more about how traversing works [here](https://github.com/nikic/PHP-Parser/blob/master/doc/2_Usage_of_basic_components.markdown#node-traversation).

## Create Visitor
To create AST visitor you must implement interface provided by Stempler engine - `Spiral\Stempler\VisitorInterface`. 
We will try to create visitor which automatically adds `alt` attribute to all `img` tags found in your templates:

```php
use Spiral\Stempler\Node\HTML;
use Spiral\Stempler\VisitorContext;
use Spiral\Stempler\VisitorInterface;

class AltImageVisitor implements VisitorInterface
{
    public function enterNode($node, VisitorContext $ctx)
    {
    }

    public function leaveNode($node, VisitorContext $ctx)
    {
        if ($node instanceof HTML\Tag && $node->name === 'img') {
            $alt = null;
            foreach ($node->attrs as $attr) {
                if ($attr->name === 'alt') {
                    $alt = $attr;
                    break;
                }
            }

            if ($alt === null) {
                $node->attrs[] = new HTML\Attr('alt', '"this is image alt"');
            }
        }
    }
}
```

## Register Visitor
Call `StemplerBootloader`->`addVistitor` to register visitor in the template engine. We can do it using application bootloader:

```php
namespace App\Bootloader;

use App\Visitor\AltImageVisitor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class AltImageBootloader extends Bootloader
{
    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addVisitor(AltImageVisitor::class);
    }
}
```

> You have clean view cache to view newly applied changes.

Now all the `img` tags will always include `alt` attribute.
