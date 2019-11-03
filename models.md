# 模型

要討論 Laravel 的各種便利功能，勢必不可避免需要探討的項目之一，就是 MVC 功能之一，用來和資料庫溝通的 ORM 模型了

在 Laravel 框架內，這個 Model 通常被稱為 Eloquent Model。

下面我們就來聊聊 Laravel 內 Eloquent Model 的運作方式。

## 切入點

幸運的是， Laravel 預設就提供我們一個 `User` 的物件了，我們可以從這個物件作為起點開始研讀。

首先我們到 `app/User.php` 看這個物件

```php
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
```

找到 `Illuminate\Foundation\Auth\User` 我們會看到

```php
<?php

namespace Illuminate\Foundation\Auth;

use Illuminate\Auth\Authenticatable;
use Illuminate\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Auth\Passwords\CanResetPassword;
use Illuminate\Foundation\Auth\Access\Authorizable;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
use Illuminate\Contracts\Auth\Access\Authorizable as AuthorizableContract;
use Illuminate\Contracts\Auth\CanResetPassword as CanResetPasswordContract;

class User extends Model implements
    AuthenticatableContract,
    AuthorizableContract,
    CanResetPasswordContract
{
    use Authenticatable, Authorizable, CanResetPassword, MustVerifyEmail;
}

```

其實做非常簡單，甚至可以說簡單的有點誇張。只有繼承了 `Illuminate\Database\Eloquent\Model` 實作了幾個介面，然後用了四個 trait 就把事情做完了。
 
雖然繼續研究這些東西非常有趣，不過我們在乎的是針對 Eloquent Model 的實作，所以我們來看看 `Illuminate\Database\Eloquent\Model` 的程式碼。
 
我們先看看他的宣告
 
 ```php
abstract class Model implements ArrayAccess, Arrayable, Jsonable, JsonSerializable, QueueableEntity, UrlRoutable
{
    use Concerns\HasAttributes,
        Concerns\HasEvents,
        Concerns\HasGlobalScopes,
        Concerns\HasRelationships,
        Concerns\HasTimestamps,
        Concerns\HidesAttributes,
        Concerns\GuardsAttributes,
        ForwardsCalls;
 ```
 
可以看到，這是一個 `abstract class`，一樣繼承了大量的介面，並使用大量的 trait 來減少程式碼。

不過，把這些 trait 和 `Illuminate\Database\Eloquent\Model` 裡面的一千多行程式碼一次看完，顯然不是一個好的研究方式。

所以我們將問題縮減一下，一次只針對一個問題進行鑽研。

我們這裡先選擇一個問題：

```php
<?php

$user = User::find(1);
return $user->id;
```

當我們使用 Eloquent Model，比方說透過 `User::find` 建立的物件 `$user`，存取 `$user->id` 時，到底經過哪些流程呢？

## 物件參數存取

問題看起來很簡單，不過我們實際下手之後，可能會發現有一點問題：

根據 PHP 的邏輯，你寫了 `$user->id` 要取得 `$user` 物件的參數 `$id`，起碼你要寫類似 `public $id` 之類的宣告吧。

但是很顯然這不是 Laravel 的處理方式，那麼該怎麼繼續下去呢？

這部分會用到 PHP 的一個魔術方法： `__get()`。

我們看看官方網站對 `__get()` 的解說，在 [https://www.php.net/manual/en/language.oop5.overloading.php#object.get] 裡面提到：

> __get() is utilized for reading data from inaccessible (protected or private) or non-existing properties.

我們看看 `Illuminate\Database\Eloquent\Model` 內 `__get()` 的實作

```php
/**
 * Dynamically retrieve attributes on the model.
 *
 * @param  string  $key
 * @return mixed
 */
public function __get($key)
{
    return $this->getAttribute($key);
}
```

往下我們看 `getAttribute()` 的實作，這個函式實作的位置在 `trait HasAttributes` 裡面

```php
/**
 * Get an attribute from the model.
 *
 * @param  string  $key
 * @return mixed
 */
public function getAttribute($key)
{
    if (! $key) {
        return;
    }

    // If the attribute exists in the attribute array or has a "get" mutator we will
    // get the attribute's value. Otherwise, we will proceed as if the developers
    // are asking for a relationship's value. This covers both types of values.
    if (array_key_exists($key, $this->attributes) ||
        $this->hasGetMutator($key)) {
        return $this->getAttributeValue($key);
    }

    // Here we will determine if the model base class itself contains this given key
    // since we don't want to treat any of those methods as relationships because
    // they are all intended as helper methods and none of these are relations.
    if (method_exists(self::class, $key)) {
        return;
    }

    return $this->getRelationValue($key);
}
```

這裡我們可以看到幾個段落，還有一些針對邏輯所做的詳細說明，我們來看看他們說了什麼。

第一段落

```php
if (! $key) {
    return;
}
```

這一個段落非常好懂，所以註解上也沒有多作解釋。

下一個段落

```php
// If the attribute exists in the attribute array or has a "get" mutator we will
// get the attribute's value. Otherwise, we will proceed as if the developers
// are asking for a relationship's value. This covers both types of values.
if (array_key_exists($key, $this->attributes) ||
    $this->hasGetMutator($key)) {
    return $this->getAttributeValue($key);
}
```

這裡我們暫時忽略對「"get" mutator」的註解，專注於對取值的研究。

如果 `array_key_exists($key, $this->attributes)` 為真，就會運行 `$this->getAttributeValue($key)`

我們來看看 `getAttributeValue()` 的實作是什麼

```php
$value = $this->getAttributeFromArray($key);

// If the attribute has a get mutator, we will call that then return what
// it returns as the value, which is useful for transforming values on
// retrieval from the model to a form that is more useful for usage.
if ($this->hasGetMutator($key)) {
    return $this->mutateAttribute($key, $value);
}

// If the attribute exists within the cast array, we will convert it to
// an appropriate native PHP type dependant upon the associated value
// given with the key in the pair. Dayle made this comment line up.
if ($this->hasCast($key)) {
    return $this->castAttribute($key, $value);
}

// If the attribute is listed as a date, we will convert it to a DateTime
// instance on retrieval, which makes it quite convenient to work with
// date fields without having to create a mutator for each property.
if (in_array($key, $this->getDates()) &&
    ! is_null($value)) {
    return $this->asDateTime($value);
}

return $value;
```

繼續往下追蹤 `getAttributeFromArray()` 的實作，我們會看到

```php
/**
 * Get an attribute from the $attributes array.
 *
 * @param  string  $key
 * @return mixed
 */
protected function getAttributeFromArray($key)
{
    if (isset($this->attributes[$key])) {
        return $this->attributes[$key];
    }
}
```

到這邊，我們可以知道這些值主要是從 `$attributes` 這個陣列來的。

那麼，我們的問題就變成了，一開始建立 Eloquent Model 時，是怎麼把值放到 `$attributes` 的呢？

我們接著往下看其他的魔法函式：`__call()` 和 `__callStatic()`

## `__call`

一般我們建立 Eloquent Model 物件時，必定會呼叫類似 `find()` 之類的函式來建立該物件。

很奇妙的是，這種時候即使 Model 裡面並沒有宣告 `find()` 函式，甚至有的 IDE 都會提示這個問題，不過實際上程式依舊可以執行

這時候就得看到 `__call()` 和 `__callStatic()` 這兩個魔術方法了！我們先來看看 `__callStatic()`

```php
/**
 * Handle dynamic static method calls into the method.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public static function __callStatic($method, $parameters)
{
    return (new static)->$method(...$parameters);
}
```

這段程式會在呼叫某靜態方法的時候，如果不存在該靜態方法，新建立該物件，呼叫同名的方法，並將參數原封不動的傳輸進去。

那，如果連同名的方法都沒有的話呢？這時候就會嘗試透過 `__call()` 進行處理

```php
/**
 * Handle dynamic method calls into the model.
 *
 * @param  string  $method
 * @param  array  $parameters
 * @return mixed
 */
public function __call($method, $parameters)
{
    if (in_array($method, ['increment', 'decrement'])) {
        return $this->$method(...$parameters);
    }

    return $this->forwardCallTo($this->newQuery(), $method, $parameters);
}
```

上面過濾了 `increment` 和 `decrement` 兩個函式，除了這兩個函式之外，都會進入到 `forwardCallTo()` 函式

那麼，這個函式的實作又是什麼呢？這段函式是在 `ForwardsCalls` 這個 trait 裡面，這個 trait 很小，我們來看看內容

```php
<?php

namespace Illuminate\Support\Traits;

use Error;
use BadMethodCallException;

trait ForwardsCalls
{
    /**
     * Forward a method call to the given object.
     *
     * @param  mixed  $object
     * @param  string  $method
     * @param  array  $parameters
     * @return mixed
     *
     * @throws \BadMethodCallException
     */
    protected function forwardCallTo($object, $method, $parameters)
    {
        try {
            return $object->{$method}(...$parameters);
        } catch (Error | BadMethodCallException $e) {
            $pattern = '~^Call to undefined method (?P<class>[^:]+)::(?P<method>[^\(]+)\(\)$~';

            if (! preg_match($pattern, $e->getMessage(), $matches)) {
                throw $e;
            }

            if ($matches['class'] != get_class($object) ||
                $matches['method'] != $method) {
                throw $e;
            }

            static::throwBadMethodCallException($method);
        }
    }

    /**
     * Throw a bad method call exception for the given method.
     *
     * @param  string  $method
     * @return void
     *
     * @throws \BadMethodCallException
     */
    protected static function throwBadMethodCallException($method)
    {
        throw new BadMethodCallException(sprintf(
            'Call to undefined method %s::%s()', static::class, $method
        ));
    }
}
```

這個 trait 最主要做的事情，就是 `return $object->{$method}(...$parameters);`，將收到的函式往下一個類別同名的函式丟過去。

所以 `Illuminate\Database\Eloquent\Model` 是怎麼利用這個函式的呢？我們來看看 `$this->forwardCallTo($this->newQuery(), $method, $parameters);` 裡面的 `$this->newQuery()` 實作

```php
/**
 * Get a new query builder for the model's table.
 *
 * @return \Illuminate\Database\Eloquent\Builder
 */
public function newQuery()
{
    return $this->registerGlobalScopes($this->newQueryWithoutScopes());
}
```

這兩個函式的邏輯套疊比較複雜，不過從註解我們可以看出，最後所回傳的是 `\Illuminate\Database\Eloquent\Builder` 這個物件。

看起來這個物件應該會有我們想要找到的 `find` 函式實作？我們搜尋看看確實有：

```php
/**
 * Find a model by its primary key.
 *
 * @param  mixed  $id
 * @param  array  $columns
 * @return \Illuminate\Database\Eloquent\Model|\Illuminate\Database\Eloquent\Collection|static[]|static|null
 */
public function find($id, $columns = ['*'])
{
    if (is_array($id) || $id instanceof Arrayable) {
        return $this->findMany($id, $columns);
    }

    return $this->whereKey($id)->first($columns);
}
```

前面是針對傳入 array 的方式所做的，也就是官網所說的 `$flights = App\Flight::find([1, 2, 3]);` 這種用法。

如果是像之前所麽所想探討的 `User::find(1);` 那就會進入到 `$this->whereKey($id)->first($columns)` 這串函式

我們先看看 `whereKey()` 的實作

```php
/**
 * Add a where clause on the primary key to the query.
 *
 * @param  mixed  $id
 * @return $this
 */
public function whereKey($id)
{
    if (is_array($id) || $id instanceof Arrayable) {
        $this->query->whereIn($this->model->getQualifiedKeyName(), $id);

        return $this;
    }

    return $this->where($this->model->getQualifiedKeyName(), '=', $id);
}
```

前面也是針對陣列情況的過濾，以我們的狀況，會運作的是 `$this->where($this->model->getQualifiedKeyName(), '=', $id);` 這段程式

根據名稱我們可以先推測，這邊應該是先利用某個時間點建立的 `$this->model` 然後 `getQualifiedKeyName()` 取出 id 的名稱（預設很可能就是 `id`），最後利用 `$this->where()` 來對資料庫進行取值。

我們往下看看這樣的推測是否正確，先看 `getQualifiedKeyName()`

```php
/**
 * Get the table qualified key name.
 *
 * @return string
 */
public function getQualifiedKeyName()
{
    return $this->qualifyColumn($this->getKeyName());
}
```

然後往下看 `qualifyColumn()` 

```php
/**
 * Qualify the given column name by the model's table.
 *
 * @param  string  $column
 * @return string
 */
public function qualifyColumn($column)
{
    if (Str::contains($column, '.')) {
        return $column;
    }

    return $this->getTable().'.'.$column;
}
```

和 `getKeyName()`

```
