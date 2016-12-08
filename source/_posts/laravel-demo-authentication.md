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
Laravel 已經提供了使用者註冊、使用者登入登出、忘記密碼寄送 email、重設密碼等功能，並且已經實現好了各功能的 Controller 部分。
要啟用 Laravel 內建的使用者驗證功能，首先需要執行
```shell
php artisan make:auth 
```
和
``` shell
php artisan migrate
```
第一個命令會建立這些功能的 views 和 routes，第二個命令會建立所需的資料庫。
因為驗證使用者的功能必須要有一個存放使用者相關資訊的資料庫來做為驗證之用，因此 Laravel 也提供了建立這些資料庫的語法。
這些語法以 migrations 的形式存在 database/migrations 裡，有了 migrations 我們甚至可以不用連進 MySQL 就能對各資料表的結構進行管理以及版本控制。
但是在使用 migrations 之前，要先建立好資料庫，因為 migrations 只有建立資料表的部分，而資料表必須要在一個資料庫內建立。
因此我們必須先手動建好資料庫，這邊的假設資料庫的名字是 laravel_demo，連進 MySQL 執行以下 SQL:
```
CREATE DATABASE laravel_demo;
```
然後修改環境設定檔 .env，設定關於資料庫連線的部分。
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_demo
DB_USERNAME=your_username
DB_PASSWORD=your_password
```
因為當使用者忘記密碼會需要寄送 email 通知使用者重設密碼，因此這些功能會用到 email。
因此還需要在 .env 檔內設定關於連線 mail server 的部分。
mail server 的部分我使用我的帳號登入 gmail 來使用 gmail 收發電子郵件。
在自己開發的應用程式中如果需要使用 google 的 gmail 服務需要先跟 google 申請 gmail 的應用程式密碼，作為應用程式登入之用。
當然也可以使用 postfix 來寄送 email，這邊不詳細介紹，有興趣的人可以看[鳥哥的Linux私房菜](http://linux.vbird.org/linux_server/0380mail.php)了解如何設定 postfix。
如果要使用其他的 mail server 需要參考該 mail server 的使用方式。

如果應用程式要使用 google 帳戶來使用 gmail 服務的話，需要先跟 google 申請應用程式密碼給應用程式使用。要在 Laravel 中使用 gmail 的應用程式密碼可以在 .env 的 MAIL_PASSWORD 的部分填入申請好的應用程式密碼。
```
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your_gamil_account
MAIL_PASSWORD=your_gamil_app_password
MAIL_ENCRYPTION=tls
```
修改完 .env 檔案，需要重啟 server。

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

# 小結：
大部分的網頁應用程式都需要能辨認使用者，在電子商務類型的網站這幾乎是必備的功能。
Laravel 為我們提供了大部分常用的使用者驗證功能，剩下的只需要客製化出需要的功能就好。
想了解更多如何客製化使用者驗證的方式可以參考 [Laravel 官網](https://laravel.com/docs/5.3/authentication)。
