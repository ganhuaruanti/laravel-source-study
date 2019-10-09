# 回傳

複習一下，Laravel 裡面 `index.php` 的運作邏輯是

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

$response->send();

$kernel->terminate($request, $response);
```

前面兩段看完了，下一段就是 `$response->send()` 囉！我們來看看這段的運作邏輯

## `send()`

追蹤過之後，我們會發現 `$kernel->handle()`，除非我們抽換了 `Illuminate\Contracts\Http\Kernel` 所對應的實體類別，不然回傳的會是 `Illuminate\Http\Response` 類別的物件。

我們來看看 `Illuminate\Http\Response` 這個類別

```php
class Response extends BaseResponse
{
    use ResponseTrait, Macroable {
        Macroable::__call as macroCall;
    }

    /**
     * Set the content on the response.
     *
     * @param  mixed  $content
     * @return $this
     */
    public function setContent($content)
    {
        $this->original = $content;

        // If the content is "JSONable" we will set the appropriate header and convert
        // the content to JSON. This is useful when returning something like models
        // from routes that will be automatically transformed to their JSON form.
        if ($this->shouldBeJson($content)) {
            $this->header('Content-Type', 'application/json');

            $content = $this->morphToJson($content);
        }

        // If this content implements the "Renderable" interface then we will call the
        // render method on the object so we will avoid any "__toString" exceptions
        // that might be thrown and have their errors obscured by PHP's handling.
        elseif ($content instanceof Renderable) {
            $content = $content->render();
        }

        parent::setContent($content);

        return $this;
    }

    /**
     * Determine if the given content should be turned into JSON.
     *
     * @param  mixed  $content
     * @return bool
     */
    protected function shouldBeJson($content)
    {
        return $content instanceof Arrayable ||
               $content instanceof Jsonable ||
               $content instanceof ArrayObject ||
               $content instanceof JsonSerializable ||
               is_array($content);
    }

    /**
     * Morph the given content into JSON.
     *
     * @param  mixed   $content
     * @return string
     */
    protected function morphToJson($content)
    {
        if ($content instanceof Jsonable) {
            return $content->toJson();
        } elseif ($content instanceof Arrayable) {
            return json_encode($content->toArray());
        }

        return json_encode($content);
    }
}
```

這麼重要的類別，但是裡面竟然沒有多少內容！而且還沒有前面所宣告的 `send()`

這其中的重點，就在 `class Response extends BaseResponse` 這句話裡面。

往上面追蹤，我們會找到 `use Symfony\Component\HttpFoundation\Response as BaseResponse;` 這段。

這下，我們就可以清楚看到，這又是一個利用 Symfony 套件來減少需要做工的選擇。利用 Symfony 已經做好過的回應功能，來減少框架開發時所需要的工作。

要弄清楚到底做了什麼，我們繼續往下看 `Symfony\Component\HttpFoundation\Response` 對 `send()` 的實作。

```php
/**
 * Sends HTTP headers and content.
 *
 * @return $this
 */
public function send()
{
    $this->sendHeaders();
    $this->sendContent();

    if (\function_exists('fastcgi_finish_request')) {
        fastcgi_finish_request();
    } elseif (!\in_array(\PHP_SAPI, ['cli', 'phpdbg'], true)) {
        static::closeOutputBuffers(0, true);
    }

    return $this;
}
```

繼續檢查，我們可以找到 php.net [對 fastcgi_finish_request 的說明](https://www.php.net/manual/en/function.fastcgi-finish-request.php)

所以我們可以下結論，如果 `fastcgi_finish_request()` 存在，Symfony 可以利用 `fastcgi_finish_request()` 將資料回傳給用戶，不然就透過 `closeOutputBuffers()` 回傳。其實做為

```php
/**
 * Cleans or flushes output buffers up to target level.
 *
 * Resulting level can be greater than target level if a non-removable buffer has been encountered.
 *
 * @final
 */
public static function closeOutputBuffers(int $targetLevel, bool $flush)
{
    $status = ob_get_status(true);
    $level = \count($status);
    $flags = PHP_OUTPUT_HANDLER_REMOVABLE | ($flush ? PHP_OUTPUT_HANDLER_FLUSHABLE : PHP_OUTPUT_HANDLER_CLEANABLE);

    while ($level-- > $targetLevel && ($s = $status[$level]) && (!isset($s['del']) ? !isset($s['flags']) || ($s['flags'] & $flags) === $flags : $s['del'])) {
        if ($flush) {
            ob_end_flush();
        } else {
            ob_end_clean();
        }
    }
}
```

如果沒法使用 `fastcgi_finish_request()` 那就是利用 output buffers 系列的函式，比方說 `ob_get_status()`，`ob_end_flush()`，`ob_end_clean()` 來進行處理。

回到 Laravel 的 Response 物件，我們前面可以注意到，Laravel 似乎只覆寫了 `sendContent()`，我們先看看 Symfony 的實作

```php
/**
 * Sends content for the current web response.
 *
 * @return $this
 */
public function sendContent()
{
    echo $this->content;

    return $this;
}
```

非常單純的 `echo`，那麼，我們來比對 Laravel 的實作

```php
/**
 * Set the content on the response.
 *
 * @param  mixed  $content
 * @return $this
 */
public function setContent($content)
{
    $this->original = $content;

    // If the content is "JSONable" we will set the appropriate header and convert
    // the content to JSON. This is useful when returning something like models
    // from routes that will be automatically transformed to their JSON form.
    if ($this->shouldBeJson($content)) {
        $this->header('Content-Type', 'application/json');

        $content = $this->morphToJson($content);
    }

    // If this content implements the "Renderable" interface then we will call the
    // render method on the object so we will avoid any "__toString" exceptions
    // that might be thrown and have their errors obscured by PHP's handling.
    elseif ($content instanceof Renderable) {
        $content = $content->render();
    }

    parent::setContent($content);

    return $this;
}
```

搭配上註解，我們可以看出來，這是基於 Symfony 的基礎上，加上針對 json 和 Renderable 的實作。

覆寫了原本的 `setContent()` 之後，我們就可以

### json

我們先看看 json 的部分。`shouldBeJson()` 非常明顯，是一個回傳 `bool` 的判斷式

```
/**
 * Determine if the given content should be turned into JSON.
 *
 * @param  mixed  $content
 * @return bool
 */
protected function shouldBeJson($content)
{
    return $content instanceof Arrayable ||
           $content instanceof Jsonable ||
           $content instanceof ArrayObject ||
           $content instanceof JsonSerializable ||
           is_array($content);
}
```

`$this->header('Content-Type', 'application/json');` 這段也不難理解。我們來看看 `morphToJson()` 的實作

```php
/**
 * Morph the given content into JSON.
 *
 * @param  mixed   $content
 * @return string
 */
protected function morphToJson($content)
{
    if ($content instanceof Jsonable) {
        return $content->toJson();
    } elseif ($content instanceof Arrayable) {
        return json_encode($content->toArray());
    }

    return json_encode($content);
}
```

這裏利用 `json_encode()` 來處理 `Arrayable` 的 json 回傳。


## 



