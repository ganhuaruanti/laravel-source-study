# 開始

## 程式起始點

研讀所有程式原始碼時，面對滿山滿谷的原始碼，通常都是先從程式的起始點開始找起。

也就是說，我們嘗試從這麼多的程式碼裡面，找出 **這支程式從哪裡開始？** 這個問題的答案。

以 Laravel 來說，就是 `public/index.php`，這也是為什麼當你在設置環境的時候，Laravel 的官方教學說

```
Public Directory

After installing Laravel, you should configure your web server's document / web root to be the public directory. The index.php in this directory serves as the front controller for all HTTP requests entering your application.
```

找到這個進入點之後，我們就可以非常快速地看到程式的概觀，並粗略的理解整個程式流程為何。

### `index.php`

我們來看看這個檔案寫了些什麼

```php
/**
 * Laravel - A PHP Framework For Web Artisans
 *
 * @package  Laravel
 * @author   Taylor Otwell <taylor@laravel.com>
 */

define('LARAVEL_START', microtime(true));
```

這段定義了程式開啟的時間

```php
/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader for
| our application. We just need to utilize it! We'll simply require it
| into the script here so that we don't have to worry about manual
| loading any of our classes later on. It feels great to relax.
|
*/

require __DIR__.'/../vendor/autoload.php';
```

這一段註冊了 `composer` 裡面的程式

```php
/*
|--------------------------------------------------------------------------
| Turn On The Lights
|--------------------------------------------------------------------------
|
| We need to illuminate PHP development, so let us turn on the lights.
| This bootstraps the framework and gets it ready for use, then it
| will load up this application so that we can run it and send
| the responses back to the browser and delight our users.
|
*/

$app = require_once __DIR__.'/../bootstrap/app.php';
```

這裡面用到了 `bootstrap/app.php` 來建立 `$app` 物件。至於細節是什麼，我們暫且不需理會，往下找程式運作的主要邏輯。

```php
/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
| Once we have the application, we can handle the incoming request
| through the kernel, and send the associated response back to
| the client's browser allowing them to enjoy the creative
| and wonderful application we have prepared for them.
|
*/

$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

$response->send();

$kernel->terminate($request, $response);
```

是否很意外，竟然只有這麼短短的幾行程式碼？

雖然只有這麼短短的幾行，但是到這裡，我們可以紮實地看到 Laravel 框架運作的流程以及概念。

首先，透過 `$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);`，我們建立了處理 `$request` 的核心 `$kernel`

接著，透過 `Illuminate\Http\Request::capture()` 收到此次請求的 `$request` 之後，我們透過 `$kernel->handle()` 處理需求，並回傳 `$response` 物件。

接著，我們透過 `$response->send();` 回傳處理過後的回應。然後執行 `$kernel->terminate($request, $response);`

到這裡，我們就可以大致上掌握住程式的概略邏輯了。不管背後有多麽複雜的操作邏輯，這一些邏輯必定會在這幾行程式碼內的某個位置呼叫。我們可以依序地往下追蹤，來找出所有的功能。
