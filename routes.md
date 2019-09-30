# 路由

`$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);` 建立了一個

結果我們發現，`Illuminate\Contracts\Http\Kernel` 是一個介面！

這下就麻煩了，介面只告訴了我們 `handle()` 函式的存在，並沒有告訴我們裡面的實作，那我們怎麼繼續追蹤其中的邏輯呢？

幸好，透過找哪些類別實作了這個介面，我們只找到了 `Illuminate\Foundation\Http\Kernel` 這個類別

```php
use Illuminate\Contracts\Http\Kernel as KernelContract;

class Kernel implements KernelContract
...
```

我們可以推測出，這個類別應該就是 `$app->make(Illuminate\Contracts\Http\Kernel::class)` 實際做出來的物件類別。

另外，我們也可以透過改寫 `index.php` 來檢查這個事情

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
var_dump($kernel);
```

我們會看到

```text
object(App\Http\Kernel)#38 (7) { [...
```

到 `App\Http\Kernel` 這個類別一看，他並沒有實作任何函式，但是 `App\Http\Kernel` 繼承了 `Illuminate\Foundation\Http\Kernel`，所以我們可以知道，實作 `handle()` 函式的實際位置，是在 `Illuminate\Foundation\Http\Kernel` 裡面。

現在我們來看看 `Illuminate\Foundation\Http\Kernel` 這個類別。

這個類別針對 `handle()` 的實作如下

```
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

看起來好像很多，不過我們可以發現到，這裡面大多數都是例外處理。`$request->enableHttpMethodParameterOverride();` 所做的事情是設置參數。

所以真正的邏輯是在 `$response = $this->sendRequestThroughRouter($request);` 裡面。

我們到這個函式裡面再看看

```
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

往下追我們會找到

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

前面用了 `Collection` 的 `partition()`，將

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


#### 一般情況

#### 找不到路徑
##### 只是動詞錯誤

##### 完全找不到路徑

### `runRoute()`

進到 `runRoute()` 之後，我們就可以一探實際運作框架邏輯的起源了。

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

如果像我們測試常做的回傳純字串，那麼就會被包裝成 `SymfonyResponse` 然後回傳。

