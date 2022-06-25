# kafka Set





## 1) download strimzi



[GitHub releases page](https://github.com/strimzi/strimzi-kafka-operator/releases).



```sh
unzip strimzi-<version>.zip

```





## 3) Installing Strimzi

stimzi crd 생성 



```sh
kubectl create ns kafka

```





```sh
sed -i 's/namespace: .*/namespace: kafka/' install/cluster-operator/*RoleBinding*.yaml

```



Give permission to the Cluster Operator to watch the kafka namespace

```sh
kubectl create -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n kafka


kubectl create -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n kafka

```







# 3. Kafka Cluster 생성



## 3.1 ephemeral sample

### 1) kafka cluster 생성(no 인증)

- 기본생성

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.2.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.2"
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {} 
```







### 2) kafka cluster 생성(인증)

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.2.0
    replicas: 3
    authorization:
      type: simple
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
        authentication:
          type: scram-sha-512
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.2"
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {} 
```

- 인증메커니즘

  - SASL 은 인증 멫 보안 서비스를 제공하는 프레임워크이다.
  - 위 yaml 파일의 인증방식은 scram-sha-512  방식인데 이는 SASL 이 지원하는 메커니즘 중 하나이며 Broker 를 SASL 구성로 구성한다.

- Client 에서 접속하는 두가지 방법

  - sasl.jaas.config 설정

  - JAAS Configuration

    - 애플리케이션 구동시 아규먼트 이용

    ```
    -Djava.security.auth.login.config={JAAS File} 
    ```

  - Kafka client 0.10.2.X 이상 버전 일때는 JAAS Config 방식을 사용하지 않고 클라이언트의 특성에서 직접 SASL 인증을 구성

- security.protocol 설정

  - tls 방식은 아니므로 _SASL_PLAINTEXT_로 설정한다.

- cluster 생성

```
$ oc -n kafka create -f 11.kafka-sa-cluster.yaml
```

- deleteClaim
  - kafka cluster 가 undeployed 되었을때 데이터를 삭제할지를 묻는다.









# 4.  KafkaUser

- KafkaUser 를 생성하면 secret 에 Opaque 가 생성되며 향후 인증 password 로 사용됨
- 어떤 topic 에 접근 가능할지를 명시할 수 있다.
- 그러므로 특정 user로 namespace별 topic 간 경계설정이 가능하다.



## 4.1. User 정책



- user 정책

```
[Part명]-user
[Part명]-[서비스명]-user
[Part명]-[서비스명]-[서브도메인]-user
```



- sample user 별 설명

```
ㅇ my-user
my 로 시작하는 모든 topic을 처리할 수 있음
my 로 시작하는 모든 group을 Consume 가능

ㅇ order-user
order로 시작하는 모든 topic을 처리할 수 있음
order로 시작하는 모든 group을 Consume 가능

ㅇ order-user-readonly
order로 시작하는 모든 topic을 읽을 수 있음
order로 시작하는 모든 group을 Consume 가능
```



## 4.2 user 생성

```sh
cd /home/mobaxterm/song/kafka/strimzi/kafkaUser 
cat > 11.KafkaUser-my-bridge-user-my-topic.yaml
---
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaUser
metadata:
  name: my-user
  labels:
    strimzi.io/cluster: arsenal-cluster
  namespace: kafka
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - operation: All
        resource:
          type: topic
          name: my
          patternType: prefix     # 1)
      - operation: All
        resource:
          name: my            # 2)
          patternType: prefix
          type: group
      - operation: All
        resource:
          name: my           # 3)
          patternType: prefix
          type: group
---
```

- 1) order로 시작하는 topic 을 모두 처리가능

  - ex) order-intl-board-create,  order-intl-board-update

- 2) consumer group 은 항상 동일하다. 

  - ex) order-intl-board-group

- 3) default consumer-group 을 위해서 생성한다.



```sh
$ oc -n kafka create -f 11.KafkaUser-order-user-my-topic.yaml
$ oc -n kafka get secret order-user
NAME             TYPE      DATA      AGE
order-user   Opaque    1         26d

$ kubectl -n kafka get secret my-user


# user/pass 
  my-user / RUqfDsDzvZdL
  
```





## 4.3 참고 yaml 

### 1) 참고 - my-bridge-user

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-bridge-user
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: rater       <--- rater.my-topic
          patternType: literal
          #patternType: prefix
        operation: All
        #operation: Read
      - operation: All
        resource:
          name: my-topic
          patternType: literal
          type: topic
      - operation: All
        resource:
          name: bridge-consumer-group
          patternType: literal
          type: group
      - operation: All
        resource:
          name: console
          patternType: prefix
          type: group
---
```



### 2) 참고 - my-user

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - operation: All
        resource:
          name: my
          patternType: prefix
          type: topic
      - operation: All
        resource:
          name: my
          patternType: prefix
          type: group

```







### 3) 참고 - order-user

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: order-user
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    acls:
      - operation: All
        resource:
          name: order
          patternType: prefix
          type: topic
      - operation: All
        resource:
          name: order
          patternType: prefix
          type: group
    type: simple

## 
```





# 5. KafkaTopic



## 5.1. Topic 정책 

- topic 정책

```
[Part명]-[서비스명]-[서브도메인]-[사용자정의]
```



- sample topic 

```
order-intl-board-create
order-intl-board-update
order-intl-board-delete

bill-intl-board-create
bill-intl-board-update
bill-intl-board-delete

rater-intl-board-create
rater-intl-board-update
rater-intl-board-delete
```





## 5.2 Topic 생성

### 1) Topic 생성

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-my-topic1
  labels:
    strimzi.io/cluster: sa-cluster
  namespace: kafka
spec:
  partitions: 3
  replicas: 1
  config:
    #retention.ms: 7200000      # 2시간
    retention.ms: 86400000      # 24시간
    segment.bytes: 1073741824   # 1GB

```

- partitions 1이면 producer 수행시 아래 메세지 발생할 수 있음.
  -  LEADER_NOT_AVAILABLE



## 5.3. 참고 yaml



### 1) 참고 yaml

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-intl-board-create
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  partitions: 3
  replicas: 3
  topicName: order-intl-board-create
  config:
    #retention.ms: 7200000      # 2시간
    retention.ms: 86400000      # 24시간
    segment.bytes: 1073741824   # 1GB
  
```



### 2) 참고 yaml

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-intl-board-create
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  partitions: 3
  replicas: 3
  topicName: order-intl-board-create
  config:
    #retention.ms: 7200000      # 2시간
    retention.ms: 86400000      # 24시간
    segment.bytes: 1073741824   # 1GB
 
 
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-intl-board-update
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  partitions: 3
  replicas: 3
  topicName: order-intl-board-update
  config:
    #retention.ms: 7200000      # 2시간
    retention.ms: 86400000      # 24시간
    segment.bytes: 1073741824   # 1GB
  
  
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-intl-board-delete
  namespace: kafka-system
  labels:
    strimzi.io/cluster: sa-cluster
spec:
  partitions: 3
  replicas: 3
  topicName: order-intl-board-delete
  config:
    #retention.ms: 7200000      # 2시간
    retention.ms: 86400000      # 24시간
    segment.bytes: 1073741824   # 1GB
  
```































# Node port Setting



## 1) node port



### (1) Cluster Setting

```


...
listeners:
  # ...
  
  - name: external
    port: 9094
    type: nodeport
    tls: false
    configuration:
      brokers:
      - broker: 0
        advertisedHost: my-cluster.kafka.ktcloud.211.254.212.105.nip.io
      - broker: 1
        advertisedHost: my-cluster.kafka.ktcloud.211.254.212.105.nip.io
      - broker: 2
        advertisedHost: my-cluster.kafka.ktcloud.211.254.212.105.nip.io
        



211.254.212.105:31482
211.254.212.105:30084
211.254.212.105:30536
211.254.212.105:32129


my-cluster.kafka.ktcloud.211.254.212.105.nip.io:31482
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30084
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30536
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:32129

```





```sh

$ cat << EOF | kubectl create -n kafka -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: nodeport
        tls: false
    storage:
      type: ephemeral
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF



## clean up
$ kubectl -n kafka delete my-cluster


```



```sh
# kkf get svc
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
my-cluster-kafka-0                    NodePort    10.43.81.63     <none>        9094:30084/TCP                        11m
my-cluster-kafka-1                    NodePort    10.43.147.189   <none>        9094:30536/TCP                        5m52s
my-cluster-kafka-2                    NodePort    10.43.161.37    <none>        9094:32129/TCP                        5m52s
my-cluster-kafka-bootstrap            ClusterIP   10.43.88.79     <none>        9091/TCP,9092/TCP,9093/TCP            11m
my-cluster-kafka-brokers              ClusterIP   None            <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   11m
my-cluster-kafka-external-bootstrap   NodePort    10.43.68.229    <none>        9094:31482/TCP                        11m
my-cluster-zookeeper-client           ClusterIP   10.43.81.121    <none>        2181/TCP                              11m
my-cluster-zookeeper-nodes            ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            11m

```

node port 31482









### (2) kafkacat 테스트



```sh
ku run kafkacat --image=kafkacat -- sleep 365d
```



```sh
$ kubectl -n kafka run kafka-consumer -ti \
    --image=quay.io/strimzi/kafka:0.29.0-kafka-3.2.0 \
    --rm=true --restart=Never \
    -- bin/kafka-console-consumer.sh \
    --bootstrap-server my-cluster-kafka-bootstrap:9092 \
    --topic my-topic \
    --from-beginning


```







```sh

# 참고
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:31482,
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30084,
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30536,
my-cluster.kafka.ktcloud.211.254.212.105.nip.io:32129



## nodeport 로 확인
export BROKERS=my-cluster.kafka.ktcloud.211.254.212.105.nip.io:31482,my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30084,my-cluster.kafka.ktcloud.211.254.212.105.nip.io:30536,my-cluster.kafka.ktcloud.211.254.212.105.nip.io:32129
export TOPIC=my-topic

  
## topic list
kafkacat -b $BROKERS -L


## producer
kafkacat -b $BROKERS \
  -t $TOPIC -P -X acks=1

← 확인불가







```

