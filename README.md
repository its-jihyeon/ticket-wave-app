# Ticket Wave — APP
### EKS 기반 고가용성 티켓팅 서비스

> 이 저장소는 멋쟁이사자처럼 AWS 기반 DevOps 엔지니어 과정 팀 프로젝트(5인)의 fork 입니다. <br>
> 본 README는 담당 영역을 중심으로 재작성했습니다. <br>

<br>

원본 저장소
* **App 레포** : [team5-ticket-app](https://github.com/CLD-05/team5-ticket-app)
* **Config 레포** : [team5-ticket-config](https://github.com/CLD-05/team5-ticket-config)
* **Infra 레포** : [team5-ticket-infra](https://github.com/CLD-05/team5-ticket-infra) <br>

<br>

## 프로젝트 개요

예매 오픈 시점에 트래픽이 순간적으로 집중되는 티켓팅 서비스 특성상, 

단순 서버 배포가 아니라 트래픽 완충·비동기 처리·데이터 정합성·오토스케일링·운영 관측성을 함께 고려한 인프라가 필요했습니다. 

이를 EKS 기반으로 구축하고, dev/prod 환경을 분리해 Terraform으로 관리했습니다.

<br>

- 기간 : 2026.06.02 ~ 2026.07.08
- 팀 구성 : 5인
- 담당 : EKS 접근 통제 체계 구축 · 실시간 좌석 API 개발

<br>

## 시스템 아키텍처 및 비동기 처리 흐름

```mermaid
flowchart TD
    Client([사용자 / Client]) --> API["Web API Pod (ticket-service)"]

    API -- "대기열 진입 / 토큰 검증" --> Redis[(ElastiCache Redis)]
    API -- "좌석 조회 / 좌석 선점" --> Redis
    API -- "예매 요청 적재 (202 Accepted)" --> SQS["AWS SQS (Booking Queue)"]

    SQS -- "메시지 소비" --> Worker["Booking Worker Pod"]
    Worker -- "예매 확정 처리" --> Proxy["RDS Proxy"]
    Proxy --> RDS[(RDS MySQL)]
```

<br>

## 내가 담당한 부분

### 실시간 좌석 선점 API
- Redis Lua Script 기반 좌석 선점 로직 구현

  → 기존 캐시 조회와 저장의 분리된 연산으로 인해 발생할 수 있는 중복 선점을 원자적으로 차단
- Redis multiGet 기반 좌석 상태 병합 조회 최적화
- k6 테스트(10석 / 37,500 req) 결과 중복 선점 0건 검증
- 관련 코드 : `src/.../SeatService.java`
  
<br>

**[EKS 접근 통제 체계 구축](https://github.com/its-jihyeon/ticket-wave-infra/blob/main/README.md)**

<br>

## 트러블슈팅
### [Redis multiGet 조회 최적화]
- 상황 : 프론트엔드 예매 페이지 진입 시 실시간 좌석 데이터를 불러오는 과정에서 응답 지연 현상 발생
- 원인 분석 : 좌석 조회 로직 코드 리뷰를 진행한 결과 Redis 단건으로 360번 반복 요청에 따라 발생한 잦은 네트워크 왕복 통신이 지연의 핵심 원인임을 파악
- 시도한 방법 : 단건 조회의 비효율성을 개선하기 위해 캐시 통신 횟수를 줄이고 다건을 한 번에 처리할 수 있는 최적화 방안 리서치
- 최종 해결 : Redis의 multiGet을 새롭게 학습 및 적용하여 360번의 단건 조회 요청을 단 1번의 네트워크 통신으로 병합 조회하도록 최적화
- 배운 점 : 단순 조회 속도뿐만 아니라 인프라 간의 네트워크 통신 최소화가 성능 최적화의 핵심임을 체감

<br>

## 기술 스택
AWS (EKS, RDS, IAM, SSM) · Terraform · Redis · Java

