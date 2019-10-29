# 模型

要討論 Laravel 的各種便利功能，勢必不可避免需要探討的項目之一，就是 MVC 功能之一，用來和資料庫溝通的 ORM 模型了

在 Laravel 框架內，這個 Model 通常被稱為 Eloquent Model。

下面我們就來聊聊 Laravel 內 Eloquent Model 的運作方式。

## 切入點

幸運的是， Laravel 預設就提供我們一個 `User` 的物件了，我們可以從這個物件作為起點開始研讀。

首先我們到 `app/User.php` 看這個物件

