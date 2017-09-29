## Infrastructure
![Infrastructure](https://github.com/leeyan85/docker-elk/blob/master/ELK%20Architecture.png)

## Environment
### server functions
Host  | IP | application | note
---|---|---|---|
logs1  | 10.58.90.66 | logstash |
kafka1 | 10.58.90.56 | kafka + zookeeper| 
es0    | 10.58.90.77|elasticsearch| 	Coordinating-only Elasticsearch
es1    | 10.58.90.68|elasticsearch | master node
es2    | 10.58.90.69|elasticsearch | data node
kibana1| 10.58.90.55| kibana| kibana
### ansible init /etc/hosts
```
    cd ansible/
    ansible-play -i hosts group.yml
```    
## Kafka
### kafka config
**/opt/kafka/config/server.properties**

```
broker.id=0
listeners=PLAINTEXT://kafka1:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/opt/kafka/logs
num.partitions=1
num.recovery.threads.per.data.dir=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=6000
delete.topic.enable=true
```
### start kafka

```
nohup /opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeper.properties
nohup /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```



## Elasticsearch
### Elasticsearch config
Elasticsearch have 3 mode, and all mode configure file is ** /etc/elasticsearch/elasticsearch.yml**

Tunning JVM can be done by setting JVM heap size in **/etc/elsticsearch/jvm.options**

1. Elasticsearch master node
    
    ```
    
    cluster.name: scm-logging
    node.name: es1
    node.master: true
    node.data: true
    path.data: /opt/elasticsearch/data
    path.logs: /opt/elasticsearch/log
    network.host: [_eth0_, _local_]
    http.cors.enabled: true
    http.cors.allow-origin: "*"
    discovery.zen.ping.unicast.hosts: ["es1"]
    ```

2. Elasticsearch data node

    ```
    cluster.name: scm-logging
    node.name: es2  # Change node.name to make it unique in the cluster
    node.master: false
    node.data: true
    path.data: /opt/elasticsearch/data
    path.logs: /opt/elasticsearch/log
    network.host: [_eth0_, _local_]
    discovery.zen.ping.unicast.hosts: ["es1"]
    ```

3. Elasticsearch coordinating-only node

    ```
    cluster.name: scm-logging
    node.name: es0
    node.master: false
    node.data: false
    node.ingest: false
    path.data: /opt/elasticsearch/data
    path.logs: /opt/elasticsearch/log
    network.host: [_eth0_, _local_]
    discovery.zen.ping.unicast.hosts: ["es1"]
    ```

### Elasticsearch template
1. create a template which define data types or formats for elasticserch indicies. Here is an example. **/data/elasticsearch/gerrit.template.json**
    
    ```
    {
      "template": "gerrit-*",
      "order": 1,
      "settings" : {
            "number_of_shards" : "3",
            "number_of_replicas" : "1"
      },
      "mappings": {
        "_default_": {
          "_all": {
            "enabled": false
          }
        }, "syslog" : {
          "properties" : {
            "@timestamp" : { "type" : "date", "format" : "date_time" },
            "@version" : { "type" : "string" , "index" : "not_analyzed" },
            "beat" : {
              "properties" : {
                "hostname" : {"type" : "string" },
                "name" : { "type" : "string" }
              }
            },
            "count" : { "type" : "long" },
            "host" : { "type" : "string", "index" : "not_analyzed" },
            "info" : { "type" : "string" },
            "input_type" : { "type" : "string" },
            "offset" : { "type" : "long" },
            "source" : { "type" : "string" },
            "pid" : { "type" : "string", "index" : "not_analyzed" },
            "program": { "type" : "string" },
            "syslog_timestamp" : { "type" : "string" },
            "tags" : { "type" : "string" },
            "type" : { "type" : "string" }
          }
        }
      }
    }   
    ```
2. put the template to Elasticsearch.
    
    ```
    curl -XPUT 'http://es0:9200/_template/gerrit?pretty' -d@/data/elasticsearch/gerrit.template.json
    ```

### Elasticsearch-head
elasticsearch-head is a web front end for browsing and interacting with an Elasticsearch cluster.

how to install elasticsearch-head

    ```
    sudo add-apt-repository ppa:chris-lea/node.js
    sudo apt-get update
    sudo apt-get install nodejs npm
    git clone git://github.com/mobz/elasticsearch-head.git
    cd elasticsearch-head
    npm install
    npm run start
    ```

## logstash
Put Logstash plugin configurations in /etc/logstash/conf.d/

[grok debug URL](http://grokdebug.herokuapp.com/)

**/etc/logstash/conf.d/apache_access.conf**


```
input {
    kafka {
      bootstrap_servers => "kafka1:9092"
      topics => ["logstash", "filebeat-apache-access"] 
      auto_offset_reset => "earliest"
      codec => json
    }
}

filter {
  if [type] == "apache_access" {
    grok {
      match => {
        "message" => "%{COMBINEDAPACHELOG}"
      }
    }

    date {
      match => [ "timestamp", "dd/MMM/YYYY:HH:mm:ss ZZ" ]
    }
  }
 else if [type] == "opengrok"{
     grok {
      match => {
        "message" => [
          #"%{COMBINEDAPACHELOG}"
          "%{IPORHOST:clientip} %{USER} %{USER} \[%{HTTPDATE:timestamp}\] \"%{WORD:verb} /%{WORD:platform}/xref(?:/%{DATA:file}) HTTP/%{NUMBER}\" %{NUMBER:response} (?:%{NUMBER:bytes}|-)",
          "%{IPORHOST:clientip} %{USER} %{USER} \[%{HTTPDATE:timestamp}\] \"%{WORD:verb} /%{WORD:platform}/search\?q=%{DATA} HTTP/%{NUMBER}\" %{NUMBER:response} (?:%{NUMBER:bytes}|-)"
        ]
      }
    }
    date {
      match => [ "timestamp", "dd/MMM/YYYY:HH:mm:ss ZZ" ]
    }
    if "_grokparsefailure" in [tags] { drop {} }
  }

}


output {
  if [type]=="apache_access" {
     elasticsearch {
        hosts => ["es1:9200", "es2:9200", "es3:9200"]
        manage_template => false
        index => "apache_access-%{+YYYY.MM.dd}"
        document_type => "%{[@metadata][type]}"
     }
   }

  else if [type]=="gerrit_access"{
        elasticsearch {
        hosts => ["es1:9200", "es2:9200", "es3:9200"]
        manage_template => false
        index => "gerrit_access-%{+YYYY.MM.dd}"
        document_type => "%{[@metadata][type]}"
     }
  }
  else if [type]=="opengrok_access"{
      elasticsearch {
      hosts => ["es1:9200", "es2:9200", "es3:9200"]
      index => "opengrok_access-%{+YYYY.MM.dd}"
      manage_template => false
      document_type => "%{[@metadata][type]}"
      #template_name => "opengrok"
    }

  }
}

```
## kibana
### kibana configure
configure file **/etc/kibana/kibana.yml**
```
server.host: "0.0.0.0"
elasticsearch.url: "http://es0:9200"
```
### 启动kibana
`service kibana restart`

## filbeat
filebeat is installed on the servers which need collect the log, and send the log to kafka

### config filebeat
config filebeat send message to kafka.

**/etc/filebeat/filebeat.yml**
```
filebeat:
  prospectors:
    #-
     #paths:
        #- /var/log/syslog
       #input_type: log
       #document_type: syslog
    -
      paths:
        - /var/log/apache2/log/access*.log
      input_type: log
      document_type: apache_access
      
output.kafka:
    hosts: ["kafka1:9200"]
    version: 0.10.0
    topic: "filebeat-apache-access"
    required_acks: 1
    compression: gzip
    max_message_bytes: 1000000
logging:
  to_files: true
  files:
    path: /var/log/filebeat.log
    name: filebeat.log
    rotateeverybytes: 10485760
```


