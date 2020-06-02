[![Latest Version on Packagist](https://img.shields.io/packagist/v/juhedata/laravel-samlidp.svg?style=flat-square)](https://packagist.org/packages/juhedata/laravel-samlidp)
[![Total Downloads](https://img.shields.io/packagist/dt/juhedata/laravel-samlidp.svg?style=flat-square)](https://packagist.org/packages/juhedata/laravel-samlidp)

# Laravel SAML IdP

This package allows you to implement your own Identification Provider (idP) using the SAML 2.0 standard to be used with supporting SAML 2.0 Service Providers (SP).
该组件可以让你实现基于SAML 2.0协议的idP端（idP端提供身份验证，用户在此登录）。

## Version 版本

^1.0

- Laravel 5.X required

^2.0

- PHP 7.2+ required
- Laravel 6.X required

## Installation 安装

Require this package with composer:
用composer安装本组件：

```shell
composer require juhedata/laravel-samlidp:^2.0
```

Publish config 发布配置文件

```shell
php artisan vendor:publish --tag="samlidp_config"
```

FileSystem configuration 文件系统配置

```php
// config/filesystem.php

'disks' => [

        ...

        'samlidp' => [
            'driver' => 'local',
            'root' => storage_path() . '/samlidp',
        ]
],
```

Use the following command to create a self signed certificate for your IdP. If you change the certname or keyname to anything other than the default names, you will need to update your `config/samlidp.php` config file to reflect those new file names.
用以下命令生成自签证书

```shell
php artisan samlidp:cert [--days <days> --keyname <name> --certname <name>]
```

```shell
Options:
  --days=<days>      Days to add for the expiration date [default: 7800]
  --keyname=<name>   Name of the certificate key file [default: key.pem]
  --certname=<name>  Name of the certificate file [default: cert.pem]
```

## Usage

Within your login view, probably `resources/views/auth/login.blade.php` add the SAMLRequest directive beneath the CSRF directive:
在登录页面（如`resources/views/auth/login.blade.php`），在CSRF directive后增加SAMLRequest directive

```php
@csrf
@samlidp
```

The SAMLRequest directive will fill out the hidden input automatically when a SAMLRequest is sent by an HTTP request and therefore initiate a SAML authentication attempt. To initiate the SAML auth, the login and redirect processes need to be intervened. This is done using the Laravel events fired upon authentication.
SAMLRequest directive会自动检查当前的HTTP请求是否含有SAML相关的参数，若有则补充SAML相关的参数到登录的表单中。
相关中间件会处理表单中的SAML请求，并将用户重定向到SP。

## Config

After you publish the config file, you will need to set up your Service Providers. The key for the Service Provider is a base 64 encoded Consumer Service (ACS) URL. You can get this information from your Service Provider, but you will need to base 64 encode the URL and place it in your config. This is due to config dot notation.

You may use this command to help generate a new SAML Service Provider:
一个idP可以对应多个SP，用以下命令生成SP配置代码：

```shell
php artisan samlidp:sp
```

Example SP in `config/samlidp.php` file:
可参考 `config/samlidp.php` 文件中的SP配置示例：

```php
<?php

return [
    // The URI to your login page
    'login_uri' => 'login',
    // The URI to the saml metadata file, this describes your idP
    'issuer_uri' => 'saml/metadata',
    // List of all Service Providers
    'sp' => [
        // Base64 encoded ACS URL
        'aHR0cHM6Ly9teWZhY2Vib29rd29ya3BsYWNlLmZhY2Vib29rLmNvbS93b3JrL3NhbWwucGhw' => [
            // ACS URL of the Service Provider
            'destination' => 'https://example.com/saml/acs',
            // Simple Logout URL of the Service Provider
            'logout' => 'https://example.com/saml/sls',
        ]
    ]

];
```

## Log out of IdP after SLO

If you wish to log out of the IdP after SLO has completed, set `LOGOUT_AFTER_SLO` to `true` in your `.env` perform the logout action on the Idp.

```
// .env

LOGOUT_AFTER_SLO=true
```

## Redirect to SLO initiator after logout

If you wish to return the user back to the SP by which SLO was initiated, you may provide an additional query parameter to the `/saml/logout` route, for example:

```
https://idp.com/saml/logout?redirect_to=mysp.com
```

After all SP's have been logged out of, the user will be redirected to `mysp.com`. For this to work properly you need to add the `sp_slo_redirects` option to your `config/samlidp.php` config file, for example:

```php
<?php

// config/samlidp.php

return [
    // If you need to redirect after SLO depending on SLO initiator
    // key is beginning of HTTP_REFERER value from SERVER, value is redirect path
    'sp_slo_redirects' => [
        'mysp.com' => 'https://mysp.com',
    ],

];
```

## Attributes (optional) 其他属性（可选）

Service providers may require more additional attributes to be sent via assertion. Its even possible that they require the same information but as a different Claim Type.
SP可能要求提供更多的属性。

By Default this package will send the following Claim Types:

`ClaimTypes::EMAIL_ADDRESS` as `auth()->user()->email`
`ClaimTypes::GIVEN_NAME` as `auth()->user()->name`

This is because Laravel migrations, by default, only supply email and name fields that are usable by SAML 2.0.

To add additional Claim Types, you can subscribe to the Assertion event:

`CodeGreenCreative\SamlIdp\Events\Assertion`

Subscribing to the Event:
可以通过订阅相关事件，来向SP传递更多属性：

In your `App\Providers\EventServiceProvider` class, add to the already existing `$listen` property...
在 `App\Providers\EventServiceProvider` 中订阅这个事件：`CodeGreenCreative\SamlIdp\Events\Assertion`

```php
protected $listen = [
    'App\Events\Event' => [
        'App\Listeners\EventListener',
    ],
    'CodeGreenCreative\SamlIdp\Events\Assertion' => [
        'App\Listeners\SamlAssertionAttributes'
    ]
];
```

Sample Listener:
Listener示例：

```php
<?php

namespace App\Listeners;

use LightSaml\ClaimTypes;
use LightSaml\Model\Assertion\Attribute;
use CodeGreenCreative\SamlIdp\Events\Assertion;

class SamlAssertionAttributes
{
    public function handle(Assertion $event)
    {
        $event->attribute_statement
            ->addAttribute(new Attribute(ClaimTypes::PPID, auth()->user()->id))
            ->addAttribute(new Attribute(ClaimTypes::NAME, auth()->user()->name));
    }
}

```
