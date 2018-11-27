# Logstash 

Logstash 你可以想像是一個 數據處理的地方 他主要處理如下圖:

把各種beat 的資料，在去區分不同的欄位，最後在放到Elasticsearch內儲存

![image](../images/Logstash%20Config/structure.png)

我們可以把原始的字串 變成區分成不同欄位在elasticsearch 並且在 kibana上面看到

![image](../images/Logstash%20Config/data.png)

所以對Logstash 來說 他會有

基礎 由 三塊組合而成

* input : 用來定義你輸入的來源 EX: 從檔案來的或是beat , redis 以及相對的port
* filter: 你要把資料怎麼區分成不同的欄位
* output: 定義你想要把資料輸出到哪邊

![image](../images/Logstash%20Config/basic_logstash_pipeline.png)


在教學pattern之前 我們最好先了解一下elasticsearch 儲存的資料 名稱

* index => DB
* document => 每一筆的資料(檔案 儲存成json格式)

## ELK 5 (or before) VS ELK 6  Different

為什麼會特別講這篇呢? 是因為 過去在elk 5版的時候

你會發現一個index 底下 會有許多的type
=>index => db , type => table 

![image](../images/Logstash%20Config/document.png)

但是就會造成很多問題，舉一個例子來說在同一個index 底下,有books這個type 上面有一個 欄位是id 他是數字 ,但是magzines的type上面的id 卻是字串，這樣會造成誤會
並且也會影響ES的壓縮效率，所以在ELK6 他就限制一個INDEX只有一個TYPE 
> https://www.elastic.co/blog/removal-of-mapping-types-elasticsearch

所以之後會長像這樣，會建議先想好要怎麼去定義同一個index底下要放什麼樣的資料
![image](../images/Logstash%20Config/elk6type.png)


# Logstash Config Pattern


### input

這裡可以定義我們常用的輸入port
ex: file,syslog,redis,beat .... 



* 以下是filebeat or metricbeat ... 這些 都會走我們設定的5044 port
````
input {

    beats {
          port => 5044
    }
}
````
* 我們也可以定義讀取本機端的log 檔案並且設定開始的地點

````
file {
        path => ["/var/log/*.log", "/var/log/message"]
        start_position => "beginning"
}
````
如果有需要可以自己尋找適合的plugin以及設定
https://www.elastic.co/guide/en/logstash/current/input-plugins.html

** 這裡可以設定將收到的多行的 event 合併，但是其實不建議使用，可以參考Performance or Case Study , 因為logstash 通常台數都不會多，通常這種我們會在input前會先整理好EX: filebeat or beats 處理掉，避免影響效能

** 下面範例是ISO 8601 判斷換行的格式

````
input {
    beats {
        port => 5044
        codec => multiline {
        pattern => "^[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}[\.,][0-9]{3,7} "
        negate => true
            what => "previous"
        }
    }
}
````

### filter

這是我們最重要的設定的地方

這裡列出一些常用的filter

* grok : 將收到的資料 根據一些內建的pattern 將欄位分離
* mutate: 可以將欄位rename , 改變型態，修改值,刪除資料
* drop: 刪除整筆events
* clone: 複製一份相同的資料
* geoip: 將欄位的資訊轉換成地理欄位的資料
* date: 將我們從grok拿到的日期欄位，變成日期欄位格式以及設定他相對應的時區







reference:

https://www.elastic.co/blog/removal-of-mapping-types-elasticsearch

https://www.elastic.co/blog/do-you-grok-grok

https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html