 # 輔助方法 - 字串
 
 跟陣列差不多的，Laravel 提供了 `Illuminate\Support\Str` 物件，來協助一些字串的處理。我們來看看這個物件的實作
 
 ```php
 use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidFactory;
use Illuminate\Support\Traits\Macroable;
use Ramsey\Uuid\Generator\CombGenerator;
use Ramsey\Uuid\Codec\TimestampFirstCombCodec;

class Str
{
    use Macroable;
    // 其他內容
}
 ```
 
 裡面有大約七百多行，顯然不是一次可以看完的內容，我們下面依據官網的列出的順序，一個一個閱讀這些函式實作的方式
 
 ## `Str::after()`

```php
/**
 * Return the remainder of a string after a given value.
 *
 * @param  string  $subject
 * @param  string  $search
 * @return string
 */
public static function after($subject, $search)
{
    return $search === '' ? $subject : array_reverse(explode($search, $subject, 2))[0];
}
```

## `Str::before()`

```php
/**
 * Get the portion of a string before a given value.
 *
 * @param  string  $subject
 * @param  string  $search
 * @return string
 */
public static function before($subject, $search)
{
    return $search === '' ? $subject : explode($search, $subject)[0];
}
```

## `Str::camel()`

```php
/**
 * Convert a value to camel case.
 *
 * @param  string  $value
 * @return string
 */
public static function camel($value)
{
    if (isset(static::$camelCache[$value])) {
        return static::$camelCache[$value];
    }

    return static::$camelCache[$value] = lcfirst(static::studly($value));
}
```

```php
/**
 * Convert a value to studly caps case.
 *
 * @param  string  $value
 * @return string
 */
public static function studly($value)
{
    $key = $value;

    if (isset(static::$studlyCache[$key])) {
        return static::$studlyCache[$key];
    }

    $value = ucwords(str_replace(['-', '_'], ' ', $value));

    return static::$studlyCache[$key] = str_replace(' ', '', $value);
}
```

## `Str::contains()`

```php
/**
 * Determine if a given string contains a given substring.
 *
 * @param  string  $haystack
 * @param  string|array  $needles
 * @return bool
 */
public static function contains($haystack, $needles)
{
    foreach ((array) $needles as $needle) {
        if ($needle !== '' && mb_strpos($haystack, $needle) !== false) {
            return true;
        }
    }

    return false;
}
```

## `Str::containsAll()`

```php
/**
 * Determine if a given string contains all array values.
 *
 * @param  string  $haystack
 * @param  array  $needles
 * @return bool
 */
public static function containsAll($haystack, array $needles)
{
    foreach ($needles as $needle) {
        if (! static::contains($haystack, $needle)) {
            return false;
        }
    }

    return true;
}
```

## `Str::endsWith()`

```php
/**
 * Determine if a given string ends with a given substring.
 *
 * @param  string  $haystack
 * @param  string|array  $needles
 * @return bool
 */
public static function endsWith($haystack, $needles)
{
    foreach ((array) $needles as $needle) {
        if (substr($haystack, -strlen($needle)) === (string) $needle) {
            return true;
        }
    }

    return false;
}
```

## `Str::finish()`

```php
/**
 * Cap a string with a single instance of a given value.
 *
 * @param  string  $value
 * @param  string  $cap
 * @return string
 */
public static function finish($value, $cap)
{
    $quoted = preg_quote($cap, '/');

    return preg_replace('/(?:'.$quoted.')+$/u', '', $value).$cap;
}
```