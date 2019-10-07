上次在控制器處理裡面，我們看到了 `Illuminate\Routing\Route` 的 `getController()` 裡面有

```php
$this->controller = $this->container->make(ltrim($class, '\\'));
```

這一段。

根據程式的邏輯，我們可以猜到這一段大概是根據 `$class` 來產生對應類別的物件。

可是，這件事情是怎麼做到的呢？我們一起追蹤 `$this->container->make()` 看看

## `make()`
