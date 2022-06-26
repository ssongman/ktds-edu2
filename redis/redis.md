


# Redis Cluster Setting



# 1. 개요



## 1.1 redis 와 redis-cluster

Redis는 애플리케이션을 위한 확장 가능한 오픈 소스 분산 인메모리 캐시이다. 문자열, 해시, 목록, 집합 및 정렬된 집합의 형태로 데이터를 저장하고 제공하는 데 사용할 수 있다.

일단 Redis 와 Redis-cluster 와의 차이점은 아래와 같다.

| Redis                                                  | Redis Cluster                                                |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| 여러 데이터베이스 지원                                 | 한개의 데이터베이스 지원                                     |
| Single write point (single master)                     | Multiple write points (multiple masters)                     |
| ![redis-topology.png](redis.assets/redis-topology.png) | ![redis-cluster-topology.png](redis.assets/redis-cluster-topology.png) |







# 2. Redis Cluster Install Using Helm

참조link : https://github.com/bitnami/charts/tree/master/bitnami/redis-cluster



## 1) helm chart download

helm chart 를download 해보자.



### (1) Repo add

redis-cluster chart 를 가지고 있는 bitnami repogistory 를  helm repo 에 추가한다.

```sh
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```



### (2) Chart Search

추가된 bitnami repo에서 redis-cluster 를 찾는다.

```sh
$ helm search repo redis
bitnami/redis                                   16.13.0         6.2.7           Redis(R) is an open source, advanced key-value ...
bitnami/redis-cluster                           7.6.3           6.2.7           Redis(R) is an open source, scalable, distribut...
prometheus-community/prometheus-redis-exporter  4.8.0           1.27.0          Prometheus exporter for Redis metrics

```

우리가 사용할 redis-cluster 버젼은 chart version 7.6.3( app version: 6.2.7) 이다.



### (3) Chart Fetch

helm chart 를 fetch 받는다.

```sh
# chart 를 저장할 적당한 위치로 이동
$ cd ~/song/helm3/charts

$ helm fetch bitnami/redis-cluster

$ ls
redis-cluster-7.6.3.tgz

$ tar -xzvf redis-cluster-7.6.3.tgz
...

$ cd redis-cluster

$ ls -ltr
drwxr-xr-x 5 root root  4096 Jun 26 05:37 ./
drwxr-xr-x 4 root root  4096 Jun 26 05:37 ../
-rw-r--r-- 1 root root   220 Jun 10 16:31 Chart.lock
drwxr-xr-x 3 root root  4096 Jun 26 05:37 charts/
-rw-r--r-- 1 root root   761 Jun 10 16:31 Chart.yaml
-rw-r--r-- 1 root root   333 Jun 10 16:31 .helmignore
drwxr-xr-x 2 root root  4096 Jun 26 05:37 img/
-rw-r--r-- 1 root root 67832 Jun 10 16:31 README.md
drwxr-xr-x 2 root root  4096 Jun 26 05:37 templates/
-rw-r--r-- 1 root root 39649 Jun 10 16:31 values.yaml

```



### (4) namespace 생성



```sh

$ kubectl create ns redis-system

```







## 2) install - without pv



### (1) emptyDir 설정

chart 의 기본은 pv/pvc 를 참조하도록 설정되어 있다.

아직 pv/pvc 가 준비되어 있지 않다면 emptydir 로 설정한후 install 을 시도해야 한다.  그렇지 않으면 pvc 를 찾지못해 오류 발생한다.

아래 chart 의  파일을 찾아서 일부 내용을 변경해야 한다.

```sh
$ cd ~/song/helm/charts/redis-cluster/templates

$ ll
-rw-r--r-- 1 root root 90053 Jun 10 16:31 configmap.yaml
-rw-r--r-- 1 root root   117 Jun 10 16:31 extra-list.yaml
-rw-r--r-- 1 root root   903 Jun 10 16:31 headless-svc.yaml
-rw-r--r-- 1 root root  8774 Jun 10 16:31 _helpers.tpl
-rw-r--r-- 1 root root  2599 Jun 10 16:31 metrics-prometheus.yaml
-rw-r--r-- 1 root root  1520 Jun 10 16:31 metrics-svc.yaml
-rw-r--r-- 1 root root  2424 Jun 10 16:31 networkpolicy.yaml
-rw-r--r-- 1 root root  6195 Jun 10 16:31 NOTES.txt
-rw-r--r-- 1 root root   921 Jun 10 16:31 poddisruptionbudget.yaml
-rw-r--r-- 1 root root  1223 Jun 10 16:31 prometheusrule.yaml
-rw-r--r-- 1 root root  1508 Jun 10 16:31 psp.yaml
-rw-r--r-- 1 root root   833 Jun 10 16:31 redis-rolebinding.yaml
-rw-r--r-- 1 root root  1082 Jun 10 16:31 redis-role.yaml
-rw-r--r-- 1 root root   954 Jun 10 16:31 redis-serviceaccount.yaml
-rw-r--r-- 1 root root 22299 Jun 10 16:31 redis-statefulset.yaml          <--- 변경파일
-rw-r--r-- 1 root root  2568 Jun 10 16:31 redis-svc.yaml
-rw-r--r-- 1 root root  3279 Jun 10 16:31 scripts-configmap.yaml
-rw-r--r-- 1 root root   699 Jun 10 16:31 secret.yaml
-rw-r--r-- 1 root root  2055 Jun 10 16:31 svc-cluster-external-access.yaml
-rw-r--r-- 1 root root  1511 Jun 10 16:31 tls-secret.yaml
-rw-r--r-- 1 root root 16247 Jun 10 16:31 update-cluster.yaml

```



- 변경내용 정리

해당 파일 내의 volumeClaimTemplates 부분을 삭제하고 volumes 에 아래와 같이 emptyDir 내용을 추가해야 한다.

        - name: redis-data
          emptyDir: {}



- 변경전

```yaml

      volumes:
        - name: scripts
          configMap:
            name: {{ include "common.names.fullname" . }}-scripts
            defaultMode: 0755
        {{- if .Values.usePasswordFile }}
        - name: redis-password
          secret:
            secretName: {{ include "redis-cluster.secretName" . }}
            items:
              - key: {{ include "redis-cluster.secretPasswordKey" . }}
                path: redis-password
        {{- end }}
        - name: default-config
          configMap:
            name: {{ include "common.names.fullname" . }}-default
        {{- if .Values.sysctlImage.mountHostSys }}
        - name: host-sys
          hostPath:
            path: /sys
        {{- end }}
        - name: redis-tmp-conf
          emptyDir: {}
        {{- if .Values.redis.extraVolumes }}
        {{- include "common.tplvalues.render" ( dict "value" .Values.redis.extraVolumes "context" $ ) | nindent 8 }}
        {{- end }}
        {{- if .Values.tls.enabled }}
        - name: redis-certificates
          secret:
            secretName: {{ include "redis-cluster.tlsSecretName" . }}
            defaultMode: 256
        {{- end }}
  volumeClaimTemplates:
    - metadata:
        name: redis-data
        labels: {{- include "common.labels.matchLabels" . | nindent 10 }}
        {{- if .Values.persistence.annotations }}
        annotations: {{- include "common.tplvalues.render" (dict "value" .Values.persistence.annotations "context" $) | nindent 10 }}
        {{- end }}
      spec:
        accessModes:
        {{- range .Values.persistence.accessModes }}
          - {{ . | quote }}
        {{- end }}
        resources:
          requests:
            storage: {{ .Values.persistence.size | quote }}
        {{- include "common.storage.class" (dict "persistence" .Values.persistence "global" .Values.global) | nindent 8 }}
        {{- if or .Values.persistence.matchLabels .Values.persistence.matchExpressions }}
        selector:
        {{- if .Values.persistence.matchLabels }}
          matchLabels:
          {{- toYaml .Values.persistence.matchLabels | nindent 12 }}
        {{- end -}}
        {{- if .Values.persistence.matchExpressions }}
          matchExpressions:
          {{- toYaml .Values.persistence.matchExpressions | nindent 12 }}
        {{- end -}}
        {{- end }}
{{- end }}

```



- 변경후

```yaml
      volumes:
        - name: scripts
          configMap:
            name: {{ include "common.names.fullname" . }}-scripts
            defaultMode: 0755
        {{- if .Values.usePasswordFile }}
        - name: redis-password
          secret:
            secretName: {{ include "redis-cluster.secretName" . }}
            items:
              - key: {{ include "redis-cluster.secretPasswordKey" . }}
                path: redis-password
        {{- end }}
        - name: default-config
          configMap:
            name: {{ include "common.names.fullname" . }}-default
        {{- if .Values.sysctlImage.mountHostSys }}
        - name: host-sys
          hostPath:
            path: /sys
        {{- end }}
        - name: redis-tmp-conf
          emptyDir: {}
        {{- if .Values.redis.extraVolumes }}
        {{- include "common.tplvalues.render" ( dict "value" .Values.redis.extraVolumes "context" $ ) | nindent 8 }}
        {{- end }}
        {{- if .Values.tls.enabled }}
        - name: redis-certificates
          secret:
            secretName: {{ include "redis-cluster.tlsSecretName" . }}
            defaultMode: 256
        {{- end }}
        - name: redis-data                     <--- 이부분 으로 대체
          emptyDir: {}
{{- end }}
```





### (2) helm install

```sh
$ cd  ~/song/helm/charts/redis-cluster

## dry-run 으로 실행
$ helm -n redis-system install my-release . \
    --set image.registry=docker.io \
    --set cluster.nodes=6 \
    --set cluster.replicas=1 \
    --set password=new1234 \
    --debug --dry-run=true > dry-run_1.yaml


## 기본값으로 실행
$ helm -n redis-system install my-release . \
    --set image.registry=docker.io \
    --set cluster.nodes=6 \
    --set cluster.replicas=1 \
    --set password=new1234


To get your password run:
    export REDIS_PASSWORD=$(kubectl get secret --namespace "redis-system" my-release-redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

You have deployed a Redis&reg; Cluster accessible only from within you Kubernetes Cluster.INFO: The Job to create the cluster will be created.To connect to your Redis&reg; cluster:

1. Run a Redis&reg; pod that you can use as a client:
kubectl run --namespace redis-system my-release-redis-cluster-client --rm --tty -i --restart='Never' \
 --env REDIS_PASSWORD=$REDIS_PASSWORD \
--image docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 -- bash

2. Connect using the Redis&reg; CLI:

redis-cli -c -h my-release-redis-cluster -a $REDIS_PASSWORD




## 확인
$ helm -n redis-system ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
my-release      redis-system    1               2022-06-26 05:45:14.961024747 +0000 UTC deployed        redis-cluster-7.6.3     6.2.7     




$ helm -n redis-system status my-release
NAME: my-release
LAST DEPLOYED: Sun Jun 26 05:45:14 2022
NAMESPACE: redis-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: redis-cluster
CHART VERSION: 7.6.3
APP VERSION: 6.2.7** Please be patient while the chart is being deployed **


To get your password run:
    export REDIS_PASSWORD=$(kubectl get secret --namespace "redis-system" my-release-redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

You have deployed a Redis&reg; Cluster accessible only from within you Kubernetes Cluster.INFO: The Job to create the cluster will be created.To connect to your Redis&reg; cluster:

1. Run a Redis&reg; pod that you can use as a client:
kubectl run --namespace redis-system my-release-redis-cluster-client --rm --tty -i --restart='Never' \
 --env REDIS_PASSWORD=$REDIS_PASSWORD \
--image docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 -- bash

2. Connect using the Redis&reg; CLI:

redis-cli -c -h my-release-redis-cluster -a $REDIS_PASSWORD

```





## 3) Redis pod/svc 확인

먼저 아래와 같이 동일한 Namespace 에 redis-client 를 실행한다.

```sh
## redis cluster 를 구성하고 있는 pod 를 조회
$ kubectl -n redis-system get pod -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
my-release-redis-cluster-0      1/1     Running   0          12m     10.42.1.108   master02   <none>           <none>
my-release-redis-cluster-1      1/1     Running   0          12m     10.42.4.131   worker02   <none>           <none>
my-release-redis-cluster-2      1/1     Running   0          12m     10.42.2.160   master03   <none>           <none>
my-release-redis-cluster-3      1/1     Running   0          12m     10.42.5.114   worker03   <none>           <none>
my-release-redis-cluster-4      1/1     Running   0          12m     10.42.0.121   master01   <none>           <none>
my-release-redis-cluster-5      1/1     Running   0          12m     10.42.3.126   worker01   <none>           <none>
...



$ kubectl -n redis-system get svc
NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
my-release-redis-cluster            ClusterIP   10.43.35.97   <none>        6379/TCP             16m
my-release-redis-cluster-headless   ClusterIP   None          <none>        6379/TCP,16379/TCP   16m


```







## 4) CleanUp & Update

### (1) delete

helm delete 명령을 이용하면 helm chart 로 설치된 모든 리소스가 한꺼번에 삭제된다.

```sh

## 삭제하기
$ helm3 -n redis-system delete my-release

```



### (2) update 

node 추가와 같은 helm 기반 update 가 필요할때는 아래와 같은 방식으로 update 를 수행한다.

```sh

## update sample 1
$ helm3 upgrade --timeout 600s my-release \
    --set "password=${REDIS_PASSWORD},cluster.nodes=7,cluster.update.addNodes=true,cluster.update.currentNumberOfNodes=6" bitnami/redis-cluster

## update sample 2
$ helm upgrade <release> \
  --set "password=${REDIS_PASSWORD}
  --set cluster.externalAccess.enabled=true
  --set cluster.externalAccess.service.type=LoadBalancer
  --set cluster.externalAccess.service.loadBalancerIP[0]=<loadBalancerip-0>
  --set cluster.externalAccess.service.loadBalancerIP[1]=<loadbalanacerip-1>
  --set cluster.externalAccess.service.loadBalancerIP[2]=<loadbalancerip-2>
  --set cluster.externalAccess.service.loadBalancerIP[3]=<loadbalancerip-3>
  --set cluster.externalAccess.service.loadBalancerIP[4]=<loadbalancerip-4>
  --set cluster.externalAccess.service.loadBalancerIP[5]=<loadbalancerip-5>
  --set cluster.externalAccess.service.loadBalancerIP[6]=
  --set cluster.nodes=7
  --set cluster.init=false bitnami/redis-cluster



```













# 3. Internal Access

redis client를 동 cluster 에 실행하여 접근하는 방법을 알아보자.



## 1) Internal Access





### (1) Redis-clinet 실행

먼저 아래와 같이 동일한 Namespace 에 redis-client 를 실행한다.

```sh
## redis-client 용도로 deployment 를 실행한다.
$ kubectl -n redis-system create deploy redis-client --image=docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 -- sleep 365d
deployment.apps/redis-client created


## redis client pod 확인
$ kubectl -n redis-system get pod
NAME                            READY   STATUS    RESTARTS   AGE
redis-client-7cdd56bb6c-njjls   1/1     Running   0          5s     <--- redis client pod


## redis-client 로 접근한다.
## okd web console 에서 해당 pod 의 terminal 로 접근해도 된다.
$ kubectl -n redis-system exec -it deploy/redis-client -- bash

```



### (2) Redis-cluster 상태 확인

```sh

## service 명으로 cluster mode 접근
$ redis-cli -h my-release-redis-cluster -c -a new1234

## cluster node 를 확인
my-release-redis-cluster:6379> cluster nodes
e24b97c55fe808bb8cec4d3c84dded7b8b997fdc 10.42.3.126:6379@16379 slave 414a8ae9ed8d19b0c2beecdebead7c3149cf0aa2 0 1656222687801 2 connected
414a8ae9ed8d19b0c2beecdebead7c3149cf0aa2 10.42.4.131:6379@16379 myself,master - 0 1656222686000 2 connected 5461-10922
b310995bc9db36ebe51c1f25b09d01885a66cc86 10.42.2.160:6379@16379 master - 0 1656222685794 3 connected 10923-16383
506d16c194f3582ecf290ecf83602f1cef672dae 10.42.1.108:6379@16379 master - 0 1656222687000 1 connected 0-5460
bdff18b3bacefcd328c33fb37e905db8dc7486cc 10.42.0.121:6379@16379 slave 506d16c194f3582ecf290ecf83602f1cef672dae 0 1656222688000 1 connected
3008b31ef8be6e456112242e2a8c92582f2536c7 10.42.5.114:6379@16379 slave b310995bc9db36ebe51c1f25b09d01885a66cc86 0 1656222688806 3 connected
## master 3개, slave가 3개 사용하는 모습을 볼 수가 있다.


## cluster info 확인
my-release-redis-cluster:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:2
cluster_stats_messages_ping_sent:396
cluster_stats_messages_pong_sent:366
cluster_stats_messages_meet_sent:1
cluster_stats_messages_sent:763
cluster_stats_messages_ping_received:366
cluster_stats_messages_pong_received:397
cluster_stats_messages_received:763
## cluster state 가 OK 인 것을 확인할 수 있다.

```



### (3) set / get 확인

```sh

## service 명으로 cluster mode 접근
$ redis-cli -h my-release-redis-cluster -c -a new1234



## set 명령 수행
my-release-redis-cluster:6379> set a 1
OK
my-release-redis-cluster:6379> set b 2
-> Redirected to slot [3300] located at 10.42.1.108:6379
OK
10.42.1.108:6379> set c 3
-> Redirected to slot [7365] located at 10.42.4.131:6379
OK
10.42.4.131:6379> set d 4
-> Redirected to slot [11298] located at 10.42.2.160:6379
OK
10.42.2.160:6379> set e 5
OK
10.42.2.160:6379> set f 6
-> Redirected to slot [3168] located at 10.42.1.108:6379
OK
10.42.1.108:6379> set g 7
-> Redirected to slot [7233] located at 10.42.4.131:6379
OK
10.42.4.131:6379> set h 8
-> Redirected to slot [11694] located at 10.42.2.160:6379
OK
10.42.2.160:6379> set i 9
OK
10.42.2.160:6379> set j 10
-> Redirected to slot [3564] located at 10.42.1.108:6379
OK

## Set 명령수행시 master node 를 변경하면서 set 하는 모습을 확인할 수 있다.



# get 명령 수행
my-release-redis-cluster:6379> get a
"1"
my-release-redis-cluster:6379> get b
-> Redirected to slot [3300] located at 10.42.1.108:6379
"2"
10.42.1.108:6379> get c
-> Redirected to slot [7365] located at 10.42.4.131:6379
"3"
10.42.4.131:6379> get d
-> Redirected to slot [11298] located at 10.42.2.160:6379
"4"
10.42.2.160:6379> get e
"5"
10.42.2.160:6379> get f
-> Redirected to slot [3168] located at 10.42.1.108:6379
"6"
10.42.1.108:6379> get g
-> Redirected to slot [7233] located at 10.42.4.131:6379
"7"
10.42.4.131:6379> get h
-> Redirected to slot [11694] located at 10.42.2.160:6379
"8"
10.42.2.160:6379> get i
"9"
10.42.2.160:6379> get j
-> Redirected to slot [3564] located at 10.42.1.108:6379
"10"

## get 명령을 실행하면 해당 데이터가 존재하는 master pod 로 redirectred 되는 것을 확인할 수 있다.



```













# 4. External Access

External (Cluster 외부) 에서 access 하기 위해서 node port 를 이용할 것이다.

하지만 redis-cluster 는 원격지에서 접속시 cluster node 들에 대한 DNS 를 지원하지 않는다.

그러므로 redis-cluster 가 아닌 redis 로 설치 하여 테스트를 진행한다.



## 1) Redis(Single Master) Install



### (1)   redis cluster 초기화

```sh
# redis cluster 초기화
$ helm -n redis-system delete my-release
```



### (2)  Redis Install(Single Master)



#### helm search

추가된 bitnami repo에서 redis-cluster 를 찾는다.

```sh
$ helm search repo redis
bitnami/redis                                   16.13.0         6.2.7           Redis(R) is an open source, advanced key-value ...
bitnami/redis-cluster                           7.6.3           6.2.7           Redis(R) is an open source, scalable, distribut...
prometheus-community/prometheus-redis-exporter  4.8.0           1.27.0          Prometheus exporter for Redis metrics

```

bitnami/redis



#### Chart Fetch

helm chart 를 fetch 받는다.

```sh
# chart 를 저장할 적당한 위치로 이동
$ cd ~/song/helm/charts

$ helm fetch bitnami/redis

$ ls
redis-16.13.0.tgz

$ tar -xzvf redis-16.13.0.tgz
...

$ cd redis

$ ls -ltr
-rw-r--r-- 1 root root    220 Jun 24 17:57 Chart.lock
drwxr-xr-x 3 root root   4096 Jun 26 07:15 charts/
-rw-r--r-- 1 root root    773 Jun 24 17:57 Chart.yaml
-rw-r--r-- 1 root root    333 Jun 24 17:57 .helmignore
drwxr-xr-x 2 root root   4096 Jun 26 07:15 img/
-rw-r--r-- 1 root root 100896 Jun 24 17:57 README.md
drwxr-xr-x 5 root root   4096 Jun 26 07:15 templates/
-rw-r--r-- 1 root root   4483 Jun 24 17:57 values.schema.json
-rw-r--r-- 1 root root  68558 Jun 24 17:57 values.yaml

```



#### helm install

```sh

$ cd ~/song/helm/charts/redis   

# helm install
# node 2, replicas 1 이므로 Master / Slave 한개씩 사용됨
$ helm -n redis-system install my-release . \
    --set global.redis.password=new1234 \
    --set image.registry=docker.io \
    --set master.persistence.enabled=false \
    --set master.service.type=NodePort \
    --set master.service.nodePorts.redis=32200 \
    --set replica.replicaCount=3 \
    --set replica.persistence.enabled=false \
    --set replica.service.type=NodePort \
    --set replica.service.nodePorts.redis=32210

##
my-release-redis-master.redis-system.svc.cluster.local for read/write operations (port 6379)
my-release-redis-replicas.redis-system.svc.cluster.local for read-only operations (port 6379)



$ helm -n redis-system ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
my-release      redis-system    1               2022-06-26 06:59:30.08278938 +0000 UTC  deployed        redis-cluster-7.6.3     6.2.7


# 삭제시
$ helm -n redis-system delete my-release

```





### (3) pod / svc 확인

```sh
$ krs get pod
NAME                            READY   STATUS    RESTARTS   AGE
my-release-redis-master-0       1/1     Running   0          57s
my-release-redis-replicas-0     1/1     Running   0          57s
my-release-redis-replicas-1     1/1     Running   0          27s
my-release-redis-replicas-2     0/1     Running   0          2s


$ krs get svc
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-release-redis-headless   ClusterIP   None            <none>        6379/TCP         27s
my-release-redis-master     NodePort    10.43.203.127   <none>        6379:32200/TCP   27s
my-release-redis-replicas   NodePort    10.43.44.77     <none>        6379:32210/TCP   27s


```



### (4) redis client 확인

#### docker redis client

local pc 에서 access 테스트를 위해 docker redis client 를 설치하자.

```sh
## redis-client 용도로 docker client 를 실행한다.
$ docker run --name redis-client -d --rm --user root docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 sleep 365d

## docker 내에 진입후
$ docker exec -it redis-client bash

## service 명으로 cluster mode 접근
$ redis-cli -h 211.254.212.105 -c -a new1234 -p 32200

## service 명으로 none cluster mode 접근
$ redis-cli -h 211.254.212.105 -a new1234 -p 32200

```



#### set/get 확인

```

211.254.212.105:32200> set a 1
OK
211.254.212.105:32200> set b 2
OK
211.254.212.105:32200> set c 3
OK
211.254.212.105:32200> get a
"1"
211.254.212.105:32200> get b
"2"
211.254.212.105:32200> get c

```





## 9) local redis client set

local pc 에서 access 테스트를 위해 docker redis client 를 설치하자.

```sh

## redis-client 용도로 docker client 를 실행한다.
$ docker run --name redis-client -d --rm --user root docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 sleep 365d

## docker 내에 진입후
$ docker exec -it redis-client bash


```











# 5. ACL

Redis 6.0 이상부터는 계정별 access 수준을 정의할 수 있다.  

이러한 ACL 기능을 이용해서 아래와 같은 계정을 관리 할 수 있다.

- 읽기전용 계정생성도 가능

- 특정 프리픽스로 시작하는 Key 만 access 가능하도록 하는 계정 생성



## 1) ACL 기본명령



```sh
$ redis-cli -h 211.254.212.105 -c -a new1234 -p 32200

# 계정 목록
211.254.212.105:32200> acl list
1) "user default on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"


# 계정 추가
211.254.212.105:32200> acl setuser supersong on >new1234 allcommands allkeys
OK
211.254.212.105:32200> acl setuser tempsong on >new1234 allcommands allkeys
OK


211.254.212.105:32200> acl list
1) "user default on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
2) "user supersong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
3) "user tempsong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"


# 계정 전환
211.254.212.105:32200> acl whoami
"default"

211.254.212.105:32200> auth supersong new1234
OK
211.254.212.105:32200> acl whoami
"supersong"

211.254.212.105:32200> auth default new1234
OK

# 계정 삭제
211.254.212.105:32200> acl deluser tempsong
(integer) 1

```





## 2)읽기전용 계정 생성

- 읽기전용 계정 테스트

```sh

# 계정생성
211.254.212.105:32200> acl setuser readonlysong on >new1234 allcommands allkeys -set +get
OK

211.254.212.105:32200> acl list
1) "user default on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
2) "user readonlysong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all -set"
3) "user supersong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"


# 읽기는 가능
127.0.0.1:6379> get a
"1"

# 쓰기는 불가능
211.254.212.105:32200> set a 1
(error) NOPERM this user has no permissions to run the 'set' command or its subcommand

```





## 3) 특정 key만 접근 허용

- song으로 로그인 하면 song으로 시작하는 key 만 get/set 가능하도록 설정

```sh
# song 으로 시작하는 key 만 접근가능하도록 설정


211.254.212.105:32200> acl setuser song on >new1234 allcommands allkeys
OK
211.254.212.105:32200> acl list
1) "user default on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
2) "user readonlysong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all -set"
3) "user song on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
4) "user supersong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"


211.254.212.105:32200> acl setuser song resetkeys ~song*
OK


211.254.212.105:32200> acl list
1) "user default on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"
2) "user readonlysong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all -set"
3) "user song on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~song* &* +@all"
4) "user supersong on #65fd3b5c243ea857f91daef8e3d5c203fa045f33e034861998b9d74cc42ceb24 ~* &* +@all"


211.254.212.105:32200> auth song new1234
OK

211.254.212.105:32200> acl whoami
"song"


# set 명령 테스트
211.254.212.105:32200> set a 1
(error) NOPERM this user has no permissions to access one of the keys used as arguments

211.254.212.105:32200> set song_a 1
OK

# get 명령 테스트
211.254.212.105:32200> get a
(error) NOPERM this user has no permissions to access one of the keys used as arguments


211.254.212.105:32200> get song_a
"1"


```







# 6. redis-ui(p3x)



```

https://www.electronjs.org/apps/p3x-redis-ui

https://github.com/patrikx3/redis-ui/blob/master/k8s/manifests/service.yaml

```





# 7. Java redis test



## 1) Jedis vs Lettuce



Java 의 Redis Client 는 크게 두 가지가 있습니다.

Jedis 와 Lettuce 인데요.

원래 Jedis 를 많이 사용했으나 여러 가지 단점 (멀티 쓰레드 불안정, Pool 한계 등등..) 과 Lettuce 의 장점 (Netty 기반이라 비동기 지원 가능) 때문에 Lettuce 로 추세가 넘어가고 있었습니다.

그러다 결국 Spring Boot 2.0 부터 Jedis 가 기본 클라이언트에서 deprecated 되고 Lettuce 가 탑재되었습니다.







pom.xml

```xml
...
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
...
```







```
spring:
  redis:
    host: localhost
    port: 6379
```





```java
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private int port;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }
}

```
















# 9. python test - 작성중


- pod로 실행
```
oc -n redis-system run pythonfortest --image=ktis-bastion01.container.ipc.kt.com:5000/admin/python:3.7 -- sleep 365d
```

- deploy로 실행
```
oc -n redis-system create deploy pythonfortest --image=ktis-bastion01.container.ipc.kt.com:5000/admin/python:3.7 -- sleep 365d
```





