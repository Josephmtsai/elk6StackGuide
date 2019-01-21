# Metricbeat

Metricbeat 是一個輕量化的資料收集器 ，他提供了很多 module，可以把系統的狀態 或是 application 的狀態直接送到 elasticsearch or logstash,並且提供許多內建的 Dashboard 給我們看

MetricBeat Dashboard List:
![image](../images/metricbeat/dashboard.png)

System Overview Dasboard
![image](../images/metricbeat/systemoverviewdashboard.png)

System Dasboard
![image](../images/metricbeat/sysdashboard.png)

Service Dashbobard
![image](../images/metricbeat/servicedashboard.png)

Redis Dashboard
![image](../images/metricbeat/redisdashboard.png)

## Metricbeat Setting

### Windows Setting

### Linux Setting

透過 我們可以知道 預設只有 system 這個模組被開啟

```
metricbeat modules list
```

![image](../images/metricbeat/moduleslist.png)

所以我們需要 enable 內建的一些模組必須要下以下的指令

```
metricbeat modules enable logstash
metricbeat modules enable elasticsearch
metricbeat modules enable redis
```

## Metricbeat Export Dashboard to ELK

簡單來講就是透過任何一個安裝好的 metricbeat agent 把裡面內建的 dashboard 匯出到 elk

```
metricbeat setup -e `-E output.elasticsearch.host=[xxxx:9200] -E setup.kibana.host'xxxx:5601' `  --dashboards
```

![image](../images/metricbeat/exportdashboard.png)

成功後應該會再 elk 看面看到
MetricBeat Dashboard List:
![image](../images/metricbeat/dashboard.png)
