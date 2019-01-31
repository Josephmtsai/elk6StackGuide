# Filebeat Config

Filebeat 主要由兩個組件組成，prospector ,harvester,並且當達到預設 log 數量的時候(spooler 設定預設是 1024 筆)一次送給 es or logstash 如圖片所示，

- prospector 負責管理每一個 filebeat 設定的檔案們，以及 harvester 他會根據 type:log,將所有找到的 log 檔案各自去啟動一個 harvester 去做收集 log 的動作

![images](https://www.elastic.co/guide/en/beats/filebeat/6.0/images/filebeat.png)

### Reference

https://www.elastic.co/guide/en/beats/filebeat/6.0/how-filebeat-works.html

https://hk.saowen.com/a/b536b806db32b4a305323ded8139161bf9d7dcc83e902ac673f6111954f049bd

http://docs.flycloud.me/docs/ELKStack/beats/file.html
