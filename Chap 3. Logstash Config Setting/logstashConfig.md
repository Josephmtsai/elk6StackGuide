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

這裡我們先介紹各個區塊的logstash  設定 ，最後再給大家一個完整的例子

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
* kv: key value => 你可以根據某一個格式 將類似json的格式拆分出來，不過不是很建議使用這種方法，因為這樣算是動態欄位 ，有可能會對效能造成影響 如果資料量大的話
**grok**

Grok Patterns 有兩種寫法 

第一種寫法可以參考
[Grok Patterns](https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns)
以及他的Type 可以參考
[FieldType](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)
* %{Pattern名稱:欄位名稱:型別(可以填 預設是text)}
 > %{POSINT:procesid:int}
* (?<欄位名稱>正規表示式) 
  > (?<request>[A-Z\-a-z]+)  
  > (?<useragentOS>[iOS|Android|None]*) 

假設我們有一個log得一行資料我們想要做測試

原始資料如下
````
2018-11-27 11:35:10,105 [7] [INFO] [AppDataManager] [PERFORMANCE][BankingManager][150ms]
````

我們寫了一段Grok pattern

````
%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\] \[(?<PERFORMANCE>[A-Z\-a-z]+)\]\[(?<method>[A-Z\-a-z . /=?0-9:()%]+)\]\[%{NUMBER:performancetime:int}ms\]%{GREEDYDATA:messages}
````
我們可以用下面這兩個網站來測試,或是你可以用kibana 上面的Tab 
Dev Tools => Grok Debugger 來撰寫

![image](../images/Logstash%20Config/grokdebug.png)

[grokdebug](https://grokdebug.herokuapp.com/)

[grokconstructor](http://grokconstructor.appspot.com/do/match)

測試成功後應該就會將欄位成功爬出來

那我們在看兩個簡單的例子 grok

**break_on_match => false**

代表是每一個pattern都會依照順序從上到下跑一次，第一行從
message先爬 是因為從filebeat 來預設的欄位都是message ，接著我們可以定義像是我們剩餘的欄位長什麼樣子

用途:
> 用於格式類似的pattern

ex:
像這兩筆資料 只有最後的欄位不同 我們就可以考慮把他用這個寫法處理

````
2018-11-27 10:47:52,331 [27] [INFO] [PageBaseController][REQUEST][/page-not-found][GET][127.0.0.1][hkczyqbnpz4ewyhjq5csnsxe]
2018-11-27 10:47:52,331 [27] [INFO] [PageBaseController][REQUEST][/request][GET][127.0.0.1][memberCode][hkczyqbnpz4ewyhjq5csnsxe]
````


````
grok{
    break_on_match => false			
    match => {"message" => "^%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\]%{GREEDYDATA:messagebody}$"}
    match => {"messagebody" => "^\[(?<request>[A-Z\-a-z]+)\]\[(?<method>[A-Z\-a-z . /=?0-9\[\]:()%]+)\]\[(?<httpmethod>[A-Z\-a-z /.\-]+)\]\[%{IP:clientip}\]%{GREEDYDATA:requestMessage}$"}
    
}
````

**break_on_match => true**

這就是你只要match到就會把他break掉,我會建議如果設定成true,盡量只用一個 match,來避免 假設你寫了
Match A,B,C,D這四種格式的Pattern,他可能每次最多都跑四次,會影響效能

````
grok{
    break_on_match => true			
    match => {"message" => "^%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\] \[(?<Feature>(FEATURE))\]\[(?<method>[A-Z\-a-z . /=?0-9:()]+)\]\[(?<result>[A-Z\-a-z . /=?0-9:()]+)\]\[(?<membercode>[A-Z\-a-z . /=?0-9:()]*)\]%{GREEDYDATA:messages}"}
}
````

**mutate**

用法像下面這樣

> add_tag => [ "%{subcategory}"  ] =>  %{欄位名稱的值}
> lowercase => [ "欄位名稱" ]  將值全部小寫
>  rename => { "useragentOS" => "[useragent][OS]"	} 換欄位名稱 => 這樣寫會變成{'useragent': {"OS": "iOS"}} 格式
````
mutate {
     remove_field => [ "logdate","message" ]
     remove_tag => ["beats_input_codec_plain_applied", beats_input_codec_multiline_applied"]
     add_tag => [ "%{subcategory}"  ]
     lowercase => [ "method" ]
     rename => { "useragentOS" => "[useragent][OS]"	}
}
````

**date**

主要是讓他可以做時區的轉換  以及將轉換成日期格式

>我們可以針對爬出來的欄位 取代@timestamp, @timestamp預設是log input到es的時間
>timezone是指定他這個log是哪一個時區
>match你可以到哪一個格式 ex: ISO8601

````
date {
	timezone => "UTC"
	match => ["logdate", "yyyy-MM-dd HH:mm:ss"]
	target => "@timestamp"
}
````

**geoip**

先指定欄位 以及目標的欄位，然後因為[geoip][coordinates] 這欄位需要預設是浮點數
我們要另外做處理，

````
geoip {
	source => "clientip"
	target => "geoip"
	add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
	add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
}
mutate {
	convert => [ "[geoip][coordinates]", "float"]
    convert => [ "[geoip][location]", "geo_point"]
}
````

如果我們要在kibana上面看到地圖，除了上面的設定，一定要將裡面的[geoip][location]設定成geo_point 型態才可以使用

![image](../images/Logstash%20Config/map.png)

### output 

**預設index都要是小寫**
>ww代表周 ,+xxxx 代表年度
>2018.51 代表51周
>%{tempIndex} 代表是使用之前我們爬出來的欄位


````
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{tempIndex}-%{+xxxx.ww}"
    document_type => "%{[@metadata][type]}"
   }
}

````


### logstash example code






### reference:

https://www.elastic.co/blog/removal-of-mapping-types-elasticsearch

https://www.elastic.co/blog/do-you-grok-grok

https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html

https://blog.johnwu.cc/article/elk-logstash-grok-filter.html

https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/filter/kv.html