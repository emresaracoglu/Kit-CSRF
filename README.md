# Secure CSRF validator #

*CSRF* is a PHP library for preventing [Cross-Site Request Forgery]
(https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29) attacks.
A CSRF attack takes advantage of authenticated users by sending them to a
malicious website that sends carefully crafted requests to the targeted website
in order to modify content on that website. The attack uses the authenticated
user's browser to send the request to bypass any authentication. This library
prevents these attacks by requiring a CSRF token in each POST, PUT and DELETE
request. These tokens are not known by the attacker, which prevents them from
sending malicious requests.

This library supports storing the CSRF tokens using either cookies or sessions.
The token can also be submitted using either a hidden form field in POST
requests or using a HTTP header, which makes it easier to pass the token in
ajax requests.

In order to provide additional security against different forms of attacks
against the CSRF tokens, this library uses constant time string comparisons to
prevent timing attacks and generates random encrypted tokens in each request to
prevent BREACH attacks. On top of that, all tokens are generated using a secure
random byte generator.

The API documentation is available at: http://kit.riimu.net/api/csrf/

[![Travis](https://img.shields.io/travis/Riimu/Kit-CSRF.svg?style=flat-square)](https://travis-ci.org/Riimu/Kit-CSRF)
[![Scrutinizer](https://img.shields.io/scrutinizer/g/Riimu/Kit-CSRF.svg?style=flat-square)](https://scrutinizer-ci.com/g/Riimu/Kit-CSRF/)
[![Scrutinizer Coverage](https://img.shields.io/scrutinizer/coverage/g/Riimu/Kit-CSRF.svg?style=flat-square)](https://scrutinizer-ci.com/g/Riimu/Kit-CSRF/)
[![Packagist](https://img.shields.io/packagist/v/riimu/kit-csrf.svg?style=flat-square)](https://packagist.org/packages/riimu/kit-csrf)

## Requirements ##

  * The minimum supported PHP version is 5.6
  * The library depends on the following external PHP libraries:
    * [Kit-SecureRandom](https://packagist.org/packages/riimu/kit-securerandom) (`^1.0`)

## Installation ##

### Installation with Composer ###

The easiest way to install this library is to use Composer to handle your
dependencies. In order to install this library via Composer, simply follow
these two steps:

  1. Acquire the `composer.phar` by running the Composer
     [Command-line installation](https://getcomposer.org/download/)
     in your project root.

  2. Once you have run the installation script, you should have the `composer.phar`
     file in you project root and you can run the following command:

     ```
     php composer.phar require "riimu/kit-csrf:^2.5"
     ```

After installing this library via Composer, you can load the library by
including the `vendor/autoload.php` file that was generated by Composer during
the installation.

### Adding the library as a dependency ###

If you are already familiar with how to use Composer, you may alternatively add
the library as a dependency by adding the following `composer.json` file to your
project and running the `composer install` command:

```json
{
    "require": {
        "riimu/kit-csrf": "^2.5"
    }
}
```

### Manual installation ###

If you do not wish to use Composer to load the library, you may also download
the library manually by downloading the [latest release](https://github.com/Riimu/Kit-CSRF/releases/latest)
and extracting the `src` folder to your project. You may then include the
provided `src/autoload.php` file to load the library classes.

Please note that using Composer will also automatically download the other
required PHP libraries. If you install this library manually, you will also need
to make those other required libraries available.

## Usage ##

The idea of this library is to make security as convenient as possible. You only
really need two methods provided by the `CSRFHandler` class. The method
`validateRequest()` should be called at the very beginning of each request. This
method will only validate POST, PUT and DELETE requests so you can safely call
it on every request. The method `getToken()` can be used to retrieve the token
that should be included in each submitted form using a hidden field named
`csrf_token`.

If the submitted token does not match against the secret token stored in the
cookie or session, the `validateRequest()` method will send a HTTP 400 (bad
request) header and kill the script execution. This should not affect the normal
usage of your website, but it will prevent any CSRF attack attempts against
your website.

As an example, here is a simple web page that has one form that can be
submitted:

```php
<?php

require 'vendor/autoload.php';
$csrf = new \Riimu\Kit\CSRF\CSRFHandler();
$csrf->validateRequest();
$token = $csrf->getToken();

?>
<!DOCTYPE html>
<html>
 <head>
  <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
  <title>Simple Form</title>
 </head>
 <body>
<?php

if (!empty($_POST['my_name'])) {
    printf("  <p>Hello <strong>%s</strong>!</p>" . PHP_EOL, htmlspecialchars($_POST['my_name'], ENT_QUOTES | ENT_HTML5, 'UTF-8'));
}

?>
  <form method="post"><div>
   <input type="hidden" name="csrf_token" value="<?=htmlspecialchars($token, ENT_QUOTES | ENT_HTML5, 'UTF-8')?>" />
   What is your name?
   <input type="text" name="my_name" />
   <input type="submit" />
  </div></form>
 </body>
</html>
```

### Using Sessions ###
 
By default, the library will save the secret token to a cookie. If you prefer
to save the token to a session instead, you can initialize the `CSRFHandler` by
setting the constructor parameter to `false`. For example:

```php
<?php

session_start();
require 'vendor/autoload.php';
$csrf = new \Riimu\Kit\CSRF\CSRFHandler(false);
```

### Handling Invalid Tokens ###

If you wish to have more control over what happens when the request sends an
invalid csrf token, you can set the parameter passed to `validateRequest()` to
`true`. This will cause the method to throw an `InvalidCSRFTokenException`
instead of killing the script. For example:

```php
<?php

require 'vendor/autoload.php';
$csrf = new \Riimu\Kit\CSRF\CSRFHandler();

try {
    $csrf->validateRequest(true);
} catch (\Riimu\Kit\CSRF\InvalidCSRFTokenException $ex) {
    header('HTTP/1.0 400 Bad Request');
    exit('Bad CSRF Token!');
}
```

### Using Token Headers ###

Note that if you are building a REST api to your website or you are using ajax
requests to send POST, PUT or DELETE requests, you may also provide the csrf
token using a header.

To provide the token using a header, simply include a header named `X-CSRF-Token`
which contains the same value you would include in the `csrf_token` form field.

### Using nonces ###

A *nonce* is a token that can be used only once. Turning CSRF tokens into nonces
provides protection against [Replay Attacks](https://en.wikipedia.org/wiki/Replay_attack).
It is important to note, however, that the best defense against such attacks is
using a secure HTTPS connection. However, if you do not have the luxury of an
encrypted connection at your disposal, it may be possible to [use nonces]
(http://blog.ircmaxell.com/2013/02/preventing-csrf-attacks.html) to prevent
these attacks.

This library provides a way to implement nonces by using the `NonceValidator`
class. This class works exactly the same as `CSRFHandler` except that it accepts
each token generated by `getToken()` only once. Even if the attacker can spy
on the connection, they cannot resend the http request because the token only
works once.

You can use the `NonceValidator` the same way as you would use the `CSRFHandler`,
for example:

```php
<?php

require 'vendor/autoload.php';

session_start();
$csrf = new \Riimu\Kit\CSRF\NonceValidator();
$csrf->validateRequest();
$token = $csrf->getToken();
```

Note that `NonceValidator` always uses sessions to store the CSRF token. In
addition to that, it will also store which tokens have been used up and cannot
be used again. If you have a website that relies on a large number of form
submissions, this array of invalidated tokens can grow quite large. To clear this
array, simply regenerate the token using `regenerateToken()`. For example:
 
```php
<?php

require 'vendor/autoload.php';

session_start();
$csrf = new \Riimu\Kit\CSRF\NonceValidator();
$csrf->validateRequest();

if ($csrf->getNonceCount() > 100) {
    $csrf->regenerateToken();
}

$token = $csrf->getToken();
```

### Manual Usage ###

If you wish to have more control over the token validation, this library
provides several methods that allows you to manually manage several aspects of
the library. For your convenience, the `CSRFHandler` provides the following
methods:

  * `isValidatedRequest()` tells if the current request is a POST, PUT or
    DELETE request which should be validated.
    
  * `validateRequest($throw = false)` validates the request and kills the script
    or throws an exception if the token is invalid. The token is only validated
    on POST, PUT and DELETE requests.
  
  * `validateRequestToken()` validates the token sent in the request. True is
    returned if the token exists and it matches against the secret token.
    
  * `validateToken($token)` can be used to validate tokens manually. The token
    passed to the method should be the one that has been returned by `getToken()`
    
  * `getToken()` returns a valid base64 encoded token.
  
  * `regenerateToken()` regenerates the secret CSRF token and invalidates all
    the tokens returned previously by `getToken()`
    
  * `getTrueToken()` returns the stored secret CSRF token that is used to
    validate the tokens submitted by the user.
    
  * `getRequestToken()` returns the token sent in the request.

## Securing Your Website ##

Even this library does not prevent CSRF attacks if you fail to utilize the
tokens correctly. It is very important that each request is properly validated
and that the token is sent with each submitted form. However, there are still
couple of pitfalls that you should be aware of.

If you're not using nonces, in some rare cases it may also be possible to use
[Session Fixation] (https://www.owasp.org/index.php/Session_fixation) attack to
determine the CSRF token used by the authenticated user. Even if the session ID
is regenerated upon login, the attacker may still take advantage of the known
CSRF token. To prevent this, it is simply advisable to regenerate the token
upon authentication by calling `regenerateToken()`.

In order to create a website that is impervious to CSRF attacks, you must also
remember that only POST, PUT and DELETE requests should change the state of the
website. A CSRF token should be never be supplied in a GET parameter, because
this can be leaked using various different attacks. Thus, GET requests should
never affect the state. For example, allowing users to be deleted using a simple
GET request would make your website vulnerable to CSRF attacks.

If you truly want to create a secure site, however, you must also only use
encrypted connections, i.e. you must use HTTPS. This is the only effective
measure against [Man-in-the-middle](https://www.owasp.org/index.php/Man-in-the-middle_attack)
attacks, but it also helps in preventing replay attacks.

Finally, remember that CSRF tokens only protect you from external requests. They
offer no protection against [Cross-site Scripting](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29)
attacks. If the attacker is capable of running javascript on your website, the
CSRF tokens offer no additional protection. Using a XSS attack, the attacker is
always capable of finding out the CSRF token. The security of your website only
as strong as the weakest link.

## Anatomy of the CSRF tokens ##

All the tokens generated by this library are random 32 byte strings. These
strings have been generated by using the SecureRandom library in order to ensure
that they have been generated using a secure random source. However, the tokens
returned by `getToken()` are more than double that in length. This is because
they are base64 encoded strings that also contain a hashed version of the token
using the secret token as a salt.

In order to prevent BREACH attacks, each token returned by `getToken()` is
different, because a static token can be used to break the encryption used by a
HTTPS connection. In order to achieve this, the returned token actually consists
of a random generated token and an encrypted version of that token that has been
encrypted using HMAC-SHA256 using the secret token as the key. (which also makes
it infeasible to reverse the operation to find out the secret token). This
allows each token to be different, but still valid until `regenerateToken()` is
called. Thus, the actual length of the returned decoded string is 64 bytes.

Note that a new random token is generated every time `getToken()` is called.
Thus, each string returned by that method is different. If you have a large
number of forms on your web page, it may be more efficient to use `SingleToken`
class, which loads the token only once and it can be casted to a string.

## Credits ##

This library is copyright 2014 - 2015 to Riikka Kalliomäki.

See LICENSE for license and copying information.

Implementation of this library is based on ideas from Go library
[nosurf](https://github.com/justinas/nosurf) by Justinas Stankevicius
