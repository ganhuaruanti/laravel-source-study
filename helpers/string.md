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




