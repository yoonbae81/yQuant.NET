# Race Condition Fix - Position and Deposit Updates

## 문제 (Problem)

### 위험도: 높음 (High Severity)

BrokerGateway의 `UpdatePositionLocalAsync` 및 `UpdateDepositLocalAsync` 메서드에서 **Race Condition (경쟁 상태)** 문제가 발생할 수 있었습니다.

### 원인 (Root Cause)

기존 코드는 **Get → Modify → Set** 패턴을 사용했습니다:

```csharp
// ❌ 문제가 있는 코드 (Before)
var posJson = await db.HashGetAsync(key, order.Ticker);  // 1. Read
// ... 값 수정 ...                                        // 2. Modify
await db.HashSetAsync(key, order.Ticker, ...);           // 3. Write
```

이 패턴은 **원자적(Atomic)이지 않아** 다음과 같은 시나리오에서 **Lost Update** 문제가 발생합니다:

#### 시나리오 1: 부분 체결 (Partial Fills)
```
시간 →
Thread A: Read(Qty=100) → Modify(+10) → Write(Qty=110)
Thread B:      Read(Qty=100) → Modify(+5) → Write(Qty=105) ❌
```
**결과**: Thread A의 업데이트가 손실됨. 최종 수량은 105 (올바른 값: 115)

#### 시나리오 2: 동시 주문 처리
```
시간 →
Order 1: Read(Balance=10000) → Modify(-1000) → Write(9000)
Order 2:      Read(Balance=10000) → Modify(-500) → Write(9500) ❌
```
**결과**: Order 1의 업데이트가 손실됨. 최종 잔액은 9500 (올바른 값: 8500)

## 해결책 (Solution)

### Redis Lua Script를 사용한 원자적 업데이트

Lua 스크립트는 Redis 서버에서 **단일 원자적 작업**으로 실행되므로 Race Condition을 완전히 방지합니다.

### 구현 파일

1. **`RedisLuaScripts.cs`** - Lua 스크립트 정의
   - `UpdatePositionScript`: 포지션 원자적 업데이트
   - `UpdateDepositScript`: 입금/출금 원자적 업데이트

2. **`Worker.cs`** - 수정된 메서드
   - `UpdatePositionLocalAsync`: Lua 스크립트 사용
   - `UpdateDepositLocalAsync`: Lua 스크립트 사용

3. **`RaceConditionTests.cs`** - 포괄적인 테스트
   - 동시 매수 주문 테스트
   - 부분 체결 시뮬레이션
   - 매수/매도 혼합 테스트
   - 스트레스 테스트 (100개 동시 주문)

## 주요 변경사항 (Key Changes)

### Before (문제 있는 코드)
```csharp
private async Task UpdatePositionLocalAsync(IDatabase db, string alias, Order order)
{
    // ❌ Non-atomic: Get → Modify → Set
    var posJson = await db.HashGetAsync(key, order.Ticker);
    Position position = /* deserialize and modify */;
    await db.HashSetAsync(key, order.Ticker, JsonSerializer.Serialize(position));
}
```

### After (수정된 코드)
```csharp
private async Task UpdatePositionLocalAsync(IDatabase db, string alias, Order order)
{
    // ✅ Atomic: Single Lua script execution
    var result = await db.ScriptEvaluateAsync(
        RedisLuaScripts.UpdatePositionScript,
        new RedisKey[] { key },
        new RedisValue[] { ticker, action, qty, price, ... }
    );
}
```

## Lua Script 동작 원리

### Position Update Script

```lua
-- 1. 기존 포지션 읽기 (또는 새로 생성)
local posJson = redis.call('HGET', key, ticker)
local position = posJson and cjson.decode(posJson) or { Qty = 0, AvgPrice = 0, ... }

-- 2. 매수/매도에 따라 수정
if action == 'Buy' then
    -- 평균 단가 재계산
    position.AvgPrice = ((oldQty * oldAvgPrice) + (newQty * price)) / (oldQty + newQty)
    position.Qty = position.Qty + qty
elseif action == 'Sell' then
    position.Qty = position.Qty - qty
end

-- 3. 수량이 0이면 삭제, 아니면 저장
if position.Qty <= 0 then
    redis.call('HDEL', key, ticker)
else
    redis.call('HSET', key, ticker, cjson.encode(position))
end
```

**핵심**: 이 모든 작업이 **단일 원자적 트랜잭션**으로 실행됩니다.

## 테스트 커버리지

### 1. 동시 매수 주문 테스트
```csharp
[TestMethod]
public async Task UpdatePosition_ConcurrentBuyOrders_ShouldNotLoseUpdates()
```
- 10개의 동시 매수 주문 실행
- 모든 주문이 정확히 반영되는지 검증
- 평균 단가가 올바르게 계산되는지 검증

### 2. 부분 체결 시뮬레이션
```csharp
[TestMethod]
public async Task UpdatePosition_ConcurrentPartialFills_ShouldNotLoseUpdates()
```
- 20개의 부분 체결을 동시에 처리
- Lost Update가 발생하지 않는지 검증

### 3. 매수/매도 혼합 테스트
```csharp
[TestMethod]
public async Task UpdatePosition_ConcurrentBuyAndSell_ShouldMaintainCorrectQuantity()
```
- 매수와 매도 주문을 동시에 실행
- 최종 수량이 정확한지 검증

### 4. 스트레스 테스트
```csharp
[TestMethod]
public async Task UpdatePosition_StressTest_100ConcurrentOrders()
```
- 100개의 무작위 주문을 동시에 실행
- 데이터 무결성 검증

### 5. 입금/출금 테스트
```csharp
[TestMethod]
public async Task UpdateDeposit_ConcurrentBuyOrders_ShouldNotLoseUpdates()
```
- 동시 거래 시 잔액이 정확히 업데이트되는지 검증

## 테스트 실행 방법

### 전제 조건
Redis 서버가 로컬에서 실행 중이어야 합니다:
```bash
# Docker를 사용하는 경우
docker run -d -p 6379:6379 redis:latest

# 또는 Windows용 Redis 설치
```

### 테스트 실행
```bash
# 전체 테스트 실행
dotnet test tests/yQuant.App.BrokerGateway.Tests/yQuant.App.BrokerGateway.Tests.csproj

# 특정 테스트만 실행
dotnet test --filter "FullyQualifiedName~RaceConditionTests"
```

## 성능 영향

### Lua Script의 장점
1. **원자성 보장**: Race Condition 완전 방지
2. **네트워크 왕복 감소**: Get + Set 2번 → ScriptEvaluate 1번
3. **서버 측 실행**: 네트워크 지연 최소화

### 예상 성능
- **기존**: ~2-4ms (Get + 로직 + Set, 2 round trips)
- **개선**: ~1-2ms (ScriptEvaluate, 1 round trip)
- **동시성**: 무제한 (Redis의 단일 스레드 특성으로 자동 직렬화)

## 추가 고려사항

### 1. Lua Script 캐싱
Redis는 Lua 스크립트를 자동으로 캐싱하므로, 반복 실행 시 성능이 더욱 향상됩니다.

### 2. 에러 처리
Lua 스크립트 내에서 에러가 발생하면 전체 작업이 롤백되어 데이터 일관성이 보장됩니다.

### 3. 모니터링
```csharp
if (_logger.IsEnabled(LogLevel.Debug))
{
    _logger.LogDebug("Updated position for {Ticker} in {Alias}: {Position}", 
        order.Ticker, alias, result.ToString());
}
```

## 결론

이 수정으로 BrokerGateway는 다음을 보장합니다:

✅ **데이터 무결성**: Lost Update 문제 완전 해결  
✅ **동시성 안전**: 무제한 동시 주문 처리 가능  
✅ **성능 향상**: 네트워크 왕복 50% 감소  
✅ **테스트 커버리지**: 포괄적인 Race Condition 테스트  

---

**작성일**: 2025-12-09  
**작성자**: Antigravity AI  
**관련 파일**:
- `src/03.Applications/yQuant.App.BrokerGateway/RedisLuaScripts.cs`
- `src/03.Applications/yQuant.App.BrokerGateway/Worker.cs`
- `tests/yQuant.App.BrokerGateway.Tests/RaceConditionTests.cs`
