# Secure CSRF validator #

*CSRF* is a PHP library for preventing [Cross-Site Request Forgery]
(https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29) attacks.
A CSRF attack takes advantage of authenticated users by sending them to a
malicious website that sends specially crafted requests to the targeted website
in order to modify content on that website. The attack uses the authenticated
user's browser to send the request to bypass any authentication. This library
prevents these attacks by requiring a CSRF token in each POST, PUT and DELETE
request. These tokens are not known by the attacker, which prevents them from
sending malicious requests.

This library can store the CSRF tokens using cookies or sessions. In order to
facilitate different kinds of websites and applications, the library allows
submission of the secret token in a hidden form field or using a HTTP header.

In order to provide additional security against different forms of attacks
against the CSRF tokens, this library uses constant time string comparisons to
prevent timing attacks and encrypts each token with a random string to prevent
BREACH attacks. On top of that, each CSRF token and encryption key is generated
using a secure random byte source.

The API documentation, which can be generated using Apigen, can be read online
at: http://kit.riimu.net/api/csrf/

[![Build Status](https://img.shields.io/travis/Riimu/Kit-CSRF.svg?style=flat)](https://travis-ci.org/Riimu/Kit-CSRF)
[![Coverage Status](https://img.shields.io/coveralls/Riimu/Kit-CSRF.svg?style=flat)](https://coveralls.io/r/Riimu/Kit-CSRF?branch=master)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/Riimu/Kit-CSRF.svg?style=flat)](https://scrutinizer-ci.com/g/Riimu/Kit-CSRF/?branch=master)

## Requirements ##

In order to use this library, the following requirements must be met:

  * PHP version 5.4
  * [SecureRandom](https://github.com/Riimu/Kit-SecureRandom) library is required

## Installation ##

This library can be installed via [Composer](http://getcomposer.org/). To do
this, download the `composer.phar` and require this library as a dependency. For
example:

```
$ php -r "readfile('https://getcomposer.org/installer');" | php
$ php composer.phar require riimu/kit-csrf:2.*
```

Alternatively, you can add the dependency to your `composer.json` and run
`composer install`. For example:

```json
{
    "require": {
        "riimu/kit-csrf": "2.*"
    }
}
```

Any library that has been installed via Composer can be loaded by including the
`vendor/autoload.php` file that was generated by Composer.

It is also possible to install this library manually. To do this, download the
[latest release](https://github.com/Riimu/Kit-CSRF/releases/latest) and
extract the `src` folder to your project folder. To load the library, include
the provided `src/autoload.php` file.

Note that if you install this library manually, you must also install the
dependencies by yourself.

## Usage ##

The idea of this library is to make security as convenient as possible. You only
really need two methods provided by the `CSRFHandler` class. The method
`validateRequest()` should be called at the very beginning of each request. This
method will only validate POST, PUT and DELETE requests so you can safely call
it on every request. The method `getToken()` can be used to retrieve the token
that should be included in each submitted form using a hidden field named
`csrf_token`.

If the token submitted in the form does not match the token stored in the user
cookie, the `validateRequest()` method will send a HTTP 400 (bad request) header
and kill the script execution. In normal use, this will not affect normal users,
but it will prevent any CSRF attacks against your website.

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
 
By default, the library will save the secret token to a cookie. If you prefer
to save the token to a session instead, you can initialize the `CSRFHandler` by
setting the constructor parameter to `false`. For example:

```php
<?php

session_start();
require 'vendor/autoload.php';
$csrf = new \Riimu\Kit\CSRF\CSRFHandler(false);
```

If you wish to have more control over what happens when the request sends an
invalid token, you can set the parameter passed to `validateRequest()` to
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

Note that if you are using ajax requests on your website, you may also provide
the CSRF token using a `X-CSRF-Token` header. This is particularly useful, if
you are creating a RESTful api that takes advantage of PUT and DELETE requests.

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
addition to that, it will also store which tokens have been used up and cannot be
used again. If you have a website that relies on a large number of form
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

### Securing Your Website ###

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

If you truly want to create a secure site, however, you must also employ SSL
encryption, i.e. use a HTTPS connection. This is the only effective measure
against [Man-in-the-middle](https://www.owasp.org/index.php/Man-in-the-middle_attack)
attacks, but it also helps in preventing replay attacks.

Finally, remember that CSRF tokens only protect you from external requests. They
offer no protection against [Cross-site Scripting](https://www.owasp.org/index.php/Cross-site_Scripting_%28XSS%29)
attacks. If the attacker is capable of running javascript on your website, the
CSRF tokens offer no additional protection. Using a XSS attack, the attacker is
always capable of finding out the CSRF token. The security of your website only
as strong as the weakest link.

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
    returned if the token exists and it matches the secret token.
    
  * `validateToken($token)` can be used to validate tokens manually. The token
    passed to the method should be the one that has been returned by `getToken()`
    
  * `getToken()` returns a valid base64 encoded token.
  
  * `regenerateToken()` regenerates the secret CSRF token and invalidates all
    the tokens returned previously by `getToken()`
    
  * `getTrueToken()` returns the stored secret CSRF token that is used to
    validate the tokens submitted by the user.
    
  * `getRequestToken()` returns the token sent in the request.

## Anatomy of the CSRF tokens ##

All the CSRF tokens generated by this library are random 32 byte strings. These
strings have been generated by using the SecureRandom library in order to
ensure that they have been generated using a secure random source. However, the
tokens returned by `getToken()` are very different in length. This is because
they are base64 encoded strings that also contain an encryption key for the
token.

In order to prevent BREACH attacks, each token returned by `getToken()` is
different, because a static token can be used to break the encryption used by
HTTPS connection. In order to achieve this, the returned token actually consists
of a random encryption key and a token that has been encrypted using HMAC-SHA256
(which also makes it infeasible to reverse the operation and find out the secret
CSRF token). This allows each token to be different, but still valid until
`regenerateToken()` is called. Thus, the actual length of the returned decoded 
string is 64 bytes.

Note that a new encryption key is generated every time `getToken()` is called.
Thus, each string returned by that method is different. If you have a large
number of forms on your web page, it may be more efficient to call that method
only once and store the result into a variable.

## Credits ##

This library is copyright 2014 - 2015 to Riikka Kalliomäki.

See LICENSE for license and copying information.

Implementation of this library is based on ideas from Go library
[nosurf](https://github.com/justinas/nosurf) by Justinas Stankevicius
