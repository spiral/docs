# Security - User Authentication
The framework includes the set of components to authorize users via temporary or permanent tokens from different
sources and safely manage user context.

> The component does not enforce any specific User entity interface and does not limit application to HTTP scope only 
> (GRPC auth is possible as well).

## Principle of Work
The authentication extension will create IoC scope for `Spiral\Auth\AuthContextInterface` which points to the currently authorized actor 
(User, API Client). The actor is fetched from `Spiral\Auth\ActorProviderInterface` using `Spiral\Auth\TokenInterface`.

The token is managed by `Spiral\Auth\TokenStorageInterface` and always includes the payload (for example `["userID" => $id]`, LDAP creds, etc.). 
The token payload use to find current application user via `Spiral\Auth\ActorProviderInterface`.

Token storage can either store token in external source (such as database, redis or file) or decode it on a fly. Framework
includes multiple token implementations out of the box for easier use. 

> You can use multiple token and actor providers inside one application.

## Installation and Configuration
To install authorization extension for Web bundle:

```bash
$ composer require spiral/auth spiral/auth-http
```

The package `spiral/auth` provides common interfaces without the relation to any specific dispatching method, while
`spiral/auth-http` includes HTTP Middleware, Token transport (Cookie, Header) and Firewall components.

// todo: keep writing
