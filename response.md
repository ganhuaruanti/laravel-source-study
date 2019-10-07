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

這下，我們就清楚了，這又是一個利用 Symfony 套件來減少需要做工的選擇。利用 Symfony 已經做好過的回應功能，來減少框架開發時所需要的工作。

要弄清楚到底做了什麼，我們繼續往下看 `Symfony\Component\HttpFoundation\Response` 對 `handle()` 的實作。

