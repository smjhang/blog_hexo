---
title: Laravel 初探 - 使用者驗證
date: 2016-12-01 14:26:42
tags: 
- Laravel
- PHP

---
# 簡介
很多的網站應用程式都需要能辨識使用者，以記錄使用者的操作以及為不同身分的使用者提供不同的功能。為了要達成這個需求，網站應用程式必須要先驗證使用者後才能繼續進一步的操作。
由於這個驗證使用者的功能是如此的常見，因此很多框架、套件都有提供這個功能。本篇將介紹如何使用 Laravel 內建的使用者驗證功能。

<!-- more -->

# Laravel 提供的使用者驗證相關功能
Laravel 已經提供了使用者註冊、使用者登入登出、忘記密碼寄送 email、重設密碼等功能，並且已經實現好了各功能的 Controller 部分，也提供了使用者的 Model 和建立資料庫的 migrations。
要啟用 Laravel 內建的使用者驗證功能，首先需要執行
```shell
php artisan make:auth 
```
和
``` shell
php artisan migrate
```
第一個命令會建立這些功能的 views 和 routes，第二個命令會建立所需的資料庫。
然後還要改 .env 檔來設定要連線的資料庫和 email 的傳輸方式，這些功能會用到 email，因為當使用者忘記密碼會需要寄送 email 通知使用者重設密碼。
因為 migrations 只能建立資料表，不能建立資料庫，所以要先建立好資料庫並且設定好資料庫的連線。
這邊假設已建立好一個 laravel_demo 的資料庫，相關的 .env 設定如下：
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_demo
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

然後這邊使用 gmail 來給使用者傳輸密碼郵件，當然也可以使用 sendemail 搭配 postfix 來寄送 email，這邊不詳細介紹，有興趣的人可以看[鳥哥的Linux私房菜](http://linux.vbird.org/linux_server/0380mail.php)了解如何設定 postfix 以及 [Laravel 官網](https://laravel.com/docs/5.3/mail) 了解如何設定 mail server 的連線。
如果應用程式要使用 gamil 的話，需要先跟 gmail 申請 app password 給應用程式使用。要在 laravel 中使用 gmail 的應用程式密碼可以在 .env 的 MAIL_PASSWORD 的部分填入申請好的應用程式密碼。
如果要使用其他的 mail server 需要參考該 mail server 的使用方式。

```
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your_gamil_account
MAIL_PASSWORD=your_gamil_app_password
MAIL_ENCRYPTION=tls
```
查看頁面:
{% asset_img auth_index.jpg 首頁 %}
註冊頁面:
{% asset_img auth_register.jpg 註冊頁 %}
登入頁面:
{% asset_img auth_login.jpg 登入頁 %}
登入後頁面:
{% asset_img auth_logined.jpg 登入後頁面 %}
密碼重設頁面:
{% asset_img auth_reset.jpg 密碼重設頁面 %}
寄發的 email:
{% asset_img auth_email.jpg 寄發的 email %}

小結：
大部分的網頁應用程式都需要能辨認使用者，在電子商務類型的網站這幾乎是必備的功能。
Laravel 為我們提供了大部分常用的使用者驗證功能，剩下的只需要客製化出需要的功能就好。
想了解更多如何客製化使用者驗證的方式可以參考 [Laravel 官網](https://laravel.com/docs/5.3/authentication)。
