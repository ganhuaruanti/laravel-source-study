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

看到這麼複雜的實作方式，感覺很有疑問，不過看官網說明也沒看出什麼端倪。

這時候，我們可以看看該函式對應的測試，找看看有沒有什麼線索：

```php
public function testFinish()
{
    $this->assertEquals('abbc', Str::finish('ab', 'bc'));
    $this->assertEquals('abbc', Str::finish('abbcbc', 'bc'));
    $this->assertEquals('abcbbc', Str::finish('abcbbcbc', 'bc'));
}
```

## `Str::is()`

```php
/**
 * Determine if a given string matches a given pattern.
 *
 * @param  string|array  $pattern
 * @param  string  $value
 * @return bool
 */
public static function is($pattern, $value)
{
    $patterns = Arr::wrap($pattern);

    if (empty($patterns)) {
        return false;
    }

    foreach ($patterns as $pattern) {
        // If the given value is an exact match we can of course return true right
        // from the beginning. Otherwise, we will translate asterisks and do an
        // actual pattern match against the two strings to see if they match.
        if ($pattern == $value) {
            return true;
        }

        $pattern = preg_quote($pattern, '#');

        // Asterisks are translated into zero-or-more regular expression wildcards
        // to make it convenient to check if the strings starts with the given
        // pattern such as "library/*", making any string check convenient.
        $pattern = str_replace('\*', '.*', $pattern);

        if (preg_match('#^'.$pattern.'\z#u', $value) === 1) {
            return true;
        }
    }

    return false;
}
```

## `Str::kebab()`

```php
/**
 * Convert a string to kebab case.
 *
 * @param  string  $value
 * @return string
 */
public static function kebab($value)
{
    return static::snake($value, '-');
}
```

```php
/**
 * Convert a string to snake case.
 *
 * @param  string  $value
 * @param  string  $delimiter
 * @return string
 */
public static function snake($value, $delimiter = '_')
{
    $key = $value;

    if (isset(static::$snakeCache[$key][$delimiter])) {
        return static::$snakeCache[$key][$delimiter];
    }

    if (! ctype_lower($value)) {
        $value = preg_replace('/\s+/u', '', ucwords($value));

        $value = static::lower(preg_replace('/(.)(?=[A-Z])/u', '$1'.$delimiter, $value));
    }

    return static::$snakeCache[$key][$delimiter] = $value;
}
```

## `Str::limit()`

```php
/**
 * Limit the number of characters in a string.
 *
 * @param  string  $value
 * @param  int     $limit
 * @param  string  $end
 * @return string
 */
public static function limit($value, $limit = 100, $end = '...')
{
    if (mb_strwidth($value, 'UTF-8') <= $limit) {
        return $value;
    }

    return rtrim(mb_strimwidth($value, 0, $limit, '', 'UTF-8')).$end;
}
```

## `Str::orderedUuid()`

```php
/**
 * Generate a time-ordered UUID (version 4).
 *
 * @return \Ramsey\Uuid\UuidInterface
 */
public static function orderedUuid()
{
    $factory = new UuidFactory;

    $factory->setRandomGenerator(new CombGenerator(
        $factory->getRandomGenerator(),
        $factory->getNumberConverter()
    ));

    $factory->setCodec(new TimestampFirstCombCodec(
        $factory->getUuidBuilder()
    ));

    return $factory->uuid4();
}
```

## `Str::plural()`

```php
/**
 * Get the plural form of an English word.
 *
 * @param  string  $value
 * @param  int     $count
 * @return string
 */
public static function plural($value, $count = 2)
{
    return Pluralizer::plural($value, $count);
}
```

```php
/**
 * Get the plural form of an English word.
 *
 * @param  string  $value
 * @param  int     $count
 * @return string
 */
public static function plural($value, $count = 2)
{
    if ((int) abs($count) === 1 || static::uncountable($value)) {
        return $value;
    }

    $plural = Inflector::pluralize($value);

    return static::matchCase($plural, $value);
}
```
