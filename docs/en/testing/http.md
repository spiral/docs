# Testing — HTTP Tests

The Spiral Framework's testing package has a really handy way to test your application controllers. You can use the
`Spiral\Testing\Http\FakeHttp` class to send requests to your controllers and check the responses.

Like, in the example below we can see a test for a `HomeController`. We created a `FakeHttp` instance and used it to
send a `GET` request to the `/` endpoint. Then we checked if the response is `OK`.

```php tests/Feature/HomeControllerTest.php 
namespace Tests\Feature;

use Spiral\Testing\Http\FakeHttp;
use Tests\TestCase;

final class HomeControllerTest extends TestCase
{
    private FakeHttp $http;

    protected function setUp(): void
    {
        parent::setUp();
        $this->http = $this->fakeHttp();
    }

    public function testIndex(): void
    {
        $response = $this->http->get('/');
        $response->assertOk();
    }
}
```

**It's a pretty straightforward way to test your controllers.**

## Making Requests

When you use the `FakeHttp` class to send a request, it's sent straight to your controller, passing through any
middleware and interceptors just like a real request would. You can use any of the standard HTTP methods like
`GET`, `POST`, `DELETE`, `PUT` in your tests.

Instead of returning a real response  `Psr\Http\Message\ResponseInterface` object, the `FakeHttp` class returns
a `Spiral\Testing\Http\TestResponse` object. It provides a variety of useful assertions that let you check things like
the response status, headers, and body. This makes it easy to test how your application responds to different requests
and ensure it's working as expected.

### Request Methods

Еhe `FakeHttp` class provides a variety of methods for sending different types of requests to your application.

#### GET

The `get()` method is used to send a `GET` request. It takes several arguments like the `uri`, `query`, `headers`
and `cookies`.

```php
$http = $this->fakeHttp();
$response = $http->get(
    uri: '/users',
    query: ['sort' => 'desc'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
);
```

Use the `getJson` method to send a `GET` request with the following headers:

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->getJson(
    uri: '/users',
    query: ['sort' => 'desc'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
);
```

#### POST

```php
$http = $this->fakeHttp();
$response = $http->post(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **Note**
> `data` argument accepts an array or `Psr\Http\Message\StreamInterface` object.

Use the `postJson` method to send a `POST` request with the following headers:

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->postJson(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

#### PUT

```php
$http = $this->fakeHttp();
$response = $http->put(
    uri: '/users',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **Note**
> `data` argument accepts an array or `Psr\Http\Message\StreamInterface` object.

Use the `putJson` method to send a `PUT` request with the following headers:

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->putJson(
    uri: '/users',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

#### DELETE

```php
$http = $this->fakeHttp();
$response = $http->delete(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **Note**
> `data` argument accepts an array or `Psr\Http\Message\StreamInterface` object.

Use the `deleteJson` method to send a `DELETE` request with the following headers:

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->deleteJson(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

### Request Headers

Yes, you can use the `withHeaders()` and `withHeader()` methods to add/set default custom headers to the request.

#### Set an array of headers

```php
$http = $this->fakeHttp();
$http->withHeaders(['Content-type' => 'application/json']);

$http->get('/users');
```

#### Set a single header

```php
$http = $this->fakeHttp();
$http->withHeader('Content-type', 'application/json');
$http->get('/users');
```

#### Set an authorization header

```php
$http = $this->fakeHttp();
$http->withAuthorizationToken(
    token: 'xxx-xxxx', 
    type: 'Bearer' // Default value is 'Bearer'
);
```

#### Flush headers

This method will flush all default headers.

```php
$http = $this->fakeHttp();
$http->flushHeaders();
```

### Request Cookies

Yes, you can use the `withCookies()` and `withCookie()` methods to add/set default custom cookies to the request.

#### Set an array of cookies

```php
$http = $this->fakeHttp();
$http->withCookies(['theme' => 'dark']);

$http->get('/users');
```

#### Set a single cookie

```php
$http = $this->fakeHttp();
$http->withHeader('theme', 'dark');
$http->get('/users');
```

#### Flush headers

This method will flush all default cookies.

```php
$http = $this->fakeHttp();
$http->flushCookies();
```

### Session / Authentication

The Spiral Framework has some cool tools to help you interact with the session while testing your HTTP requests. One of
those tools is the `withSession()` method. It allows you to set the session data to a specific array before sending a
request to your application. This can be really handy when you need to load the session with some specific data for your
test.

For example, you can set the session data like this:

```php
$http = $this->fakeHttp();
$http->withSession(
    data: ['fav_color' => 'blue'], 
    lifetime: 3600, // Default value is 3600
    id: null // Default value is null
)->get('/users');
```

The session is often used to maintain state for the currently authenticated user. This is why the Spiral Framework
provides a `withActor` method that makes it easy to authenticate a given user as the current user during testing.

This method allows you to quickly and easily set the authenticated user for a test, by passing an instance of a user
object to the method.

```php
$http = $this->fakeHttp();

$user = new User();
$http->withActor($user)->get('/profile');
```

When you call the `withActor()` method, a `Spiral\Testing\Auth\FakeActorProvider` object is bound to the container
implementing the `Spiral\Auth\ActorProviderInterface`, and this object will return the user that you passed every time
the `getActor()` method is called. This allows the application to access the authenticated user during the test, just as
it would during normal operation.

## Testing Responses

After sending a request to your application, you can use convenient methods to test the response.

### Available Assertions

#### assertHasHeader

Assert that the response has a given header.

```php
// Assert that the response has a header named Content-Type.
$response->assertHasHeader(name: 'Content-type');

// Assert that the response has a header named Content-Type with the given value.
$response->assertHasHeader(name: 'Content-type', value: 'application/json');
```

#### assertHeaderMissing

Assert that the response does not have a given header.

```php
$response->assertHeaderMissing(name: 'Content-type');
```

#### assertStatus

Assert that the response has a given status code.

```php
$response->assertStatus(status: 200);
```

#### assertOk

Assert that the response has a 200 status code.

```php
$response->assertOk();
```

#### assertCreated

Assert that the response has a 201 status code.

```php
$response->assertCreated();
```

#### assertAccepted

Assert that the response has a 202 status code.

```php
$response->assertAccepted();
```

#### assertNoContent

Assert that the response has a 204 status code and an empty body.

```php
$response->assertNoContent(
    status: 204 // Default value is 204
);
```

#### assertNotFound

Assert that the response has a 404 status code.

```php
$response->assertNotFound();
```

#### assertForbidden

Assert that the response has a 403 status code.

```php
$response->assertForbidden();
```

#### assertUnauthorized

Assert that the response has a 401 status code.

```php
$response->assertUnauthorized();
```

#### assertUnprocessable

Assert that the response has a 422 status code.

```php
$response->assertUnauthorized();
```

#### assertBodySame

Assert that the response body is equal to the given value.

```php
$response->assertBodySame(needle: "<html>...</html>");
```

#### assertBodyNotSame

Assert that the response body is not equal to the given value.

```php
$response->assertBodyNotSame(needle: "<html>...</html>");
```

#### assertBodyContains

Assert that the response body contains the given string.

```php
$response->assertBodyContains(needle: "<div>...</div>");
```

#### assertCookieExists

Assert that the response has a given cookie.

```php
$response->assertCookieExists(key: 'theme');
```

#### assertCookieSame

Assert that the response has a given cookie with the given value.

```php
$response->assertCookieSame(key: 'theme', value: 'dark');
```

#### assertCookieMissed

Assert that the response does not have a given cookie.

```php
$response->assertCookieMissed(key: 'theme');
```

## Testing File Uploads

Yes, the `FakeHttp` class provides a `getFileFactory` method which can be used to generate dummy files or images for
testing purposes. This method returns an instance of `Spiral\Testing\Http\FileFactory`, which has several methods to
create different types of files.

This feature allows you to easily test how your application handles file uploads, and ensure that it is functioning
correctly.

```php tests/Feature/UserControllerTest.php 
namespace Tests\Feature;

use Spiral\Testing\Http\FakeHttp;
use Tests\TestCase;

final class UserControllerTest extends TestCase
{
    private FakeHttp $http;

    protected function setUp(): void
    {
        parent::setUp();
        $this->http = $this->fakeHttp();
    }

    public function testUploadAvatar(): void
    {
        // Create a fake image 640x480
        $image = $http->getFileFactory()->createImage('avatar.jpg', 640, 480);

        $response = $http->post(uri: '/user/1', files: ['avatar' => $image]);
        $response->assertOk();
    }
}
```

### Create a fake image

```php
$image = $http->getFileFactory()->createImage(
    filename: 'avatar.jpg', 
    width: 640, 
    height: 480
);
```

### Create a fake file

```php
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt'
);
```

#### With file size

You can set a file size in kilobytes:

```php
// Create a file with size - 100kb
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt', 
    kilobytes: 100
);
```

#### With mime type

You can set a mime type:

```php
// Create a file with size - 100kb
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt', 
    mimeType: 'text/plain'
);
```

#### With content

You can set a file with specific content:

```php
$file = $http->getFileFactory()->createFileWithContent(
    filename: 'foo.txt', 
    content: 'Hello world', 
    mimeType: 'text/plain'
);
```