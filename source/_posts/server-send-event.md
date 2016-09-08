---
title: HTML5 Server Send Event
date: 2016-09-08 22:49:42
tags: 
- HTML5
- ES2015

---
# 簡介
HTML 5 Server Send Event (SSE) 是一種將資料從伺服器端推送到瀏覽器端的技術。與 Ajax polling 這種多次發送 HTTP 請求的的方式相比，SSE 只需要一次的 HTTP 請求，因此能減少連線次數及傳輸的資料量、降低伺服器的負擔，並讓使用者得到更即時的資訊。
而與 Web Socket 相較，它沿用 HTTP 協定，不需要再額外建立專門處理其他協定的伺服端程式，直接沿用原本的網頁伺服器（如：Apache, Nginx）即可。所以無論是在程式開發上還是維運管理上都要簡易很多。
因此，如果只需要單向的從伺服器到瀏覽器的推送，就不需要使用 Web Socket，可以只使用 SSE。接續 [Redis Notification](/2016/07/27/redis-notification/) 中提到的 Redis 通知 PHP 程式的技術，本篇將展示如何使用 SSE 將 Redis 的更新同步到瀏覽器，並使用 ECMAScript 2015 的新功能來組織程式。

<!-- more -->

# HTML5 Server Send Event (SSE)
實作 SSE 有三個要點，首先是伺服器端的網頁程式必須要發送 SSE 專屬的內容類型，然後是傳輸的內容必須依照 SSE 的格式撰寫，最後是瀏覽器接收到訊息後的處理。
## 伺服器端 PHP 程式範例
```php
<?php
# 將回傳的內容類型指定為 text/event-stream
header('Content-Type: text/event-stream');
# 避免瀏覽器快取 SSE 訊息，以保證最新的資訊能被接收到
header('Cache-Control: no-cache'); 

/**
 * 將要回傳的訊息整理程指定的格式並輸出
 *
 * @param string $data 要送出的資料
 */
function sendData($data) {
  echo "data: $data \n\n";
  // 避免 PHP 緩衝輸出，讓資料能夠即時輸出
  ob_flush();
  flush();
}

while (1) {
    $serverTime = time();
    sendData('server time: ' . date("h:i:s", time()));
    // 避免輸出太快
    sleep(1);
}
```
## 資料格式
SSE 的每個訊息由以下欄位構成：
event - 如果該訊息是一個事件通知，則此欄位代表事件的名稱，非必要欄位。
data - 訊息的資訊，必要欄位，一個訊息可以有多個 data 欄位。
id - 指定訊息的 id，可以讓接收者辨識目前的訊息，以便了解當前接收進度，非必要欄位。
retry - 指定從該訊息後，當瀏覽器斷線時，重新連接的間格時間，單位是毫秒，非必要欄位。
其中每個欄位以一個換行符號 \n 分隔，而每個訊息最後以兩個換行符號分隔。
範例：
```
event: userconnect
data: {"username": "bobby", "time": "02:33:48"}

data: Here's a system message of some kind that will get used
data: to accomplish some task.

event: usermessage
data: {"username": "bobby", "time": "02:34:11", "text": "Hi everyone."}

```
## 瀏覽器端程式範例


