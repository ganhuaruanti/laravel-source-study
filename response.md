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

所以我們可以下結論，如果 `fastcgi_finish_request()` 存在，Symfony 可以利用 `fastcgi_finish_request()` 將資料回傳給用戶，不然就透過 `closeOutputBuffers()` 回傳。`closeOutputBuffers()` 其實做為

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

所以，更詳細的說，如果系統沒法使用 `fastcgi_finish_request()` 的話，那就是利用 output buffers 系列的函式，比方說 `ob_get_status()`，`ob_end_flush()`，`ob_end_clean()` 來進行處理。

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

覆寫了原本的 `setContent()` 之後，我們就可以處理回傳了，以下我們針對幾種不同的回傳形式來追蹤

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


## Renderable

再來我們看到下一段

```php
// If this content implements the "Renderable" interface then we will call the
// render method on the object so we will avoid any "__toString" exceptions
// that might be thrown and have their errors obscured by PHP's handling.
elseif ($content instanceof Renderable) {
    $content = $content->render();
}
```

`Renderable` 的全部路徑是 `Illuminate\Contracts\Support\Renderable`，可以推測這個是另一個 Laravel 建立的介面。

我們來看看哪些物件實作或者延伸這個介面。

我們會看到三個部分：

* `interface View extends Renderable`
* `class Mailable implements MailableContract, Renderable`
* `class MailMessage extends SimpleMessage implements Renderable`

這三個部分顯然最重要的是 `View` 這個介面，因為往下延伸的話，可以看到整個 `View` 的處理過程，這是 MVC 框架設計另一個很大的邏輯。

不過今天我們先專注於 response 的處理，有關 `View` 的處理先打住。

## `parent::setContent()`

繼續往下我們看到

```php
parent::setContent($content);

return $this;
```

這樣的設計結構可能對一些人有點陌生。簡單的說，當你繼承了一個有商業邏輯的類別，你只想要在該類別的某些邏輯上增減東西，但是不想重新複製所有的內容時，就會選擇這樣的做法：先處理一些自己的邏輯，之後呼叫父類別的函式，如果做完了之後你還想處理部分內容，你可以在後面加上自己的邏輯。

我們來看看 `parent::setContent()` 的實作

```php
/**
 * Sets the response content.
 *
 * Valid types are strings, numbers, null, and objects that implement a __toString() method.
 *
 * @param mixed $content Content that can be cast to string
 *
 * @return $this
 *
 * @throws \UnexpectedValueException
 */
public function setContent($content)
{
    if (null !== $content && !\is_string($content) && !is_numeric($content) && !\is_callable([$content, '__toString'])) {
        throw new \UnexpectedValueException(sprintf('The Response content must be a string or object implementing __toString(), "%s" given.', \gettype($content)));
    }

    $this->content = (string) $content;

    return $this;
}
```

這裡主要是進行錯誤處理，如果被強迫設置無法轉成字串的內容時，拋出一個 PHP 標準 library（PHP SPL）的 `UnexpectedValueException`。

接著是 `$this->content = (string) $content;` 的轉型，然後 `return $this;` 回傳自己。

這種設計方式被稱為流式接口（fluent interface），簡單的說就是將資料包裝起來，然後每個對資料的操作最後不僅僅是回傳資料或不回傳，而是回傳物件自身。這樣的話，使用該類別時就可以寫出類似的結構：

```php
$foo->doBar()
    ->doBaz()
    ->setQux('qux')
    ->otherCall()
    ->getAllTheThings();
```

不過我們看到，Laravel 的實作並沒有實際利用這個回傳。僅僅是呼叫了之後，一樣回傳了 `$this`，也就是一個 `Illuminate\Http\Response` 物件。

到這邊，整個流程大概就走過一輪了，也提升了我們對整個框架的基本認識。
