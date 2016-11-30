---
title: Laravel 初探 - 使用者驗證
date: 2016-07-27 22:49:42
tags: 
- Redis

---
# 簡介
很多的網站應用程式都需要能辨識使用者，以記錄使用者的操作以及為不同身分的使用者提供不同的功能。為了要達成這個需求，網站應用程式必須要先驗證使用者後才能繼續進一步的操作。
由於這個驗證使用者的功能是如此的常見，因此很多框架、套件都有提供這個功能。本篇將介紹如何使用 Laravel 內建的使用者驗證功能。

<!-- more -->

# Laravel 提供的使用者驗證相關功能
商品的資料儲存在 product:#id 的 hash，
並建立一個 set 用來維護商品清單。

|data type | key          | description  |
|----------|--------------|------------  |
|set       | product_ids  | 商品總類清單   |
|hash      | product:#id  | 商品細節      |
<caption>表1: 資料結構規劃</caption>

<caption>建立 Redis 資料</capion>

    HMSET product:1 name "多力多滋組合包-綜合 54g*4包/組" price 55 stock 256
    HMSET product:2 name "【洋芋片】Lays樂事瑞士香濃起司 97g/包" price 32 stock 179
    HMSET product:3 name "【蔓莓纖果】萬歲牌蔓莓纖果150G"  price 59 stock 326
    HMSET product:4 name "【蒜香青豆】盛香珍蒜香青豆 240g/包" price 42 stock 324
    HMSET product:5 name "【義美】義美小泡芙（檸檬風味）171g/盒" price 64 stock 64
    HMSET product:6 name "【洋芋片】波的多超厚切洋芋片-蚵仔煎味" price 53 stock 182
    HMSET product:7 name "【義美】義美小蛋卷（原味）" price 49 stock 313
    HMSET product:8 name "品客碳烤BBQ口味洋芋片" price 55 stock 158
    SADD products 1
    SADD products 2
    SADD products 3
    SADD products 4
    SADD products 5
    SADD products 6
    SADD products 7
    SADD products 8

# Redis keyspace notification 
為了讓物品庫存數量能即時反應到應用程式，必須讓資料庫將資料的異動反應給應用程式，
這邊說明如何使用 Redis 的 keyspace notification 功能來實現異動事件通知。
至於更詳細的說明可以看 [官網說明文件][1]。
[1]: http://redis.io/topics/notifications  "Redis Keyspace Notifications"

keyspace notification 可讓應用程式訂閱 keyspace 更動的事件，
當 Redis 的資料有異動的時候有兩種類型的事件會被觸發：
1. 第一種讓我們可以監聽某個 key 是否被異動，當它被異動的時候，我們可以得知異動這個 key 的命令類型，稱為 keyspace 事件。
2. 第二種讓我們可以監聽是否有某個命令類型被執行，當它被執行時，透過這個事件，我們可以得知被這個命令影響到的 key，稱為 keyevent 事件。

Redis 的事件通知是透過 PUB/SUB 來進行的，因此再使用前需先了解 Redis PUB/SUB 是如何進行的。
可以參考官網的文件 [Redis PUB/SUB][2]。簡單來說就是一個客戶端訂閱了某個 channel, 另外一個客戶端可以發佈訊息到這個 channel, 
然後前面那個訂閱的客戶端就可以收到第二個客戶端發送過來的訊息了。在 Redis keyspace notification 的應用中，Redis 會負責發佈訊息到指定的 channel, 我們只要接收這些訊息就可以了。
後面會討論這些 channel。

[2]: http://redis.io/topics/pubsub "Redis PUB/SUB"

在使用 keyspace notification 前，需要先打開這個功能，編輯 redis.conf 將 notify-keyspace-events 設成：
    
    notify-keyspace-events "KEA"
    
根據官網的說明，
K - Keyspace events
E - Keyevent event
A - All commands
KEA 代表要訂閱所有命令類型的第一種和第二種事件。
更改設定檔要重啟 Redis 後才會生效。

Redis 的事件通知是透過 PUB/SUB 來進行的，因此當上述兩種事件發生的時候，Redis 會分別發佈訊息到以下兩種 channel 上：
    1. PUBLISH __keyspace@<db id>__:<key name> <command type>
    2. PUBLISH __keyevent@<db id>__:<command type> <key name> 

我們可以透過 PSUBSCRIBE 來訂閱這些事件：
    1. PSUBSCRIBE __keyspace@*__:*
    2. PSUBSCRIBE __keyevent@*__:*

假設我們要觀察 del test1 test2 test3 test4 指令對 keyspace 和 keyevent 的影響可以分別訂閱以下兩個 channel

    127.0.0.1:6379> PSUBSCRIBE __keyspace@*__:test*
    Reading messages... (press Ctrl-C to quit)
    1) "psubscribe"
    2) "__keyspace@*__:test*"
    3) (integer) 1
    
    127.0.0.1:6379> PSUBSCRIBE __keyevent@*__:del
    Reading messages... (press Ctrl-C to quit)
    1) "psubscribe"
    2) "__keyevent@*__:del"
    3) (integer) 1
    
設定要處理的 key
    
    127.0.0.1:6379> set test1 test1
    OK
    127.0.0.1:6379> set test2 test2
    OK
    127.0.0.1:6379> set test3 test3
    OK
    127.0.0.1:6379> set test4 test4
    
注意我這邊開了三個 terminal，分別執行三個 redis-cli，一個監聽對 test* 的異動，一個監聽 del 操作是否被執行，最後一個負責下各種指令。 

在建立過程中，可以看到對 keyspace 的監聽訊息如下：

    127.0.0.1:6379> PSUBSCRIBE __keyspace@*__:test*
    Reading messages... (press Ctrl-C to quit)
    1) "psubscribe"
    2) "__keyspace@*__:test*"
    3) (integer) 1
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test1"
    4) "set"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test2"
    4) "set"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test3"
    4) "set"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test4"
    4) "set"

注意 127.0.0.1:6379> PSUBSCRIBE __keyevent@*__:del 那個 redis-cli 沒有任何更新，因為沒有任何 del 操作被執行。
現在刪除 test, tests, test3, test4：

    127.0.0.1:6379> del test1 test2 test3 test4
    (integer) 4

可以看到對 keyspace (127.0.0.1:6379> PSUBSCRIBE __keyspace@*__:test*) 的監聽如下：
    
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test1"
    4) "del"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test2"
    4) "del"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test3"
    4) "del"
    1) "pmessage"
    2) "__keyspace@*__:test*"
    3) "__keyspace@0__:test4"
    4) "del"

對 keyevent (127.0.0.1:6379> PSUBSCRIBE __keyevent@*__:del)的監聽如下：
    
    1) "pmessage"
    2) "__keyevent@*__:del"
    3) "__keyevent@0__:del"
    4) "test1"
    1) "pmessage"
    2) "__keyevent@*__:del"
    3) "__keyevent@0__:del"
    4) "test2"
    1) "pmessage"
    2) "__keyevent@*__:del"
    3) "__keyevent@0__:del"
    4) "test3"
    1) "pmessage"
    2) "__keyevent@*__:del"
    3) "__keyevent@0__:del"
    4) "test4"

從上面的結果我們可以知道一個命令對多個 key 的操作，是會引發多次 keyspace 事件和 keyevent 事件的。
然後我們透過這些事件只能知道是甚麼 key 被影響、被甚麼指令影響，而不知道 key 被異動後的資料，如果想要知道 key 的最新資料則要自己再去讀取那個 key。

# 使用 Predis 在 PHP 中操作 Redis

建立一個 redis-notify 專案
    
    mkdir redis-notofy

我們這邊使用 composer 來取得 Predis Async, 使用非同步版本的 Predis 比較適合這種監聽事件的任務。
關於 composer 的使用可以參考 [composer 官網說明文件][3]。
關於 Predis Async 的介紹可以參考 [Predis Async][4]。
[3]: https://getcomposer.org/  "Composer"
[4]: https://packagist.org/packages/predis/predis
在該專案內建立 composer.json 如下：

    {
      "require":{"predis/predis-async":"dev-master"}
    }

執行  composer install  來從 Packagist 下載 Predis-Async

    compsoer install

可以開始用 Predis 了，編輯程式檔 notify.php 如下：

```php
<?php
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
        echo "新增 product_store 項目:\n";
        echo "id: $id\n";
        echo "name: {$product['name']}\n";
        echo "price: {$product['price']}\n";
        echo "stock: {$product['stock']}\n\n";
        $this->product_store[$id] = $product;
    }


    /**
     * 處理更改事件
     * @param $id
     * @param $product
     */
    function updateHandler($id, $product)
    {
        echo "更改 product_store 項目:\n";
        echo "id: $id\n";
        if ($this->product_store[$id]['name'] !== $product['name']) {
            echo "name: {$product['name']}\n";
            $this->product_store[$id]['name'] = $product['name'];
        }
        if ($this->product_store[$id]['price'] !== $product['price']) {
            echo "price: {$product['price']}\n";
            $this->product_store[$id]['price'] = $product['price'];
        }
        if ($this->product_store[$id]['stock'] !== $product['stock']) {
            echo "stock: {$product['stock']}\n";
            $this->product_store[$id]['stock'] = $product['stock'];
        }
        echo "\n";
    }

    /**
     * 處理刪除事件
     * @param $id
     * @param $product
     */
    function deleteHandler($id, $product)
    {
        echo "刪除 product_store 項目:\n";
        echo "id: $id\n\n";
        unset($this->product_store[$id]);
    }

    /**
     * 顯示 product_store
     */
    function showStore()
    {
        foreach ($this->product_store as $id => $product) {
            echo "id: $id\n";
            echo "name: {$product['name']}\n";
            echo "price: {$product['price']}\n";
            echo "stock: {$product['stock']}\n";
            echo "----------------------------------------------\n";
        }
    }

}

$client = new Predis\Async\Client('tcp://127.0.0.1:6379');
$client_sync = new Predis\Client('tcp://127.0.0.1:6379');

$local_storage = new LocalStorage();
$local_storage->init($client_sync);
$local_storage->showStore();


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
        }
    });
});
// 開始監聽 keyspace 異動事件
$client->getEventLoop()->run();
```

這個程式維護一個本地的商品清單，並且隨時接收 Redis 的最新異動來更新本地清單。
首先先建立一個 LocalStorage 物件來管理本地商品清單的增刪查改。
然後使用 Predis Async 來訂閱所有關於 product：* 的異動，
Redis 的 keyspace notification 對要監聽的每個 key 都建立一個 channel
要訂閱的 channel 樣式如下：
    
    __keyspace@*__:product:*

然後當資料異動事件發生的時候，可以取得 channel 的名稱和從 channel 傳遞過來的訊息：

    channel 的名稱： __keyspace@0__:product:1
    channel 的訊息： hset

因此我們可以從 channel 的名稱取得被異動的 key 的名稱 product:1，當然也可以只取得部份的 key 名稱 1，來作為 id。
然後可以從 channel 傳遞過來的訊息得知是甚麼操作異動了 key，這裡是 hset。
關於各種類型的操作會傳遞甚麼訊息可以查看官網的說明 [官網說明文件][1]。
詳細的程式可以在 [我的 github 上的 demos 專案][5] 下載，放在 redis-notify 資料夾內。

[5]: https://github.com/smjhang/demos

執行程式可以看到以下結果：
    
    simon@simon:~/demos/redis-notify$ php notify.php 
    id: 1
    name: 多力多滋組合包-綜合 54g*4包/組
    price: 55
    stock: 256
    ----------------------------------------------
    id: 2
    name: 【洋芋片】Lays樂事瑞士香濃起司 97g/包
    price: 32
    stock: 179
    ----------------------------------------------
    id: 3
    name: 【蔓莓纖果】萬歲牌蔓莓纖果150G
    price: 59
    stock: 326
    ----------------------------------------------
    id: 4
    name: 【蒜香青豆】盛香珍蒜香青豆 240g/包
    price: 42
    stock: 324
    ----------------------------------------------
    id: 5
    name: 【義美】義美小泡芙（檸檬風味）171g/盒
    price: 64
    stock: 64
    ----------------------------------------------
    id: 6
    name: 【洋芋片】波的多超厚切洋芋片-蚵仔煎味
    price: 53
    stock: 182
    ----------------------------------------------
    id: 7
    name: 【義美】義美小蛋卷（原味）
    price: 49
    stock: 313
    ----------------------------------------------
    id: 8
    name: 品客碳烤BBQ口味洋芋片
    price: 55
    stock: 158
    ----------------------------------------------

另外開一個 Redis client 來異動資料：
    
    simon@simon:~/demos$ redis-cli
    127.0.0.1:6379> HMSET product:9 name '旺旺 仙貝經濟包' price 469 stock 79
    OK
    127.0.0.1:6379> HMSET product:9 name '旺旺 仙貝經濟包' price 469 stock 75
    OK
    127.0.0.1:6379> DEL product:9
    (integer) 1
    127.0.0.1:6379> 

可以在 notify.php 的輸出畫面看到新的訊息：

    新增 product_store 項目:
    id: 9
    name: 旺旺 仙貝經濟包
    price: 469
    stock: 79
    
    更改 product_store 項目:
    id: 9
    stock: 75
    
    刪除 product_store 項目:
    id: 9


# 結論
這邊介紹 Redis 如何實現資料異動通知的功能。不過要注意的是 Redis 不會保存通知過的訊息，因此如果對 Redis 的連線斷線的話，斷線的應用程式是無法再取得斷線期間的異動通知。
如果非常在意事件一定要通知到的話，要自己想辦法把事件保留起來，官網上說未來 Redis 可能會將這些通知保留再另外的 SET 內，不過現階段還沒有，可能要自己實作保存事件的部份。