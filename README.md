# JWT Authentication Middleware for Slim

[![Latest Version](https://img.shields.io/packagist/v/tuupola/slim-jwt-auth.svg?style=flat-square)](https://packagist.org/packages/tuupola/slim-jwt-auth)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Build Status](https://img.shields.io/travis/tuupola/slim-jwt-auth/master.svg?style=flat-square)](https://travis-ci.org/tuupola/slim-jwt-auth)
[![HHVM Status](https://img.shields.io/hhvm/tuupola/slim-jwt-auth.svg?style=flat-square)](http://hhvm.h4cc.de/package/tuupola/slim-jwt-auth)
[![Coverage](http://img.shields.io/codecov/c/github/tuupola/slim-jwt-auth.svg?style=flat-square)](https://codecov.io/github/tuupola/slim-jwt-auth)

This middleware implements JSON Web Token Authentication for Slim Framework. It does **not** implement OAuth 2.0 authorization server nor does it provide ways to generate, issue or store authentication tokens. It only parses and authenticates a token when passed via header or cookie. This is useful for example when you want to use [JSON Web Tokens as API keys](https://auth0.com/blog/2014/12/02/using-json-web-tokens-as-api-keys/).

## Install

Install Slim 3 version using [composer](https://getcomposer.org/).

``` bash
$ composer require tuupola/slim-jwt-auth
```

Also add the following to the `.htaccess` file. Otherwise PHP wont have access to `Authorization: Bearer` header.

``` bash
RewriteRule .* - [env=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Usage

Configuration options are passed as an array. The only mandatory parameter is `secret` which is used for verifying then token signature. For simplicity's sake examples show `secret` hardcoded in code. In real life you should store it somewhere else. Good option is environment variable. You can use [dotenv](https://github.com/vlucas/phpdotenv) or something similar for development.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

An example where your secret is stored as an environment variable:

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => getenv("JWT_SECRET")
]));
```

When a request is made, the middleware tries to validate and decode the token. If a token is not found or there is an error when validating and decoding it, the server will respond with `401 Unauthorized`.

Validation errors are triggered when the token has been tampered with or the token has expired. For all possible validation errors, see [JWT library ](https://github.com/firebase/php-jwt/blob/master/Authentication/JWT.php#L44) source.


## Optional parameters
### Path

The optional `path` parameter allows you to specify the protected part of your website. It can be either a string or an array. You do not need to specify each URL. Instead think of `path` setting as a folder. In the example below everything starting with `/api` will be authenticated.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "path" => "/api", /* or ["/api", "/admin"] */
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

### Passthrough


With optional `passthrough` parameter you can make exceptions to `path` parameter. In the example below everything starting with `/api` and `/admin`  will be authenticated with the exception of `/api/token` and `/admin/ping` which will not be authenticated.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "path" => ["/api", "/admin"],
    "passthrough" => ["/api/token", "/admin/ping"],
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

### Environment

By default middleware tries to find the token from `HTTP_AUTHORIZATION` and `REDIRECT_HTTP_AUTHORIZATION` environments. You can change this using `environment` parameter.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "environment" => ["HTTP_BRAWNDO", "HTTP_ELECTROLYTE"],
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

### Cookie

If token is not found from environment middleware tries to find it from cookie name `token`. You can change cookie name using `cookie` parameter.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "cookie" => "nekot",
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

### Attribute

When the token is decoded successfully and authentication succees the contents of decoded token as saved as `token` attribute to the `$request` object. You can change this with. `attribute` parameter. Set to `null` or `false` to disable this behavour

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "attribute" => "jwt",
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));

/* ... */

$decoded = $request->getAttribute("jwt");
```

### Logger

The optional `logger` parameter allows you to pass in a PSR-3 compatible logger to help with debugging or other application logging needs.

``` php
$app = new \Slim\App();

$logger = \Monolog\Logger("slim");
$rotating = new RotatingFileHandler(__DIR__ . "/logs/slim.log", 0, Logger::DEBUG);
$logger->pushHandler($rotating);

$app->add(new \Slim\Middleware\JwtAuthentication([
    "path" => "/api",
    "logger" => $logger,
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

### Callback

Callback is called only when authentication succeeds. It receives decoded token in arguments. If callback returns boolean `false` authentication is forced to be failed.

You can also use callback for storing the value of decoded token for later use.

```php
$app = new \Slim\App();

$container = $app->getContainer();

$container["jwt"] = function ($container) {
    return new StdClass;
};

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => "supersecretkeyyoushouldnotcommittogithub",
    "callback" => function ($request, $response, $arguments) use ($container) {
        $container["jwt"] = $arguments["decoded"];
    }
]));
```

### Error

Error is called when authentication fails. It receives last error message in arguments.

```php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => "supersecretkeyyoushouldnotcommittogithub",
    "error" => function ($request, $response, $arguments) {
        return $response->write("Error");
    }
]));
```

### Rules

The optional `rules` parameter allows you to pass in rules which define whether the request should be authenticated or not. Rule is a callable which receives the Slim app as parameter. If the callable returns boolean `false` request will not be authenticated.

By default middleware configuration looks like this. All paths are authenticated with all request methods except `OPTIONS`.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "rules" => [
        new \Slim\Middleware\JwtAuthentication\RequestPathRule([
            "path" => "/",
            "passthrough" => []
        ]),
        new \Slim\Middleware\JwtAuthentication\RequestMethodRule([
            "passthrough" => ["OPTIONS"]
        ])
    ]
]));
```

RequestPathRule contains both a `path` parameter and a `passthrough` parameter. Latter contains paths which should not be authenticated. RequestMethodRule contains `passthrough` parameter of request methods which also should not be authenticated. Think of `passthrough` as a whitelist.

Example use case for this is an API. Token can be retrieved via [HTTP Basic Auth](https://github.com/tuupola/slim-basic-auth) protected address. There also is an unprotected url for pinging. Rest of the API is protected by the JWT middleware.

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "logger" => $logger,
    "secret" => "supersecretkeyyoushouldnotcommittogithub",
    "rules" => [
        new RequestPathRule([
            "path" => "/api",
            "passthrough" => ["/api/token", "/api/ping"]
        ]),
        new \Slim\Middleware\JwtAuthentication\RequestMethodRule([
            "passthrough" => ["OPTIONS"]
        ])
    ]
]));

$app->add(new \Slim\Middleware\HttpBasicAuthentication([
    "path" => "/api/token",
    "users" => [
        "user" => "password"
    ]
]));

$app->post("/token", function () {
  /* Here generate and return JWT to the client. */
});
```

## Security

JSON Web Tokens are essentially passwords. You should treat them as such and you should always use HTTPS. If the middleware detects insecure usage over HTTP it will throw a `RuntimeException`. This rule is relaxed for requests on localhost. To allow insecure usage you must enable it manually by setting `secure` to `false`.

``` php
$app->add(new \Slim\Middleware\JwtAuthentication([
    "secure" => false,
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

Alternatively you can list your development host to have relaxed security.

``` php
$app->add(new \Slim\Middleware\JwtAuthentication([
    "secure" => true,
    "relaxed" => ["localhost", "dev.example.com"],
    "secret" => "supersecretkeyyoushouldnotcommittogithub"
]));
```

## Authorization

By default middleware only authenticates. This is not very interesting. Beauty of JWT is you can pass extra data in the token. This data can include for example scope which can be used for authorization.

**It is up to you to implement how token data is stored or possible authorization implemented.**

Let assume you have token which includes data for scope. In middleware callback you store the decoded token data to `$app->jwt` and later use it for authorization.

``` php
[
    "iat" => "1428819941",
    "exp" => "1744352741",
    "scope" => ["read", "write", "delete"]
]
```

``` php
$app = new \Slim\App();

$container = $app->getContainer();

$container["jwt"] = function ($container) {
    return new StdClass;
};

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => "supersecretkeyyoushouldnotcommittogithub",
    "callback" => function ($request, $response, $arguments) use ($container) {
        $container["jwt"] = $arguments["decoded"];
    }
]));

$app->delete("/item/{id}", function ($request, $response, $arguments) {
    if (in_array("delete", $this->jwt->scope)) {
        /* Code for deleting item */
    } else {
        /* No scope so respond with 401 Unauthorized */
        return $response->withStatus(401);
    }
});
```

## Blacklisting (not implemented yet)

Tokens are like passwords and should be treated like one. Sometimes bad things happen. Maybe you accidentally leak internal token with full admin rights scope and expiration set to 10 years. Since there is no need to store tokens in database how can you revoke one? Changing the secret key would revoke all tokens in the wild.

**Again it is up to you to implement how token blacklisting is done. This middleware intentionally only provides interface for blacklisting.**

You can blacklist tokens by passing a callable `blacklist` parameter. This callable receives the decoded token and instance of Slim application as parameters. If callable returns boolean `true` the token is assumed to be blacklisted and server will response with `401 Unauthorized`.

One way of support blacklisting is to include `jti` ([JWT ID](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html#jtiDef)) claim in the token. This claim is unique identifier for the token and can be used for blacklisting.

``` php
[
    "iat" => "1428819941",
    "exp" => "1744352741",
    "jti" => "24d4e5c5-5727-4b7f-bd1d-a8f0733f160b",
    "scope" => ["read", "write", "delete"]
]
```

``` php
$app = new \Slim\App();

$app->add(new \Slim\Middleware\JwtAuthentication([
    "secret" => "supersecretkeyyoushouldnotcommittogithub",
    "blacklist" => function ($request, $response, $arguments) use ($app) {
        $decoded = $arguments["decoded"];
        return "24d4e5c5-5727-4b7f-bd1d-a8f0733f160b" === $decoded["jti"];
    }
]));
```

## Testing

You can run tests either manually...

``` bash
$ vendor/bin/phpunit
$ vendor/bin/phpcs --standard=PSR2 src/ -p
```

... or automatically on every code change.

``` bash
$ npm install
$ grunt watch
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email tuupola@appelsiini.net instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
