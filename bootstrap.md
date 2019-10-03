`public/index.php` 裡面有

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
$app = require_once __DIR__.'/../bootstrap/app.php';
```

這一段。

看起來很簡短，不過我們往下挖掘一下，來看看這段是怎麼產生 `$app` 的

打開 `/bootstrap/app.php`

```php
/*
|--------------------------------------------------------------------------
| Create The Application
|--------------------------------------------------------------------------
|
| The first thing we will do is create a new Laravel application instance
| which serves as the "glue" for all the components of Laravel, and is
| the IoC container for the system binding all of the various parts.
|
*/
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
```

這裡建立了 `$app`，根據註解看起來還做了很多綁定的工作。不過我們先不分心，繼續往下看。

```php
/*
|--------------------------------------------------------------------------
| Bind Important Interfaces
|--------------------------------------------------------------------------
|
| Next, we need to bind some important interfaces into the container so
| we will be able to resolve them when needed. The kernels serve the
| incoming requests to this application from both the web and CLI.
|
*/

$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```

 這裡綁定了三個「重要的介面」，我們先不理會，繼續往下看
 
 ```php
 /*
|--------------------------------------------------------------------------
| Return The Application
|--------------------------------------------------------------------------
|
| This script returns the application instance. The instance is given to
| the calling script so we can separate the building of the instances
| from the actual running of the application and sending responses.
|
*/

return $app;
 ```
 
這邊回傳了 `$app`。
 
根據註解，我們可以看出整個 `bootstrap.php` 設計的意義。藉由將應用的建立放在這邊，我們可以將建立應用和操作應用兩件事情切割開。

看完整體，我們開始往下仔細解剖

## `new Illuminate\Foundation\Application`

針對

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
```

我們往下看看這個物件的建構，做了什麼操作。往下看 `Illuminate\Foundation\Application` 的建構子

```php
/**
 * Create a new Illuminate application instance.
 *
 * @param  string|null  $basePath
 * @return void
 */
public function __construct($basePath = null)
{
    if ($basePath) {
        $this->setBasePath($basePath);
    }

    $this->registerBaseBindings();
    $this->registerBaseServiceProviders();
    $this->registerCoreContainerAliases();
}
```
 
### `registerBaseServiceProviders()`

我們來看看 `$this->registerBaseServiceProviders()` 的實作

```php
/**
 * Register all of the base service providers.
 *
 * @return void
 */
protected function registerBaseServiceProviders()
{
    $this->register(new EventServiceProvider($this));
    $this->register(new LogServiceProvider($this));
    $this->register(new RoutingServiceProvider($this));
}
```

這邊登記了三個「基本的 Service Providers」。

我們注意到，這邊就已經紀錄了對網頁路徑很重要的路由服務了。有關路由服務的細節部分，會在講解路由的文章特別說明。
