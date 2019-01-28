# Logstash Config Structure

由於在 ELK Stack 6 後 把 type 欄位拿掉了，以往 index 對應到多個 type 的情況就沒有了，因此我們想了一個比較大的架構，讓不同的 Team 可以維護各自的 index

目標有以下

- 讓不同的 Team 的 index 不會互相影響
- 容易維護以及管理
- filebeat 預設只能選擇一個 index send data 如果要區分不同的 index 需要另外去判斷
  ![image](../images/Logstash%20Config/filebeatSetting.png)

所以針對以上需求設計了這個架構

![image](../images/Logstash%20Config/logstashflow.png)

實際攤出來的架構會像這樣

```
/etc/logstash/conf.d 裡面的檔案結構
```

![image](../images/Logstash%20Config/logstashconfiglist.png)

> ** logstash 預設是讀取 conf.d 這一個資料夾底下的檔案，她可以把檔案拆出來放，但是請按照順序擺放 input => filter => output**
> 我們前面放的數字就是讓她預設排序可以依照順序去排

這裡比較特別的用共用的 input 是這個

15.filter.index.conf

```
filter {
    if [@metadata][beat] {
        mutate {
            add_field => {
                "index_prefix" => "%{[@metadata][beat]}"
            }
        }
    }
}
```

> 這是為了把輸入進來的 filebeat index 改成這個欄位，讓各 team 自行去使用這個 index
> 所以 ouput 也會根據這個 index_prefix 去判斷

30.output.conf

```
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{index_prefix}-%{+xxxx.ww}"
   }
}
```

我們可以先舉一個例子來看

這是我們定義好的 log 格式 ,我們要先整理好格式 才知道怎麼設計我們的 log pattern

![image](../images/Logstash%20Config/logformat.png)

假設我們是 Team A

```
20.filter.A-Team.default.index.conf
21.filter.A-Team.application.conf
21.filter.A-Team.iis.conf
21.filter.A-Team.jetty.conf
22.filter.A-Team.final.conf
```

先大概講解一下這幾個檔案的用途

### 20.filter.A-Team.default.index.conf

```
filter {
    if [@metadata][beat] =~ "A-Team" and [indexfield] {
			mutate {
				replace => {
					"index_prefix" => "%{[indexfield]}"
				}
			}
    }
}
```

> 預設 從 filebeat 進來都是 [@metadata][beat] 這個欄位當 index
> ex: A-Team-application , 但是就剛剛講的，針對 filebeat 出去的 index 只能有一種，
> 所以我們需要多加一個欄位判斷，假設他是有設定 indexfield,就優先使用這個 indexfield

### 21.filter.A-Team.application.conf

```
filter {
	if  [index_prefix] =~ "188-application-"   {
		if [index_prefix] =~ "error" {
			grok{
				break_on_match => false
				match => {"message" => "^%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\]%{GREEDYDATA:messages}$"}
			}
		}
		if [index_prefix] =~ "request" {
			grok{
				break_on_match => false
				match => {"message" => "^%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\]%{GREEDYDATA:messagebody}$"}
				match => {"messagebody" => "^\[(?<request>[A-Z\-a-z]+)\]\[(?<method>[A-Z\-a-z . /=?0-9\[\]:()%]+)\]\[(?<httpmethod>[A-Z\-a-z /.\-]+)\]\[%{IP:clientip}\]%{GREEDYDATA:requestMessage}$"}
				match => {"requestMessage" => "^\[(?<sessionid>[A-Z\-a-z . /=?0-9]*)\]\[(?<membercode>[A-Z\-a-z . /=?0-9]*)\]\[(?<refererurl>[A-Z\-a-z /.\-:?=&0-9_]*)\]%{GREEDYDATA:messages}$"}
				match => {"requestMessage" => "^\[(?<useragentOS>[iOS|Android|None]*)\]%{GREEDYDATA:messages}$"}

			}
			if [useragentOS] {
				mutate {
					rename => {
						"useragentOS" => "[useragent][OS]"
					}
				}
			}
		}


		if [index_prefix] =~ "performance" {
			grok{
				break_on_match => true
				match => {"message" => "^%{TIMESTAMP_ISO8601:logdate} (\[%{POSINT:procesid:int}\]) \[(?<level>(INFO|DEBUG|WARNING|ERROR))\] \[(?<logger>[A-Z\-a-z .]+)\] \[(?<PERFORMANCE>[A-Z\-a-z]+)\]\[(?<method>[A-Z\-a-z . /=?0-9:()%]+)\]\[%{NUMBER:performancetime:int}ms\]%{GREEDYDATA:messages}$"}
			}
		}

		# customize timestamp
		date {
			#timezone => "America/Anguilla"
			match => ["logdate", "ISO8601"]
			target => "@timestamp"
		}
		mutate {
				remove_field => [ "PERFORMANCE","EVENTMESSAGE","Feature" ,"request","messagebody","describe","requestMessage" ,"markfield" ]
		}
	}
}
```

> 這是處理 所有 application log 的格式的地方，我們透過 index_prefix
> **index_prefix** 重要就是我們用 index_prefix 去判斷他是要走哪一個 log
> 這些都是根據我們之前 excel 設計的格式去判斷

### 21.filter.A-Team.iis.conf

```
filter {
    if [index_prefix] =~ "A-Team-iis" {

        grok {
			break_on_match => true
            match => { "message" => "^%{TIMESTAMP_ISO8601:logdate} %{WORD:httpmethod} %{NOTSPACE:method} %{NOTSPACE:querystring} %{NUMBER:port:int} %{NOTSPACE:requestusername} %{NOTSPACE:sourceuseragent} %{NOTSPACE:refererurl} %{NOTSPACE:hostname} %{NUMBER:statuscode} %{NUMBER:substatuscode} %{NUMBER:responsetime:int}$" }
        }
        useragent {
            source => "sourceuseragent"
            target => "useragent"
        }
        #customize timestamp
        date {
            timezone => "UTC"
            match => ["logdate", "yyyy-MM-dd HH:mm:ss"]
            target => "@timestamp"
        }

		mutate {
            lowercase => [ "method" ]
            remove_field => ["sourceuseragent"]
            replace => {"index_prefix" => "A-Team-weblog"}
        }
    }
}
```

> 這是處理 iis log 的區域，我們會根據不同的 log type 讓它分開處理 ，避免複雜度 也好維護，
> **index_prefix** 重要就是我們用 index_prefix 去判斷他是要走哪一個 log

### 21.filter.A-Team.jetty.conf

```
filter {
    if [index_prefix] =~ "A-Team-jetty" {

        grok {
			break_on_match => true
            match => { "message" => '^%{IP:clientip} - - \[%{HTTPDATE:logdate}\] "%{WORD:httpmethod} %{NOTSPACE:method} %{NOTSPACE:protocol}" %{NUMBER:statuscode} %{NUMBER:responsesize:int} "%{NOTSPACE:refererurl}" "(?<sourceuseragent>[A-Z\-a-z /.\-:;,.?=()&0-9_]*)" - %{NUMBER:responsetime:int}$' }
        }
        useragent {
            source => "sourceuseragent"
            target => "useragent"
        }
        #customize timestamp
        date {
            match => ["logdate", "dd/MMM/yyyy:HH:mm:ss Z"]
            target => "@timestamp"
        }
		mutate {
            remove_field => ["sourceuseragent","clientip"]
            remove_tag => ["beats_input_codec_plain_applied", "beats_input_codec_multiline_applied"]
            replace => {"index_prefix" => "A-Team-weblog"}
        }
    }
}
```

> 這是處理 jetty log 的區域，我們會根據不同的 log type 讓它分開處理 ，避免複雜度 也好維護，
> **index_prefix** 重要就是我們用 index_prefix 去判斷他是要走哪一個 log

### 22.filter.A-Team.final.conf

```
filter {
    if [index_prefix] =~ "A-Team~" {
        if [method]{
			mutate {
				lowercase => [ "method" ]
			}
		}
        if [subcategory]{
			mutate {
				add_tag => [ "%{subcategory}"  ]
			}
		}
		if [clientip]{
			geoip {
				  source => "clientip"
				  target => "geoip"
				  add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
				  add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
			}
            mutate {
				convert => [ "[geoip][coordinates]", "float"]
		    }
		}
		mutate {
			remove_field => [ "logdate","modulefield","indexfield","subcategory","typefield","source","[host][name]","[beat][name]","[input][type]" ]
			remove_tag => ["beats_input_codec_plain_applied", "beats_input_codec_multiline_applied"]
		}

        if "_grokparsefailure" not in [tags] {
            mutate {
                remove_field => [message]
            }
        }

    }

}
```

> 這支檔案最主要是在處理 把一些像是原始資料 message or 不必要的欄未刪除 或是做轉換的功能
> 因為 message 這原始資料通常都不一定需要 而且很占空間
> \_grokparsefailure 代表是 grok parse 有問題 這時候可以保留 message 這個欄位

** 根據上面的例子，最主要是 index_prefix 用來區別各種不同的 index 要走哪一些檔案**

這樣我們就可以設計 filebeat 可以讓收集各種不同的 index

![image](../images/Logstash%20Config/filebeatsample.png)
