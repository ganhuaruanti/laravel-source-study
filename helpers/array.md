# 輔助方法 - 陣列

Laravel 提供了 `Illuminate\Support\Arr` 物件，我們來看看這個物件的實作

```php
use ArrayAccess;
use InvalidArgumentException;
use Illuminate\Support\Traits\Macroable;

class Arr
{
    use Macroable;
    // 其他內容
}
```

裡面有六百多行，顯然不是可以一次閱讀完的份量，我們下面依據官網的列出的順序，一個一個閱讀這些函式實作的方式

## `Arr::add`

```php
/**
 * Add an element to an array using "dot" notation if it doesn't exist.
 *
 * @param  array   $array
 * @param  string  $key
 * @param  mixed   $value
 * @return array
 */
public static function add($array, $key, $value)
{
    if (is_null(static::get($array, $key))) {
        static::set($array, $key, $value);
    }

    return $array;
}
```

所以會先嘗試 `get()` 看看，如果為 `null` 就執行 `set`，並且最後一定回傳原本陣列。

我們看看 `get()` 和 `set()` 實作，首先看 `get()`

```php
/**
 * Get an item from an array using "dot" notation.
 *
 * @param  \ArrayAccess|array  $array
 * @param  string  $key
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

這邊我們可以看到一個很有趣的設計：「"dot" notation」的實作，這段在前面 `add()` 的註解裡面也有提到。實作的內容如下：

```php
foreach (explode('.', $key) as $segment) {
    if (static::accessible($array) && static::exists($array, $segment)) {
        $array = $array[$segment];
    } else {
        return value($default);
    }
}
```

看完這段邏輯，
