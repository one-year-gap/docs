---
title: 'AWS Step Function 실행 실패와 ECS 정상 상태 모순: CDK 배포 불일치와 Security Group Egress 누락 추적기'
date: 2026-03-14
domain: 인프라
severity: Major
status: resolved
author: 김도연
thumbnail: /docs/assets/images/uploads/스크린샷 2026-03-14 11.54.36.png
tags:
  - AWS Security Group
  - Step Function
---

# 1. 문제 발생

최근 변경된 On-Demand 워크플로우를 구현하며 운영환경에서 배포 이후, 실행 과정(KST 기준: 2026년 3월 14일 11시 2분 15.769초)에서 예상하지 못한 장애를 직면했다.

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2011.54.36.png)

해당 워크플로우는 `EventBridge`에서 시작되어 `Step Functions`, `Lambda(Readiness Probe)`, `ECS RunTask`, `DynamoDB Lock`으로 이어지는 구조를 가진다. 워크플로우의 핵심 목적은 본격적인 배치 작업 실행 전에 대상 시스템인 `analysis-server`가 실제로 준비되었는지 `/ready` 및 `/health` 엔드포인트를 통해 확인하고, 검증이 완료된 후 후속 작업을 수행하는 것이다.

실패한 내역은 아래와 같다.

- 상태 머신: AnalysisBatchWorkflow
- 실행 시각: 2026-03-14 11:02:15 KST
- 종료 시각: 2026-03-14 11:06:56 KST
- 최종 상태: FAILED

# 2. 가설 세우기

장애가 발생한 워크플로우는 AWS CDK in Java로 작성하였으며 OnDemandWorkflowStack.java에 정의되어 있다. 특히, Readiness Probe 역할을 수행하는 Lamda 함수는 어플리케이션 내부 로직이 아닌, CDK를 통해 주입받은 `VPC`, `Subnet`, `Security Group(SG)` 설정에 의해 네트워크 통신 경로가 결정된다.

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.16.18.png)

장애 발생 지점은 파악을 했으므로 해당 단계에서 발생할 수 있는 4가지 가설을 세웠다.

1. analysis-server의 기동이 지연되어 /ready 응답에서의 timeout이 발생했을 가능성.
2. 응답은 정상적으로 수신했으나 릴리스 태그나 계약 조건이 불일치할 가능성
3. Lamda가 analysis-server로의 네트워크 연결 자체를 못하는 가능성
4. AWS CDK 의 코드 내용과 실제 AWS 인프라 환경의 배포 상태가 다를 가능성

# 3. 가설 검증 및 원인 추적

## 1) `Step Functions`에서 실패 지점 파악

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.06.59.png)

가장 먼저` AWS console - Step Functions` 에서 실행 이력을 분석하여 실패 지점을 좁혔다. 전체 워크 플로우 중 배치 실행 단계가 아니라 준비 상태를 확인하는 `ProbeAnalysisServerReady` 단계에서 에러가 발생했음을 확인했다.![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.09.13.png)

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.16.18.png)

해당 단계에서는`probeIntervalSeconds`와 `probeMaxAttempts` 설정에 따라 지정된 횟수만큼 재시도를 수행하도록 설계되어 있었으며, 설정된 재시도를 모두 소진한 후 최종 실패로 처리되고 있다.

## 2) Lamda 로그에서 timeout 패턴 확인

다음으로 정확한 실패 사유를 분석하기 위해 Readiness Probe Lamda 실행 로그를 분석했다. Lamda의 Python-urllib를 이용하여 대상 서버에 GET 요청을 보내고 HTTP 상태 코드 및 Payload의 정합성을 검증하도록 작성했다.

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.29.50.png)

로그 분석 결과, HTTP 500이나 404와 같은 서버 에러나 릴리즈 태그 불일치 에러는 발견되지 않았다. 다만 네트워크 레벨에서의 `urlopen timeout` 예외가 5초 간격으로 누적되어 발생하고 있었다. 이는` Lamdbda`가 대상 서버로 연결조차 하지 못하는 상태임을 의한다. 이를 근거로 `가설2`는 배제했다.

실제로 오류가 발생하는 시점의 코드는 아래와 같다.

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.30.03.png)

## 3) Analysis ECS Service Task 상태 확인

이어서, 타겟 서버인` analysis-server` 자체의 업-다운 여부 확인을 위해 `ECS Task` 상태를 확인했다. 확인 결과 ECS Task는 `Step Functions`가 실행하기 이전부터 정상적으로 기동되어 `RUNNING` 상태임을 확인했다.

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.33.07.png)

따라서, Analysis Server task가 늦게 기동되어 `timeout`이 발생했을 것이라는 첫 번째 가설은 아님을 입증했다.

## 3) Security Group 규칙 확인

네트워크 통신 간 보안 문제 확인을 위해 각각 할당된` Security Group`의 규칙을 확인했다. 타겟 서버인 `Analysis-Server`의 `인바운드` 규칙은 올바르게 설정되어 접근을 허용하고 있었다. ![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.34.13.png)

하지만,` Security Group` 간 통신이 가능하려면 `인바운드` 뿐만 아니라 출발지의 `아웃바운드` 규칙 역시 설정되어있어야 한다. AWS CDK 코드상으로` Probe Lambda`가 속한 `AnalysisServerSg`의 8000번 포트로 향하는 Egress 규칙이 명확히 정의되어 있었다. 하지만 실제 AWS 환경에 프로비저닝된 리소스를 확인해 보니 해당 아웃바운드 규칙이 누락되어 있었다. 이로 인해 TCP 3-Way Handshake조차 이루어지지 않아 타임아웃이 발생했던 것이다.

## 4) CloudFormation 스택 상태 확인

코드에는 존재하는 규칙이 실제 인프라에 반영되지 않은 이유를 찾기 위해 `CloudFormation`의 스택 배포 상태를 확인했다. 확인 결과, 워크플로우 로직을 담은 `OnDemandWorkflowStack`은 정상적으로 배포(`UPDATE_COMPLETE`)되었으나, 네트워크 규칙 변경을 포함하는 `NetworkStack`은 모종의 이유로 롤백(`UPDATE_ROLLBACK_COMPLETE`)된 상태였다. 

![](/docs/assets/images/uploads/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-03-14%2012.48.13.png)

결과적으로 Readiness Probe 기능 자체는 추가되었으나, 해당 기능이 의존하는 네트워크 계층의 변경 사항이 반영되지 않아 코드와 실제 인프라 간의 드리프트(Drift)가 발생한 것이 장애의 근본 원인이었다.
