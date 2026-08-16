---
title: "Spring Boot + Quartz Scheduler: @Scheduled로는 안 되는 지점"
date: 2025-10-04
categories: [개발일지, Quartz]
tags: [spring-boot, devops]
---

## 스케줄러와 배치는 다른 질문에 답한다

스케줄러는 "언제 실행할지"를 관리하고, 배치는 "무엇을 어떻게 처리할지"를 관리한다.
둘을 섞어서 생각하면 설계가 꼬인다 — 예를 들어 "매일 새벽 2시에 전체 회원 데이터를
분석한다"는 요구사항은 스케줄링(새벽 2시)과 배치(전체 데이터 분석)가 결합된 것이지,
하나의 개념이 아니다. Quartz는 전자(언제)를 담당하는 도구다.

## 왜 @Scheduled가 아니라 Quartz인가

Spring의 `@Scheduled`는 코드에 스케줄이 박혀 있다 — cron 표현식이 어노테이션에
하드코딩되고, 애플리케이션을 재시작하지 않으면 주기를 바꿀 수 없다. Job을 동적으로
등록·삭제·조회해야 하는 요구사항(예: 사용자가 API로 알림 주기를 직접 설정하는 기능)에는
맞지 않는다.

| | Spring @Scheduled | Quartz |
|---|---|---|
| 장점 | 간단하고 즉시 사용 가능 | 동적 Job 등록/삭제, 반복 주기 관리, Job 상태 추적 |
| 단점 | 복잡한 반복 주기·동적 Job 등록 어려움 | 초기 설정이 다소 복잡 |

핵심 차이는 "Job이 코드 배포와 묶여 있는가, 런타임에 독립적으로 관리되는가"다. 반복
횟수 관리, TriggerListener를 통한 실시간 상태 갱신처럼 **런타임에 Job의 생애주기를
제어**해야 하는 요구사항에는 Quartz 쪽이 맞다.

## TimerInfo — Job 설정을 코드가 아니라 데이터로 다루기

```java
TimerInfo info = new TimerInfo();
info.setTotalFireCount(5);        // 총 실행 횟수
info.setRemainingFireCount(5);    // 남은 실행 횟수
info.setRepeatIntervalMs(5000);   // 반복 간격 (ms)
info.setInitialOffsetMs(1000);    // 최초 시작 지연 시간 (ms)
info.setCallbackData("My callback data");
```

이 DTO의 존재 이유는 Job의 "실행 방법"(코드)과 "실행 조건"(데이터)을 분리하는 것이다.
Quartz의 `JobDataMap`에 이 정보를 실어 보내면, 같은 Job 클래스를 재사용하면서도
인스턴스마다 다른 주기·횟수로 실행할 수 있다.

## JobKey / TriggerKey — group으로 Job을 논리적으로 묶기

```java
JobKey emailJob = new JobKey("sendEmail", "dailyJobs");
JobKey reportJob = new JobKey("generateReport", "dailyJobs");
JobKey cleanupJob = new JobKey("cleanup", "weeklyJobs");
```

group을 쓰는 이유는 운영 편의성이다. Job이 수십~수백 개로 늘어나면 "매일 실행되는
것들만 조회", "이번 주 배치만 전부 삭제" 같은 조회/삭제가 필요해지는데, group이 없으면
Job을 하나씩 이름으로 찾아야 한다. `GroupMatcher.jobGroupEquals("dailyJobs")`로
그룹 단위 조회가 가능해지는 게 핵심 이점이다.

## TriggerListener로 실행 횟수 관리하기

```java
@Override
public void triggerFired(Trigger trigger, JobExecutionContext context) {
    final String timerId = trigger.getKey().getName();
    final JobDataMap jobDataMap = context.getJobDetail().getJobDataMap();
    final TimerInfo info = (TimerInfo) jobDataMap.get(timerId);

    if (!info.isRunForever()) {
        int remainingFireCount = info.getRemainingFireCount();
        if (remainingFireCount == 0) {
            return;
        }
        info.setRemainingFireCount(remainingFireCount - 1);
    }

    schedulerService.updateTimer(timerId, info);
}
```

Quartz의 `withRepeatCount`만으로도 반복 횟수 제한은 가능하지만, "남은 횟수를 실시간으로
외부에 노출"하려면(예: API로 "이 Job이 몇 번 더 실행되나" 조회) Quartz 내부 상태를
직접 조회하는 API가 마땅치 않다. 그래서 `JobDataMap`에 상태를 직접 들고 다니면서
`TriggerListener`로 매 실행마다 갱신하는 방식을 썼다 — Quartz의 스케줄링 능력과
애플리케이션 레벨의 상태 관리를 분리한 것이다.

## 실무에서 놓치기 쉬운 것들

- **Misfire 정책**: 서버가 잠깐 멈춰 있는 동안 트리거가 발동해야 했는데 놓친 경우
  (misfire), Quartz는 기본적으로 그걸 어떻게 처리할지 정책을 별도로 설정해야 한다
  (즉시 실행할지, 건너뛸지). 기본값을 그대로 쓰면 서버 재시작 직후 밀린 Job이 한꺼번에
  몰려 실행되는 상황이 생길 수 있다.
- **클러스터 환경에서의 중복 실행**: 인스턴스를 여러 대 띄우면, 별도 설정 없이는
  같은 Job이 인스턴스마다 중복 실행된다. `JDBCJobStore`로 클러스터 모드를 켜서 DB
  락으로 "한 번만 실행됨"을 보장해야 한다 — 인메모리 `RAMJobStore` 기준이라면
  애플리케이션을 재시작하면 등록된 Job이 전부 사라진다는 한계도 있다.
- **JobDataMap에 상태를 직접 들고 다니는 방식의 트레이드오프**: 위 TimerInfo 접근은
  구현이 간단하지만, JobDataMap은 기본적으로 직렬화되어 JobStore에 저장되므로 상태가
  커지거나 자주 갱신될수록 저장 비용이 커진다. 실행 이력을 자주 조회해야 한다면
  차라리 별도 테이블에 상태를 저장하고 Quartz는 "트리거만" 담당하게 하는 편이 나을 수 있다.

## 배운 점

Quartz는 단순 반복 작업 이상으로 "Job의 생애주기를 런타임에 동적으로 제어"해야 할 때
진가를 발휘한다. 다만 그 유연성은 공짜가 아니다 — misfire, 클러스터링, 상태 저장
방식까지 직접 설계해야 하므로, 요구사항이 단순히 "매일 새벽 2시에 배치 하나 돌리기"
수준이라면 `@Scheduled` + cron이 오히려 더 나은 선택일 수 있다.
