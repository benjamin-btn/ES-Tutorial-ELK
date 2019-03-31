# ELK Stack Tutorial(Elasticsearch Logstash Kibana)

ES 기술 유닛의 마지막 회차 [ELK Stack 의 튜토리얼](https://kakao.agit.in/g/300003763/wall/315995175) 을 기술합니다.

## Product 별 버전 상세
```
Latest ELK Version. 6.5.x(2018/12/10 기준 Latest Ver.)
```
* [Elasticsearch](http://kin.is.kakaocorp.com/SE/rpms/elasticsearch-6.5.2.rpm)
* [Logstash](http://kin.is.kakaocorp.com/SE/rpms/logstash-6.5.2.rpm)
* [Kibana](http://kin.is.kakaocorp.com/SE/rpms/kibana-6.5.2-x86_64.rpm)
* [Filebeat](http://kin.is.kakaocorp.com/SE/rpms/filebeat-6.5.2-x86_64.rpm)

최신 버전은 [Elasticsearch 공식 홈페이지](https://www.elastic.co/downloads) 에서 다운로드 가능합니다.

## ELK Product 설치

이 튜토리얼에서는 tar.gz 파일을 이용하여 실습합니다.
설치는 non-root 계정으로 설치합니다.
linux 배포판과 Mac OS 에 대해 패키지 설치를 지원합니다.

```bash
# su - deploy
$ git clone https://github.daumkakao.com/benjamin-butn/ELK.git
$ cd ELK
$ ./instelk
```

## ELK Tutorial 1 - Elasticsearch, Kibana, Filebeat 세팅

### Elasticsearch
* packages/elasticsearch/config/elasticsearch.yml
  - network.host, http.cors.enabled, http.cors.allow-origin 추가설정

* packages/elasticsearch/config/jvm.options
  - Xms1g, Xmx1g 를 물리 메모리의 절반으로 수정

```bash
$ vi packages/elasticsearch/config/elasticsearch.yml
+ network.host: 0.0.0.0
+ http.cors.enabled: true
+ http.cors.allow-origin: "*"

$ vi packages/elasticsearch/config/jvm.options

- -Xms1g
+ -Xms4g
- -Xmx1g
+ -Xmx4g
```

### Kibana
* packages/kibana/config/kibana.yml
  - server.host: "0.0.0.0" -> 외부에서 접근 가능하도록 변경
  - elasticsearch.url: "http://localhost:9200" -> 주석해제
  - kibana.index: ".kibana" -> 주석해제

```bash
$ vi packages/kibana/config/kibana.yml
- #server.host: "localhost"
+ server.host: "0.0.0.0"
- #elasticsearch.url: "http://localhost:9200"
+ elasticsearch.url: "http://localhost:9200"
- #kibana.index: ".kibana"
+ kibana.index: ".kibana"
```

### Filebeat
* packages/filebeat/config/filebeat.yml
  - /home/deploy/ELK/sample/ 밑에 .log 파일을 스트리밍 하도록 추가
  - output.elasticsearch: 에 hosts: ["localhost:9200"] 추가

```bash
$ vi packages/filebeat/config/filebeat.yml
+ filebeat.inputs:
+ - type: log
+   enabled: true
+   paths:
+     - /home/deploy/ELK/sample/*.log
+ output.elasticsearch:
+   hosts: ["localhost:9200"]
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
    "number" : "6.5.2",
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
  - path.config: "/home/deploy/ELK/packages/logstash/config/\*.conf" -> 주석해제 및 conf 설정

```bash
$ vi packages/logstash/config/logstash.yml
- # config.reload.automatic: false
+ config.reload.automatic: true
- # config.reload.interval: 3s
+ config.reload.interval: 3s

$ vi packages/logstash/config/pipelines.yml
- # - pipeline.id: another_test
+ - pipeline.id: systemd-pipe
- #   path.config: "/tmp/logstash/*.config"
+ path.config: "/home/deploy/ELK/packages/logstash/config/*.conf"
```
> [logstash patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns) 를 사용하여 filter 를 정의
> [logstash plugins](https://github.com/logstash-plugins) 를 사용하여 inputs, filters, outputs 를 정의

## Smoke Test

### Logstash

```bash
$ curl localhost:9600

{"host":"ben-lectes-data02.pg1.krane.9rum.cc","version":"6.5.2","http_address":"127.0.0.1:9600","id":"f56379d1-e521-436e-85f0-890ca0368548","name":"ben-lectes-data02.pg1.krane.9rum.cc","build_date":"2018-11-30T00:45:25+00:00","build_sha":"c7710db828f75e2d3f1682a28a3579901d9b73e6","build_snapshot":false}

$ ./tuto2 1
Hello Benji
{
      "@version" => "1",
       "message" => "Hello Benji",
          "host" => "ben-lectes-data02.pg1.krane.9rum.cc",
    "@timestamp" => 2018-12-17T16:09:19.743Z
}

$ ./tuto2 2
Hello Benji
{
      "@version" => "1",
          "host" => "ben-lectes-data02.pg1.krane.9rum.cc",
          "name" => "Benji",
    "@timestamp" => 2018-12-17T16:22:56.552Z,
       "message" => "Hello Benji"
}
```


