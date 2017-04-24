# Vault
Vault administration panel provides ability use regular application controllers inside [Security enviroment](https://github.com/spiral/security) with set of pre-created visual elements like grids, tabs, forms and other. 
Vault module is based on a set of [Materialize CSS](http://materializecss.com/) styles.

## Installation
```
composer require spiral/vault
spiral register spiral/vault
```

Add following bootloader to your application:
```php
[
    \Spiral\Vault\Bootloaders\VaultBootloader::class
]
```

> You can tweak Vault behaviour (route, middlewares), create new navigation sections or register your own controllers via `app/config/modules/vault.php` configuration file.

If you wish to play with Vault without configuring security rules (development only):

```php
[
    \Spiral\Vault\Bootloaders\InsecureBootloader::class
]
```

## Configuration
In order to add your custom controller to be run under vault and enable authorization check simply register it into `app/config/modules/vault` config:
 
```php
'controllers' => [
    'sample' => SampleController::class
],
```

To create navigation item modify `navigation` section of vault config:

```php
'navigation'  => [
    'my-section' => [
        'title' => 'My Controllers',
        'icon'  => 'tab',
        'items' => [
            'sample' => ['title' => 'Sample Controller'],
        ]
    ],
],
```

Once completed you can access your controller via `/vault/sample` URL.

### Permissions
Controller access will be managed by [security](/components/security.md) component. Vault will automatically generate permission based on registered controller alias and vault guard namespace (see config) - `vault.sample`.

In order to allow access to this controller only for specific user roles:

```php
public function boot(PermissionsInterface $permissions)
{
    $permissions->addRole('admin');
   
    $permissions->associate('admin', "vault.*", AllowRule::class);
    
    //or
    $permissions->associate('admin', "vault.sample", AllowRule::class);    
}
```

## Elements included
Vault panel includes set of view widgets available when you extend layout `vault.layout`:

```php
<extends:vault.layout title="[[Sample Dashboard]]"/>
```

In-Vault Uri tag (URL automatically resolved via Vault route):

```html
<vault:uri target="controller:action" options="<?= ['id' => 123] ?>" icon="icon" class="...">
    label
</vault:uri>
```

Cards and blocks:

```html
<vault:card color="blue-grey darken-2" text="white">
    <p>There is an issue with something.</p>
</vault:card>

<vault:block title="Title">
    content
</vault:block>
```

Forms (ajax):

```html
<vault:form action="<?= vault()->uri('countries:edit', ['id' => $entity->id]) ?>">
    <div class="row">
        <div class="col s7">
            <form:input label="[[Country:]]" name="name" value="<?= e($entity->name) ?>"/>
        </div>
        <div class="col s5">
            <form:input label="[[Country Code:]]" name="code" value="<?= e($entity->code) ?>"/>
        </div>
    </div>
    <div class="right-align">
        <input type="submit" value="[[UPDATE]]" class="btn teal waves-effect waves-light"/>
    </div>
</vault:form>
```

![Form](https://raw.githubusercontent.com/spiral/guide/master/resources/vault-form.png)

Tabs:

```html
<extends:vault:layout title="[[Vault]]"/>

<block:content>
    <tab:wrapper>

        <!--Primary information about user account-->
        <tab:item id="info" title="User Information" icon="user">
            <div class="row">
                <div class="col s6">
                    <vault:block>
                        <spiral:form>
                            <form:input label="abc"/>
                        </spiral:form>
                    </vault:block>
                </div>

                <div class="col s6">
                    <vault:card color="blue-grey darken-2" text="white">
                        <p>There is an issue with something.</p>
                    </vault:card>
                </div>
            </div>
        </tab:item>

        <!--Additional information about user account-->
        <tab:item id="extra" title="Extra Information">
            extra user information <vault:uri target="controller:action">link</vault:uri>
        </tab:item>

    </tab:wrapper>
</block:content>
```

![Animation](https://raw.githubusercontent.com/spiral/guide/master/resources/albus.gif)

Grids:

```php
/**
 * @return string
 */
protected function indexAction(PostsSource $source)
{
    return $this->views->render('admin/posts/list', [
        'posts' => $source->findActive()->paginate(5)
    ]);
}
```

```html
<extends:vault:layout title="Posts"/>

<define:content>
    <vault:grid source="<?= $posts ?>" as="post">
        <grid:cell label="ID:" value="<?= $post->id ?>"/>
        <grid:cell label="Time Created:" value="<?= $post->time_created ?>"/>
        <grid:cell label="Title:" value="<?= $post->title ?>"/>

        <grid:cell.bool label="Published:" value="<?= $post->isPublished() ?>"/>

        <grid:cell style="text-align: right">
            <vault:uri target="posts:edit" options="<?= ['id' => $post->id] ?>"
                       class="waves-effect btn-flat" icon="edit"/>
        </grid:cell>
    </vault:grid>
</define:content>
```

![Grid](https://raw.githubusercontent.com/spiral/guide/master/resources/grid.png)

> There is also Listing implementation with search and filtering abilities.
