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

