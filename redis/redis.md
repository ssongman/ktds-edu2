


# Redis On Kubernetes



# 1. Redis 개요

## 1) Redis 개요

- [Redis](https://redis.io/) (REmote DIctionary Server의 약자)는 데이터베이스, 캐시 또는 메시지 브로커로 자주 사용되는 오픈 소스 인메모리 DB 


- list, map, set, and sorted set 과 같은 고급 데이터 유형을 저장하고 조작할 수 있음
- Redis는 다양한 형식의 키를 허용하고 서버에서 직접 수행되므로 클라이언트의 작업 부하를 줄일 수 있음 

- 기본적으로 DB 전체를 메모리에 보유하며 Disk 는 지속성을 위해서만 사용됨 

- Redis는 인기 있는 데이터 스토리지 솔루션이며 GitHub, Pinterest, Snapchat, Twitter, StackOverflow, Flickr 등과 같은 거대 기술 기업에서 사용됨



### Redis를 사용하는 이유

- 아주 빠름. ANSI C로 작성되었으며 Linux, Mac OS X 및 Solaris와 같은 POSIX 시스템에서 실행됨
- Redis는 종종 가장 인기 있는 키/값 데이터베이스 및 컨테이너와 함께 사용되는 가장 인기 있는 NoSQL 데이터베이스로 선정됨
- 캐싱 솔루션은 클라우드 데이터베이스 백엔드에 대한 호출 수를 줄임
- 클라이언트 API 라이브러리를 통해 애플리케이션에서 액세스할 수 있음
- Redis는 인기 있는 모든 프로그래밍 언어에서 지원됨
- 오픈 소스이며 안정적임



### 실제 세계에서 Redis 사용

- Twitter는 Redis 클러스터 내의 모든 사용자에 대한 타임라인을 저장함
- Pinterest는 데이터가 수백 개의 인스턴스에 걸쳐 샤딩되는 Redis Cluster에 사용자 팔로어 그래프를 저장함
- Github은 Redis를 대기열로 사용함





## 2) Redis Cluster



### (1) Redis Cluster

- [Redis 클러스터](https://redis.io/topics/cluster-tutorial) 는 DB를 분할하여 데이터베이스를 확장하여 복원력을 향상시키도록 설계된 Redis Instance 들의 집합임

- 만약 Master에 연결할 수 없으면 해당 Slave가 Master로 승격됨 
- 3개의 Master노드로 구성된 최소 Redis 클러스터에서 각 Master 노드에는 단일 Slave 노드가 있습니다(최소 장애 조치 허용)
- 각 Master노드에는 0에서 16,383 사이의 해시 슬롯 범위가 할당됨 
- 노드 A는 0에서 5000까지의 해시 슬롯, 노드 B는 5001에서 10000까지, 노드 C는 10001에서 16383까지 포함됨 

- 클러스터 내부 통신은 internal bus를 통해 이루어지며 클러스터에 대한 정보를 전파하거나 새로운 노드를 발견하기 위해 gossip protocol을 사용. 

- 데이터는 여러 노드 간에 자동으로 분할되므로 노드의 하위 집합에 장애가 발생하거나 클러스터의 나머지 부분과 통신할 수 없는 경우에도 안정적인 서비스 제공





![kubernetes-deployment](redis.assets/rancher_blog_deploying_kubernetes_1.png)





### (2) Redis 와 Redis-cluster 와의 차이점

| Redis                                                  | Redis Cluster                                                |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| 여러 데이터베이스 지원                                 | 한개의 데이터베이스 지원                                     |
| Single write point (single master)                     | Multiple write points (multiple masters)                     |
| ![redis-topology.png](redis.assets/redis-topology.png) | ![redis-cluster-topology.png](redis.assets/redis-cluster-topology.png) |











# 3. Install 준비

WSL 환경에서 istio 를 설치해 보자.



## 1) helm install

쿠버네티스에 서비스를 배포하는 방법이 다양하게 존재하는데 그중 대표적인 방법중에 하나가 Helm chart 방식 이다.



- helm chart 의 필요성

일반적으로 Kubernetes 에 서비스를 배포하기 위해 준비되는 Manifest 파일은 정적인 형태이다. 따라서 데이터를 수정하기 위해선 파일 자체를 수정해야 한다.  잘 관리를 한다면야 큰 어려움은 없겠지만, 문제는 CI/CD 등 자동화된 파이프라인을 구축해서 애플리케이션 라이프사이클을 관리할 때 발생한다.  

보통 애플리케이션 이미지를 새로 빌드하게 되면, 빌드 넘버가 변경된다. 이렇게 되면 새로운 이미지를 사용하기 위해 Kubernetes Manifest의 Image도 변경되어야 한다.  하지만 Kubernetes Manifest를 살펴보면, 이를 변경하기 쉽지 않다. Image Tag가 별도로 존재하지 않고 Image 이름에 붙어있기 때문입니다. 이를 자동화 파이프라인에서 변경하려면, sed 명령어를 쓰는 등의 힘든 작업을 해야 한다.

Image Tag는 굉장히 단적인 예제이다.  이 외에 도 Configmap 등 배포시마다 조금씩 다른 형태의 데이터를 배포해야 할때 Maniifest 파일 방식은 너무나 비효율적이다.  Helm Chart 는 이런 어려운 점을 모두 해결한 훌륭한 도구이다.  비단,  사용자가 개발한 AP 뿐아니라 kubernetes 에 배포되는 오픈소스 기반 솔루션들은 거의 모두 helm chart 를 제공한다.

Istio 도 마찬가지로 helm 배포를 위한 chart 를 제공해 준다.



### (1) helm architecture



![helm-architecure](redis.assets/helm-architecure.png)



### (2) helm client download

helm client 를 local 에 설치해 보자.

```sh
# root 권한으로 수행
## 임시 디렉토리를 하나 만들자.
$ mkdir -p ~/helm/
$ cd ~/helm/

$ wget https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
$ tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
$ mv linux-amd64/helm /usr/local/bin/helm

$ ll /usr/local/bin/helm*
-rwxr-xr-x 1 song song 46182400 May 19 01:45 /usr/local/bin/helm*


# 권한정리
$ sudo chmod 600  /home/song/.kube/config


# 확인
$ helm version
version.BuildInfo{Version:"v3.9.0", GitCommit:"7ceeda6c585217a19a1131663d8cd1f7d641b2a7", GitTreeState:"clean", GoVersion:"go1.17.5"}

$ helm -n user01 ls
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION

```





## 2) namespace 생성

```sh
$ kubectl create ns redis-system

$ alias krs='kubectl -n redis-system'

```



# 2. Redis Cluster Install

kubernetes 기반에서 Redis 를 설치해보자.

참조link : https://github.com/bitnami/charts/tree/master/bitnami/redis-cluster



## 1) helm chart download



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





## 3) pod/svc 확인

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





## 4) Internal Access

redis client를 cluster 내부에서 실행후 접근하는 방법을 알아보자.

### (1) Redis client 실행

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









## 5) Clean Up & Update

### (1) delete

helm delete 명령을 이용하면 helm chart 로 설치된 모든 리소스가 한꺼번에 삭제된다.

```sh
## 삭제하기
$ helm -n redis-system delete my-release

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





# 3. Redis Install

External (Cluster 외부) 에서 access 하기 위해서 node port 를 이용해야 한다.

하지만 Redis Cluster 의 경우 접근해야 할 Master Node 가 두개 이상이며 해당 데이터가 저장된 위치를 찾아 redirect 된다.

이때 redirect 가 정확히 이루어지려면 Client 가 인식가능한 Node 주소를 알아야 한다.

하지만 Redis Cluster 는 원격지 Client 가 인식가능한 Node 들의 DNS를 지원하지 않는다.

결국 Redis Cluster 는 PRD환경과 같이 Kubernetes Cluster 내에서는 사용가능하지만 

개발자 PC에서 연결이 필요한 DEV환경에서는 적절치 않다.

그러므로 redis-cluster 가 아닌 redis 로 설치 하여 테스트를 진행한다.



## 1) Redis(Single Master) Install



### (1)  Redis Install



#### helm search

추가된 bitnami repo에서 redis-cluster 를 찾는다.

```sh
$ helm search repo redis
bitnami/redis                                   16.13.0         6.2.7           Redis(R) is an open source, advanced key-value ...
bitnami/redis-cluster                           7.6.3           6.2.7           Redis(R) is an open source, scalable, distribut...
prometheus-community/prometheus-redis-exporter  4.8.0           1.27.0          Prometheus exporter for Redis metrics

```

bitnami/redis



#### helm install

```sh
$ cd ~/song/helm/charts/redis

# helm install
# master 1, slave 3 실행
$ helm -n redis-system install my-release bitnami/redis \
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



To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace redis-system my-release-redis -o jsonpath="{.data.redis-password}" | base64 -d)

To connect to your Redis&reg; server:

1. Run a Redis&reg; pod that you can use as a client:

   kubectl run --namespace redis-system redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:6.2.7-debian-11-r9 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client \
   --namespace redis-system -- bash

2. Connect using the Redis&reg; CLI:
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h my-release-redis-master
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h my-release-redis-replicas

To connect to your database from outside the cluster execute the following commands:

    export NODE_IP=$(kubectl get nodes --namespace redis-system -o jsonpath="{.items[0].status.addresses[0].address}")
    export NODE_PORT=$(kubectl get --namespace redis-system -o jsonpath="{.spec.ports[0].nodePort}" services my-release-redis-master)
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h $NODE_IP -p $NODE_PORT


# 확인
$ helm -n redis-system ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
my-release      redis-system    1               2022-07-01 01:21:32.6958631 +0900 KST   deployed        redis-16.13.1   6.2.7


```

my-release-redis-master 는 read/write 용도로 사용되며 my-release-redis-replicas 는 read-only 용도로 사용된다.





### (2) Chart Fetch 이후 Install

설치 과정에서 chart 를 다운 받지 못한다면 Chart 를 fetch 받아서 설치하자.



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

my-release-redis-master 는 read/write 용도로 사용되며 my-release-redis-replicas 는 read-only 용도로 사용된다.



### (3) pod / svc 확인

```sh
$ krs get pod
NAME                          READY   STATUS    RESTARTS   AGE
my-release-redis-master-0     1/1     Running   0          6m30s
my-release-redis-replicas-0   1/1     Running   0          6m30s
my-release-redis-replicas-1   1/1     Running   0          5m34s
my-release-redis-replicas-2   1/1     Running   0          5m9s


$ krs get svc
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-release-redis-headless   ClusterIP   None            <none>        6379/TCP         6m40s
my-release-redis-master     NodePort    10.108.249.49   <none>        6379:32200/TCP   6m40s
my-release-redis-replicas   NodePort    10.96.105.76    <none>        6379:32210/TCP   6m40s

```



### (4) Catch up

```sh

# 삭제시
$ helm -n redis-system delete my-release

```





## 2) Internal Access

redis client를 cluster 내부에서 실행후 접근하는 방법을 알아보자.

### (1) Redis client 확인

#### docker redis client

local pc 에서 access 테스트를 위해 docker redis client 를 설치하자.

```sh
## redis-client 용도로 docker client 를 실행한다.
$ docker run --name redis-client -d --rm --user root docker.io/bitnami/redis-cluster:6.2.7-debian-11-r3 sleep 365d

## docker 내에 진입후
$ docker exec -it redis-client bash

## Local PC IP로 cluster mode 접근
$ redis-cli -h 192.168.31.1 -c -a new1234 -p 32200



```



### (2) set/get 확인

```

192.168.31.1:32200>  set a 1
OK
192.168.31.1:32200> set b 2
OK
192.168.31.1:32200> set c 3
OK
192.168.31.1:32200> get a
"1"
192.168.31.1:32200> get b
"2"
192.168.31.1:32200> get c
"3"

```





# 4. P3X Redis UI

참고링크
https://www.electronjs.org/apps/p3x-redis-ui

https://github.com/patrikx3/redis-ui/blob/master/k8s/manifests/service.yaml

Redis DB 관리를 위한  편리한 데이터베이스 GUI app이며  WEB  UI 와 Desktop App 에서 작동한다.

P3X Web UI 를 kubernetes 에 설치해 보자.



## 1) redis-ui deploy

아래 yaml  manifest file을 활용하여 configmap, deployment, service, ingress 를 일괄 실행한다.

```sh
$ cd ~/githubrepo/ktds-edu2


$ cat ./redis/redisui/11.p3xredisui.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: p3x-redis-ui-settings
data:
  .p3xrs-conns.json: |
    {
      "list": [
        {
          "name": "cluster",
          "host": "my-release-redis-master",
          "port": 6379,
          "password": "new1234",
          "id": "unique"
        }
      ],
      "license": ""
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: p3x-redis-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: p3x-redis-ui
  template:
    metadata:
      labels:
        app.kubernetes.io/name: p3x-redis-ui
    spec:
      containers:
      - name: p3x-redis-ui
        image: patrikx3/p3x-redis-ui
        ports:
        - name: p3x-redis-ui
          containerPort: 7843
        volumeMounts:
        - name: p3x-redis-ui-settings
          mountPath: /settings/.p3xrs-conns.json
          subPath: .p3xrs-conns.json
      volumes:
      - name: p3x-redis-ui-settings
        configMap:
          name: p3x-redis-ui-settings
---
apiVersion: v1
kind: Service
metadata:
  name: p3x-redis-ui-service
  labels:
    app.kubernetes.io/name: p3x-redis-ui-service
spec:
  ports:
  - port: 7843
    targetPort: p3x-redis-ui
    name: p3x-redis-ui
  selector:
    app.kubernetes.io/name: p3x-redis-ui
  type: NodePort
---

# install
$ kubectl -n redis-system apply -f ./redis/redisui/11.p3xredisui.yaml

# 삭제시
$ kubectl -n redis-system delete -f ./redis/redisui/11.p3xredisui.yaml

```



## 2) ui 확인

http://p3xredisui.redis-system.ktcloud.211.254.212.105.nip.io/main/key/people

![image-20220626181624749](redis.assets/image-20220626181624749.png)











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





## 2) 읽기전용 계정 생성

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







# 6. Java Sample



## 1) Jedis vs Lettuce

참고: https://jojoldu.tistory.com/418

- Java 의 Redis Client 는 크게 Jedis 와 Lettuce  가 있음.

- 초기에는 Jedis 를 많이 사용했으나 현재는 Lettuce 를 많이 사용하는 추세임.

- Jedis 의 단점
  -  멀티 쓰레드 불안정, Pool 한계 등
- Lettuce 의 장점
  - Netty 기반으로 비동기 지원 가능 등

- 결국 Spring Boot 2.0 부터 Jedis 가 기본 클라이언트에서 deprecated 되고 Lettuce 가 탑재되었음





## 2) Spring Boot Sample

sample source github link





- pom.xml

```xml
...
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
...
```



- application.yaml

```yaml
spring:
  redis:
    lettuce:
      pool:
        max-active: 10
        max-idle: 10
        min-idle: 2
    host: localhost
    port: 6379
    password: 'new1234'
```



- 참고 : 각 항목들에 대한 설명

| 변수                         | 기본값                             | 설명                                                         |
| ---------------------------- | ---------------------------------- | ------------------------------------------------------------ |
| spring.redis.database        | 0                                  | 커넥션  팩토리에 사용되는 데이터베이스 인덱스                |
| spring.redis.host            | localhost                          | 레디스  서버 호스트                                          |
| spring.redis.password        | 레디스  서버 로그인 패스워드       |                                                              |
| spring.redis.pool.max-active | 8                                  | pool에  할당될 수 있는 커넥션 최대수 (음수로 하면 무제한)    |
| spring.redis.pool.max-idle   | 8                                  | pool의  "idle" 커넥션 최대수 (음수로 하면 무제한)            |
| spring.redis.pool.max-wait   | -1                                 | pool이  바닥났을 때 예외발생 전에 커넥션 할당 차단의 최대 시간 (단위: 밀리세컨드, 음수는 무제한 차단) |
| spring.redis.pool.min-idle   | 0                                  | 풀에서  관리하는 idle 커넥션의 최소 수 대상 (양수일 때만 유효) |
| spring.redis.port            | 6379                               | 레디스  서버 포트                                            |
| spring.redis.sentinel.master | 레디스  서버 이름                  |                                                              |
| spring.redis.sentinel.nodes  | 호스트:포트  쌍 목록 (콤마로 구분) |                                                              |
| spring.redis.timeout         | 0                                  | 커넥션  타임아웃 (단위: 밀리세컨드)                          |



Redis에 Connection을 하기 위한 RedisConnectionFactory 생성

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

**RedisConnectionFactory 인터페이스를 통해 LettuceConnectionFactory를 생성하여 반환합니다.**



```java
@Getter
@RedisHash(value = "people", timeToLive = 30)
public class Person {

    @Id
    private String id;
    private String name;
    private Integer age;
    private LocalDateTime createdAt;

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
        this.createdAt = LocalDateTime.now();
    }
}
```

- Redis 에 저장할 자료구조인 객체를 정의함

- 일반적인 객체 선언 후 @RedisHash 를 붙임
  - value  값이 Redis 의 key prefix 로 사용됨
  - timeToLive : 만료시간을 seconds 단위로 설정할 수 있음 
    - 기본값은 만료시간이 없는 -1L 임.
- @Id 어노테이션이 붙은 필드가 Redis Key 값이 되며 null 로 세팅하면 랜덤값이 설정됨
  - keyspace 와 합쳐져서 레디스에 저장된 최종 키 값은 keyspace:id 가 됨
    - key 생성형식: "people:{id}"








# 9. python test - 작성중


- pod로 실행

```
oc -n redis-system run pythonfortest --image=ktis-bastion01.container.ipc.kt.com:5000/admin/python:3.7 -- sleep 365d
```

- deploy로 실행

```
oc -n redis-system create deploy pythonfortest --image=ktis-bastion01.container.ipc.kt.com:5000/admin/python:3.7 -- sleep 365d
```





