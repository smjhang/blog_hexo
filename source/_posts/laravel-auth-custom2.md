---
title: Laravel 客製化使用者驗證功能-修改登入驗證流程
date: 2016-12-12 13:09:59
tags: 
- Laravel
- PHP

---
# 簡介
延續 [Laravel 客製化使用者驗證功能-增加使用者資料欄位][1] 的內容，在該篇文章中提到了新增一個 status 欄位用來判斷使用者是否停權，使得我們可以阻擋被停權的使用者登入系統。
透過這樣的設計，我們可以在後續開發的功能中視需要對使用者進行停權或復原的作業。
然而 Laravel 預設的登入驗證流程只有判斷登入名稱是否存在，然後比較密碼是否正確而已，因此在本篇中將對登入驗證的流程進行修改。

[1]: /2016/12/07/laravel-auth-custum1.md
<!-- more -->
# 修改登入功能
登入功能是放在 ~/laravel_demo/app/Http/Controllers/Auth/LoginController.php 中，可以看到他用了一個 trait:
```
use AuthenticatesUsers;
```
所有登入所需的功能都是在這個 trait 內實現。
一般來說，使用 trait 就是直接將該 trait 內程式碼，即該 trait 內實現的各個私有或公開方法和成員變數原封不動的貼到目前的類別中達到程式碼共享的目的。
而這邊看起來使用 trait 的目的應該是為了將 Laravel 內建的程式碼與用戶自行開發的程式分離，使得將來 Laravel 的升級或修改不會直接影響到用戶的程式碼。
正常情況下，在使用 Laravel 的時候並不會直接去改 Laravel 內建的功能，而是透過實作介面、修改設定檔、覆寫繼承的方法等方式來進行客製化。
所以這邊透過在 LoginController 中覆寫(overwrite)要客製化的部分來達成我們需要的修改。
首先大致瀏覽 Illuminate\Foundation\Auth\AuthenticatesUsers 內的程式來找出要客製化的部分。
本次目標是對登入驗證的部分進行修改，經過分析可以發現以下方法是登入驗證的關鍵部分：
```PHP
<?php
    /**
     * Attempt to log the user into the application.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return bool
     */
    protected function attemptLogin(Request $request)
    {
        return $this->guard()->attempt(
            $this->credentials($request), $request->has('remember')
        );
    }
```
所以在 LoginController 覆寫該方法來達成需要的功能如下：
~/laravel_demo/app/Http/Controllers/Auth/LoginController.php
```PHP
<?php

namespace App\Http\Controllers\Auth;

use Illuminate\Http\Request;
use App\Models\User;
use App\Http\Controllers\Controller;
use Illuminate\Foundation\Auth\AuthenticatesUsers;

class LoginController extends Controller
{
    /*
    |--------------------------------------------------------------------------
    | Login Controller
    |--------------------------------------------------------------------------
    |
    | This controller handles authenticating users for the application and
    | redirecting them to your home screen. The controller uses a trait
    | to conveniently provide its functionality to your applications.
    |
    */

    use AuthenticatesUsers;

    /**
     * Where to redirect users after login.
     *
     * @var string
     */
    protected $redirectTo = '/home';

    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('guest', ['except' => 'logout']);
    }

    /**
     * Get the login username to be used by the controller.
     *
     * @return string
     */
    public function username()
    {
        return 'login_name';
    }

    /**
     * Attempt to log the user into the application.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return bool
     */
    protected function attemptLogin(Request $request)
    {
        $active_user = User::where($this->username(), $request[$this->username()])
            ->where('status', User::STATUS_ACTIVE)->first();
        if ($active_user !== null) {
            return $this->guard()->attempt(
                $this->credentials($request), $request->has('remember')
            );
        }
        return false;
    }
}

```
username() 方法是使用者用來登入的欄位，在[Laravel 客製化使用者驗證功能-增加使用者資料欄位][1]中已經提過。
而在這次增加的 attemptLogin 中，基本上跟 Illuminate\Foundation\Auth\AuthenticatesUsers 中的方法大同小異，不同的地方是在登入之前先將 User 的資料從資料庫中取出來檢驗，
即先判斷 User 的 status 欄位是否是 'active' 來決定要不要繼續之後的登入過程。
為了之後可能會增加不同的 status 的值，或修改原有 status 的值。這邊不寫死 status 的值，而是將 status 所有的可能的值寫進 User 的常數，
之後對 status 的值的改變只要改 User 就好了。
User 的修改如下：
~/laravel_demo/app/Models/User.php
```PHP
<?php

namespace App\Models;

use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    use Notifiable;
    const STATUS_ACTIVE = 'active';
    const STATUS_IDLE = 'idle';

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
最後再改一下登入失敗時的提示就可以了，因為現在登入失敗除了密碼錯誤、帳號不存在以外還多了帳號已被停權。
# 測試
先將資料庫內的 user 的 status 的值改成 idle:
{% asset_img idle_user.jpg 修改 status 為 idle %}
可以看到程式阻擋了使用者的登入:
{% asset_img prevent_login.jpg 阻擋登入%}
再將資料庫內的 user 的 status 的值改回 active:
{% asset_img restore_user.jpg 將 status 改回 active %}
登入成功:
{% asset_img login_success.jpg 登入成功 %}
# 小結
本篇文章介紹了如何在登入驗證的流程中增加一些判斷，
這裡不並是直接更改 Laravel 內建的驗證方式，
而是在執行驗證之前，先進行一些判斷來決定要不要進入預設的驗證過程。
對於登入流程需要額外的資訊做判斷的情況，這種方式是較為簡易的，並不需要大幅度修改程式碼。
