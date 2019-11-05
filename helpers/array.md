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

## `Arr::add()`

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

這裡的實作有點有趣，利用 `array_shift()` 和 `&$array[]` 的存取方式。

看到這裏，基本上就已經看完這個函式了，我們可以往下一個去看。

## `Arr::collapse()`


```php
/**
 * Collapse an array of arrays into a single array.
 *
 * @param  array  $array
 * @return array
 */
public static function collapse($array)
{
    $results = [];

    foreach ($array as $values) {
        if ($values instanceof Collection) {
            $values = $values->all();
        } elseif (! is_array($values)) {
            continue;
        }

        $results = array_merge($results, $values);
    }

    return $results;
}
```

這裡有個小細節比較值得提出，如果遇到的是 Laravel 提供的 Collection 物件，會改用 `all()` 取出陣列。其他則利用 `array_merge()` 合併

## `Arr::divide()`

```php
/**
 * Divide an array into two arrays. One with keys and the other with values.
 *
 * @param  array  $array
 * @return array
 */
public static function divide($array)
{
    return [array_keys($array), array_values($array)];
}
```

## `Arr::dot()`

```php
/**
 * Flatten a multi-dimensional associative array with dots.
 *
 * @param  array   $array
 * @param  string  $prepend
 * @return array
 */
public static function dot($array, $prepend = '')
{
    $results = [];

    foreach ($array as $key => $value) {
        if (is_array($value) && ! empty($value)) {
            $results = array_merge($results, static::dot($value, $prepend.$key.'.'));
        } else {
            $results[$prepend.$key] = $value;
        }
    }

    return $results;
}
```

這段函式比較有趣的地方，個人認為是竟然用遞迴的方式實作。

## `Arr::except()`

```php
/**
 * Get all of the given array except for a specified array of keys.
 *
 * @param  array  $array
 * @param  array|string  $keys
 * @return array
 */
public static function except($array, $keys)
{
    static::forget($array, $keys);

    return $array;
}
```

要弄清楚只好看看 `forget()` 的實作

```php
/**
 * Remove one or many array items from a given array using "dot" notation.
 *
 * @param  array  $array
 * @param  array|string  $keys
 * @return void
 */
public static function forget(&$array, $keys)
{
    $original = &$array;

    $keys = (array) $keys;

    if (count($keys) === 0) {
        return;
    }

    foreach ($keys as $key) {
        // if the exact key exists in the top-level, remove it
        if (static::exists($array, $key)) {
            unset($array[$key]);

            continue;
        }

        $parts = explode('.', $key);

        // clean up before each pass
        $array = &$original;

        while (count($parts) > 1) {
            $part = array_shift($parts);

            if (isset($array[$part]) && is_array($array[$part])) {
                $array = &$array[$part];
            } else {
                continue 2;
            }
        }

        unset($array[array_shift($parts)]);
    }
}
```

## `Arr::first()`

```php
/**
 * Return the first element in an array passing a given truth test.
 *
 * @param  array  $array
 * @param  callable|null  $callback
 * @param  mixed  $default
 * @return mixed
 */
public static function first($array, callable $callback = null, $default = null)
{
    if (is_null($callback)) {
        if (empty($array)) {
            return value($default);
        }

        foreach ($array as $item) {
            return $item;
        }
    }

    foreach ($array as $key => $value) {
        if (call_user_func($callback, $value, $key)) {
            return $value;
        }
    }

    return value($default);
}
```

比較複雜的地方應該是納入 `$callback` 的部分，其他地方還好。

## `Arr::flatten()`

```php
/**
 * Flatten a multi-dimensional array into a single level.
 *
 * @param  array  $array
 * @param  int  $depth
 * @return array
 */
public static function flatten($array, $depth = INF)
{
    $result = [];

    foreach ($array as $item) {
        $item = $item instanceof Collection ? $item->all() : $item;

        if (! is_array($item)) {
            $result[] = $item;
        } elseif ($depth === 1) {
            $result = array_merge($result, array_values($item));
        } else {
            $result = array_merge($result, static::flatten($item, $depth - 1));
        }
    }
}
```

跟 `Arr::dot()` 一樣，用遞迴的方式實作。

## `Arr::forget()`

前面 `except()` 時有看過了，這邊不再贅述。

## `Arr::get()`

前面 `add()` 時有看過了，這邊不再贅述。

## `Arr::has()`

```php
/**
 * Check if an item or items exist in an array using "dot" notation.
 *
 * @param  \ArrayAccess|array  $array
 * @param  string|array  $keys
 * @return bool
 */
public static function has($array, $keys)
{
    $keys = (array) $keys;

    if (! $array || $keys === []) {
        return false;
    }

    foreach ($keys as $key) {
        $subKeyArray = $array;

        if (static::exists($array, $key)) {
            continue;
        }

        foreach (explode('.', $key) as $segment) {
            if (static::accessible($subKeyArray) && static::exists($subKeyArray, $segment)) {
                $subKeyArray = $subKeyArray[$segment];
            } else {
                return false;
            }
        }
    }

    return true;
}
```

可以看出實作上用到了 `exists()` 和 `accessible()` 我們來看看這兩個函式

首先看 `exists()`

```php
/**
 * Determine if the given key exists in the provided array.
 *
 * @param  \ArrayAccess|array  $array
 * @param  string|int  $key
 * @return bool
 */
public static function exists($array, $key)
{
    if ($array instanceof ArrayAccess) {
        return $array->offsetExists($key);
    }

    return array_key_exists($key, $array);
}
```

這邊應該是為了解決如果輸入值是某個實作 PHP 提供的 `ArrayAccess` 介面，不是 PHP 原生的陣列物件，那麼無法用 `array_key_exists()` 的問題。

看完之後，我們來看看 `accessible()`

```php
/**
 * Determine whether the given value is array accessible.
 *
 * @param  mixed  $value
 * @return bool
 */
public static function accessible($value)
{
    return is_array($value) || $value instanceof ArrayAccess;
}
```

兩個都是 PHP 原生語法，就不繼續追蹤下去了。

## `Arr::last()`

```php
/**
 * Return the last element in an array passing a given truth test.
 *
 * @param  array  $array
 * @param  callable|null  $callback
 * @param  mixed  $default
 * @return mixed
 */
public static function last($array, callable $callback = null, $default = null)
{
    if (is_null($callback)) {
        return empty($array) ? value($default) : end($array);
    }

    return static::first(array_reverse($array, true), $callback, $default);
}
```

沒想到竟然用了 `array_reverse()`。

## `Arr::only()`

```php
/**
 * Get a subset of the items from the given array.
 *
 * @param  array  $array
 * @param  array|string  $keys
 * @return array
 */
public static function only($array, $keys)
{
    return array_intersect_key($array, array_flip((array) $keys));
}
```

很簡短的作法，不過看懂了之後，會覺得很有巧思。

## `Arr::pluck()`

pluck（拔出）這個字我比較陌生，所以我會從官網先看看實作的目標

    The Arr::pluck method retrieves all of the values for a given key from an array:

    use Illuminate\Support\Arr;

    $array = [
        ['developer' => ['id' => 1, 'name' => 'Taylor']],
        ['developer' => ['id' => 2, 'name' => 'Abigail']],
    ];

    $names = Arr::pluck($array, 'developer.name');

    // ['Taylor', 'Abigail']
    
    You may also specify how you wish the resulting list to be keyed:

    use Illuminate\Support\Arr;

    $names = Arr::pluck($array, 'developer.name', 'developer.id');

    // [1 => 'Taylor', 2 => 'Abigail']

這樣我們大概有點理解，看起來就是從原本的陣列中，依據傳入的 key 拔出對應的 value。

我們往下看看實作

```php
/**
 * Pluck an array of values from an array.
 *
 * @param  array  $array
 * @param  string|array  $value
 * @param  string|array|null  $key
 * @return array
 */
public static function pluck($array, $value, $key = null)
{
    $results = [];

    [$value, $key] = static::explodePluckParameters($value, $key);

    foreach ($array as $item) {
        $itemValue = data_get($item, $value);

        // If the key is "null", we will just append the value to the array and keep
        // looping. Otherwise we will key the array using the value of the key we
        // received from the developer. Then we'll return the final array form.
        if (is_null($key)) {
            $results[] = $itemValue;
        } else {
            $itemKey = data_get($item, $key);

            if (is_object($itemKey) && method_exists($itemKey, '__toString')) {
                $itemKey = (string) $itemKey;
            }

            $results[$itemKey] = $itemValue;
        }
    }

    return $results;
}
```

第一個我們會遇到的是 `explodePluckParameters()` 我們來看看其實作

```php
/**
 * Explode the "value" and "key" arguments passed to "pluck".
 *
 * @param  string|array  $value
 * @param  string|array|null  $key
 * @return array
 */
protected static function explodePluckParameters($value, $key)
{
    $value = is_string($value) ? explode('.', $value) : $value;

    $key = is_null($key) || is_array($key) ? $key : explode('.', $key);

    return [$value, $key];
}
```

看完怎麼剖析傳入參數後，我們來看看 `data_get()`

```php
if (! function_exists('data_get')) {
    /**
     * Get an item from an array or object using "dot" notation.
     *
     * @param  mixed   $target
     * @param  string|array|int  $key
     * @param  mixed   $default
     * @return mixed
     */
    function data_get($target, $key, $default = null)
    {
        if (is_null($key)) {
            return $target;
        }

        $key = is_array($key) ? $key : explode('.', $key);

        while (! is_null($segment = array_shift($key))) {
            if ($segment === '*') {
                if ($target instanceof Collection) {
                    $target = $target->all();
                } elseif (! is_array($target)) {
                    return value($default);
                }

                $result = [];

                foreach ($target as $item) {
                    $result[] = data_get($item, $key);
                }

                return in_array('*', $key) ? Arr::collapse($result) : $result;
            }

            if (Arr::accessible($target) && Arr::exists($target, $segment)) {
                $target = $target[$segment];
            } elseif (is_object($target) && isset($target->{$segment})) {
                $target = $target->{$segment};
            } else {
                return value($default);
            }
        }

        return $target;
    }
}
```

















