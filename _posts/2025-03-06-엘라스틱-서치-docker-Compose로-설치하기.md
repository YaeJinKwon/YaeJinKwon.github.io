---
title: 엘라스틱 서치 docker Compose로 설치하기
date: 2025-03-06 17:59:00 +0900
categories: [Tech, Elastic Search]
tags: [doker, elasticsearch, java, spring]     # TAG names should always be lowercase
---
## 엘라스틱 서치 여정기
<hr/>
엘라스틱,,, 참 어려웠다. 
프로젝트 진행 했을때 먼저 로컬에서는 엘라스틱 서치를 도커를 사용해서 설치하였다.  
그 후 서버에 올리려고 했으나, 서버에서 SSL 인증 하는 것이 너무 어려웠다.
그래서 ElasticCloud를 사용해서 검색 기능을 제공하였으나,,, 비용 이슈 때문에 현재는 다 내려간 상태이다.  
어쨌든! 더 깊은 엘라스틱 공부를 위해 다시 로컬에다가 설치 것이다!.  
그 과정을 잊지 않도록....블로그에다가 끄적여본다.

<br/>
정말 감사하게도 엘라스틱 공식 홈페이지에서 잘 정리가 되어있다.
*[참고](https://www.elastic.co/kr/blog/getting-started-with-the-elastic-stack-and-docker-compose)*  
그럼에도 정리를 해보겠다!.


### 파일 구조
<hr/>
먼저 파일 구조이다.

├── .env

├── docker-compose.yml

├── filebeat.yml

├── logstash.conf

└── metricbeat.yml
  
난 여기서 filebeat, metricbeat는 설치 하지 않았다.  
추후에 서버가 커지고 사용자가 많아진다면 필요하겠지만, 지금 당장 로그를 수집할 필요가 없어서  
ELK(Elastic, Logstash, Kibana)만 설치를 할 것이다.

### .env
<hr/>
- .env 파일을 생성한다.  
- doker-compose에 전달할 변수들을 정의하는 곳이다.  
- changeme는 각자의 비밀번호로 변경 해야한다.  
- 포트도 필요시 조정한다.


```yml
# Project namespace (defaults to the current folder name if not set)
#COMPOSE_PROJECT_NAME=myproject


# Password for the 'elastic' user (at least 6 characters)
ELASTIC_PASSWORD=changeme


# Password for the 'kibana_system' user (at least 6 characters)
KIBANA_PASSWORD=changeme


# Version of Elastic products
STACK_VERSION=8.7.1


# Set the cluster name
CLUSTER_NAME=docker-cluster


# Set to 'basic' or 'trial' to automatically start the 30-day trial
LICENSE=basic
#LICENSE=trial


# Port to expose Elasticsearch HTTP API to the host
ES_PORT=9200


# Port to expose Kibana to the host
KIBANA_PORT=5601


# Increase or decrease based on the available host memory (in bytes)
ES_MEM_LIMIT=1073741824
KB_MEM_LIMIT=1073741824
LS_MEM_LIMIT=1073741824


# SAMPLE Predefined Key only to be used in POC environments
ENCRYPTION_KEY=c34d38b3a14956121ff2170e5030b471551370178f43e5626eec58b04a30fae2

```

### docker-compose.yml

#### 1. setup 컨테이너

- Elasticsearch & Kibana 보안 통신을 위한 인증서 발급과 패스워드 설정을 자동화  

docker-compose.yml

```yml
version: "3.8"


volumes:
 certs:
   driver: local
 esdata01:
   driver: local
 kibanadata:
   driver: local
 metricbeatdata01:
   driver: local
 filebeatdata01:
   driver: local
 logstashdata01:
   driver: local


networks:
 default:
   name: elastic
   external: false


services:
 setup:
   image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
   volumes:
     - certs:/usr/share/elasticsearch/config/certs
   user: "0"
   command: >
     bash -c '
       if [ x${ELASTIC_PASSWORD} == x ]; then
         echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
         exit 1;
       elif [ x${KIBANA_PASSWORD} == x ]; then
         echo "Set the KIBANA_PASSWORD environment variable in the .env file";
         exit 1;
       fi;
       if [ ! -f config/certs/ca.zip ]; then
         echo "Creating CA";
         bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
         unzip config/certs/ca.zip -d config/certs;
       fi;
       if [ ! -f config/certs/certs.zip ]; then
         echo "Creating certs";
         echo -ne \
         "instances:\n"\
         "  - name: es01\n"\
         "    dns:\n"\
         "      - es01\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         "  - name: kibana\n"\
         "    dns:\n"\
         "      - kibana\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         > config/certs/instances.yml;
         bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
         unzip config/certs/certs.zip -d config/certs;
       fi;
       echo "Setting file permissions"
       chown -R root:root config/certs;
       find . -type d -exec chmod 750 \{\} \;;
       find . -type f -exec chmod 640 \{\} \;;
       echo "Waiting for Elasticsearch availability";
       until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
       echo "Setting kibana_system password";
       until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
       echo "All done!";
     '
   healthcheck:
     test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
     interval: 1s
     timeout: 5s
     retries: 120
```
#### 2. es01 컨테이너
<hr/>
- Elasticsearch의 단일 노드 클러스터
- Elasticsearch 서버를 실행하는 곳!  
  
docker-compose.yml

```yml
 es01:
   depends_on:
     setup:
       condition: service_healthy
   image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
   labels:
     co.elastic.logs/module: elasticsearch
   volumes:
     - certs:/usr/share/elasticsearch/config/certs
     - esdata01:/usr/share/elasticsearch/data
   ports:
     - ${ES_PORT}:9200
   environment:
     - node.name=es01
     - cluster.name=${CLUSTER_NAME}
     - discovery.type=single-node
     - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
     - bootstrap.memory_lock=true
     - xpack.security.enabled=true
     - xpack.security.http.ssl.enabled=true
     - xpack.security.http.ssl.key=certs/es01/es01.key
     - xpack.security.http.ssl.certificate=certs/es01/es01.crt
     - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
     - xpack.security.transport.ssl.enabled=true
     - xpack.security.transport.ssl.key=certs/es01/es01.key
     - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
     - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
     - xpack.security.transport.ssl.verification_mode=certificate
     - xpack.license.self_generated.type=${LICENSE}
   mem_limit: ${ES_MEM_LIMIT}
   ulimits:
     memlock:
       soft: -1
       hard: -1
   healthcheck:
     test:
       [
         "CMD-SHELL",
         "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
       ]
     interval: 10s
     timeout: 10s
     retries: 120
```

#### 3. 인증서 테스트!
1) 도커 컨테이너 이름 확인
```bash
    docker ps
 ```
- names 열에 있는 것이 이름!

2) 현재 작업 중인 폴더에 인증서 복사
 ```bash
    docker cp picky-ber-es01-1:/usr/share/elasticsearch/config/certs/ca/ca.crt .
```

3) 인증서가 다운로드 되면 엘라스틱 서치 실행!
 ```bash
    curl --cacert ./ca.crt -u elastic:changeme https://localhost:9200
```
- changeme는 엘라스틱 서치 비밀번호

**또한 인증서를 신뢰하도록 설정해야함**
  

1) cacerts 위치 찾기
```yml
    find /Library/Java/JavaVirtualMachines -name cacerts
```
2) 인증서 등록
```yml
sudo keytool -importcert \
 -trustcacerts \
 -alias elasticsearch-ca \
 -file ./ca.crt \
 -keystore /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home/lib/security/cacerts \
 -storepass changeit
```
- 기본 비밀번호는 changeit -> 변경하기  
  


이렇게 나왔다면 성공!!

```bash
{
  "name" : "es01",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "xT1U6rt_RfyEWrLh_Ox2sA",
  "version" : {}
    "number" : "8.17.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2b6a7fed44faa321997703718f07ee0420804b41",
    "build_date" : "2024-12-11T12:08:05.663969764Z",
    "build_snapshot" : false,
    "lucene_version" : "9.12.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```


#### 4. kibana
- 엘라스틱 서치가 잘 실행되는지 확인 하고 kibana 설치  

docker-compose.yml
```yml
kibana:
   depends_on:
     es01:
       condition: service_healthy
   image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
   labels:
     co.elastic.logs/module: kibana
   volumes:
     - certs:/usr/share/kibana/config/certs
     - kibanadata:/usr/share/kibana/data
   ports:
     - ${KIBANA_PORT}:5601
   environment:
     - SERVERNAME=kibana
     - ELASTICSEARCH_HOSTS=https://es01:9200
     - ELASTICSEARCH_USERNAME=kibana_system
     - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
     - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
     - XPACK_SECURITY_ENCRYPTIONKEY=${ENCRYPTION_KEY}
     - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${ENCRYPTION_KEY}
     - XPACK_REPORTING_ENCRYPTIONKEY=${ENCRYPTION_KEY}
   mem_limit: ${KB_MEM_LIMIT}
   healthcheck:
     test:
       [
         "CMD-SHELL",
         "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
       ]
     interval: 10s
     timeout: 10s
     retries: 120
```

컨테이너가 정상이라면 http://localhost:5601 접속해서, 접속 가능한지 확인!  

#### 5. Logstash

docker-compose.yml  
```yml
 logstash01:
   depends_on:
     es01:
       condition: service_healthy
     kibana:
       condition: service_healthy
   image: docker.elastic.co/logstash/logstash:${STACK_VERSION}
   labels:
     co.elastic.logs/module: logstash
   user: root
   volumes:
     - certs:/usr/share/logstash/certs
     - logstashdata01:/usr/share/logstash/data
     - "./logstash_ingest_data/:/usr/share/logstash/ingest_data/"
     - "./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro"
   environment:
     - xpack.monitoring.enabled=false
     - ELASTIC_USER=elastic
     - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
     - ELASTIC_HOSTS=https://es01:9200
```
- logstash.conf 파일 만들기  

logstash.conf

```yml
input {
 file {
   #https://www.elastic.co/guide/en/logstash/current/plugins-inputs-file.html
   #default is TAIL which assumes more data will come into the file.
   #change to mode => "read" if the file is a compelte file.  by default, the file will be removed once reading is complete -- backup your files if you need them.
   mode => "tail"
   path => "/usr/share/logstash/ingest_data/*"
 }
}


filter {
}


output {
 elasticsearch {
   index => "logstash-%{+YYYY.MM.dd}"
   hosts=> "${ELASTIC_HOSTS}"
   user=> "${ELASTIC_USER}"
   password=> "${ELASTIC_PASSWORD}"
   cacert=> "certs/ca/ca.crt"
 }
}

```

일단 여기까지 하면 기본적인 설치는 끝났다.  
아직 단일 노드이고, filebeat, Metricbeat과 같은 여러 기능을 사용하지 않았다.  
실제 환경에서는 더 추가 해야할 것들이 많다.  
하지만 일단 난 로컬 환경에서 테스트용으로 사용할 예정이므로 이정도까지만 설치할 것이다.
