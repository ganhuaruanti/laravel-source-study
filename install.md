# 安裝
## 取得 Laravel 6.0.0 原始碼

要研究任何東西之前，總是要先有研究對象。

那麼，要怎麼取得程式碼呢？

在 github 發達的現在，取得框架的程式碼並不是什麼難事，我們只要 google 「laravel github」 就可以找到 Laravel 的 repo 位置了。

然後，我們找到有關 Laravel 專案的 repository，在 https://github.com/laravel/laravel 。

然後，我們在開發的資料夾下，利用 `git clone` 指令來下載程式碼，因為我們要確定安裝的是 `v6.0.0` 版本，所以要加上 `--branch v6.0.0 --single-branch` 參數。

所以，我們執行 `git clone git@github.com:laravel/laravel.git --branch v6.0.0 --single-branch` 這個指令，就可以把程式碼下載到指定的位置囉！

## 確認相依 framework 版本號

`laravel/laravel` 相依於 `laravel/framework`，因為我們會追蹤這部分的程式碼，所以也得把這部分的版本固定下來。

進到 `composer.json` 把 `"laravel/framework": "^6.0",` 改成 `"laravel/framework": "6.0.0",` 。

這樣一來，我們就可以固定 `laravel/framework` 的程式碼版本，不用擔心實際看到的程式碼和這一系列文章的不同了。
