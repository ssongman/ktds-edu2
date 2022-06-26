# Kakfa / Redis on Kubernetes

> ktds Container 기반 Kafka, Redis 교육자료

본 교육은 Container를 기반으로 한 Kafka, Redis 를 학습하는 과정으로 kubernetes 환경에서 kafka/Redis  install수행하는 방법과  각각 모니터 솔루션을 설치하는 방법을 알아보며 Spring Boot / Python 으로 실습한다.

문의: 송양종( yj.song@kt.com / ssongmantop@gmail.com )








# 1. 시작전에 ( [가이드 문서 보기](./beforebegin/beforebegin.md) )  



## 1) 실습 환경 준비(개인PC)

- 실습에 필요한 tool 준비
  - mobaxterm 확인
  - wsl2
  - docker desktop 확인

- 교육자료 Download



## 2) 실습 환경 준비(KT Cloud)

- KT Cloud 이해
- ssl 접속 확인
- 수강생별 계정 매핑
- 교육자료 Download






# 2. Class1: Kafka on Kubernetes ( [가이드 문서 보기](./kafka/kafka.md) )  



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