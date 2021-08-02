## ELK日志系统

### 背景：
- 随着服务系统的不断扩大与拆分，排查问题变得十分复杂，将所有服务的日志采集到一起，并且各服务间的日志可以串起来，这项工作就变得很重要了。
  日志的串接属于trace的范畴，在此不展开。

### 选型：
- Flume （这个我没有研究，好像用的也比较多）
- ELK （es+logstash+kibana）变种（filebeat+kafka+logstash+es+kibana）。后者用的比较多。
- 需要注意的是ELK各组件的版本一定要一致，本文中的版本是7.10.0
- 部署全部使用Docker，部署更简单，也更干净


### 采集工具-Filebeat
- logstash采集日志 普遍反映性能不行。后来推出了beats，日志采集就是filebeat
- filebeat可以采集文本日志，docker标准输出的日志。
- 但是docker的配置的是容器id，由于docker发生重启会导致容器id变化，该方式不灵活，战略上弃用。
- 或者自动发现所有的容器，事情突然变得不可控了起来。有一些docker运行的日志我们并不想要，所以也不建议使用。
- 最终方案就是服务将日志输出的文件，每台服务器上部署一个filebeat采集日志。即使由于各种问题导致日志没有采集到，本地也还有保留，可用性也是更高的。
- filebeat会在文件中记录当前读的offset，不用担心filebeat挂掉导致日志丢失与重复消费的问题。
- filebeat新版镜像内置了kibana，所以其他的组件都可以省了。可用性很低，不建议使用
- filebeat可以将日志输出到kafka、logstash、es。在此建议先到kafka，再到es 理由如下
    - 写入kafka后，后面的组件挂了数据不会丢
    - kafka具备一定的抗压能力
    - 可以多分支消费，万一你还有别的用处呢
    - kafka->es的流程需要借助logstash，这下所有组件就都用上了
    - kafka->es的流程也可以借助flink完成，本人没有研究，在此就不展开了

- 为什么使用host模式 这样会使用宿主机的时区 不然的话会使用utc
- 如果要采集docker日志，需要（docker in docker）模式，即需要将宿主机 /var/run/docker.sock 映射到容器内
- Filebeat Docker-compose 文件如下

```yaml
version: '3.3' 
services: 
  fb: 
    image: docker.elastic.co/beats/filebeat:7.10.0 
    container_name: fb 
    network_mode: host 
    ulimits: 
      nofile: 
        soft: "102400" 
        hard: "102400" 
    volumes: 
      - $PWD/config/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro 
      - $PWD/registry:/usr/share/filebeat/data/registry 
      - /data/logs:/data/logs 
    restart: always 
    logging: 
      driver: "json-file" 
      options: 
        max-size: "1g" 
```

- 配置文件 config/filebeat.docker.yml
```yaml
setup.ilm.enabled: false 
setup.template.enabled: true 
setup.template.name: "qingserverlog" 
setup.template.pattern: "qingserverlog_*" 
 
output.kafka: 
  enabled: true 
  hosts: ["127.0.0.1:9092"] 
  topic: "filebeat_log_%{[fields.service]}" 
  required_acks: 1 
  compression: gzip 
  max_message_bytes: 1000000 

  
filebeat.inputs: 
- type: log 
  paths: 
    - /data/logs/github.com/ClearGrass/mqttRoute/server.log 
    - /data/logs/github.com/ClearGrass/mqttPlugin/server.log 
  fields: 
    service: qingServer 
```

### Logstash 数据管道
- logstash 负责消费kafka的数据到es
- docker-compose文件如下：

```yaml
version: '3.3' 
services: 
  logstash: 
    image: docker.elastic.co/logstash/logstash:7.10.0 
    container_name: logstash 
    restart: always 
    ulimits: 
      nofile: 
        soft: "102400" 
        hard: "102400" 
    environment: 
      - MONITORING_ELASTICSEARCH_HOSTS=http://127.0.0.1:9200
    volumes: 
      - $PWD/logstash/pipeline:/usr/share/logstash/pipeline:rw 
      - $PWD/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro 
    logging: 
      driver: "json-file" 
      options: 
        max-size: "1g" 
```

- 配置文件 pipline/logstash.conf
```yaml
input { 
  kafka { 
    bootstrap_servers => ["127.0.0.1:9092"] 
    group_id => "logstash-server-log" 
    topics => ["filebeat_log_qingDaily"] 
    codec => json 
 } 
} 
 
filter { 
 
} 
 
output { 
  elasticsearch { 
    hosts => ["http://127.0.0.1:9200"] 
    index => "serverlog-%{+YYYY-MM-dd}" 
  } 
} 
```

- 配置文件 config/logstash.yml
```yaml
http.host: "0.0.0.0" 
monitoring.elasticsearch.hosts: [ "http://127.0.0.1:9200" ] 
```


### 存储与看板
- 存储就是用es，看板就是kibana
- es存在的问题就是，官方提供的加密认证x-pack收费。所以我就不用了，简单用nginx对kibana做个认证
- 过期日志的清理，index结尾都是根据日期命名的，所以直接调用es的接口模糊删除索引，然后把任务加入cron
- es的docker-compose文件如下。es是单点的

```yaml
version: '3.3' 
services: 
  es: 
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.0 
    network_mode: host 
    container_name: es 
    user: root 
    ulimits: 
      nofile: 
        soft: "102400" 
        hard: "102400" 
    volumes: 
      - $PWD/esdata/data:/usr/share/elasticsearch/data:rw 
      - $PWD/esdata/logs:/usr/share/elasticsearch/logs:rw 
    environment: 
      - discovery.type=single-node 
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" 
    logging: 
      driver: "json-file" 
      options: 
        max-size: "1g" 
 
  kibana: 
    image: docker.elastic.co/kibana/kibana:7.10.0 
    container_name: kibana 
    ports: 
      - "5601:5601" 
    ulimits: 
      nofile: 
        soft: "102400" 
        hard: "102400" 
    environment: 
      - ELASTICSEARCH_HOSTS=http://127.0.0.1:9200 
    volumes: 
      - $PWD/kbdata/data:/usr/share/kibana/data:rw 
      - $PWD/kbdata/plugins:/usr/share/kibana/plugins:rw 
    logging: 
      driver: "json-file" 
      options: 
        max-size: "1g" 
```

- kibana的nginx+htpasswd认证
```yaml
server { 
           server_name localhost; 
           listen 5602; 
           location / { 
                proxy_pass http://127.0.1:5601; 
                auth_basic "登陆验证"; 
                auth_basic_user_file /etc/nginx/pwddb/kibana.db; 
           } 
} 
```


### 清理过期日志
- 没有什么好的办法 调用es的删除index的接口，编写脚本，任务加入crontab

```python3
import datetime,os 
 
def cleanIndex(daysKeep): 
    daysKeep = 0- daysKeep 
    t = datetime.datetime.now() + datetime.timedelta(days=daysKeep) 
    par = t.strftime('%Y-%m-%d') 
 
    url = 'http://127.0.0.1:9200/' + '*' + par 
 
    cmd = 'curl -X DELETE ' + url 
    print('do cmd:',cmd) 
    os.popen(cmd).readline() 
 
# 清理三天前的日志
cleanIndex(3) 
```