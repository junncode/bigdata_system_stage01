# Stage 1. Simple realtime visualization
- 사용자의 접속로그를 logstsh로 수집하여 Elasticsearch로 저장한 후, kibana를 이용하여 빠르게 시각화

### - What are customer doing?
 * 시간별 사용량 추이를 어떠한가? 그 중 mobile 접속자는 어느정도 되는가?
 * 가장 많이 접속하는 지역은 어디인가?
 * 지역별로 어떤 사용자들이 접속하는가? (customer_id만 조회가능)
 * 사용자들이 언제 어떤 음악을 듣는가? (customer_id와 track_id만 조회가능)

### - software 구성도
![stage1 architecture](https://github.com/freepsw/demo-spark-analytics/blob/master/resources/images/stage1.png)


## [Prerequisite] Install system library



### 실습에 필요한 라이브러리 설치
```
# Java 설치
> sudo yum install -y java

# console에 JAVA_HOME 설정
> export JAVA_HOME=$(alternatives --display java | grep current | sed 's/link currently points to //' | sed 's|/bin/java||')
> echo $JAVA_HOME
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/jre

# user shell에 JAVA_HOME 설정
> vi ~/.bash_profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/jre

> source ~/.bash_profile
```

### Download git project
```
> cd ~
> sudo yum install -y wget git
> git clone https://github.com/freepsw/demo-spark-analytics.git
> cd demo-spark-analytics
> mkdir sw
```

## [STEP 1] install elasticsearch and kibana

- elasticsearh [link](https://github.com/freepsw/demo-spark-analytics/blob/master/01.installed_sw/elasticsearch.md)

```
> cd ~/demo-spark-analytics/sw

# download
> wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.4.0/elasticsearch-2.4.0.tar.gz
> tar xvf elasticsearch-2.4.0.tar.gz

# install plugin
> cd elasticsearch-2.4.0
> bin/plugin install mobz/elasticsearch-head
```

- config : server와 client가 다른 ip가 있을 경우, 외부에서 접속할 수 있도록 설정을 추가해야함.
```
> cd ~/demo-spark-analytics/sw/elasticsearch-2.4.0
> vi config/elasticsearch.yml
# bind ip to connect from client  (lan이 여러개 있을 경우 외부에서 접속할 ip를 지정할 수 있음.)
# bind all ip server have "0.0.0.0"
network.host: 0.0.0.0   (":" 다음에 스페이스를 추가해야 함.)
```


- kibana [link](https://github.com/freepsw/demo-spark-analytics/blob/master/01.installed_sw/kibana.md)

```
> cd ~/demo-spark-analytics/sw
> wget https://download.elastic.co/kibana/kibana/kibana-4.6.2-linux-x86_64.tar.gz
> tar xvf kibana-4.6.2-linux-x86_64.tar.gz
```

- logstash [link](https://github.com/freepsw/demo-spark-analytics/tree/master/01.logstash)

```
> cd ~/demo-spark-analytics/sw
> wget https://download.elastic.co/logstash/logstash/logstash-2.4.0.tar.gz
> tar xvf logstash-2.4.0.tar.gz
```

 - set logstash path to $path
```
> vi ~/.bash_profile
# 마지막 line에 추가
export PATH=$PATH:~/demo-spark-analytics/sw/logstash-2.4.0/bin

> source ~/.bash_profile
```


## [STEP 2] run elasticsearch and kibana
```
> cd ~/demo-spark-analytics/sw/elasticsearch-2.4.0
> bin/elasticsearch

> cd ~/demo-spark-analytics/sw/kibana-4.6.2-linux-x86_64
> bin/kibana
```

- open with web browser
  - open elasticsearch : http://localhost:9200/_plugin/head/
  - open kibana : http://localhost:5601


## [STEP 3] run logstash (logstash_stage1.conf)
- tracks_live.csv 파일을 읽어서온 후, 필드별로 type을 지정하고 elasticsearch에 저장한다.

### configuration (collect logs and save to ES)
- "demo-spark-analytics/00.stage1/logstash_stage1.conf"
- <PATH> 부분을 각자의 경로로 수정한다.

```javascript
> vi ~/demo-spark-analytics/00.stage1/logstash_stage1.conf
input {  
  file {
    path => "/home/rts/demo-spark-analytics/00.stage1/tracks_live.csv"
    sincedb_path => "/dev/null"
    start_position => "beginning"
  }
}

filter {
  csv {
    columns => ["event_id","customer_id","track_id","datetime","ismobile","listening_zip_code"]
    separator => ","
  }

  date {
    match => [ "datetime", "YYYY-MM-dd HH:mm:ss"]
    target => "datetime"
  }

  mutate {
    convert => { "ismobile" => "integer" }
  }
}

output {
  stdout {
    codec => rubydebug{ }
  }

  elasticsearch {
    hosts => "http://localhost:9200"
    index => "ba_realtime"
    document_type => "stage1"
  }
}
```

- input
  - path : 읽어올 파일이 절대 경로를 입력한다. (새로운 내용이 파일에 입력되면 즉시 해당 내용을 읽어온다. tail -f 와 동일한 기능)
  - start_position : 처음 파일을 읽어올때 기존 내용을 전부 읽을 경우(beginning), 마지막 내용만 읽어올 경우(end, default)

- filter
  - csv
  > - csv파일의 내용을 명시한 field명으로 매핑하여 저장(elasticsearch에 저장될 field명)
  > - seperator : 구분
  - date
  > - match : 지정한 field(지정한 date format을 가진)를 timestamp로 지정. (만약 아래 target이 별도로 지정되지 않는 경우.)
  > - target : 위에서 매핑한 date를 elasticsearch의 기본 timestamp로 사용하지 않고, datetime으로 저장함. (만약 target이 없으면 datetime을 timestamp로 사용)
  > - locale : log에 저장된 날짜 type이 영어권 지역인 경우, 지역에 맞는 parsing locale을 지정해야 한다.
  - mutate
  - convert : 해당 field의 type을 지정해 준다. (integer로 해야 kibana등에서 sum/avg등의 연산이 가능함)

- output
  - hosts : elasticsearch가 실행중인 서버의 ip
  - index : elasticsearch의 index (없으면 자동으로 생성됨)
  - document_type : elasticsearch의 document_type (없으면 자동으로 생성됨)

#### run logstash
```
> cd ~/demo-spark-analytics/00.stage1
> ~/demo-spark-analytics/sw/logstash-2.4.0/bin/logstash -f logstash_stage1.conf
```

#### check result
- check using elasticsearch query
```javascript
//open with web browser
http://localhost:9200/ba_realtime/_count

// 아래와 같이 count가 증가하면 정상적으로 ES에 입력되고 있음을 확인
{
"count": 16,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  }
}
```

- check using elasticsearch plugin "head"
```
//open with web brower
http://localhost:9200/_plugin/head/
// Overview menu에 생성한 index와 document type이 존재하는지 확인
```

## [STEP 4] run data generator (data_generator.py)
### source code
- 실시간으로 데이터가 유입될 수 있도록 data generator에서 특정 file에 write (random time period)
- 이는 실시간으로 사용자들이 접속하는 log를 재연하기 위한 용도로 사용.
- tracks.csv -> data generator(date를 현재 시간으로 조정) -> tracks_live.log
- 1,000,000건이 저장된 tracks.csv에서 파일을 읽어서, 랜덤한 시간 간격으로 tracks_live.csv에 쓴다.
- data_generator.py
```python
#-*- coding: utf-8 -*-
import time
import random

r_fname = "tracks.csv"
w_fname = "tracks_live.csv"
rf = open(r_fname)
wf = open(w_fname, "a+")

try:
  num_lines = sum(1 for line in rf)
  rf.seek(0)
  lines = 0

  while (1):
    line = rf.readline()
    wf.write(line)
    wf.flush()

    # sleep for weighted time period
    stime = random.choice([1, 1, 1, 0.5, 0.5, 0.8, 0.3, 2, 0.1, 3])
    time.sleep(stime)
    lines += 1

    # exit if read all lines
    if(lines == num_lines):
      break

finally:
  rf.close()
  wf.close()
  print "close file"
```

### run data_generator.py
```
> cd ~/demo-spark-analytics/00.stage1
> python data_generator.py
```


## [STEP 5] visualize collected data using kibana
- kibana를 이용한 시각화 가이드는 아래의 link에 있는 ppt파일을 참고
- https://github.com/freepsw/demo-spark-analytics/blob/master/00.stage1/kibana_visualization_guide_stage1.pptx

####- index 추가
 * http://localhost:5601/ 접속
 * [Settings] > [Indeices]로 이동
 * index name 입력 (kibana에서 시각화할 index)
 * timestamp 선택 (데이터를 시각화할 기본 timestamp)

####- Discover
 * discovery에서 입력되는 count 확인
 * 각 record의 상세 데이터 확인
 * 조회할 기간 변경
 * filter를 이용해서 검색어를 입력하여 원하는 데이터만 조회
 * 화면에 표시될 필드 선택하여 조회
 * 위의 bar 그래프에서 원하는 영역을 drag하여 drill down 조회
 * 사용자가 시각화한 포맷을 저장한다. "save"아이콘 활용

####- visualization 메뉴를 이용하여 시각화
 * [Visualize] > [Pie Chart]
  - 가장 많이 접속하는 지역과, 해당지역에서 접속하는 사용자 비율
  - Split Sliices > Terms > listening_zip_code
  - Split Slices > Terms > customer_id
  - Options > View options > check Donut
  - Save as "BA_S1: pie_chart Customer count per region"

 * [Visualize] > [Line Chart]
  - X-Axis > Aggregation (Date Histogram) > Field(@timestamp) > Interval(Auto)
  - Y-Axis > Aggregation (Sum) > Field(ismobile)
  - Dot Size > Aggregation(Unique Count) > Field(customer_id)

 * [Visualize] > [Metrics]

 * [Visualize] > [Bar chart]

#### - Dashboard 생성
 * [Dashboard] > [+] button
 * [Visualize]메뉴에서 저장한 chart를 선택하여 자신만의 dashboard를 생성한다.
 * 생성한 dashboard를 저장한다.
 * 이후 다른 kibana web에서 dashboard를 보고싶다면 export하여 json파일로 저장한다.

#### - github에 포함된 json파일을 이용하여 dashboard 생성
 * [Setting] > [Objects] 메뉴에서 "import"를 클릭
 * ~/demo-spark-analytics/00.stage1 아래에 있는 json 파일을 선택
 * kibana_dashboard_s1.json, kibana_search_s1.json, kibana_visualization_s1.json
 * [Dashboard] > [+] button을 클릭하여 import한 시각화 객체를 불러온다. (또는 Dashboard 객체)
