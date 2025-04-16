# 🚀 Hyundai Card Coupon-Issue Optimization

[![Java](https://img.shields.io/badge/Java-backend-%23ED8B00.svg?style=for-the-badge&logo=java&logoColor=white)]()
[![Oracle](https://img.shields.io/badge/Oracle-DB-%23F80000.svg?style=for-the-badge&logo=oracle&logoColor=white)]()
[![SQL](https://img.shields.io/badge/SQL-Optimization-%23007ACC.svg?style=for-the-badge&logo=sqlite&logoColor=white)]()

> 현대카드 쿠폰 발급 API에서 발생한 **동시성 문제 및 Deadlock 이슈**를 해결한 최적화 프로젝트입니다.

> **Note**: 본 프로젝트는 사내 전용 시스템으로 실제 코드는 공개하지 않습니다.  
> 기술 구조 및 역할 위주로 정리된 문서 기반 포트폴리오입니다.  
> 본 문서의 코드는 **설명용 예시**이며, 실제 구현 코드와는 다를 수 있습니다.

---

## 🧩 문제 개요

A 커머스 고객사의 **실시간 난수 쿠폰 발행 API**에서  
다수 트랜잭션 동시 인입 시 Deadlock으로 인해 거래 실패가 빈번히 발생.

- 기존 트랜잭션 속도: 약 **150ms**
- 쿠폰 업데이트/삽입 시 **Lock 경합 → Deadlock 발생**
- 결과적으로 쿠폰 발급 실패 및 dup 오류 증가

---

## ⚠️ 주요 원인 분석

### 1. 동일 쿠폰 ID를 동시에 업데이트
```sql
SELECT * FROM coupon WHERE id = 100 FOR UPDATE;
UPDATE coupon SET is_used = 1 WHERE id = 100;
```

### 2. PK 충돌 (idx+1 방식)
```sql
SELECT MAX(id) INTO @newId FROM coupon;
INSERT INTO coupon (id) VALUES (@newId + 1);
```

### 3. `is_used = 0` 쿠폰 조회 후 업데이트
```sql
SELECT * FROM coupon WHERE is_used = 0 LIMIT 1;
UPDATE coupon SET is_used = 1 WHERE id = ?;
```

### 4. Java 트랜잭션 처리 중 경쟁 조건
```java
conn.setAutoCommit(false);
PreparedStatement ps = conn.selectDB1000A;
ResultSet rs = ps.executeQuery();
```

---

## 🔧 해결 전략

### ✅ 트랜져션 시간 로깅 및 병목 제거  

### ✅ 랜덤 쿠폰 추출 로직 전환  
- **LAG() + SYSTIMESTAMP** 기반 추출 (중복 방지)
- **DBMS_RANDOM**을 활용한 난수 기반 쿠폰 선택
- **남은 쿠폰 수 동적 체크** 후 방식 전환

```sql
-- LAG() 기반 추출
SELECT id FROM (
  SELECT id, LAG(id) OVER (ORDER BY SYSTIMESTAMP)
  FROM coupon WHERE is_used = 0
) WHERE ROWNUM = 1;
```

### ✅ 파티셔닝을 통한 동시성 처리

```sql
CREATE TABLE coupon (
  id BIGINT PRIMARY KEY,
  ...
) PARTITION BY HASH(id) PARTITIONS 10;
```
> ➔ 파티셔닝 분산으로 Lock 충돌 완화 및 대량 요청 분산 처리

---

## 📊 참조 성과
| 항목 | 감정 전 | 감정 후 |
|------|---------|---------|
| 트랜져션 평균 속도 | 150ms | **70~100ms** |
| 쿠포드 dup 오류율 | 높음 | **대편 감소** |
| 동시 처리 성능 | 낮음 | **파티셔닝 기반 향상** |

- **트랜잭션 처리 속도 약 33% 단축**
- **쿠폰 발급 오류 최소화 및 안정성 확률**
- 일부 중복 이슈는 이력 테이블 관리 및 별도 알림 처리

---

## 📀 결론

대규모 트랜잭션 처리 시스템에서의 Deadlock은  
다른 하위 시스템으로 전달되기 위해
대규모 실시간 서비스를 가정하고 분석하는 것이 해결 관건입니다.

본 프로젝트에서는  
1. **랜덤 쿠폰 추출 방식**,  
2. **트랜잭션 충돌 회피 설계**,  
3. **데이터 파티셔닝 기반 확장성**을 통해  
**동시성 문제를 구조적으로 해결**했습니다.

> 실제 운용 환경에서도 안정적으로 적용 완료되어있으며,  
> 실시간 대량 트랜잭션에서의 문제 해결 경험으로 기록합니다.