---
title: Laravel 開發實踐 - 客製化使用者驗證功能(1)
date: 2016-12-07 06:08:59
tags: 
- Laravel
- PHP

---
# 簡介
在 [Laravel 初探 - 使用者驗證][1] 中提到了 Laravel 預設的使用者驗證相關功能的使用方式。
本篇將要來介紹如何客製化這些功能，例如將原本以 email 來登入的方式改成用一個登入帳號來登入、提供使用者名稱來做為公開顯示用。
即登入用的帳號和公開顯示的使用者名稱分開，如此可避免其他使用者透過公開的 email 或使用者名稱來嘗試登入他人的帳號。
此外還新增一個狀態欄位來判斷一個帳號是否被停權，可用於登入時的檢測。
[1]: /2016/12/01/laravel-demo-authentication/
<!-- more -->

# Laravel 內建使用者驗證功能概覽
Laravel 提供了使用者登入、重設密碼、註冊使用者、忘記密碼寄送重設密碼 email 等功能。分別放在以下 controller:
App\Http\Controllers\Auth\LoginController
App\Http\Controllers\Auth\ResetPasswordController
App\Http\Controllers\Auth\RegisterController
App\Http\Controllers\Auth\ForgotPasswordController
從這些 controller 進入可以找到實作這些功能的相關邏輯及介面。

本次的目標是新增使用者相關資料 login_name 做為登入用、user_name 做為顯示用、status 做為帳號停權時的判斷欄位，
並將原本以 email 登入的行為改成用 login_name 登入。
因此需要修改資料庫欄位、修改登入功能和註冊功能，並調整相關功能的介面。

# 利用 migrations 修改資料表欄位定義
使用 migrations 的好處是可以對資料庫的修改進行紀錄，並根據這些紀錄進行資料庫的版本控制。
這個功能只建議在開發環境使用，因為對資料庫欄位的修改通常會需要修改既有的資料。
在開發環境的測試資料都是可以自行填充的，還可以搭配 [Database: Seeding](https://laravel.com/docs/5.3/seeding) 來建立填充資料的腳本。
Migrations 的價值是團隊開發時同步各開發成員的開發環境，使得大家都用相同的資料庫欄位定義和測試資料來開發、測試。
由於 Laravel 已經提供了建立 users 資料表所需的 migrations，而在 [Laravel 初探 - 使用者驗證][1] 中也執行該 migrations 將 users 建立起來了。
因此本篇將建立用來修改 users 資料表的 migration，執行以下命令來建立 migration:
```
php artisan make:migration change_users_table
```
如此便可以看到在 database/migrations 多了一個檔案 2016_12_07_135158_change_users_table.php，前面的日期是執行這個命令時自動加的，可以幫助 Laravel 判斷 migrations 的先後順序。
修改新建的檔案 ~/laravel_demo/database/migrations/2016_12_07_135158_change_users_table.php:
```PHP
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class ChangeUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
		    // 修改 users 資料表的 name 欄位，將其改名為 login_name
            $table->renameColumn('name', 'login_name');
			// 將 login_name 增加 unique 索引，確保 login_name 的值都是唯一的
            $table->unique('login_name');
			// 移除 email 的 unique 索引
            $table->dropUnique('users_email_unique');
			// 在 name 後面新增 user_name 欄位，這邊的欄位順序似乎是參考未執行 migrate 時的狀態，
			// 此時 name 尚未改成 login_name，因此如果不用 name 而改用 login_name 的話會發生找不到欄位的錯誤
            $table->string('user_name')->after('name');
			// 在 email 欄位後面新增 status 欄位，預設值為 'active'
            $table->string('status')->after('email')->default('active');
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
		    
            $table->dropColumn('status');
            $table->dropColumn('user_name');
			$table->unique('email');
            $table->dropUnique('users_login_name_unique');
            $table->renameColumn('login_name', 'name');
        });
    }
}
```
這個類別提供了兩個方法，up 和 down 分別是在執行資料表變更和還原變更時要執行的方法。
在本次的變更中，我需要將原本做為使用者名稱顯示的 name 欄位改成 login_name 做為登入用的欄位，即該欄位的值必須是唯一的，如此才能用來辨識登入的使用者。 
然後取消原本 email 欄位用作登入的用途，即將 email 欄位的唯一性移除。
最後再新增一個 status 欄位做為使用者帳號的狀態。
在 down 的部分我習慣將 up 執行過的動作，依相反的順序執行相反的動作。
然後 unique 索引在新增和移除的過程中會檢查資料表的資料是否符合唯一性限制，因此我習慣將資料清空再執行 migrate。
如需要更多關於 migration 的指令可以參考 [Database: Migrations](https://laravel.com/docs/5.3/migrations)。
migrations 建好後可以執行以下命令來修改資料庫：
```
php artisan migrate
```
# 修改 Users 資料模型
Laravel 預設是使用 Eloquent ORM 來存取資料庫，因此需要有一些模型物件(Model)來表示資料庫的資料，一個模型對應一個資料表，而模型的實例(instance)對應資料表一行(one row)的資料。
Laravel 預設已有提供 App/User 模型了，不過我習慣開一個 ~/laravel_demo/app/Models 目錄來放模型，然後把 App/User 的檔案搬進去 app/Models 目錄內。
而因為檔案的位置改了，為了遵守 Laravel 自動載入檔案的規則需要將 App/User 的命名空間改成 App/Models。
修改 ~/laravel/app/Models/User.php 如下:
```PHP
<?php

namespace App\Models;

use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'login_name', 'user_name', 'email', 'password',
    ];

    /**
     * The attributes that should be hidden for arrays.
     *
     * @var array
     */
    protected $hidden = [
        'password', 'remember_token',
    ];
}
```
由於變更了命名空間，所以需要把整個專案用到 App/User 的地方都重構成 App/Models/User。
$fillable 屬性指定在整批設定(mass assignment)的時候哪些欄位可以直接存入資料庫，這是一個為了資料庫的安全性而做的設定，
因為有些應用會直接把外部資料整批設定後存入資料庫，而有些不希望被使用者自行修改的欄位資料(例如：權限、身分)就這樣被修改了。
例如：
```PHP
<?php
$user = new User(Input::all());
$user->save();
```
只有在 $fillable 內的欄位才可以透過整批設定的方式存進資料庫。
由於新增了 'login_name', 'user_name' 所以要把這兩個欄位加進去，否則將資料寫進資料庫了。
這邊沒有 status 的理由是因為 status 是控制帳號是否停權的欄位，不允許使用者自行修改，因此不讓它可以在整批設定的情況下被寫入資料庫。
$hidden 屬性指定在模型做序列化(Serialization)給外界讀取時需要隱藏的資料欄位。由於 password 是機密資訊，不允許外洩，所以這邊就有 password 了，
而 remember_token 是自動登入用的資訊同樣也不允許外洩。
