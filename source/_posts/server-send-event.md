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
因此，如果只需要單向的從伺服器到瀏覽器的推送，就不需要使用 Web Socket，可以只使用 SSE。接續 [Redis Notification][1] 中提到的 Redis 通知 PHP 程式的技術，本篇將展示如何使用 SSE 將 Redis 的更新同步到瀏覽器，並使用 ECMAScript 2015 的新功能來組織程式。
[1]: /2016/07/27/redis-notification/
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
```HTML
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
</head>
<body>
  <script>
    // 設定發送 SSE 訊息的伺服器端程式的 URL
    var source = new EventSource('server.php');
    // 設定當沒有指定 event 的 SSE 訊息發送過來時的處理函數
    source.onmessage = function(e) {
      document.body.innerHTML += e.data + '<br>';
    };
    // 設定當 event 欄位的值是 usermessage 的 SSE 訊息發送過來時的處理函數
    source.addEventListener('usermessage', function(e) {
      var data = JSON.parse(e.data);
      console.log(data.msg);
    }, false);
    
  </script>
</body>
</html>
```
詳細的 EventSource 用法可參考 [Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)

# Redis Notification 即時顯示
在 [Redis Notification][1] 中展示了將 Redis 的異動通知 PHP 程式的技術，現在要應用 SSE 進一步將異動傳達給客戶端。
首先將伺服器端的程式 notify.php 改寫為可以發送 SSE 訊息的程式，將原本的輸出轉換成 SSE 格式並停用輸出緩衝，還要注意 Web Server 如果會緩衝輸出也要先停用。
改寫後的程式 getUpdates.php 如下：
```php
<?php
/**
 * Author: simon
 */
require __DIR__.'/vendor/autoload.php';

class LocalStorage
{
    private $product_store = []; // 目前所有 products

    /**
     * 使用前更新目前所有 products
     * @param \Predis\Client $client_sync
     */
    public function init(\Predis\Client $client_sync)
    {
        $all_product_keys = $client_sync->keys('product:*');
        foreach ($all_product_keys as $product_key) {
            // 抓取 id
            preg_match('/product:(\d+)/', $product_key, $matches);
            $id = $matches[1];
            // 設定 id=>product 關聯
            $this->product_store[$id] = $client_sync->hgetall($product_key);
        }
        // 排序 product_store
        ksort($this->product_store);
    }

    /**
     * 判斷 id 是否在 product_store 內
     * @param $id
     * @return bool
     */
    public function contains($id)
    {
        if (isset($this->product_store[$id])) {
            return true;
        }
        return false;
    }
    
    /**
     * 處理新增事件
     * @param $id
     * @param $product
     */
    function insertHandler($id, $product)
    {
        echo "event: add" . PHP_EOL;
        echo 'data: {"id": "'.$id.'", "name": "'.$product['name'].'", "price": "'.$product['price'].'", "stock": "'.$product['stock'].'"}';
        echo PHP_EOL.PHP_EOL;
        ob_flush();
        flush();
        $this->product_store[$id] = $product;
    }
    
    /**
     * 處理更改事件
     * @param $id
     * @param $product
     */
    function updateHandler($id, $product)
    {
        $data = [];
        $data['id'] = $id;
        if ($this->product_store[$id]['name'] !== $product['name']) {
            $data['name'] = $product['name'];
            $this->product_store[$id]['name'] = $product['name'];
        }
        if ($this->product_store[$id]['price'] !== $product['price']) {
            $data['price'] = $product['price'];
            $this->product_store[$id]['price'] = $product['price'];
        }
        if ($this->product_store[$id]['stock'] !== $product['stock']) {
            $data['stock'] = $product['stock'];
            $this->product_store[$id]['stock'] = $product['stock'];
        }
        echo "event: update".PHP_EOL;
        echo "data: ";
        echo json_encode($data);
        echo PHP_EOL.PHP_EOL;
        ob_flush();
        flush();
    }

    /**
     * 處理刪除事件
     * @param $id
     * @param $product
     */
    function deleteHandler($id, $product)
    {
        $product = $this->product_store[$id];
        $data = [];
        $data['id'] = $id;
        $data['name'] = $product['name'];
        $data['price'] = $product['price'];
        $data['stock'] = $product['stock'];
        echo "event: delete".PHP_EOL;
        echo "data: ";
        echo json_encode($data);
        echo PHP_EOL.PHP_EOL;
        ob_flush();
        flush();
        unset($this->product_store[$id]);
    }
}
$client = new Predis\Async\Client('tcp://127.0.0.1:6379');
$client_sync = new Predis\Client('tcp://127.0.0.1:6379');

$local_storage = new LocalStorage();
$local_storage->init($client_sync);

header("Content-Type: text/event-stream"); // 設定 Server Send Event 的 Content-Type
header("Cache-Control: no-cache"); // 避免瀏覽器快取 Server Send Event 的內容
header("X-Accel-Buffering: no"); // 停用 Nginx 輸出緩衝控制


/**
 * 註冊處理 keyspace 異動的事件，並根據事件的訊息做相應的處理
 */
$client->connect(function ($client) use ($client_sync, $local_storage) {

    // 使用 psubscribe 訂閱 product:#id 這種樣式的 key 被異動的事件
    $client->pubSubLoop(['psubscribe'=>'__keyspace@*__:product:*'],
        function ($event, $pubsub) use ($client_sync, $local_storage) {
            // 當 product:#id 被異動的時候，根據事件發生的 channel 的名稱取得 key 的名稱和 product 的 id
            if (preg_match('/__keyspace@\d+__:(product:(\d+))/', $event->channel, $matches)) {
                $product_key = $matches[1];
                $product_id = $matches[2];
                // 取得被異動後，最新的 product 資料
                $product = $client_sync->hgetall($product_key);
                // 根據事件傳來的訊息得知操作 key 的類型
                $op = $event->payload;
                if ($op === 'del') {
                    $local_storage->deleteHandler($product_id, $product);
                } else if ($op === 'hset') {
                    // 當操作類型是 hset 的時候，需要從目前的 product store 去判斷是新增還是修改
                    if ($local_storage->contains($product_id)) {
                        $local_storage->updateHandler($product_id, $product);
                    } else {
                        $local_storage->insertHandler($product_id,$product);
                    }
                }
                if ($op === 'quit') {
                    $pubsub->quit();
                }
            }
        });
});
// 開始監聽 keyspace 異動事件
$client->getEventLoop()->run();

```
為了要讓後來連線的程式能 follow 目前最新的結果，需要有一隻程式 getProductList.php 可以顯示目前最新的產品清單如下：

```php
<?php
/**
 * Author: simon
 */
require __DIR__.'/vendor/autoload.php';
$client_sync = new Predis\Client('tcp://127.0.0.1:6379');
$all_product_keys = $client_sync->keys('product:*');
$product_store = [];
foreach ($all_product_keys as $product_key) {
    // 抓取 id
    preg_match('/product:(\d+)/', $product_key, $matches);
    $id = $matches[1];
    // 設定 id=>product 關聯
    $product_store[$id] = $client_sync->hgetall($product_key);
}
$output = [];
foreach ($product_store as $id => $product) {
    $output[] = ["id" => $id, "name" => $product['name'], "price" => $product['price'], "stock" => $product['stock']];
}
echo json_encode($output);
```
現在可以開始寫瀏覽器端的程式了，需要的程式檔案列表如下：

|file name                            | description                                     |
|-------------------------------------|-------------------------------------------------|
|main.js                              | 負責整合各部分程式碼並操作 DOM                       |
|ProductList.js                       | 負責管理商品清單的資料                              |
|ProductListException.js              | 代表 ProductList 發生異常的物件                |
|ServerSendEventHandler.js            | 負責處理產品清單異動的事件                            |
|ServerSendEventHandlerException.js   | 代表 ServerSendEventHandler 發生異常的物件      |

main.js:
```JavaScript
/**
 * Created by simon on 9/4/16.
 */
import ServerSendEventHandler from './ServerSendEventHandler.js';
import ProductList from './ProductList.js';

window.onload = function () {
    let product_list = new ProductList();
    let sse_handler = new ServerSendEventHandler();
    // 新連線的頁面先更新最新產品清單
    product_list.loadList('./getProductList.php');
    // 將產品清單的資訊顯示在頁面上
    let tbody = document.getElementById("product_list");
    makeTableFromProductList(tbody, product_list.getProductList());
    // 設定當產品異動時要進行的各種處理，這邊主要是更新產品清單資料 product_list 和修改頁面上顯示的資訊
    sse_handler.addEventHandler(ServerSendEventHandler.ADD, product_list.addItem.bind(product_list));
    sse_handler.addEventHandler(ServerSendEventHandler.ADD, insertRow.bind(undefined, tbody));
    sse_handler.addEventHandler(ServerSendEventHandler.UPDATE, product_list.updateItem.bind(product_list));
    sse_handler.addEventHandler(ServerSendEventHandler.UPDATE, updateRow);
    sse_handler.addEventHandler(ServerSendEventHandler.DELETE, product_list.deleteItemById.bind(product_list));
    sse_handler.addEventHandler(ServerSendEventHandler.DELETE, deleteRow.bind(undefined, tbody));
    // 開始監聽產品異動事件並進行相應處理
    sse_handler.listen("./getUpdates.php");
};

/**
 * 顯示產品清單
 * @param tbody
 * @param product_list
 */
function makeTableFromProductList (tbody, product_list) {
    let docfrag = document.createDocumentFragment();
    product_list.forEach(product => insertRow(docfrag, product));
    tbody.appendChild(docfrag);
}

/**
 * 插入產品
 * @param tbody
 * @param product
 */
function insertRow (tbody, product) {
    let tr = document.createElement("tr");
    let {id, name, price, stock} = product;
    tr.setAttribute('id', id);
    let td_name = document.createElement('td');
    let td_price = document.createElement('td');
    let td_stock = document.createElement('td');
    td_name.textContent = name;
    td_name.setAttribute('name','name');
    td_price.textContent = price;
    td_price.setAttribute('name','price');
    td_stock.textContent = stock;
    td_stock.setAttribute('name','stock');
    tr.appendChild(td_name);
    tr.appendChild(td_price);
    tr.appendChild(td_stock);
    tbody.appendChild(tr);
}

/**
 * 更新產品
 * @param product
 */
function updateRow (product) {
    let id = product.id;
    let tr = document.getElementById(id);
    if (tr === null) {
        return;
    }
    for (let key in product) {
        if (Object.prototype.hasOwnProperty.call(product, key)) {
            let updating_td = tr.querySelector(`td[name="${key}"]`);
            if (updating_td !== null) {
                updating_td.textContent = product[key];
            }
        }
    }
}

/**
 * 刪除產品
 * @param tbody
 * @param product
 */
function deleteRow (tbody, product) {
    let id = product.id;
    let tr = document.getElementById(id);
    if (tr !== null) {
        tbody.removeChild(tr);
    }
}
```
ProductList.js:
```JavaScript
/**
 * Created by simon on 9/4/16.
 */
import ProductListException from './ProductListException.js';
/**
 * 維護本地產品列表的物件
 */
export default class ProductList {
    constructor () {
        this.product_list = [];
    }

    /**
     * 從 Server 取得目前所有產品列表
     * @param url Server Url
     */
    loadList (url) {
        try {
            let xhr = new XMLHttpRequest();
            // 因為這個方法必須確實取得資料才能回傳，所以使用同步版本的 xhr
            xhr.open("GET", url, false);
            xhr.send();
            let result = JSON.parse(xhr.responseText);
            this.product_list = result;
            return result;
        } catch (e) {
            throw new ProductListException(`Error #1: Cannot load product list from remote server: ${url}.`)
        }
    }

    /**
     * 取得產品列表
     * @returns {Array|*}
     */
    getProductList () {
        return this.product_list;
    }

    /**
     * 增加產品
     * @param item
     */
    addItem (item) {
         this.product_list.push(item);
    }

    /**
     * 刪除產品
     * @param deleting_item
     */
    deleteItemById (deleting_item) {
        let index = this.product_list.findIndex(item => deleting_item.id === item.id);
        if (index === -1) {
            throw new ProductListException(`Error #2: Delete product error, no such product id: ${deleting_item.id}.`);
        }
        this.product_list.splice(index,1);
    }

    /**
     * 更新產品
     * @param updating_item
     */
    updateItem (updating_item) {
        let index = this.product_list.findIndex(item => item.id === updating_item.id);
        if (index === -1) {
            throw new ProductListException(`Error #3: Update product error, no such product id: ${updating_item.id}.`);
        }
        // 更新產品部分資訊
        for (let prop in updating_item) {
            if (Object.prototype.hasOwnProperty.call(updating_item, prop)) {
                this.product_list[index][prop] = updating_item[prop];
            }
        }
    }
}

```
ProductListException.js:
```JavaScript
/**
 * Created by simon on 9/4/16.
 */

/**
 * ProductList 專屬 Exception
 */
export default class ProductListException  extends Error {
    constructor(message) {
        super(message);
        this.name = this.constructor.name;
        this.message = message;
    }
}
```
ServerSendEventHandler.js:
```JavaScript
/**
 * Created by simon on 9/4/16.
 */

import ServerSendEventHandlerException from './ServerSendEventHandlerException.js';

const ADD = 0; // 產品新增事件代碼
const UPDATE = 1; // 產品修改事件代碼
const DELETE = 2; // 產品刪除事件代碼
/**
 * Server Send Event 處理物件
 */
export default class ServerSendEventHandler {
    constructor () {
        this.update_events = [];
        this.delete_events = [];
        this.add_events = [];
    }

    /**
     * 取得產品新增事件代碼
     * @returns {number}
     * @constructor
     */
    static get ADD () {
        return ADD;
    }

    /**
     * 取得產品修改事件代碼
     * @returns {number}
     * @constructor
     */
    static get UPDATE () {
        return UPDATE;
    }

    /**
     * 取得產品刪除事件代碼
     * @returns {number}
     * @constructor
     */
    static get DELETE () {
        return DELETE;
    }

    /**
     * 開始監聽 Server Send Event 並進行處理
     * @param url
     */
    listen (url) {
        if ([this.update_events.length, this.delete_events.length, this.add_events.length].includes(0)) {
            throw new ServerSendEventHandlerException(`Error #2: Event handlers must be specified before listing event.`);
        }
        let event_source = new EventSource(url);
        event_source.addEventListener('add',  (e) => {
            let response_data = JSON.parse(e.data);
            this.add_events.forEach( event_handler => event_handler(response_data));
        }, false);
        event_source.addEventListener('update',  (e) => {
            let response_data = JSON.parse(e.data);
            this.update_events.forEach(event_handler => event_handler(response_data));
        }, false);
        event_source.addEventListener('delete', (e) => {
            let response_data = JSON.parse(e.data);
            this.delete_events.forEach(event_handler => event_handler(response_data));
        }, false);
        event_source.onerror = (e) => {
            throw new ServerSendEventHandlerException(`Error #4: Server send event failed: ${e}.`);
        };
    }

    /**
     * 附加事件處理函數
     * @param event_type
     * @param event_handler
     */
    addEventHandler (event_type, event_handler) {
        switch (event_type) {
            case ServerSendEventHandler.ADD:
                this.add_events.push(event_handler);
                break;
            case ServerSendEventHandler.UPDATE:
                this.update_events.push(event_handler);
                break;
            case ServerSendEventHandler.DELETE:
                this.delete_events.push(event_handler);
                break;
            default:
                throw new ServerSendEventHandlerException(`Error #1: Invalid event types: ${event_type}.`);
                break;
        }
    }

    /**
     * 移除事件處理函數
     * @param event_type
     * @param event_handler
     */
    removeEventHandler (event_type, event_handler) {
        let index;
        switch (event_type) {
            case ServerSendEventHandler.ADD:
                index = this.add_events.findIndex(item => Object.is(item, event_handler));
                if (index === -1) {
                    throw new ServerSendEventHandlerException(`Error #3: No such event handler.`);
                }
                this.add_events.splice(index, 1);
                break;
            case ServerSendEventHandler.UPDATE:
                index = this.update_events.findIndex(item => Object.is(item, event_handler));
                if (index === -1) {
                    throw new ServerSendEventHandlerException(`Error #3: No such event handler.`);
                }
                this.update_events.splice(index, 1);
                break;
            case ServerSendEventHandler.DELETE:
                index = this.delete_events.findIndex(item => Object.is(item, event_handler));
                if (index === -1) {
                    throw new ServerSendEventHandlerException(`Error #3: No such event handler.`);
                }
                this.delete_events.splice(index, 1);
                break;
            default:
                throw new ServerSendEventHandlerException(`Error #1: Invalid event types: ${event_type}.`);
                break;
        }
    }
}
```
ServerSendEventHandlerException.js:
```JavaScript
/**
 * Created by simon on 9/4/16.
 */

/**
 * ServerSendEventHandler 專屬 Exception
 */
export default class ServerSendEventHandlerException  extends Error {
    constructor(message) {
        super(message);
        this.name = this.constructor.name;
        this.message = message;
    }
}
```
為了讓 ECMAScript 2015 的程式碼能夠順利在瀏覽器上執行，我使用了 [Babel](https://babeljs.io/) 和 [webpack](https://webpack.github.io/) 將這些 js 轉成 ECMAScript 5 並整合成一個 bundle.js 檔案給 main.html 載入。

最後的主要進入頁面如下 main.html:
```HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="bundle.js"></script>
</head>
<body>
<table>
    <thead>
    <tr><th>產品名稱</th><th>價格</th><th>庫存</th></tr>
    </thead>
    <tbody id="product_list">
    </tbody>
</table>
</body>
</html>
```
整個專案的程式可以在 [GitHub](https://github.com/smjhang/demos/tree/master/server-send-event)下載。

## 開始測試
首先在 Redis 插入起始資料:
```
HMSET product:1 name '芭樂'   price 5 stock 99
HMSET product:2 name '西瓜'   price 5 stock 99
HMSET product:3 name '蓮霧'   price 5 stock 99
HMSET product:4 name '鳳梨'   price 5 stock 99
HMSET product:5 name '蘋果'   price 5 stock 99
HMSET product:6 name '奇異果' price 5 stock 99
HMSET product:7 name '香蕉'   price 5 stock 99
HMSET product:8 name '荔枝'   price 5 stock 99
```

進入頁面:
{% asset_img main.jpg 第一次進入 %}

在 Redis 新增資料:
```
HMSET product:9 name '小玉西瓜' price 15 stock 79
```

查看頁面:
{% asset_img insert.jpg 新增資料 %}

在 Redis 修改資料:
```
HMSET product:9 name '小玉西瓜' price 15 stock 75
```

查看頁面:
{% asset_img update.jpg 修改資料 %}

在 Redis 刪除資料:
```
DEL product:9
```

查看頁面:
{% asset_img delete.jpg 刪除資料 %}

# 小結
沿續了 [Redis Notification][1] 一文中關於 Redis 推送資料的範例，本篇進一步使用 SSE 將此資料異動通知給瀏覽器，並且嘗試使用 ES 20015 來進行開發。從測試結果可以看到 SSE 確實能即時通知瀏覽器最新的資訊。
雖然 SSE 可以沿用網頁伺服器而不需要自行開發伺服器程式，但是對於 SSE 的運作原理還是要有足夠深入了解，才能了解應該如何設定網頁伺服器，並且注意到許多安全性以及系統架構上的問題。
在投入正式產品環境之前，還是要再經過謹慎的評估和測試才能運用。