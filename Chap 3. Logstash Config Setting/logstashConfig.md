# Logstash 

Logstash 你可以想像是一個 數據處理的地方 他主要處理如下圖:

把各種beat 的資料，在去區分不同的欄位，最後在放到Elasticsearch內儲存

![image](../images/Logstash%20Config/structure.png)

所以對Logstash 來說 他會有

基礎 由 三塊組合而成

* input : 用來定義你輸入的來源 EX: 從檔案來的或是beat , redis 以及相對的port
* filter: 你要把資料怎麼區分成不同的欄位
* output: 定義你想要把資料輸出到哪邊

![image](../images/Logstash%20Config/basic_logstash_pipeline.png)

# Logstash Config Pattern







reference:

https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html