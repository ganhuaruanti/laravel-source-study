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
 
雖然繼續研究這些東西非常誘人，不過我們在乎的是針對 Eloquent Model 的實作，所以我們來看看 `Illuminate\Database\Eloquent\Model` 的程式碼。
 
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

不過，把這些 trait 和 `Illuminate\Database\Eloquent\Model` 裡面的一千多行程式碼一次看完，顯然不是一個好的研究方式。所以我們將問題縮減一下，一次只針對一個問題進行鑽研。

我們這裡先選擇一個問題：當我們使用 Eloquent Model，比方說 `User` 的物件 `$user`，存取 `$user->id` 時，到底經過哪些流程呢？

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

我們接著往下看另一個魔法函式：`__call`

## `__call`
