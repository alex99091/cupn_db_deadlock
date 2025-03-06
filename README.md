# Hyundai Card Coupon-issue optimization

### Problem
```
현대카드는 자체 업무뿐만 아니라 외부 커머스 업체의 거래도 처리함.
A 고객사의 실시간 난수 쿠폰 발행 업무에서 쿠폰발행 오류가 발생.
쿠폰 발행 과정에서 시에 다수의 트랜잭션이 동시에 거래가 인입되면
(트랜잭션속도 기존 약 150ms) DML 작업 중 데드락(Deadlock)이 발생하여 
첫 번째 거래를 제외한 나머지 거래가 자동으로 취소되는 문제.
이를 해결하기 위해 최적화가 필요.
```

### Causes
오류가 발생한 원인은 3가지 정도로 분류 가능
- 
(1) 동일한 쿠폰을 동시에 업데이트하는 트랜잭션 충돌
```sql
여러 트랜잭션이 동일한 쿠폰 ID를 동시에 업데이트하려고 
시도하면서 행(row) 잠금(Lock) 경합이 발생.
트랜잭션 간 락 해제가 늦어지면서 교착 상태(Deadlock)에 빠짐.
```
- sql 예시
```sql
-- 트랜잭션 1
BEGIN;
SELECT * FROM coupon WHERE id = 100 FOR UPDATE;
UPDATE coupon SET is_used = 1 WHERE id = 100;
COMMIT;

-- 트랜잭션 2 (거의 동시에 실행)
BEGIN;
SELECT * FROM coupon WHERE id = 100 FOR UPDATE; -- 트랜잭션 1이 완료될 때까지 대기
UPDATE coupon SET is_used = 1 WHERE id = 100;
COMMIT;
```

(2) PK(기본 키) 생성 방식이 idx+1 증가 방식으로 동시성 충돌 유발
```
여러 트랜잭션이 PK를 idx+1 방식으로 생성하려다 충돌 발생.
동일한 ID를 동시에 삽입하려는 경쟁 상태(Race Condition)로 인해 
PK 중복 오류 및 트랜잭션 충돌 발생.
```
- sql예시
```sql
-- 트랜잭션 1
BEGIN;
SELECT MAX(id) INTO @newId FROM coupon; 
INSERT INTO coupon (id, code, is_used) VALUES (@newId + 1, 'COUPON123', 0);
COMMIT;

-- 트랜잭션 2 (거의 동시에 실행)
BEGIN;
SELECT MAX(id) INTO @newId FROM coupon; 
INSERT INTO coupon (id, code, is_used) VALUES (@newId + 1, 'COUPON456', 0);
COMMIT;
```
(3) 데이터 조회 후 업데이트 수행하는 방식에서 경쟁 조건 발생
```sql
SELECT * FROM coupon WHERE is_used = 0 LIMIT 1으로 쿠폰을 조회 후, 
같은 쿠폰을 여러 트랜잭션이 동시에 업데이트하려 함.
한 트랜잭션이 업데이트를 완료하기 전에 다른 트랜잭션이 동일한 쿠폰을 조회하면서 다중 트랜잭션 충돌 및 데드락 발생.
```
- java 예시
```java
public void issueCoupon(Long couponId) {
    Connection conn = dataSource.getConnection();
    try {
        conn.setAutoCommit(false);

        // 동일한 쿠폰을 여러 스레드가 동시에 선택하려 하면 데드락 발생 가능
        PreparedStatement ps = conn.selectDB1000A;
        ps.setLong(1, couponId);
        ResultSet rs = ps.executeQuery();

        if (rs.next()) {
            PreparedStatement updatePs = conn.updateDB1000A;
            updatePs.setLong(1, couponId);
            updatePs.executeUpdate();
        }

        conn.commit();
    } catch (Exception e) {
        conn.rollback();
    } finally {
        conn.close();
    }
}
```

### Solution
(1) 거래 속도 분석 및 최적화
```java
트랜잭션 실행 시간을 로깅하여 평균 속도 분석.
CPU 및 I/O 부하를 줄이기 위해 불필요한 DB 연산 최소화.
네트워크 병목 현상 제거 (Batch Processing 적용).

long startTime = System.currentTimeMillis();
try {
    bcBCCCupnMng.issueCupn();
}
long endTime = System.currentTimeMillis();
logger.debug("Transaction Time: {} ms", (endTime - startTime));

private void issueCupn(bcBCCCupnMng01 in) {
    ...
    if("R".equals(in.getCupnCd()) {
        out = db1000.selectUpdateA(db1000In);
        if(out == null) {
            // 에러처리
        }
    } else if ("S".equals(in.getCupnCd()) {
        out = db1000.selectUpdateB(db1000In);
        if(out == null) {
            // 에러처리
        }
    }
    ...
}
```
(2) 랜덤 쿠폰 추출을 통한 PK 충돌 해결
```sql
기존 순차적 PK 증가 방식 (idx+1) 문제를 해결하기 위해 랜덤 방식으로 변경.
모수가 클수록 오라클 자체 random함수 수행속도가 증가하여, 모수단위로 sql분리
LAG() 함수를 활용하여 최근 발행된 쿠폰 데이터를 기반으로 난수 생성.

/* 2-1  LAG() 기반 랜덤 쿠폰 추출 */
LAG() 함수를 활용하여 이전에 발급된 쿠폰 데이터를 기반으로 난수 생성.
SYSTIMESTAMP를 활용하여, 밀리초(ms) 단위의 변화를 기반으로 쿠폰을 선택함.
SELECT id FROM (
    SELECT id, LAG(id) OVER (ORDER BY SYSTIMESTAMP) AS prev_id
    FROM coupon 
    WHERE is_used = 0
) WHERE ROWNUM = 1;

/* 2-2 전체 쿠폰을 1000 이하일 경우 랜덤추출 */
남은 쿠폰 모수가 1000개 이하일 경우,
해당 그룹에서 난수를 기반으로 쿠폰을 추출.
SELECT id FROM (
    SELECT id 
    FROM coupon 
    WHERE is_used = 0 
    ORDER BY MOD(DBMS_RANDOM.VALUE * 1000, 100)
) WHERE ROWNUM = 1;

/* 2-3. 남은 쿠폰 수에 따른 동적 전환 로직 */
DECLARE
    remainingCoupons NUMBER;
    selectedCouponID NUMBER;
BEGIN
    SELECT COUNT(*) INTO remainingCoupons FROM coupon WHERE is_used = 0;
    
    IF remainingCoupons > 100 THEN
        SELECT id INTO selectedCouponID FROM (
            SELECT id FROM coupon WHERE is_used = 0 ORDER BY MOD(DBMS_RANDOM.VALUE * 1000000, 100)
        ) WHERE ROWNUM = 1;
    ELSE
        SELECT id INTO selectedCouponID FROM (
            SELECT id, LAG(id) OVER (ORDER BY SYSTIMESTAMP) AS prev_id
            FROM coupon WHERE is_used = 0
        ) WHERE ROWNUM = 1;
    END IF;
    
    UPDATE coupon SET is_used = 1 WHERE id = selectedCouponID;
    COMMIT;
END;
/* 남은 쿠폰 개수를 확인 (COUNT(*) 사용).
남은 쿠폰이 1000개 이상이라면 난수화 방식 적용(2-2).
남은 쿠폰이 1000개 이하라면 LAG() 기반 SYSTIMESTAMP 방식 적용(2-1). */

/* 2-4. 데이터 파티셔닝을 통한 동시 처리 개선
메타데이터에 DB 파티션 추가  */
CREATE TABLE coupon (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(255),
    is_used BOOLEAN
) PARTITION BY HASH(id) PARTITIONS 10;

/* 쿠폰 테이블을 10개의 파티션으로 분할하여 동시성 개선.
각 트랜잭션이 서로 다른 파티션에서 실행되도록 하여 DB 락 경쟁 완화.
트랜잭션 부하를 자동으로 분배하여 대량의 요청을 동시에 처리 가능. */

종합하여,
밀리초(ms) 단위로 쿠폰을 선택하는 최적화 방식 도입.
LAG() 및 SYSTIMESTAMP를 활용하여 연속적인 중복 방지
1000개 이하 쿠폰 처리 시 성능 최적화.
데이터 파티셔닝을 적용하여 동시 처리 능력 향상.
```

#### Conclusion
```
실제 운영 서버는 API(앱 -> 백엔드) 수행이 비동기 처리로 최적화되어 있어 
API 호출 이후 로직 수행과 발송 서버 전달까지만 측정.

기존 거래속도 약 150ms 사이에 동시 쿠폰발행시 DB 데드락 현상으로 인한
쿠폰발급오류 발생 문제를 해결하기 위해 

1. 랜덤 쿠폰 추출 방식 적용, 
2. 트랜잭션 동시성 관리 최적화, 
3. 데이터 파티셔닝을 통한 부하 분산 등의 개선 작업을 수행하여 
거래속도를 70~100ms로 약 50ms(33%)정도 단축시켰으며, dup에러를 최소화 시킴.
이후에도 지속적으로 발생하는 특정 dup문제에 대해서는 이력테이블을 관리하여
해당 고객에게 별도 알림등으로 처리하여 개선
```


