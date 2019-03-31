# ELK Stack Tutorial(Elasticsearch Logstash Kibana)

ElasticSearch ELK 튜토리얼을 기술합니다.

본 스크립트는 외부 공인망을 기준으로 작성되었습니다.

## Product 별 버전 상세
```
Latest ELK Version. 6.7.0(2019/04/01 기준 Latest Ver.)
```
* [Elasticsearch](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.7.0.rpm)
* [Logstash](https://artifacts.elastic.co/downloads/logstash/logstash-6.7.0.rpm)
* [Kibana](https://artifacts.elastic.co/downloads/kibana/kibana-6.7.0-x86_64.rpm)
* [Filebeat](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.7.0-x86_64.rpm)

최신 버전은 [Elasticsearch 공식 홈페이지](https://www.elastic.co/downloads) 에서 다운로드 가능합니다.

## ELK Product 설치

이 튜토리얼에서는 tar.gz 파일을 이용하여 실습합니다.
설치는 non-root 계정으로 설치합니다.
linux 배포판에 대해 패키지 설치를 지원합니다.

```bash
[ec2-user@ip-xxx-xxx-xxx-xxx ~]$ sudo yum -y install git

[ec2-user@ip-xxx-xxx-xxx-xxx ~]$ git clone https://github.com/benjamin-btn/ES-Tutorial-ELK.git

[ec2-user@ip-xxx-xxx-xxx-xxx ~]$ cd ES-Tutorial-ELK

[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ ./tuto
##################### Menu ##############
 $ ./tuto [Command]
#####################%%%%%%##############
         1 : install elk packages
         2 : set elk
         3 : standard input/output, no filters
         4 : standard input/output, simple filter
         5 : beats input, no filter, standard output
         6 : beats input, grok filter COMBINEDAPACHELOG, standard output
         7 : beats input, grok filter COMBINEDAPACHELOG, es output
         8 : beats input, grok filter COMBINEDAPACHELOG, es output with systemd
         9 : start elk
#########################################
```

## ELK Tutorial 1 - Elasticsearch, Kibana, Filebeat 세팅

### Elasticsearch
* packages/elasticsearch/config/elasticsearch.yml
  - network.host, http.cors.enabled, http.cors.allow-origin 만 설정

* packages/elasticsearch/config/jvm.options
  - Xms1g, Xmx1g 를 물리 메모리의 절반으로 수정

```bash
[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/elasticsearch/config/elasticsearch.yml
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"

[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/elasticsearch/config/jvm.options
-Xms4g
-Xmx4g
```

### Kibana
* packages/kibana/config/kibana.yml
  - server.host: "0.0.0.0" -> 외부에서 접근 가능하도록 변경
  - elasticsearch.url: "http://localhost:9200" -> 주석해제
  - kibana.index: ".kibana" -> 주석해제

```bash
[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/kibana/config/kibana.yml
server.host: "0.0.0.0"
elasticsearch.url: "http://localhost:9200"
kibana.index: ".kibana"
```

### Filebeat
* packages/filebeat/config/filebeat.yml
  - /home/ec2-user/ES-Tutorial-ELK/sample/ 밑에 .log 파일을 스트리밍 하도록 추가
  - output.elasticsearch: 에 hosts: ["localhost:9200"] 추가

```bash
[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/filebeat/config/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /home/ec2-user/ES-Tutorial-ELK/sample/*.log
output.elasticsearch:
  hosts: ["localhost:9200"]
```

## Smoke Test

### Elasticsearch

```bash
$ curl localhost:9200

{
  "name" : "UZU8SQG",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "YP3Lt-53QP6H5Ay_M6UTjw",
  "version" : {
    "number" : "6.7.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "9434bed",
    "build_date" : "2018-11-29T23:58:20.891072Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}

$ curl -H 'Content-Type: application/json' -XPOST localhost:9200/firstindex/_doc -d '{ "mykey": "myvalue" }'
```

### Kibana
* Web Browser 에 http://{FQDN}:5601 실행

### Filebeat
* Process 확인 및 Elasticsearch 에 filebeat 인덱스 생성 여부 확인
  - http://es-head.is.daumkakao.io:9100/index.html?base_uri=http://{FQDN}:9200


## ELK Tutorial 2 - Logstash 활용

### Logstash
* tuto 2-1. filter 없이 standard input 을 logstash 가 받아 standard output 으로 출력
  - packages/logstash/bin/logstash -e 'input { stdin { } } output { stdout {} }'

* tuto 2-2. grok filter 활용, Hello 뒤에 나오는 이름에 name key 를 매칭
  - packages/logstash/bin/logstash -f logstash_conf/simple.conf

* tuto 2-3. input 에 beats 정의, filter 없이 rubydebug codec 을 통해 standard output 으로 출력
  - packages/logstash/bin/logstash -f logstash_conf/beats.conf

* tuto 2-4. input 에 beats 정의, grok filter 를 통해 COMBINEDAPACHELOG 로 message 필터링, rubydebug codec 을 통해 standard output 으로 출력
  - packages/logstash/bin/logstash -f logstash_conf/apache.conf

* tuto 2-5. input 에 beats 정의, grok filter 를 통해 COMBINEDAPACHELOG 로 message 필터링, rubydebug codec 을 통해 standard output 으로 출력과 동시에 ES 에 저장
  - packages/logstash/bin/logstash -f logstash_conf/es.conf

* tuto 2-6. systemd 를 통한 logstash service
* packages/logstash/config/logstash.yml
  - config.reload.automatic: true -> 3s 주석해제 및 true 설정
  - config.reload.interval: 3s -> 주석해제

* packages/logstash/config/pipelines.yml
  - pipeline.id: my-pipe -> 주석해제 및 pipeline id 수정
  - path.config: "/home/ec2-user/ES-Tutorial-ELK/packages/logstash/config/\*.conf" -> 주석해제 및 conf 설정

```bash
[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/logstash/config/logstash.yml
config.reload.automatic: true
config.reload.interval: 3s

[ec2-user@ip-xxx-xxx-xxx-xxx ES-Tutorial-ELK]$ vi packages/logstash/config/pipelines.yml
- pipeline.id: systemd-pipe
path.config: "/home/ec2-user/ES-Tutorial-ELK/packages/logstash/config/*.conf"
```
> [logstash patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns) 를 사용하여 filter 를 정의
> [logstash plugins](https://github.com/logstash-plugins) 를 사용하여 inputs, filters, outputs 를 정의

## Smoke Test

### Logstash

```bash
$ curl localhost:9600

{"host":"ip-xxx-xxx-xxx-xxx.ap-southeast-1.compute.internal","version":"6.7.0","http_address":"127.0.0.1:9600","id":"f56379d1-e521-436e-85f0-890ca0368548","name":"ip-xxx-xxx-xxx-xxx.ap-southeast-1.compute.internal","build_date":"2018-11-30T00:45:25+00:00","build_sha":"c7710db828f75e2d3f1682a28a3579901d9b73e6","build_snapshot":false}

$ ./tuto2 1
Hello Benji
{
      "@version" => "1",
       "message" => "Hello Benji",
          "host" => "ip-xxx-xxx-xxx-xxx.ap-southeast-1.compute.internal",
    "@timestamp" => 2018-12-17T16:09:19.743Z
}

$ ./tuto2 2
Hello Benji
{
      "@version" => "1",
          "host" => "ip-xxx-xxx-xxx-xxx.ap-southeast-1.compute.internal",
          "name" => "Benji",
    "@timestamp" => 2018-12-17T16:22:56.7.0Z,
       "message" => "Hello Benji"
}
```


