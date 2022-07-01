# Kafka Hands-in





# 1. Kafka 실습환경



## 1) Strimzi 설명





## 2) local pc에 kubernete 설치



localhost 에서 kubernetes 환경은 아래와 같이 두가지 방법으로 실행할 수 있다.



### (1) docker-desktop 에서 kubernetes 활성화 하기

dockerdesktop

위치 : Dashboard > Settings > Kubernetes

Enable Kubernetes 에 check 하기



- 확인

```
```





### (2) wsl 에서 k3s 설정





### (3) kubectl client 환경설정

kubernetes 가 기존에 설치 되어 있던 환경이라면 cluster 를 선택할 수 있다.

```sh
# context 확인
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         default          default          default
          docker-desktop   docker-desktop   docker-desktop


# docker-desktop 으로 변경
$ kubectl config set current-context docker-desktop


# context 확인
$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          default          default          default
*         docker-desktop   docker-desktop   docker-desktop



# kubectl 연결 확인
$ kubectl version -o yaml
clientVersion:
  buildDate: "2022-05-03T13:46:05Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64
kustomizeVersion: v4.5.4
serverVersion:
  buildDate: "2022-05-03T13:38:19Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64

# 위와 같이 serverVersion 이 표현되어야 정상연결 된 것이다.

```









# 2. Strimzi Cluster Operator Install

srimzi  operator 를 install 한다.



## 1) namespace 생성

strimzi operator 와 kafka cluster 를 kafka namespace 에 설치해야 한다.  worker node 를 준비한후 kafka namespace 를 생성하자.

```
$ kubectl create ns kafka
```







## 2) Strmzi download



- 해당 사이트(https://strimzi.io/downloads/) 에서 해당 버젼을 다운로드 받는다.

```sh
$ mkdir -p ~/song/strimzi

$ cd ~/song/strimzi

$ wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.29.0/strimzi-0.29.0.zip

$ unzip strimzi-0.29.0.zip

$ cd  ~/song/strimzi/strimzi-0.29.0
```



OR

교육자료를 이용해도 된다.

```sh
$ cd ~/githubrepo/

$ git clone https://github.com/ssongman/ktds-edu2

$ cd ~/githubrepo/ktds-edu2

$ cd ~/githubrepo/ktds-edu2/kafka/strimzi/strimzi-0.29.0/strimzi-0.29.0
```







## 3) single name 모드 namespace 설정

- single name 모드로 설치진행
  - strimzi operator 는 다양한 namespace 에서 kafka cluster 를 쉽게 생성할 수 있는 구조로 운영이 가능하다.  이때 STRIMZI_NAMESPACE 를 설정하여 특정 namespace 만으로 cluster 를 제한 할 수 있다.  ICIS-TR SA의 경우는 kafka-system 라는 namespace 에서만  kafka cluster 를 구성할 수 있도록 설정한다. 그러므로 아래 중 Single namespace 설정에 해당한다.

```sh
$ cd ~/githubrepo/ktds-edu2

$ sed -i 's/namespace: .*/namespace: kafka/' kafka/strimzi/install/cluster-operator/*RoleBinding*.yaml
```





## 4) deploy

- kafka namespace 를 watch 할 수 있는 권한 부여

```sh
$ cd ~/githubrepo/ktds-edu2

# kafka namespace 를 watch 할 수 있는 권한 부여
$ kubectl -n kafka create -f ./kafka/strimzi/install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml

$ kubectl -n kafka create -f ./kafka/strimzi/install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml

# Deploy the CRDs
$ kubectl -n kafka create -f ./kafka/strimzi/install/cluster-operator/



# 확인
$ kubectl -n kafka get pod
NAME                                        READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-86864b86d5-rfshw   0/1     Running   0          18s

```



## 5) clean up

```sh
$ cd ~/githubrepo/ktds-edu2

$ kubectl -n kafka delete -f ./kafka/strimzi/install/cluster-operator
```







# 3. Kafka Cluster 생성



## 1) ephemeral sample

### (1) kafka cluster 생성(no 인증)

아래는 인증없이 접근 가능한 kafka cluster 를 생성하는 yaml 이므로 참고만 하자.

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/kafka/11.kafka-ephemeral-no-auth.yaml
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


$ kubectl -n kafka apply -f  ./strimzi/kafka/11.kafka-ephemeral-no-auth.yaml

```





### (2) kafka cluster 생성(인증)

#### 생성

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/kafka/12.kafka-ephemeral-auth.yaml
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

# 생성
$ kubectl -n kafka apply -f ./kafka/strimzi/kafka/12.kafka-ephemeral-auth.yaml

```

- 인증메커니즘

  - SASL 은 인증 및 보안 서비스를 제공하는 프레임워크이다.
  - 위 yaml 파일의 인증방식은 scram-sha-512  방식인데 이는 SASL 이 지원하는 메커니즘 중 하나이며 Broker 를 SASL 구성로 구성한다.



#### 확인

```sh

$ kkf get pod
NAME                                        READY   STATUS    RESTARTS       AGE
kafkacat                                    1/1     Running   1 (7h7m ago)   7d18h
my-cluster-kafka-0                          1/1     Running   0              35s
my-cluster-kafka-1                          1/1     Running   0              35s
my-cluster-kafka-2                          1/1     Running   0              35s
my-cluster-zookeeper-0                      1/1     Running   0              59s
my-cluster-zookeeper-1                      1/1     Running   0              59s
my-cluster-zookeeper-2                      1/1     Running   0              59s
strimzi-cluster-operator-7c77f74847-llxn5   1/1     Running   1 (7h7m ago)   8d

$ kubectl -n kafka get kafka
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
my-cluster   3                        3                     True



```



#### clean up

```sh
$ kubectl -n kafka delete kafka my-cluster

```







# 4.  KafkaUser

- kafka cluster 생성시 scram-sha-512 type 의 authentication 를 추가했다면 반드시 KafkaUser 가 존재해야 한다.

- KafkaUser 를 생성하면 secret 에 Opaque 가 생성되며 향후 인증 password 로 사용된다.
- 어떤 topic 에 어떻게 접근할지 에 대한 acl 기능을 추가할 수 있다.



## 1) User 정책

아래와 같이 ACL (Access Control List) 정책을 지정할 수 있다.

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



## 2) User 생성

### (1) kafkauser생성

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/user/11.kafka-user.yaml
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
          patternType: prefix
      - operation: All
        resource:
          name: my
          patternType: prefix
          type: group
      - operation: All
        resource:
          type: topic
          name: edu
          patternType: prefix
      - operation: All
        resource:
          name: edu
          patternType: prefix
          type: group
---


# 실행
$ kubectl -n kafka apply -f ./kafka/strimzi/user/11.kafka-user.yaml

# kafkauser 확인
$ kubectl -n kafka get kafkauser
NAME      CLUSTER      AUTHENTICATION   AUTHORIZATION   READY
my-user   my-cluster   scram-sha-512    simple          True

```

- 1) my로 시작하는 topic 을 모두 처리가능

  - ex) my-board-create,  my-board-update

- 2) consumer group 은 항상 동일하다. 

  - ex) my-board-group

  

  

### (2) password 확인

```sh

$ kubectl -n kafka get secret my-user
NAME      TYPE     DATA   AGE
my-user   Opaque   2      28s

$ kubectl -n kafka get secret my-user -o jsonpath='{.data.password}' | base64 -d

pprOnk80CDfo

# user/pass 
## KT Cloud 기준 : my-user / pprOnk80CDfo
## Local 기준    : my-user / eGVNg7ZvPbi0 
## 수강생 기준    : my-user / uL5fI10uQx4m
  
  
```





### (3) clean up

```sh
$ kubectl -n kafka delete kafkauser my-user

```





# 5. KafkaTopic



## 1) Topic 정책 

앞서 KafkaUser 의 ACL 기능을 이용해서 kafka topic 을 제어하는 방법을 확인했다.  그러므로 topiic 명칭을 어떻게 정하는지에 대해서 다양한 시나리오를 생각해 볼 수 있다. 아래 특정 프로젝트의 topic name 정책을 살펴보자.



### (1) ICIS-TR Topic Name 정책

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





## 2) Topic 생성

### (1) KafkaTopic 생성

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/topic/11.kafka-topic.yaml
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
  namespace: kafka
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 7200000      # 2 hour
    #retention.ms: 86400000      # 24 hours
    segment.bytes: 1073741824   # 1GB


# topic 생성
$ kubectl -n kafka apply -f ./kafka/strimzi/topic/11.kafka-topic.yaml


# topic 생성 확인
$ kubectl -n kafka get kafkatopic my-topic
NAME       CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
my-topic   my-cluster   3            3                    True

```



### (2) Topic  상세 확인

```sh
$ kubectl -n kafka get kafkatopic my-topic -o yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"kafka.strimzi.io/v1beta2","kind":"KafkaTopic","metadata":{"annotations":{},"labels":{"strimzi.io/cluster":"my-cluster"},"name":"my-topic","namespace":"kafka"},"spec":{"config":{"retention.ms":86400000,"segment.bytes":1073741824},"partitions":3,"replicas":3}}
  creationTimestamp: "2022-06-30T12:34:43Z"
  generation: 1
  labels:
    strimzi.io/cluster: my-cluster
  name: my-topic
  namespace: kafka
  resourceVersion: "7118"
  uid: 53b4001d-1b54-48c9-b749-997a5beb8dd4
spec:
  config:
    retention.ms: 86400000
    segment.bytes: 1073741824
  partitions: 3
  replicas: 3
status:
  conditions:
  - lastTransitionTime: "2022-06-30T12:34:44.290021Z"
    status: "True"
    type: Ready
  observedGeneration: 1
  topicName: my-topic

```

- status 에서 true, ready 임을 확인하자.



### (3) clean up

```sh
$ kubectl -n kafka delete kafkatopic my-topic

```







# 6. Internal access

- 본 과정에서는 모두 인증(user/pass) 으로 테스트를 수행함.
- cluster 생성시 authentication.type이 scram-sha-512 일 경우 반드시 인증(user/pass) 가 있어야 함
- topic 관련 sh 은 zookeeper 에 연결되므로 scram-sha-512인증필요 없이 바로 접속 가능



## 1) 클러스터 내부에서 연결 매커니즘



### (1) 클라이언트와 브로커 직접연결

- 특정 파티션에 지정된 클라이언트는 해당 파티션을 호스팅하는 리더 브로커에 직접 연결함

- 그러므로 브로커간 데이터전달 불필요하며 이는 클러스터내 트래픽의 양을 줄이는데 도움이 됨



![파티션에 연결하는 클라이언트](kafka.assets/2019-04-17-connecting-to-leader.png)

일단 수신되면 클라이언트는 *메타데이터* 를 사용 하여 주어진 파티션에 쓰거나 읽으려고 할 때 연결할 위치를 파악합니다. *메타데이터* 에 사용된 브로커 주소 는 브로커가 실행되는 시스템의 호스트 이름을 기반으로 브로커 자체에서 생성됩니다. 또는 옵션을 사용하여 사용자가 구성할 수 있습니다 `advertised.listeners`. 클라이언트는 메타데이터의 주소를 사용하여 *관심* 있는 특정 파티션을 호스팅하는 브로커의 주소에 대한 하나 이상의 새 연결을 엽니다. *메타데이터* 가 클라이언트가 이미 연결하고 수신한 동일한 브로커를 가리키는 경우에도 *메타데이터*에서 여전히 두 번째 연결을 엽니다. 그리고 이러한 연결은 데이터를 생성하거나 소비하는 데 사용됩니다.



클라이언트는 어떻게 해당 브로커의 위치를 알 수 있을까?



### (2) kafka discovery protocol



Kafka에는 자체 검색 프로토콜이 있습니다. Kafka 클라이언트가 Kafka 클러스터에 연결할 때 먼저 클러스터의 구성원인 브로커에 연결하고 하나 이상의 주제에 대한 *메타데이터 를 요청합니다.* *메타데이터* 에는 주제, 해당 파티션 및 이러한 파티션을 호스팅하는 브로커에 대한 정보가 포함됩니다 . 모든 브로커는 Zookeeper를 통해 모두 동기화되기 때문에 전체 클러스터에 대해 이 데이터를 가지고 있어야 합니다. 따라서 클라이언트가 처음으로 연결된 브로커는 중요하지 않습니다. 모든 브로커가 동일한 응답을 제공합니다.

![Kafka 클라이언트와 Kafka 클러스터 간의 연결 흐름](kafka.assets/2019-04-17-connection-flow.png)

일단 수신되면 클라이언트는 *메타데이터* 를 사용 하여 주어진 파티션에 쓰거나 읽으려고 할 때 연결할 위치를 파악합니다. *메타데이터* 에 사용된 브로커 주소 는 브로커가 실행되는 시스템의 호스트 이름을 기반으로 브로커 자체에서 생성됩니다. 또는 옵션을 사용하여 사용자가 구성할 수 있습니다 `advertised.listeners`. 클라이언트는 메타데이터의 주소를 사용하여 *관심* 있는 특정 파티션을 호스팅하는 브로커의 주소에 대한 하나 이상의 새 연결을 엽니다. *메타데이터* 가 클라이언트가 이미 연결하고 수신한 동일한 브로커를 가리키는 경우에도 *메타데이터*에서 여전히 두 번째 연결을 엽니다. 그리고 이러한 연결은 데이터를 생성하거나 소비하는 데 사용됩니다.





### (3) 클러스터 내부에서 연결

Kafka 클러스터와 동일한 Kubernetes 클러스터 내에서 실행되는 클라이언트에 대해 이 작업을 수행하는 것은 실제로 매우 간단합니다. 각 포드에는 다른 애플리케이션이 직접 연결하는 데 사용할 수 있는 고유한 IP 주소가 있습니다. 이것은 일반적으로 일반 Kubernetes 애플리케이션에서 사용되지 않습니다. 그 이유 중 하나는 Kubernetes가 이러한 IP 주소를 검색하는 좋은 방법을 제공하지 않기 때문입니다. IP 주소를 찾으려면 Kubernetes API를 사용하고 올바른 포드와 해당 IP 주소를 찾아야 합니다. 그리고 이에 대한 권리가 있어야 합니다. 대신 Kubernetes는 안정적인 DNS 이름이 있는 서비스를 기본 검색 메커니즘으로 사용합니다. 그러나 Kafka에서는 자체 discovery protocol 이 있으므로 문제가 되지 않습니다. 클라이언트가 Kubernetes API에서 API 주소를 알아낼 필요가 없습니다.

그러나 Strimzi에서 사용하는 더 나은 옵션이 하나 더 있습니다. Strimzi가 Kafka 브로커를 실행하는 데 사용하는 StatefulSet의 경우 Kubernetes 헤드리스 서비스를 사용하여 각 포드에 안정적인 DNS 이름을 지정할 수 있습니다. Strimzi는 이러한 DNS 이름을 Kafka 브로커에 대해 광고된 주소로 사용하고 있습니다. 

- 초기 연결은 *메타데이터* 를 가져오기 위해 일반 Kubernetes service 를 사용하여 수행됩니다 .
- 후속 연결은 다른 headless Kubernetes service 에서 Pod에 제공한 DNS 이름을 사용하여 열립니다. 





![동일한 Kubernetes 클러스터 내에서 Kafka에 액세스](kafka.assets/2019-04-17-inside-kubernetes.png)



두 접근 방식 모두 장단점이 있습니다. DNS를 사용하면 캐시된 DNS 정보에 문제가 발생할 수 있습니다. 예를 들어 롤링 업데이트 중에 포드의 기본 IP 주소가 변경되면 브로커에 연결하는 클라이언트에 최신 DNS 정보가 있어야 합니다. 그러나 때때로 Kubernetes가 매우 적극적으로 IP 주소를 재사용하고 새 포드가 다른 Kafka 노드에서 불과 몇 초 전에 사용된 IP 주소를 가져오기 때문에 IP 주소를 사용하면 더 심각한 문제가 발생한다는 것을 알게 되었습니다.







## 2) Kafka Cluster service name 

일반 서비스와 headless 서비스를 확인할 수 있다.

```sh
$ kubectl -n kafka get svc
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
my-cluster-kafka-bootstrap    ClusterIP   10.43.79.201    <none>        9091/TCP,9092/TCP,9093/TCP            80m
my-cluster-kafka-brokers      ClusterIP   None            <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   80m
my-cluster-zookeeper-client   ClusterIP   10.43.125.232   <none>        2181/TCP                              80m
my-cluster-zookeeper-nodes    ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            80m

```

- 우리는 Cluster 내에서  my-cluster-kafka-bootstrap:9092 로 접근을 시도할 것이다.





## 3) kafkacat 로 확인

kafka 접근 가능여부를 확인하기 위해 kafka Client 용 app 인 kafkacat 을 설치하자.



### (1) kafkacat 설치

```sh
# kafka cat 설치
$ kubectl -n kafka create deploy kafkacat \
    --image=confluentinc/cp-kafkacat:latest \
    -- sleep 365d

# 설치진행 확인
$ kubectl -n kafka get pod


# pod 내부로 진입( bash 명령 수행)
$ kubectl -n kafka exec -it deploy/kafkacat -- bash


```



#### ※ 참고
windows 환경의 gitbash 를 이용해 pod 내부명령을 수행한다면 prompt 가 보이지 않을수도 있다.

이런경우 windows 에서 linux 체제와 호환이 되지 않아서 발생하는 이슈이다.

아래와 같이 winpty 를 붙인다면 prompt 가 보이니 참고하자.

```sh

# pod 내부명령 수행
$ winpty kubectl -n kafka exec -it deploy/kafkacat -- bash
```





### (2) pub/sub test

id/pass 가 필요

```sh
$ kubectl -n kafka exec -it deploy/kafkacat -- bash

export BROKERS=my-cluster-kafka-bootstrap:9092
export KAFKAUSER=my-user
export PASSWORD=uL5fI10uQx4m        ## 개인별 passwrod 붙여넣자.
export TOPIC=my-topic
 
## topic 리스트
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD -L

Metadata for all topics (from broker -1: sasl_plaintext://my-cluster-kafka-bootstrap:9092/bootstrap):
 3 brokers:
  broker 0 at my-cluster-kafka-0.my-cluster-kafka-brokers.kafka.svc:9092
  broker 2 at my-cluster-kafka-2.my-cluster-kafka-brokers.kafka.svc:9092
  broker 1 at my-cluster-kafka-1.my-cluster-kafka-brokers.kafka.svc:9092 (controller)
 1 topics:
  topic "my-topic" with 3 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
    partition 1, leader 0, replicas: 0, isrs: 0
    partition 2, leader 2, replicas: 2, isrs: 2

## broker0, 1, 2 의 주소를 잘 이해하자.
## 내부 



## consumer
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -C -o -5

<-- OK 

## consumer group
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -C \
  -X group.id=my-board-group

<-- OK 



## consumer group
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -C \
  -X group.id=order-intl-board-group -o -5

<-- OK 




## producer : 입력모드
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -P -X acks=1
 
<-- OK


## 대량 발송 모드
$ cat > msg.txt
---
{"eventName":"a","num":1,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }
---

## producer : file mode
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -P ./msg.txt

<-- OK

## producer : while
while true; do kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -P ./msg.txt; done;

<-- OK

```



### (3) Clean up

```sh

## delete deploy
$ kubectl -n kafka delete deploy kafkacat
```





# 7. External access

참고: https://strimzi.io/blog/2019/04/17/accessing-kafka-part-1/

Strimzi 는 외부에서 접근가능하도록  다양한 기능을 제공한다.  

- 특정 파티션에 지정된 클라이언트는 해당 파티션을 호스팅하는 리더 브로커에 직접 연결함

- 그러므로 브로커간 데이터전달 불필요하며 이는 클러스터내 트래픽의 양을 줄이는데 도움이 됨
- 클라이언트는 어떻게 해당 브로커의 위치를 알 수 있을까?





## 1) Node Port



### (1) Kafka Cluster NodePort 등록

- node port 는 route type 을 사용하지 못한다.(택일해야 함)
- 인증방식으로 셋팅

```sh
$ kubectl -n kafka edit kafka my-cluster

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
  ...
spec:
  ...
    listeners:
    - authentication:
        type: scram-sha-512
      name: plain
      port: 9092
      tls: false
      type: internal
    - name: tls
      port: 9093
      tls: true
      type: internal
    
    ## nodeport type 등록 - 아래모두 삽입 하자.
    - name: external
      port: 9094
      type: nodeport
      tls: false
      authentication:
        type: scram-sha-512
      configuration:
        bootstrap:
          nodePort: 32100
        brokers:
        - broker: 0
          advertisedHost: my-cluster.kafka.localhost.192.168.31.1.nip.io
          nodePort: 32000
        - broker: 1
          advertisedHost: my-cluster.kafka.localhost.192.168.31.1.nip.io
          nodePort: 32001
        - broker: 2
          advertisedHost: my-cluster.kafka.localhost.192.168.31.1.nip.io
          nodePort: 32002

...
---


```

- nodePort 를 직접 명시할 수 있다.

- AdvertisedHost 필드에는 DNS 이름이나 IP 주소를 표기할 수 있다.

- node port 를 인식할 수 있는 본인 PC 의 IP를 인식하도록 nip host 에 본인 IP 를 삽입하자.

- 참고로 본인  IP 는 명령으로 확인할 수 있다.

  ```sh
  $ ipconfig
  
  Windows IP 구성
  
  
  무선 LAN 어댑터 로컬 영역 연결* 1:
  
     미디어 상태 . . . . . . . . : 미디어 연결 끊김
     연결별 DNS 접미사. . . . :
  
  무선 LAN 어댑터 로컬 영역 연결* 10:
  
     미디어 상태 . . . . . . . . : 미디어 연결 끊김
     연결별 DNS 접미사. . . . :
  
  이더넷 어댑터 VMware Network Adapter VMnet1:
  
     연결별 DNS 접미사. . . . :
     링크-로컬 IPv6 주소 . . . . : fe80::b43c:3b41:b773:48da%9
     IPv4 주소 . . . . . . . . . : 192.168.31.1                   <=============  해당 IP 를 추출한다.
     서브넷 마스크 . . . . . . . : 255.255.255.0
     기본 게이트웨이 . . . . . . :
  
  이더넷 어댑터 VMware Network Adapter VMnet8:
  
     연결별 DNS 접미사. . . . :
     링크-로컬 IPv6 주소 . . . . : fe80::905c:f7ec:a1e4:7ca6%12
     IPv4 주소 . . . . . . . . . : 192.168.239.1
     서브넷 마스크 . . . . . . . : 255.255.255.0
     
  ```

  







### (2) 확인

```sh
$ kubectl -n kafka get kafka my-cluster
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
my-cluster   3                        3                     True




$ kubectl -n kafka get kafka my-cluster -o yaml
...
status:
...
  - addresses:
    - host: my-cluster.kafka.localhost.192.168.31.1.nip.io
      port: 32100
    bootstrapServers: my-cluster.kafka.localhost.192.168.31.1.nip.io:32100
    name: external
    type: external
---



$ kubectl -n kafka get svc
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
my-cluster-kafka-0                    NodePort    10.107.81.9      <none>        9094:32000/TCP                        49s
my-cluster-kafka-1                    NodePort    10.108.122.107   <none>        9094:32001/TCP                        49s
my-cluster-kafka-2                    NodePort    10.100.130.247   <none>        9094:32002/TCP                        49s
my-cluster-kafka-bootstrap            ClusterIP   10.104.45.188    <none>        9091/TCP,9092/TCP,9093/TCP            60m
my-cluster-kafka-brokers              ClusterIP   None             <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   60m
my-cluster-kafka-external-bootstrap   NodePort    10.98.74.30      <none>        9094:32100/TCP                        49s
my-cluster-zookeeper-client           ClusterIP   10.98.160.11     <none>        2181/TCP                              61m
my-cluster-zookeeper-nodes            ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP            61m


## 정리하면...
## 외부에서 접근시 아래 주소로 cluster내부에 있는 kafka 에 접근 할 수 있다.
my-cluster.kafka.localhost.192.168.31.1.nip.io:32100
```



### (3) kafkacat 으로 확인

Local PC(Cluster 외부) 에서  kafka 접근 가능여부를 확인하기 위해 kafkacat 을 Local PC 에 설치하자.



#### 1) docker run

kafkacat 을 docker 로 설치한다.

```sh
# 실행
$ docker run --name kafkacat -d --user root confluentinc/cp-kafkacat:latest sleep 365d


# 확인
$ docker ps
```



#### 2) pub/sub 확인

```sh
$ docker exec -it kafkacat bash

export BROKERS=my-cluster.kafka.localhost.192.168.31.1.nip.io:32100
export KAFKAUSER=my-user
export PASSWORD=eGVNg7ZvPbi0
export TOPIC=my-topic
export GROUP=my-topic-group


## topic 리스트
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD -L
  
  
Metadata for all topics (from broker -1: sasl_plaintext://my-cluster.kafka.localhost.192.168.31.1.nip.io:32100/bootstrap):
 3 brokers:
  broker 0 at my-cluster.kafka.localhost.192.168.31.1.nip.io:32000
  broker 2 at my-cluster.kafka.localhost.192.168.31.1.nip.io:32002 (controller)
  broker 1 at my-cluster.kafka.localhost.192.168.31.1.nip.io:32001
 1 topics:
  topic "my-topic" with 3 partitions:
    partition 0, leader 2, replicas: 2,1,0, isrs: 1,2,0
    partition 1, leader 1, replicas: 1,0,2, isrs: 1,2,0
    partition 2, leader 0, replicas: 0,2,1, isrs: 1,2,0

# broker0, 1, 3 을 확인하자.


## consumer
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -C -o -5

<-- OK 


## producer : 입력모드
kafkacat -b $BROKERS \
  -X security.protocol=SASL_PLAINTEXT \
  -X sasl.mechanisms=SCRAM-SHA-512 \
  -X sasl.username=$KAFKAUSER \
  -X sasl.password=$PASSWORD \
  -t $TOPIC -P -X acks=1
  
<-- OK 


```





# 8. Monitoring[실습Skip]

모니터링이 필요할 경우 exporter 를 설치후 promtheus와 연동할 수 있다. 



![img](kafka.assets/1ztl7ii-FrK0GOL8mxwVFFQ.png)





## 1) metric

모니터링이 필요할 경우 아래와 같이 exporter 를 설치후 promtheus와 연동할 수 있다. 



- exporter 기능은 Strimzi에서 제공되는 통합 이미지 내에 포함되어 있음.   

  - strimzi/kafka:0.29.0-kafka-3.2.0    <-- strimzi-cluster-operator 에서 기본 이미지 설정가능
- kafka cluster 에서 위 yaml 파일만 수정후 apply 해도 추가가능.(kafka cluster 를 재설치 하지 않아도 됨.)
- prometheus / grafana 는 일반적인 내용과 동일함.
- grafana 에서는 strimzi dashboard 를 찾아서 import 한다.
- arsenal-cluster-kafka-exporter:9404 로 접근가능



### (1) my-cluter 에 exporter 생성

```sh
$ kubectl -n kafka edit kafka my-cluster
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: sa-cluster
  namespace: kafka-system
  ...
spec:
  entityOperator:
  kafka:
  zookeeper:
  
  ## exporter 추가
  kafkaExporter:
    groupRegex: .*
    topicRegex: .*
  ...    
---


```



- exporter pod 생성 여부 확인

```sh
# 확인
$ kubectl -n kafka get pod
NAME                                          READY   STATUS              RESTARTS      AGE
...
my-cluster-kafka-exporter-79b8c986f8-wg259    0/1     ContainerCreating   0             0s
...


## my-cluster-kafka-exporter 가 추가된다.


```





### (2) exporter service 생성

exporter service 는 자동으로 생성되지 않는다.

아래와 같이 수동으로 추가해야 한다.

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/monitoring/11.my-cluster-kafka-exporter-service.yaml
---
kind: Service
apiVersion: v1
metadata:
  name: my-cluster-kafka-exporter
  namespace: kafka
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9404
  selector:    
    app.kubernetes.io/instance: my-cluster
    app.kubernetes.io/managed-by: strimzi-cluster-operator
    app.kubernetes.io/name: kafka-exporter    
  type: ClusterIP



$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/11.my-cluster-kafka-exporter-service.yaml


```



- 확인

```
kkf get svc
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
...
my-cluster-kafka-exporter             ClusterIP   10.43.246.16    <none>        80/TCP                                4s
...
```





- exporter pod 에서 metric 정상 수집 여부 확인

```sh
$ kkf exec -it my-cluster-kafka-exporter-79b8c986f8-wg259 -- bash


$ curl localhost:9404/metrics
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.00037912
go_gc_duration_seconds{quantile="0.25"} 0.00037912
go_gc_duration_seconds{quantile="0.5"} 0.00037912
go_gc_duration_seconds{quantile="0.75"} 0.00037912
go_gc_duration_seconds{quantile="1"} 0.00037912
go_gc_duration_seconds_sum 0.00037912
go_gc_duration_seconds_count 1
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 19
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.17.1"} 1


$ curl my-cluster-kafka-exporter.kafka.svc/metrics
<-- ok
```





## 2) prometheus 



### (1) 권한부여

- openshift 에서 수행시 anyuid 권한이 필요하다.

```
# 권한부여시
oc adm policy add-scc-to-user    anyuid -z prometheus-server -n kafka

# 권한삭제시
oc adm policy remove-scc-from-user anyuid  -z prometheus-server -n kafka

```



### (2) helm deploy

```sh
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

$ helm fetch prometheus-community/prometheus

$ tar -zxvf prometheus-15.10.1.tgz



$ helm -n kafka list

$ helm -n kafka install prometheus prometheus-community/prometheus \
  --set alertmanager.enabled=false \
  --set configmapReload.prometheus.enabled=false \
  --set configmapReload.alertmanager.enabled=false \
  --set kubeStateMetrics.enabled=false \
  --set nodeExporter.enabled=false \
  --set server.enabled=true \
  --set server.image.repository=quay.io/prometheus/prometheus \
  --set server.namespaces[0]=kafka \
  --set server.ingress.enabled=false \
  --set server.persistentVolume.enabled=false \
  --set pushgateway.enabled=false \
  --dry-run=true > dry-run.yaml


## 확인
$ helm3 -n kafka ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
prometheus      kafka           1               2022-06-26 00:45:11.7836364 +0900 KST   deployed        prometheus-15.10.1      2.34.0
   

## 삭제
$ helm3 -n kafka delete prometheus 
```





### (3) prometheus svc 확인 

```sh
$ kkf get svc
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
...
prometheus-server                     ClusterIP   172.30.246.251   <none>        80/TCP                                68s

```





### (4) ingress

````sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/monitoring/21.prometheus-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: "prometheus.kafka.ktcloud.211.254.212.105.nip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-server
            port:
              number: 80



$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/21.prometheus-ingress.yaml
````

- 확인

![image-20220626124323035](kafka.assets/image-20220626124323035.png)



### (5) route

openshift 환경일때는 route 를 생성한다.

````sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/monitoring/22.prometheus-route.yaml
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: prometheus-route
  namespace: kafka-system
  labels:
    app: prometheus
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-15.8.4
    component: server
    heritage: Helm
    release: prometheus
spec:
  host: prometheus-kafka.apps.211-34-231-82.nip.io
  to:
    kind: Service
    name: prometheus-server
    weight: 100
  port:
    targetPort: http
    
$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/22.prometheus-route.yaml
````







### (6) configmap

exporter 주소를 추가해야 한다.

````sh
$ kubectl -n kafka edit configmap prometheus-server
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: prometheus-server
  namespace: kafka
  ...
data:
  ...
  prometheus.yml: |
    global:
      evaluation_interval: 1m
      scrape_interval: 1m
      scrape_timeout: 10s
    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts
    scrape_configs: 
    
    ## 아래 부분 추가
    - job_name: kafka-exporter
      metrics_path: /metrics
      scrape_interval: 10s
      scrape_timeout: 10s
      static_configs:
      - targets:
        - my-cluster-kafka-exporter.kafka.svc
     ...
````

- 추가후 prometheus server 재기동

```sh
$ kkf get pod
NAME                                          READY   STATUS    RESTARTS      AGE
...
prometheus-server-5dc67b6855-cdm54            1/1     Running   0             24m
...


$kkf delete pod prometheus-server-5dc67b6855-cdm54
pod "prometheus-server-5dc67b6855-cdm54" deleted


$kkf get pod
NAME                                          READY   STATUS    RESTARTS      AGE
my-cluster-kafka-exporter-79b8c986f8-wg259    1/1     Running   1 (60m ago)   60m
prometheus-server-5dc67b6855-67xts            0/1     Running   0             9s

```



- status / target 에서 아래와 같이 kafka-exporter 가 추가된것을 확인한다.

![image-20220626124700665](kafka.assets/image-20220626124700665.png)





## 3) Grafana deploy

- deploy

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/monitoring/31.grafana-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: kafka
  labels:
    app: strimzi
spec:
  replicas: 1
  selector:
    matchLabels:
      name: grafana
  template:
    metadata:
      labels:
        name: grafana
    spec:
      containers:
      - name: grafana
        image: docker.io/grafana/grafana:7.3.7
        ports:
        - name: grafana
          containerPort: 3000
          protocol: TCP
        volumeMounts:
        - name: grafana-data
          mountPath: /var/lib/grafana
        - name: grafana-logs
          mountPath: /var/log/grafana
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
      volumes:
      - name: grafana-data
        emptyDir: {}
      - name: grafana-logs
        emptyDir: {}
        
$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/31.grafana-deployment.yaml
deployment.apps/grafana created

```




- service

```sh
$ cd ~/githubrepo/ktds-edu2

$ cat ./kafka/strimzi/monitoring/32.grafana-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kafka
  labels:
    app: strimzi
spec:
  ports:
  - name: grafana
    port: 3000
    targetPort: 3000
    protocol: TCP
  selector:
    name: grafana
  type: ClusterIP

$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/32.grafana-svc.yaml
service/grafana created



```





- ingress

```sh
$ cat ./kafka/strimzi/monitoring/33.grafana-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: "grafana.kafka.ktcloud.211.254.212.105.nip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000


$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/33.grafana-ingress.yaml
ingress.networking.k8s.io/grafana-ingress created

```



- route
  - openshift 환경에서만 사용

```yaml
$ cat ./kafka/strimzi/monitoring/34.grafana-route.yaml
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: grafana-kafka-route
  namespace: kafka
  labels:
    app: strimzi
spec:
  host: grafana-kafka.apps.211-34-231-82.nip.io
  to:
    kind: Service
    name: grafana
    weight: 100
  port:
    targetPort: grafana
  wildcardPolicy: None
$ kubectl -n kafka apply -f ./kafka/strimzi/monitoring/34.grafana-route.yaml
```



- 로그인

기본 Grafana 사용자 이름과 암호는 모두 admin 이다.



 

## 4) Grafana Monitoring

### (1) Grafana 접속

http://grafana-kafka.apps.211-34-231-82.nip.io



### (2) promehteus 연동

- 메뉴 : Data Sources / Promehteus 
- URL : prometheus-server  입력



### (3) strimzi exporter dashboard import

- 메뉴: Dashboards / Manage
- import : 11285입력
- 참고링크 : https://grafana.com/grafana/dashboards/11285-strimzi-kafka-exporter



### (4) 확인

http://grafana.kafka.ktcloud.211.254.212.105.nip.io/d/jwPKIsniz/strimzi-kafka-exporter?orgId=1&refresh=5s&from=now-30m&to=now&var-consumergroup=edu-topic-group&var-topic=edu-topic-01



- 메뉴 위치 : Dashboards > Manage > Strimzi Kafka Exporter

![image-20220626111254872](kafka.assets/image-20220626111254872.png)











# 9. [실습] python

python 을 활용하여 kafka 연결을 시도해 보자. 

PRD 환경처럼 Cluster 내부에서 연결되는 Internal 환경과  DEV 환경처럼 개발자가 Local PC 에서 연결되는 External 환경을 각각 살펴보자.



## 1) Internal Access



### (1) 준비

#### Internal access 를 위한 Cluter python 실행

```sh
## cluster 에서 실행
# python deploy
$ kubectl -n kafka create deploy python --image=python:3.9 -- sleep 365d



# python pod 확인
$ kubectl -n kafka get pod
NAME                                         READY   STATUS    RESTARTS       AGE
...
python-fb57f7bd4-4w6pz                       1/1     Running   0              32s
...



# python pod 내 진입(bash 실행)
$ kubectl -n kafka exec -it deploy/python -- bash

```



#### python library install

kafka 에 접근하기 위해서 kafka-python 을 설치해야 한다.

```bash
$ pip install kafka-python

Collecting kafka-python
  Downloading kafka_python-2.0.2-py2.py3-none-any.whl (246 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 246.5/246.5 KB 3.9 MB/s eta 0:00:00
Installing collected packages: kafka-python
Successfully installed kafka-python-2.0.2
```



#### kafka host 확인

```sh

## internal 접근을 위한 host
bootstrap: my-cluster-kafka-bootstrap.kafka.svc:9092
broker0: my-cluster-kafka-0.my-cluster-kafka-brokers.kafka.svc:9092
broker2: my-cluster-kafka-2.my-cluster-kafka-brokers.kafka.svc:9092
broker1: my-cluster-kafka-1.my-cluster-kafka-brokers.kafka.svc:9092 

## 인증값
export KAFKAUSER=my-user
export PASSWORD=eGVNg7ZvPbi0
export TOPIC=my-topic
```





### (2) consumer

consumer 실행을 위해서 python cli 환경으로 들어가자.

```sh
$ python

Python 3.9.13 (main, May 28 2022, 13:56:03)
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>

```



internal 에서 접근시에는 인증서가 없는  9092 port 접근이므로 사용되는 protocol은 SASL_PLAINTEXT 이다.

```python
from kafka import KafkaConsumer

# 개인환경으로 변경
bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc:9092'
sasl_plain_password='eGVNg7ZvPbi0'

consumer = KafkaConsumer(bootstrap_servers=bootstrap_servers,
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password,    # 개인별 password 로 변경하자.
                        auto_offset_reset='earliest',
                        enable_auto_commit= True,
                        group_id='my-topic-group')



# my-user로 확인가능한 topic 목록들을 확인할 수 있다.
consumer.topics()

# 사용할 topic 지정(구독)
consumer.subscribe("my-topic")

# 구독 확인
consumer.subscription()


# 메세지 읽기
for message in consumer:
   print("topic=%s partition=%d offset=%d: key=%s value=%s" %
        (message.topic,
          message.partition,
          message.offset,
          message.key,
          message.value))



'''
topic=my-topic partition=0 offset=38: key=None value=b'{"eventName":"a","num":88,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
topic=my-topic partition=0 offset=39: key=None value=b'{"eventName":"a","num":90,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
topic=my-topic partition=0 offset=40: key=None value=b'{"eventName":"a","num":96,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
'''
```







### (3) producer

producer 실행을 위해서 별도의 terminal 을 실행한 후 python cli 환경으로 들어가자.

```sh
# python pod 내 진입(bash 실행)
$ kubectl -n kafka exec -it deploy/python -- bash


$ python

Python 3.9.13 (main, May 28 2022, 13:56:03)
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>

```





internal 에서 접근시에는 인증서가 없는  9092 port 접근이므로 사용되는 protocol은 SASL_PLAINTEXT 이다.

```python
from kafka import KafkaProducer
from time import sleep

# 개인환경으로 변경
bootstrap_servers='my-cluster-kafka-bootstrap.kafka.svc:9092'
sasl_plain_password='eGVNg7ZvPbi0'

producer = KafkaProducer(bootstrap_servers=bootstrap_servers,
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password)
    
producer.send('my-topic', b'python test1')
producer.send('my-topic', b'python test2')
producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % 1)


for i in range(10000):
    print(i)
    sleep(1)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)


```



- 대량 발송(성능테스트)

```python

# 만건 테스트
import time
start_time = time.time() # 시작시간
for i in range(10000):
    print(i)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)


end_time = time.time() # 종료시간
print("duration time :", end_time - start_time)  # 현재시각 - 시작시간 = 실행 시간
# duration time : 19.56832265853882


```



- 참고

```python

# 2만건 테스트
for i in range(10001, 20000):
    print(i)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)
    
```



- python 종료시 : Ctrl+D 





## 2) External Access



### (1) 준비

#### External access 를 위한 docker python 실행

```sh
## docker 실행
$ docker run --name python --user root --rm -d python:3.9 sleep 365d

# python 실행
$ docker exec -it python bash

```



#### python library install

python 을 이용해서 kafka 에 접근하기 위해서는 kafka 가아닌 kafka-python 을 설치해야 한다.

```bash
$ pip install kafka-python
```



#### kafka host 확인

```sh
## external 접근을 위한 host (nodeport 기준)
my-cluster.kafka.localhost.192.168.31.1.nip.io:32100

## 인증값
export KAFKAUSER=my-user
export PASSWORD=eGVNg7ZvPbi0
export TOPIC=my-topic
```





### (3) consumer

consumer 실행을 위해서 python cli 환경으로 들어가자.

```sh
$ python

Python 3.9.13 (main, May 28 2022, 13:56:03)
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>

```



internal 에서 접근시에는 인증서가 없는  9092 port 접근이므로 사용되는 protocol은 SASL_PLAINTEXT 이다.

```python
from kafka import KafkaConsumer

# 개인환경으로 변경
bootstrap_servers='my-cluster.kafka.localhost.192.168.31.1.nip.io:32100'
sasl_plain_password='eGVNg7ZvPbi0'

consumer = KafkaConsumer(bootstrap_servers=bootstrap_servers,
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password,
                        ssl_check_hostname=True,
                        auto_offset_reset='earliest',
                        enable_auto_commit= True,
                        group_id='my-topic-group')

# my-user로 확인가능한 topic 목록들을 확인할 수 있다.
consumer.topics()

# 사용할 topic 지정(구독)
consumer.subscribe("my-topic")

# 구독 확인
consumer.subscription()


# 메세지 읽기
for message in consumer:
   print("topic=%s partition=%d offset=%d: key=%s value=%s" %
        (message.topic,
          message.partition,
          message.offset,
          message.key,
          message.value))

'''
---
topic=my-topic partition=0 offset=38: key=None value=b'{"eventName":"a","num":88,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
topic=my-topic partition=0 offset=39: key=None value=b'{"eventName":"a","num":90,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
topic=my-topic partition=0 offset=40: key=None value=b'{"eventName":"a","num":96,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }'
'''
```







### (2) producer

Nodeport 설정에는 인증서가 불필요한 SASL_PLAINTEXT 이다.

```python
from kafka import KafkaProducer
from time import sleep

# 개인환경으로 변경
bootstrap_servers='my-cluster.kafka.localhost.192.168.31.1.nip.io:32100'
sasl_plain_password='eGVNg7ZvPbi0'

producer = KafkaProducer(bootstrap_servers=bootstrap_servers,
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        ssl_check_hostname=True,
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password)
                  
producer.send('my-topic', b'python test1')
producer.send('my-topic', b'python test2')
producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % 1)


for i in range(10000):
    print(i)
    sleep(1)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)

```



- 대량 발송(성능테스트)

```python
# 만건 테스트
import time
start_time = time.time() # 시작시간
for i in range(10000):
    print(i)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)


end_time = time.time() # 종료시간
print("duration time :", end_time - start_time)  # 현재시각 - 시작시간 = 실행 시간
# duration time : 20.996120929718018


```

- tps 비교
  - internal : 1.5k TPS
  - external : 1.7k TPS
- 결론
  - 일반적으로 External 이 Internal 보다 network 부하가 심해서 속도가 훨씬 느리다.
  - 하지만 우리가 테스트한 환경은 동일 PC 에서 실행하므로 속도가 거의 동일한점을 참고하자.



- 참고

```python
# 2만건 테스트
for i in range(10001, 20000):
    print(i)
    producer.send('my-topic', b'{"eventName":"a","num":%d,"title":"a", "writeId":"", "writeName": "", "writeDate":"" }' % i)
    
```



- python 종료시 : Ctrl+D 







### (4) Consumer Group 

#### 1) List 

namesapce 를 보내서 CG list 리턴

- test1

```python
from kafka.admin import KafkaAdminClient

# 개인환경으로 변경
bootstrap_servers='my-cluster.kafka.localhost.192.168.31.1.nip.io:32100'
sasl_plain_password='eGVNg7ZvPbi0'

admin_client = KafkaAdminClient(bootstrap_servers=bootstrap_servers, 
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password,
                        #client_id='test1'
                        )

list_cg = admin_client.list_consumer_groups()
print(type(list_cg))
print(list_cg )
# [('my-topic-group', 'consumer')]

```



#### 2) Describe

CG 명을 던져서 topicname, partition, current-offset 이 리턴되어야 한다.

참조: https://kafka-python.readthedocs.io/en/master/apidoc/KafkaAdminClient.html

참조: https://github.com/dpkp/kafka-python/issues/1798



- test1

```python
from kafka.admin import KafkaAdminClient

# 개인환경으로 변경
bootstrap_servers='my-cluster.kafka.localhost.192.168.31.1.nip.io:32100'
sasl_plain_password='eGVNg7ZvPbi0'

admin_client = KafkaAdminClient(bootstrap_servers=bootstrap_servers, 
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password,
                        #client_id='test1'
                        )

# 그룹명을 인수로 보낼때는 반드시 리스트[] 로 보내야 한다.
cg_desc = admin_client.describe_consumer_groups(['my-topic-group'])
print(type(cg_desc))
print(cg_desc)

'''
[
GroupInformation(
error_code=0, 
group='my-topic-group', 
state='Stable', 
protocol_type='consumer', 
protocol='range', 
members=[MemberInformation(member_id='kafka-python-2.0.2-06e95b4b-6f67-467d-ac8e-64c34710c5a2', 
client_id='kafka-python-2.0.2', 
client_host='/192.168.65.3', 
member_metadata=ConsumerProtocolMemberMetadata(version=0, subscription=['my-topic'], user_data=b''), 
member_assignment=ConsumerProtocolMemberAssignment(version=0, 
assignment=[(topic='my-topic', partitions=[0, 1, 2])], 
user_data=b''))], 
authorized_operations=None)
]
'''



# offset 정보
cg_offsets = admin_client.list_consumer_group_offsets('my-topic-group')
print(type(cg_offsets))
print(cg_offsets)

'''
{
TopicPartition(topic='my-topic', partition=0): OffsetAndMetadata(offset=13449, metadata=''), 
TopicPartition(topic='my-topic', partition=1): OffsetAndMetadata(offset=13534, metadata=''), 
TopicPartition(topic='my-topic', partition=2): OffsetAndMetadata(offset=13151, metadata='')
}
'''

```



### (5) kafka admin Client

- topic 생성시

```python
from kafka.admin import KafkaAdminClient, NewTopic

# 개인환경으로 변경
bootstrap_servers='my-cluster.kafka.localhost.192.168.31.1.nip.io:32100'
sasl_plain_password='eGVNg7ZvPbi0'

admin_client = KafkaAdminClient(bootstrap_servers=bootstrap_servers, 
                        security_protocol="SASL_PLAINTEXT",
                        sasl_mechanism='SCRAM-SHA-512',
                        sasl_plain_username='my-user',
                        sasl_plain_password=sasl_plain_password,
                        #client_id='test1'
                        )

topic_list = []
topic_list.append(NewTopic(name="example_topic", num_partitions=1, replication_factor=1))
admin_client.create_topics(new_topics=topic_list, validate_only=False)

'''
---
python kafkaAdminClient.py
---
'''
```









# 9.  [실습] Java - SCS

Spring Cloud Stream 를 이용한 샘플을 살펴보자.

- Spring Cloud Stream 특징

  - 어떤 메시지 플랫폼을 사용하는지 상관없이 추상화해서기능을 제공

  - 메시지 플랫폼으로 Rabbit MQ 나 Kafka 를 사용한다고 하더라도 비즈니스 로직을 별도로 구분할 필요 없음

  - Spring 에서 Configuration 설정을 알아서 설정함

  - 매우 심플한 사용



 

## 1) 개인당 Kafka Cluster 접속 권한 정보

 

Topic/Group/User 정보 대해서 아래와 같이 준비되어 있다.

```
# topic
edu-topic-01 ~ edu-topic-20
 
# group - 사용자가 consum 할때 선언함
edu-group-01 ~ edu-group-20
 
# user 는 공용
my-user
```

 

### (1) 접속권한

교육을 위해 한시적으로 접속권한 생성함

| **User** | **Pass**     | **비고**                          |
| -------- | ------------ | --------------------------------- |
| my-user  | pprOnk80CDfo | 2022년 7월에만  유지되고 삭제예정 |







### (2) 개인당 Topic 할당

| **Topic**    | **Group**    | **담당자** | **비고** |
| ------------ | ------------ | ---------- | -------- |
| edu-topic-01 | edu-group-01 | 송양종     |          |
| edu-topic-02 | edu-group-02 | 김성태     |          |
| edu-topic-03 | edu-group-03 | 김자영     |          |
| edu-topic-04 | edu-group-04 | 김유석     |          |
| edu-topic-05 | edu-group-05 |            |          |
| edu-topic-06 | edu-group-06 |            |          |
| edu-topic-07 | edu-group-07 |            |          |
| edu-topic-08 | edu-group-08 |            |          |
| edu-topic-09 | edu-group-09 |            |          |
| edu-topic-10 | edu-group-10 |            |          |
| edu-topic-11 | edu-group-11 |            |          |
| edu-topic-12 | edu-group-12 |            |          |
| edu-topic-13 | edu-group-13 |            |          |
| edu-topic-14 | edu-group-14 |            |          |
| edu-topic-15 | edu-group-15 |            |          |
| edu-topic-16 | edu-group-16 |            |          |
| edu-topic-17 | edu-group-17 |            |          |
| edu-topic-18 | edu-group-18 |            |          |
| edu-topic-19 | edu-group-19 |            |          |
| edu-topic-20 | edu-group-20 |            |          |







## 2) kafka-consumer



### (1) sample import



- Github 의 kafka-consumer repo 주소 확인

```
https://github.com/ssongman/kafka-consumer.git
```

복사하여 클립보드에 기억한다.



- STS 에서 import 
  - Package Explorer 에서 우클릭 이후 아래 메뉴 선택


```
1) import - git - Project from Git(with smart import)

2) Select Repository Source
   Clone 선택

3) Source Git Repository
   URI 에 위 주소 붙여넣기
   클립보드에 기억된 git 주소로 자동 셋팅된다.

4) Branch Selection
   main 선택

5) local Destination 에서 프로젝트 위치 지정

6) Import Projects
   Maven 확인 후 finish
```





### (2) 소스내 Topic/Group 수정

- application-local.yaml 에서 아래 내용 수정

```yaml
spring:
  cloud:
    stream:
      function:
        definition: boardCreate
      ...
      bindings:
        boardCreate-in-0:
          destination: edu-topic-01     # 본인의 토픽명으로 수정할것
          group: edu-group-01           # 본인의 그룹명으로 수정할것
```

 

### (3) Consumer 실행

```
Run As - Spring Boot App 실행
```



- Console log 확인

```
edu-topic-group: partitions assigned: [edu-topic-01-2, edu-topic-01-1, edu-topic-01-0]
```





## 3) kafka-producer

 

### 1) sample import

- Github 의 kafka-producer repo 주소 확인

```
https://github.com/ssongman/kafka-producer.git
```

복사하여 클립보드에 기억한다.



- STS 에서 import
  - Package Explorer 에서 우클릭 이후 아래 메뉴 선택

```
1) import - git - Project from Git(with smart import)

2) Select Repository Source
   Clone 선택

3) Source Git Repository
   URI 에 위 주소 붙여넣기
   클립보드에 기억된 git 주소로 자동 셋팅된다.

4) Branch Selection
   main 선택

5) local Destination 에서 프로젝트 위치 지정

6) Import Projects
   Maven 확인 후 finish

```



### 2) 소스내 Topic 수정

src/main/resources/config/application-local.yaml 에서 아래 내용 수정

```yaml
spring: 
  cloud:
    stream:
      bindings:
        boardCreate-out-0:
          destination: edu-topic-01   # 본인의 토픽명으로 수정할것
```

 

### 3) producer 실행

```
Run As - Spring Boot App 실행
```

 

### 4) pub test (create api call)

postman 이나 curl 을 통해서 아래와 같이 테스트를 수행한다.

```
curl -X POST http://localhost:8081/create \
  -H "Content-Type: application/json" \
  -d '{  
  "eventName": "boardCreate",
  "num": 2,
  "title": "test insert-1",
  "contents": "test insert-1",
  "writeId": "user1",
  "writeName": "user1",
  "writeDate": "2021-09-11T16:20:41"
}'
 
 
## while - sleep 1
while true; do curl -X POST http://localhost:8081/create \
  -H "Content-Type: application/json" \
  -d '{  
  "eventName": "boardCreate",
  "num": 2,
  "title": "test insert-1",
  "contents": "test insert-1",
  "writeId": "user1",
  "writeName": "user1",
  "writeDate": "2021-09-11T16:20:41"
}'; sleep 1; echo; done
 
 
```



 

# 10. [데모]Consumer Rebalancing Round Test

 

일반적으로 Consumer group의 멤버 구성에 변화가 생기면 리소스의 재분배가 필요한데 이를 **Rebalancing Round** 라고 한다.

이 RR 이 발생하는 동안은 어떤 컨슈머들도 정상적인 데이터 처리를 하지 못한다는 문제를 가지는데 카프카에서는 이를 **Stop The World** 라는 용어로 부른다. 

이는 천 개의 Connect Task 가 그룹에 존재했을 때 그 천 개의 프로세스가 전부 정상 동작하지 못하게 되는 상황을 맞이한다. 

또한 이렇게 리벨런싱이 초래한 Stop The World 는 일반적인 하드웨어나 네트워크 손실 문제로 발생한 일시적인 client fail 과 더불어, scale up / down 의 상황이나 계획적인 클라이언트 start / stop / restart 의 상황에서 전부 발생할 수 있다. 

그럼 Consumer 갯수에 따른 Partition 매핑 관계와 Consumer Rebalancing 현상에 대해서 확인해 보자.



## 1) 시나리오1 

### (1) 설명 

- Partition 3개인 Topic 에서 Consumer 갯수에 따른 변화사항 보기

- 원할한 테스트를 위해서 python console 로 수행 한다.

### (2) Consumer 환경 

- Consumer1:  python

- Consumer2:  python

- Consumer3:  python

### (3) 테스트 절차 

- Consumer 1개로 처리 되는 현상 보기 
  - 예상되는 결과: 1개의 Consumer 가 3개의 partition 을 모두 처리한다.

- Consumer 2일때 
  - 예상되는 결과: 1번 Consumer 가 partition 1개를 , 2번 Consumer 가 partition 2개 를 처리한다.·

- Consumer  3일때 
  - 예상된는 결과: Consumer 와 partition 이 1:1 매핑되어 처리된다. 가장 이상적인 구조이다.

 

## 2) 시나리오2

### (1) 설명 

- - 위 시나리오1번 에서 Consumer 종류를 다양하게 수행한다.

### (2) Consumer 환경     
- Consumer1:  Spring boot
- Consumer2: python
- Consumer3: python

### (3) 테스트 절차
- 위와 동일





# 11. Strimzi Clean up



```sh

# client tool clean up
kubectl -n kafka delete deploy kafkacat
kubectl -n kafka delete deploy python

# kafka resource clean up
kubectl -n kafka delete kafkauser my-user
kubectl -n kafka delete kafkatopic my-topic
kubectl -n kafka delete kafka my-cluster

# strimzi clean up
$ cd ~/githubrepo/ktds-edu2
$ kubectl -n kafka delete -f ./kafka/strimzi/install/cluster-operator

# kafka namespace clean up
kubectl delete namespace kafka


```

