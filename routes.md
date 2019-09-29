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

根據命名我們就看出來了，前面處理的邏輯是跟 Middleware 相關的
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)，
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)這邊我們就不
                ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
