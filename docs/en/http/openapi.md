# HTTP — OpenAPI Documentation

Spiral Framework provides seamless integration for generating OpenAPI (Swagger) documentation from your PHP code using
attributes. The documentation is automatically generated from your controllers, filters, and response classes, staying
in sync with your actual implementation.

## Installation

Install the package via Composer:

```terminal
composer require spiral-packages/swagger-php
```

Register the bootloader:

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

Create configuration file `app/config/swagger.php`:

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
            'description' => 'API documentation',
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

### Adding Scan Paths

You can add additional scan paths programmatically:

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
        $swagger->addPath(directory('app') . '/src/Endpoint');
    }
}
```

## Routing

Configure routes to expose documentation endpoints:

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
        $routes
            ->add('swagger-ui', '/api/docs')
            ->action(DocumentationController::class, 'html');

        $routes
            ->add('swagger-json', '/api/docs.json')
            ->action(DocumentationController::class, 'json');

        $routes
            ->add('swagger-yaml', '/api/docs.yaml')
            ->action(DocumentationController::class, 'yaml');
    }
}
```

### Conditional Routes

Disable documentation in production:

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Boot\EnvironmentInterface;

final class RoutesBootloader extends BaseRoutesBootloader
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        if ($this->env->get('APP_ENV') !== 'production') {
            $routes
                ->add('swagger-ui', '/api/docs')
                ->action(DocumentationController::class, 'html');
        }
    }
}
```

## Basic Usage

OpenAPI documentation is generated from PHP attributes placed on your controllers and filters. The package uses
the [swagger-php](https://github.com/zircote/swagger-php) library attributes for defining the API specification.

> **Note**
> Refer to the [swagger-php documentation](https://zircote.github.io/swagger-php/) for detailed information about
> available attributes and their properties.

### Controller Documentation

Add OpenAPI attributes to your controller methods:

```php app/src/Endpoint/Web/UserController.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Web;

use OpenApi\Attributes as OA;
use Spiral\Router\Annotation\Route;

#[OA\Tag(name: 'Users', description: 'User management')]
class UserController
{
    #[Route(route: '/api/users', name: 'users.list', methods: 'GET')]
    #[OA\Get(
        path: '/api/users',
        operationId: 'users.list',
        summary: 'List users',
        tags: ['Users'],
        parameters: [
            new OA\QueryParameter(
                name: 'page',
                description: 'Page number',
                schema: new OA\Schema(type: 'integer', default: 1)
            ),
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: 'Success',
                content: new OA\JsonContent(
                    properties: [
                        new OA\Property(
                            property: 'data',
                            type: 'array',
                            items: new OA\Items(ref: '#/components/schemas/User')
                        ),
                    ]
                )
            ),
        ]
    )]
    public function list(): array
    {
        // Implementation
    }

    #[Route(route: '/api/users', name: 'users.create', methods: 'POST')]
    #[OA\Post(
        path: '/api/users',
        operationId: 'users.create',
        summary: 'Create user',
        requestBody: new OA\RequestBody(
            required: true,
            content: new OA\JsonContent(ref: CreateUserFilter::class)
        ),
        tags: ['Users'],
        responses: [
            new OA\Response(
                response: 201,
                description: 'Created',
                content: new OA\JsonContent(ref: '#/components/schemas/User')
            ),
        ]
    )]
    public function create(CreateUserFilter $filter): array
    {
        // Implementation
    }
}
```

> **See also**
> Learn more about routing in the [HTTP — Routing](routing.md) section.

### Filter Documentation

Filters can serve as both request validators and OpenAPI schema definitions. Add OpenAPI attributes to your filter
classes:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use OpenApi\Attributes as OA;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

#[OA\Schema(
    schema: 'CreateUserRequest',
    required: ['email', 'password', 'name'],
    properties: [
        new OA\Property(
            property: 'email',
            type: 'string',
            format: 'email',
            example: 'user@example.com'
        ),
        new OA\Property(
            property: 'password',
            type: 'string',
            format: 'password',
            minLength: 8
        ),
        new OA\Property(
            property: 'name',
            type: 'string',
            example: 'John Doe'
        ),
    ]
)]
final class CreateUserFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $email;

    #[Post]
    public string $password;

    #[Post]
    public string $name;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'email' => ['required', 'email'],
            'password' => ['required', 'string::length', 8],
            'name' => ['required', 'string'],
        ]);
    }
}
```

> **See also**
> Learn more about filters in the [Filters — Filter Object](../filters/filter.md) section.

### Response Resources

Resource classes can be documented to define response schemas:

```php app/src/Endpoint/Web/Resource/UserResource.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Web\Resource;

use App\Application\HTTP\Response\JsonResource;
use OpenApi\Attributes as OA;

#[OA\Schema(
    schema: 'User',
    required: ['id', 'email', 'name', 'created_at'],
    properties: [
        new OA\Property(
            property: 'id',
            type: 'integer',
            example: 1
        ),
        new OA\Property(
            property: 'email',
            type: 'string',
            format: 'email',
            example: 'user@example.com'
        ),
        new OA\Property(
            property: 'name',
            type: 'string',
            example: 'John Doe'
        ),
        new OA\Property(
            property: 'created_at',
            type: 'string',
            format: 'date-time',
            example: '2024-01-15T10:30:00Z'
        ),
    ]
)]
final class UserResource extends JsonResource
{
    protected function mapData(): array
    {
        return [
            'id' => $this->data->id,
            'email' => $this->data->email,
            'name' => $this->data->name,
            'created_at' => $this->data->createdAt->format('c'),
        ];
    }
}
```

### Organizing Schemas

For better organization, you can create dedicated schema classes:

```php app/src/OpenApi/Schema/ErrorSchema.php
<?php

declare(strict_types=1);

namespace App\OpenApi\Schema;

use OpenApi\Attributes as OA;

#[OA\Schema(
    schema: 'Error',
    required: ['message'],
    properties: [
        new OA\Property(
            property: 'message',
            type: 'string',
            example: 'An error occurred'
        ),
        new OA\Property(
            property: 'code',
            type: 'integer',
            example: 400
        ),
    ]
)]
final class ErrorSchema
{
}
```

Use schema references in your controllers:

```php
#[OA\Response(
    response: 400,
    description: 'Bad Request',
    content: new OA\JsonContent(ref: '#/components/schemas/Error')
)]
```

### Centralized Schema References

Create an enum for type-safe schema references:

```php app/src/OpenApi/SchemaRef.php
<?php

declare(strict_types=1);

namespace App\OpenApi;

enum SchemaRef: string
{
    case User = '#/components/schemas/User';
    case Error = '#/components/schemas/Error';
    case ValidationError = '#/components/schemas/ValidationError';
}
```

Use in controllers:

```php
#[OA\Response(
    response: 200,
    description: 'Success',
    content: new OA\JsonContent(ref: SchemaRef::User->value)
)]
```

## Security Schemes

Define authentication schemes in configuration:

```php app/config/swagger.php
return [
    'documentation' => [
        'info' => [
            'title' => 'My API',
            'version' => '1.0.0',
        ],
        'components' => [
            'securitySchemes' => [
                'bearerAuth' => [
                    'type' => 'http',
                    'scheme' => 'bearer',
                    'bearerFormat' => 'JWT',
                ],
            ],
        ],
        'security' => [
            ['bearerAuth' => []],
        ],
    ],
];
```

Use in endpoints:

```php
#[OA\Get(
    path: '/api/users',
    security: [['bearerAuth' => []]],
    // ...
)]
```

## Caching

Enable caching in production for better performance:

```php app/config/swagger.php
return [
    'use_cache' => env('APP_ENV') === 'production',
    'cache_key' => 'swagger_docs',
];
```

Clear cache when documentation changes:

```terminal
php app.php cache:clear swagger_docs
```

## Custom Parsers

Extend the generator with custom parsers for additional data sources:

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
    }
}
```

Register the parser:

```php app/config/swagger.php
return [
    'parsers' => [
        ConfigurationParser::class,
        OpenApiParser::class,
        CustomParser::class,
    ],
];
```

> **See also**
> * [HTTP — Routing](routing.md) - Configure routes for your API
> * [Filters — Filter Object](../filters/filter.md) - Create request validation filters
> * [HTTP — Middleware](middleware.md) - Add authentication and other middleware
> * [swagger-php documentation](https://zircote.github.io/swagger-php/) - Complete attribute reference
