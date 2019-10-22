# Laravel 輔助方法

輔助方法可以說是相當有趣的設計。

除了框架本身架構一定會做的事情，比方說處理資料庫連線，畫面渲染等功能之外，Laravel 這個框架還加入了很多輔助用的方法，讓你在開發時可以使用。

## 疑似全域方法所導致的問題

Laravel 有很多的輔助方法，使用情境是類似全域變數的。比方說重導向的 `()`

```php
public function store(Request $request)
{
    $post = new Post;
    $post->subject_id = 0;
    $post->user_id = Auth::id();
    $post->save();
    return redirect(route('posts.index'));
}
```

這類函式在使用上當然會很方便，但是有時也導致了使用者沒辦法區分 PHP 原本的函式和 Laravel 提供的輔助方法，
