# Kakfa / Redis on Kubernetes

> ktds Container 기반 Kafka, Redis 교육자료

본 교육은 Container를 기반으로 한 Kafka, Redis 를 학습하는 과정으로 kubernetes 환경에서 kafka/Redis  install수행하는 방법과  각각 모니터 솔루션을 설치하는 방법을 알아보며 Spring Boot / Python 으로 실습한다.

문의: 송양종( yj.song@kt.com / ssongmantop@gmail.com )








# 1. 시작전에 ( [가이드 문서 보기](./beforebegin/beforebegin.md) )  



## 1) 실습 환경 준비(개인PC)

- WSL2 설치
- Docker Desktop 설치
- MobaxTerm 설치
- Typora 설치
- 교육자료 Download






# 2. Kafka 개념 ( [가이드 문서 보기](./kafka/1.kafka-개념.md) )  



## 1) Kafka 개요

- kafka 개요와 특징확인



## 2) Kafka 기본

- Kafka 구성요소인 Broker, Message, Producer, Consumer ,Topic 의 Concept 대해 확인
- Broker
  - Producer와 Consumer 사이에 존재하는 Broker 확인
- Partition과 Consumer Group
  - 병렬처리를 가능하게 하는 Partition과 수신 애플리케이션을 담당하는 Consumer Group 
- Offset 관리
  - Conumer Group 단위로 Offset 을 관리
- Producer/Consumer Partitioning
  - Partion 별로 메세지를 수신/ 발신 한느 방법에 대한 개념 확인



## 3) Kafka Replication

- Replicas
  -  Broker 장애시 수신 메시지 분실 방지를 위한 복제(Replication)
- ISR(In-Sync Replica)
  - ISR은 **현재 동기화 상태에 있는** **리플리케이션**
- Producer Ack
  - Replication 구조에서 Producer 메시지 송신 타이밍을 결정하는 Ack 설정






# 3. Kafka Hands-in ( [가이드 문서 보기](./kafka/2.kafka-hands-in.md) )  



## 1) Kafka 개요

- 주절주절




## 2) Strimzi

- 주절주절



## 3) Kafka Cluster 생성

- 주절주절



## 4) User / Topic 생성

- 주절주절



## 4) Internal / External Access

- 주절주절



## 6) Monitoring



## 7) python 실습



## 8) Java 실습





# 3. Class2: Redis on Kubernetes ( [가이드 문서 보기](./redis/redis.md) )  



## 1) Redis 개요



## 2) Redis Cluster Set



## 3) Redis Set

- 주절주절



## 4) P3X Redis UI

- 주절주절



## 5) ACL

- 주절주절



## 6) Java Sample

- 주절주절





# 별첨. KT Cloud Setup ( [가이드 문서 보기](./ktcloud-setup/ktcloud-setup.md) )  

## 1) 서버 생성

- k3s Cluster 용도 VM 생성
- k3s 설치
- user 생성

## 2) Istio 셋팅

- helm 설치
- Istio 설치
- Monitoring 설치
  - Prometheus / Grafana / Kiali / Jeager 설치

## 3) ArgoCD 셋팅

- ArgoCD 설치
- ArgoCD CLI 설치