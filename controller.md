# 進入控制器行為

在講到[路由](/routes.md)的章節中，找到路徑之後運作的可能有兩項：

```php
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

如果 `$this->isControllerAction()` 成立，代表路由是指定某個控制器的動作，所以呼叫 `runController()`

不然就呼叫 `runCallable()`

這邊我們專注於控制器的行為，先不處理 `runCallable()`，我們來看 `runController()`。

## `runController()`

`runController()` 實作是

```php
/**
 * Run the route action and return the response.
 *
 * @return mixed
 *
 * @throws \Symfony\Component\HttpKernel\Exception\NotFoundHttpException
 */
protected function runController()
{
    return $this->controllerDispatcher()->dispatch(
        $this, $this->getController(), $this->getControllerMethod()
    );
}
```

這裡我們見到了 `controllerDispatcher()` 函式

```php
/**
 * Get the dispatcher for the route's controller.
 *
 * @return \Illuminate\Routing\Contracts\ControllerDispatcher
 */
public function controllerDispatcher()
{
    if ($this->container->bound(ControllerDispatcherContract::class)) {
        return $this->container->make(ControllerDispatcherContract::class);
    }

    return new ControllerDispatcher($this->container);
}
```

如果註冊了 `ControllerDispatcherContract` 應該對應的實作類別，那麼就會通過 `$this->container->make()` 實作出該類別，否則會實作 `ControllerDispatcher()`

這個模式很有趣，不過我們今天要鑽研的是控制器的流程，我們先看預設的 `ControllerDispatcher->dispatch()` 的實作細節。


```php
/**
 * Dispatch a request to a given controller and method.
 *
 * @param  \Illuminate\Routing\Route  $route
 * @param  mixed  $controller
 * @param  string  $method
 * @return mixed
 */
public function dispatch(Route $route, $controller, $method)
{
    $parameters = $this->resolveClassMethodDependencies(
        $route->parametersWithoutNulls(), $controller, $method
    );

    if (method_exists($controller, 'callAction')) {
        return $controller->callAction($method, $parameters);
    }

    return $controller->{$method}(...array_values($parameters));
}
```
