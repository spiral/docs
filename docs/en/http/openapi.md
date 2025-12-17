# HTTP — OpenAPI Documentation

Spiral Framework provides robust support for generating OpenAPI (Swagger) documentation from your PHP code using
attributes. This integration enables automatic API documentation that stays synchronized with your codebase, reducing
maintenance overhead and improving developer experience.

## Installation

The OpenAPI package is available through Composer:

```terminal
composer require spiral-packages/swagger-php
```

After installation, register the bootloader in your application:

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\OpenApi\Bootloader\SwaggerBootloader::class,
        // ...
    ];
}
```

> **See also**
> Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.

## Configuration

Configure OpenAPI documentation through the application configuration file or directly in a bootloader.

### Basic Configuration

Create or modify `app/config/swagger.php`:

```php app/config/swagger.php
<?php

declare(strict_types=1);

use Spiral\OpenApi\Generator\Parser\ConfigurationParser;
use Spiral\OpenApi\Generator\Parser\OpenApiParser;
use Spiral\OpenApi\Renderer\HtmlRenderer;
use Spiral\OpenApi\Renderer\JsonRenderer;
use Spiral\OpenApi\Renderer\YamlRenderer;

return [
    'documentation' => [
        'info' => [
            'title' => 'My API',
            'description' => 'Complete API documentation',
            'version' => '1.0.0',
        ],
        'servers' => [
            ['url' => 'https://api.example.com', 'description' => 'Production'],
            ['url' => 'http://localhost:8080', 'description' => 'Development'],
        ],
    ],
    
    'parsers' => [
        ConfigurationParser::class,
        OpenApiParser::class,
    ],
    
    'renderers' => [
        JsonRenderer::FORMAT => JsonRenderer::class,
        YamlRenderer::FORMAT => YamlRenderer::class,
        HtmlRenderer::FORMAT => HtmlRenderer::class,
    ],
    
    'paths' => [
        directory('app') . '/src',
    ],
    
    'exclude' => null,
    'pattern' => '*.php',
    'version' => null,
    'cache_key' => 'swagger_docs',
    'use_cache' => env('DEBUG', false) === false,
    
    'generator_config' => [
        'operationId' => [
            'hash' => true,
        ],
    ],
];
```

### Configuration Options

| Option             | Type                  | Description                                                                |
|--------------------|-----------------------|----------------------------------------------------------------------------|
| `documentation`    | `array`               | OpenAPI specification base configuration (info, servers, security schemes) |
| `parsers`          | `array`               | List of parser classes to extract OpenAPI annotations                      |
| `renderers`        | `array`               | Output format renderers (JSON, YAML, HTML)                                 |
| `paths`            | `array`               | Directories to scan for OpenAPI attributes                                 |
| `exclude`          | `string\|array\|null` | Paths or patterns to exclude from scanning                                 |
| `pattern`          | `string\|null`        | File pattern to match during scanning                                      |
| `version`          | `string\|null`        | OpenAPI specification version                                              |
| `cache_key`        | `string`              | Cache key for storing generated documentation                              |
| `use_cache`        | `bool`                | Enable caching (recommended for production)                                |
| `generator_config` | `array`               | Configuration for OpenAPI generator behavior                               |

### Configuring via Bootloader

You can also configure OpenAPI programmatically in a bootloader:

```php app/src/Application/Bootloader/OpenApiBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\OpenApi\Bootloader\SwaggerBootloader;

final class OpenApiBootloader extends Bootloader
{
    public function boot(SwaggerBootloader $swagger): void
    {
        // Add additional scan paths
        $swagger->addPath(directory('app') . '/src/Endpoint');
        $swagger->addPath(directory('app') . '/src/Controller');
    }
}
```

## Routing Setup

To expose OpenAPI documentation endpoints, configure routes in your `RoutesBootloader`:

```php app/src/Application/Bootloader/RoutesBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\OpenApi\Controller\DocumentationController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        // HTML documentation
        $routes
            ->add('swagger-ui', '/api/docs')
            ->action(DocumentationController::class, 'html');

        // JSON specification
        $routes
            ->add('swagger-json', '/api/docs.json')
            ->action(DocumentationController::class, 'json');

        // YAML specification
        $routes
            ->add('swagger-yaml', '/api/docs.yaml')
            ->action(DocumentationController::class, 'yaml');
    }
}
```

### Conditional Documentation Routes

For production environments, you may want to disable documentation routes conditionally:

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Boot\EnvironmentInterface;

final class RoutesBootloader extends BaseRoutesBootloader
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        // Only register documentation routes in non-production environments
        if ($this->env->get('APP_ENV') !== 'production') {
            $routes
                ->add('swagger-ui', '/api/docs')
                ->action(DocumentationController::class, 'html');

            $routes
                ->add('swagger-json', '/api/docs.json')
                ->action(DocumentationController::class, 'json');
        }
    }
}
```

## Custom Parsers

Create custom parsers to extract documentation from additional sources:

```php app/src/OpenApi/CustomParser.php
<?php

declare(strict_types=1);

namespace App\OpenApi;

use OpenApi\Analysis;
use OpenApi\Annotations\OpenApi;
use Spiral\OpenApi\Generator\Parser\ParserInterface;

final class CustomParser implements ParserInterface
{
    public function parse(OpenApi $openApi, Analysis $analysis): void
    {
        // Custom parsing logic
        // Modify $openApi or $analysis as needed
    }
}
```

Register the custom parser:

```php app/config/swagger.php
use App\OpenApi\CustomParser;

return [
    'parsers' => [
        ConfigurationParser::class,
        OpenApiParser::class,
        CustomParser::class,
    ],
];
```

## Caching

### Cache Configuration

OpenAPI documentation generation can be resource-intensive. Enable caching in production:

```php app/config/swagger.php
return [
    'use_cache' => env('APP_ENV') === 'production',
    'cache_key' => 'swagger_docs',
];
```

### Clearing Cache

Clear OpenAPI cache when documentation changes:

```terminal
php app.php cache:clear swagger_docs
```

Or programmatically:

```php
use Psr\SimpleCache\CacheInterface;

public function clearDocs(CacheInterface $cache): void
{
    $cache->delete('swagger_docs');
}
```

> **See also**
> * [Routing](routing.md) - Learn about route configuration
