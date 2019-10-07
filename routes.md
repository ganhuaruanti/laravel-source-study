# 路由

## 紀錄路由

在尋找路由之前，我們先來研究一下，Laravel 是怎麼知道我們所撰寫的路由的。

熟悉 Laravel 開發的各位一定知道，我們會在 `routes/web.php` 和 `routes/api.php` 之類的檔案撰寫我們的路由。

可是，Laravel 是在什麼時間點，知道這兩個檔案的內容呢？

我們來看 `config/app.php` 裡面有一段

```php
'providers' => [

    /*
     * Laravel Framework Service Providers...
     */
      ...

    /*
     * Package Service Providers...
     */

    /*
     * Application Service Providers...
     */
    App\Providers\AppServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    // App\Providers\BroadcastServiceProvider::class,
    App\Providers\EventServiceProvider::class,
    App\Providers\RouteServiceProvider::class,

],
```
我們看到這裡註冊了一個 `App\Providers\RouteServiceProvider`。我們來看看這個物件的關係

```
use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;

class RouteServiceProvider extends ServiceProvider
```


看到這裡，應該就算是清楚了。藉由撰寫 `Illuminate\Foundation\Support\Providers\RouteServiceProvider` 和 `App\Providers\RouteServiceProvider` 兩個物件，可以讓開發者簡單的透過 Service Provider 簡單的使用自己喜歡的路由檔案。

看到這裡我們知道，透過 `App\Providers\RouteServiceProvider` 是怎麼把 `routes/web.php` 等檔案變成路由的。至於 Service Provider 在 config 裡面運作的時間，這個問題我們留待下次討論。

## 尋找路由
`$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);` 這段是用來建立 `$kernel`

結果我們發現，`Illuminate\Contracts\Http\Kernel` 是一個介面！

這下就麻煩了，介面只告訴了我們 `handle()` 函式的存在，並沒有告訴我們裡面的實作，那我們怎麼繼續追蹤其中的邏輯呢？

幸好，透過找哪些類別實作了這個介面，我們只找到了 `Illuminate\Foundation\Http\Kernel` 這個類別

```php
use Illuminate\Contracts\Http\Kernel as KernelContract;

class Kernel implements KernelContract
...
```

我們可以推測出，這個類別應該就是 `$app->make(Illuminate\Contracts\Http\Kernel::class)` 實際做出來的物件類別。

這個推測正不正確呢？我們可以透過改寫 `index.php` 來檢查這個事情

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
var_dump($kernel);
```

我們會看到

```text
object(App\Http\Kernel)#38 (7) { [...
```

到 `App\Http\Kernel` 這個類別一看，他並沒有實作任何函式，但是 `App\Http\Kernel` 繼承了 `Illuminate\Foundation\Http\Kernel`，所以我們可以知道，實作 `handle()` 函式的實際位置，是在 `Illuminate\Foundation\Http\Kernel` 裡面。

這樣的設計，通常是為了提升擴充的彈性。實作的邏輯在 `Illuminate\Foundation\Http\Kernel`，也就是一個框架內的程式碼。不過實際使用的物件卻是 `App\Http\Kernel` 類別的物件。如果之後想要擴充一些框架原本沒有支援的功能，我們只要在 `App\Http\Kernel` 加上這些功能就好。

現在我們來看看 `Illuminate\Foundation\Http\Kernel` 這個類別怎麼實作 `handle()`。

這個類別針對 `handle()` 的實作如下

```php
public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Exception $e) {
        $this->reportException($e);

        $response = $this->renderException($request, $e);
    } catch (Throwable $e) {
        $this->reportException($e = new FatalThrowableError($e));

        $response = $this->renderException($request, $e);
    }

    $this->app['events']->dispatch(
        new Events\RequestHandled($request, $response)
    );

    return $response;
}
```

看起來好像很多邏輯，不過我們可以發現到，這裡面大多數都是例外處理。然後，`$request->enableHttpMethodParameterOverride();` 所做的事情是設置參數。

所以真正的邏輯是在 `$response = $this->sendRequestThroughRouter($request);` 裡面。

我們到這個函式裡面再看看

```php
protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
                ->send($request)
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                ->then($this->dispatchToRouter());
}
```

根據命名我們就看出來了，前面處理的邏輯是跟 Middleware 相關的，這邊我們就不追蹤這段了，直接針對處理路由的部分處理，也就是 `$this->dispatchToRouter()` 。

這邊的中介層有點多，最後會追蹤到 `Illuminate\Routing` 的 `dispatchToRoute()`

```php
public function dispatchToRoute(Request $request)
{
    return $this->runRoute($request, $this->findRoute($request));
}
```
這邊顧名思義，邏輯也很清楚了。透過 `findRoute()` 找到 `$request` 所對應的路由，然後再透過 `runRoute()` 實際運作邏輯，將

### `findRoute()`

首先我們找到 `findRoute()`，顧名思義，這是一個把傳輸進來的 `$request` 和 `$routes` 做比對的函式

```php
protected function findRoute($request)
{
    $this->current = $route = $this->routes->match($request);

    $this->container->instance(Route::class, $route);

    return $route;
}
```

`$this->routes` 是一個 `RouteCollection` 物件，往下追我們會找到

```php
public function match(Request $request)
{
    $routes = $this->get($request->getMethod());

    // First, we will see if we can find a matching route for this current request
    // method. If we can, great, we can just return it so that it can be called
    // by the consumer. Otherwise we will check for routes with another verb.
    $route = $this->matchAgainstRoutes($routes, $request);

    if (! is_null($route)) {
        return $route->bind($request);
    }

    // If no route was found we will now check if a matching route is specified by
    // another HTTP verb. If it is we will need to throw a MethodNotAllowed and
    // inform the user agent of which HTTP verb it should use for this route.
    $others = $this->checkForAlternateVerbs($request);

    if (count($others) > 0) {
        return $this->getRouteForMethods($request, $others);
    }

    throw new NotFoundHttpException;
}
```

看了詳細的註解說明，我們可以理解這一段的商業邏輯了。

為了區分你是完全寫錯路徑，還是只是動詞寫錯，這邊會將存取需求的動詞先抓出來，然後先針對這個需求找對應路徑，找不到再換動詞找看看。

如果換了動詞就可以找到路徑，回傳 `MethodNotAllowedHttpException`，不然就回傳 `NotFoundHttpException`。

#### `get()`

我們來看看 `get()` 的邏輯

```php
/**
 * Get routes from the collection by method.
 *
 * @param  string|null  $method
 * @return array
 */
public function get($method = null)
{
    return is_null($method) ? $this->getRoutes() : Arr::get($this->routes, $method, []);
}
```

如果沒有傳入 `$method`，那就假設是要拿全部的路由，透過 `$this->getRoutes()` 可以取得：

```php
/**
 * Get all of the routes in the collection.
 *
 * @return array
 */
public function getRoutes()
{
    return array_values($this->allRoutes);
}
```

如果有傳入 `$method`，那麼代表需要比對，呼叫 `Arr::get($this->routes, $method, [])` 來取出用這個動詞的路由。

##### `Arr::get()`

這個函式屬於 Laravel Helpers 的一項，開發者可以在自己寫的程式內使用。

我們來看看裡面的實作：

```php
/**
 * Get an item from an array using "dot" notation.
 *
 * @param  \ArrayAccess|array  $array
 * @param  string|int  $key
 * @param  mixed   $default
 * @return mixed
 */
public static function get($array, $key, $default = null)
{
    if (! static::accessible($array)) {
        return value($default);
    }

    if (is_null($key)) {
        return $array;
    }

    if (static::exists($array, $key)) {
        return $array[$key];
    }

    if (strpos($key, '.') === false) {
        return $array[$key] ?? value($default);
    }

    foreach (explode('.', $key) as $segment) {
        if (static::accessible($array) && static::exists($array, $segment)) {
            $array = $array[$segment];
        } else {
            return value($default);
        }
    }

    return $array;
}
```

這個函式實作了針對所謂「"dot" notation」的取值方式。我們看看官網的教學

```php
use Illuminate\Support\Arr;

$array = ['products' => ['desk' => ['price' => 100]]];

$flattened = Arr::dot($array);

// ['products.desk.price' => 100]

```

#### `matchAgainstRoutes()`

我們來看看 `matchAgainstRoutes()`

```php
protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
{
    [$fallbacks, $routes] = collect($routes)->partition(function ($route) {
        return $route->isFallback;
    });

    return $routes->merge($fallbacks)->first(function ($value) use ($request, $includingMethod) {
        return $value->matches($request, $includingMethod);
    });
}
```

前面用了 `Collection` 的 `partition()`，將路由分成 `$fallbacks` 和 `$routes` 兩塊。之後用 `merge()` 合併

暫時不管 `$fallbacks` 我們先看 `matches()` 的實作

```php
public function matches(Request $request, $includingMethod = true)
{
    $this->compileRoute();

    foreach ($this->getValidators() as $validator) {
        if (! $includingMethod && $validator instanceof MethodValidator) {
            continue;
        }

        if (! $validator->matches($this, $request)) {
            return false;
        }
    }

    return true;
}
```


#### 找到路徑

```php
if (! is_null($route)) {
    return $route->bind($request);
}
```

找到路徑的情況下，`$route` 會有值，所以到這裡就會進入 `bind()`

我們稍微追一下 `bind()` 的實作

```php
/**
 * Bind the route to a given request for execution.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return $this
 */
public function bind(Request $request)
{
    $this->compileRoute();

    $this->parameters = (new RouteParameterBinder($this))
                    ->parameters($request);

    $this->originalParameters = $this->parameters;

    return $this;
}
```

第一行是 `compileRoute()`，看起來要理解這段程式這是必要邏輯，我們追下去看

```php
/**
* Compile the route into a Symfony CompiledRoute instance.
*
* @return \Symfony\Component\Routing\CompiledRoute
*/
protected function compileRoute()
{
if (! $this->compiled) {
    $this->compiled = (new RouteCompiler($this))->compile();
}

return $this->compiled;
}
```

然後我們追 `RouteCompiler->compile()`

```php
/**
 * Compile the route.
 *
 * @return \Symfony\Component\Routing\CompiledRoute
 */
public function compile()
{
    $optionals = $this->getOptionalParameters();

    $uri = preg_replace('/\{(\w+?)\?\}/', '{$1}', $this->route->uri());

    return (
        new SymfonyRoute($uri, $optionals, $this->route->wheres, ['utf8' => true], $this->route->getDomain() ?: '')
    )->compile();
}
```

這邊看出一個很有趣的作法：其實所謂的 `compile` 是把資訊都編譯成一個 `SymfonyRoute` 物件。利用 Symfony 這個套件之前做過的事情，免去需要自己重新動手做的麻煩。

這樣做的優點是，可以節省掉很多開發的時間。這也相當符合軟體開發常說的「不要重新發明輪子」的說法。既然路由這件事情 Symfony 已經有做過，而且做得不錯，我們就沿用下去，然後針對我們認為需要改進的地方做調整就好。

這個做法的缺點則是，程式碼的依賴會變得更加複雜。這點可以從 Symfony github 專案內的 [composer.json](https://github.com/symfony/symfony/blob/master/composer.json) 可以看出來。Symfony 大多數都是依賴自己開發的套件，而 Laravel 框架則是依賴許多的第三方套件。

除了依賴變得複雜之外，效能上也會有所犧牲。因為在啟用服務時，你比起直接使用 Symfony 的路由，必定需要經過一段邏輯處理流程，然後才能進入 Symfony 的路由繼續處理。

在這件事上，我們可以看到在這個議題上，Laravel 選擇了使用他人套件這個做法。至於合適與否，就見仁見智了。

再來我們來看 `(new RouteParameterBinder($this))->parameters($request);` 這一段

```php
/**
 * Get the parameters for the route.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
public function parameters($request)
{
    // If the route has a regular expression for the host part of the URI, we will
    // compile that and get the parameter matches for this domain. We will then
    // merge them into this parameters array so that this array is completed.
    $parameters = $this->bindPathParameters($request);

    // If the route has a regular expression for the host part of the URI, we will
    // compile that and get the parameter matches for this domain. We will then
    // merge them into this parameters array so that this array is completed.
    if (! is_null($this->route->compiled->getHostRegex())) {
        $parameters = $this->bindHostParameters(
            $request, $parameters
        );
    }

    return $this->replaceDefaults($parameters);
}
```



#### 找不到路徑

回到前面的 `match()` 中

```php
if (! is_null($route)) {
    return $route->bind($request);
}
```

如果 `is_null($route)` 是 `true`，那麼就是找不到路徑了，會往下繼續運作，根據動詞錯誤或者根本路徑不存在分開處理。

##### 只是動詞錯誤

往下追蹤，我們會看到這段程式碼

```php
// If no route was found we will now check if a matching route is specified by
// another HTTP verb. If it is we will need to throw a MethodNotAllowed and
// inform the user agent of which HTTP verb it should use for this route.
$others = $this->checkForAlternateVerbs($request);
```

這邊可以看到大量的註解，而且是很有價值的運作邏輯註解。告訴我們 `checkForAlternateVerbs()` 是為了處理「如果是動詞用錯，我們要拋出 `MethodNotAllowed`，並告訴使用者可以用哪些動詞」這個邏輯。

我們繼續往下看看

```php
/**
 * Determine if any routes match on another HTTP verb.
 *
 * @param  \Illuminate\Http\Request  $request
 * @return array
 */
protected function checkForAlternateVerbs($request)
{
    $methods = array_diff(Router::$verbs, [$request->getMethod()]);

    // Here we will spin through all verbs except for the current request verb and
    // check to see if any routes respond to them. If they do, we will return a
    // proper error response with the correct headers on the response string.
    $others = [];

    foreach ($methods as $method) {
        if (! is_null($this->matchAgainstRoutes($this->get($method), $request, false))) {
            $others[] = $method;
        }
    }

    return $others;
}
```

利用 `array_diff()` 這個 PHP 函式，我們可以找到除了 `$request`的動詞外的其他動詞。

`Router::$verbs` 有

```php
public static $verbs = ['GET', 'HEAD', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'];
```

`matchAgainstRoutes()` 我們之前看過，在下面這段裡：

```php
public function match(Request $request)
{
    $routes = $this->get($request->getMethod());

    // First, we will see if we can find a matching route for this current request
    // method. If we can, great, we can just return it so that it can be called
    // by the consumer. Otherwise we will check for routes with another verb.
    $route = $this->matchAgainstRoutes($routes, $request);
```

這時我們就可以感覺到，為什麼 `matchAgainstRoutes()` 當初會設計成

```php
/**
 * Determine if a route in the array matches the request.
 *
 * @param  array  $routes
 * @param  \Illuminate\Http\Request  $request
 * @param  bool  $includingMethod
 * @return \Illuminate\Routing\Route|null
 */
protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
```

因為這樣設計，在現在這個場景，我們就可以共用 `matchAgainstRoutes()` 這個函式。

後面的邏輯就很簡單了，在一一比對過所有動詞後，把可以用的動詞都寫進 `$others` 陣列裡面，然後回傳。

回傳其他可用的動詞之後，我們到

```
if (count($others) > 0) {
    return $this->getRouteForMethods($request, $others);
}
```

如果有其他可用動詞，我們會進入到 `getRouteForMethods()`

```php
protected function getRouteForMethods($request, array $methods)
{
    if ($request->method() === 'OPTIONS') {
        return (new Route('OPTIONS', $request->path(), function () use ($methods) {
            return new Response('', 200, ['Allow' => implode(',', $methods)]);
        }))->bind($request);
    }

    $this->methodNotAllowed($methods, $request->method());
}
```

這邊處理了有關 `'OPTIONS'` 這個選項的邏輯。我們參考 [MDN Web Docs](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS) 如果 HTTP 動詞是 `OPTIONS`，那應該要回傳本路徑可以用的所有動詞。

如果不是 `'OPTIONS'` 這個動詞，那麼就運作 `methodNotAllowed()`

```php
/**
 * Throw a method not allowed HTTP exception.
 *
 * @param  array  $others
 * @param  string  $method
 * @return void
 *
 * @throws \Symfony\Component\HttpKernel\Exception\MethodNotAllowedHttpException
 */
protected function methodNotAllowed(array $others, $method)
{
    throw new MethodNotAllowedHttpException(
        $others,
        sprintf(
            'The %s method is not supported for this route. Supported methods: %s.',
            $method,
            implode(', ', $others)
        )
    );
}
```

到這裏，寫錯動詞的路徑設定就完整了。

##### 完全找不到路徑

如果到 `count($others) > 0` 不成立，那麼我們就會到下一段落

```php
throw new NotFoundHttpException;
```

完全找不到路徑的邏輯就這麼簡單。


### `runRoute()`

`findRoute()` 看完，進到 `runRoute()` 之後，我們就可以一探實際運作框架邏輯的起源了。

找到對應的路由之後，

```php
/**
 * Return the response for the given route.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Illuminate\Routing\Route  $route
 * @return \Symfony\Component\HttpFoundation\Response
 */
protected function runRoute(Request $request, Route $route)
{
    $request->setRouteResolver(function () use ($route) {
        return $route;
    });

    $this->events->dispatch(new Events\RouteMatched($route, $request));

    return $this->prepareResponse($request,
        $this->runRouteWithinStack($route, $request)
    );
}
```

這裡面邏輯比較複雜的是 `runRouteWithinStack()` 和 `prepareResponse()` 兩個函式。

```php
protected function runRouteWithinStack(Route $route, Request $request)
{
    $shouldSkipMiddleware = $this->container->bound('middleware.disable') &&
                            $this->container->make('middleware.disable') === true;

    $middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route);

    return (new Pipeline($this->container))
                    ->send($request)
                    ->through($middleware)
                    ->then(function ($request) use ($route) {
                        return $this->prepareResponse(
                            $request, $route->run()
                        );
                    });
}
```

這段的商業邏輯是將 `$middleware` 都找出來，然後用一個 `Pipeline` 做依序處理。

全部的 `middleware` 把事情都做完之後，才會跑到 `$route->run()`

```php
/**
 * Run the route action and return the response.
 *
 * @return mixed
 */
public function run()
{
    $this->container = $this->container ?: new Container;

    try {
        if ($this->isControllerAction()) {
            return $this->runController();
        }

        return $this->runCallable();
    } catch (HttpResponseException $e) {
        return $e->getResponse();
    }
}
```

這段的商業邏輯就很單純了，只是將收到的路徑，區分成第二個參數是匿名函式還是一個 Controller Action 字串。根據狀況分成不同的函式處理。

往下的部分下次我們再進行討論，這邊我們就追蹤到這裡。

`prepareResponse()` 的部分實作如下

```php
public function prepareResponse($request, $response)
{
    return static::toResponse($request, $response);
}
```
這只是包裝了一個靜態函式，我們看看 `static::toResponse()` 的實作

```php
public static function toResponse($request, $response)
{
    if ($response instanceof Responsable) {
        $response = $response->toResponse($request);
    }

    if ($response instanceof PsrResponseInterface) {
        $response = (new HttpFoundationFactory)->createResponse($response);
    } elseif ($response instanceof Model && $response->wasRecentlyCreated) {
        $response = new JsonResponse($response, 201);
    } elseif (! $response instanceof SymfonyResponse &&
               ($response instanceof Arrayable ||
                $response instanceof Jsonable ||
                $response instanceof ArrayObject ||
                $response instanceof JsonSerializable ||
                is_array($response))) {
        $response = new JsonResponse($response);
    } elseif (! $response instanceof SymfonyResponse) {
        $response = new Response($response);
    }

    if ($response->getStatusCode() === Response::HTTP_NOT_MODIFIED) {
        $response->setNotModified();
    }

    return $response->prepare($request);
}
```

這邊就可以看到，這段邏輯是根據收到的 `$response` 具體來說是什麼物件所進行的後續處理。

如果像我們測試常做的，回傳純字串，那麼就會被包裝成 `SymfonyResponse` 然後回傳。

如果是回傳 Model，那就會包裝成 `JsonResponse`。

到這邊，有關 

```php
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```

這段函式背後所做的事情，就算是有個大致的掌握了。
