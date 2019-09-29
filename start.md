# 開始

## 取得 Laravel 原始碼

要研究任何東西之前，總是要先有研究對象。

那麼，要怎麼取得程式碼呢？

在 github 發達的現在，取得框架的程式碼並不是什麼難事，我們只要 google 「laravel github」 就可以找到 Laravel 的 repo 位置了。

然後，我們找到有關框架的 repository，在 https://github.com/laravel/framework 。

然後，我們在開發的資料夾下，執行 `git clone git@github.com:laravel/framework.git` 這個指令，就可以把程式碼下載到指定的位置囉！

## 程式起始點

研讀所有程式原始碼時，面對滿山滿谷的原始碼，通常都是先從程式的起始點開始找起。

從程式的

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

到這裡，我們可以確實地看到 Laravel 框架運作的步驟了。

首先，透過 `$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);`，我們建立了處理 `$request` 的核心 `$kernel`

接著，透過 `Illuminate\Http\Request::capture()` 收到此次請求的 `$request` 之後，我們透過 `$kernel->handle()` 處理需求，並回傳 `$response` 物件。

接著，我們透過 `$response->send();` 回傳處理過後的回應。然後執行 `$kernel->terminate($request, $response);`

到這裡，我們就可以大致上掌握住程式的概略邏輯了。
