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

看完這段邏輯，比較有趣的小地方應該是 `value()` 的實作，我們來看看

```php
if (! function_exists('value')) {
    /**
     * Return the default value of the given value.
     *
     * @param  mixed  $value
     * @return mixed
     */
    function value($value)
    {
        return $value instanceof Closure ? $value() : $value;
    }
}
```

其實也沒什麼好說明的，很簡單的實作。

再來我們看 `set()`

```php
/**
 * Set an array item to a given value using "dot" notation.
 *
 * If no key is given to the method, the entire array will be replaced.
 *
 * @param  array   $array
 * @param  string  $key
 * @param  mixed   $value
 * @return array
 */
public static function set(&$array, $key, $value)
{
    if (is_null($key)) {
        return $array = $value;
    }

    $keys = explode('.', $key);

    while (count($keys) > 1) {
        $key = array_shift($keys);

        // If the key doesn't exist at this depth, we will just create an empty array
        // to hold the next value, allowing us to create the arrays to hold final
        // values at the correct depth. Then we'll keep digging into the array.
        if (! isset($array[$key]) || ! is_array($array[$key])) {
            $array[$key] = [];
        }

        $array = &$array[$key];
    }

    $array[array_shift($keys)] = $value;

    return $array;
}
```

這裡的實作有點有趣，利用 `array_shift()` 和 `&$array[]` 的存取方式
