# 🧪 테스트 전략과 품질 게이트

## 1. 원칙

| 원칙 | 적용 |
|---|---|
| **재현 테스트 먼저** | 버그를 고치기 전에 그 버그를 재현하는 실패 테스트를 먼저 작성(Red-Green) |
| **상태 조합은 표로** | 플랜·볼륨·호스트 상태 조합을 테이블 테스트로 고정해 회귀 방지 |
| **DB 없이 테스트** | 트랜잭션 안에서 실행되는 로직을 별도 함수로 분리해 mock만으로 검증 |
| **완료의 정의** | lint → build → test 결과가 있어야만 완료로 인정 |

```go
// 설명용 예시 (실제 코드 아님)
// 트랜잭션 경계와 비즈니스 로직을 분리하면, 로직은 DB 없이 테스트할 수 있다.
func (s *Service) StopPlan(ctx context.Context, id PlanID) error {
    return s.db.WithTx(ctx, func(tx Tx) error {
        return s.stopPlanInTx(ctx, tx, id)
    })
}

func (s *Service) stopPlanInTx(ctx context.Context, tx Tx, id PlanID) error {
    // 상태 확인 → 전이 → 기록 (테스트에서는 tx를 mock으로 대체)
    ...
}
```

## 2. 규모와 결과

- 컷오버·복제 시작·상태 조회 시나리오 단위 테스트 **2,700건 이상** (mock repository·replicator 기반)
- 멀티 스토리지 복제 성공·부분 실패 롤백 트레이스 테스트
- SonarQube **new_coverage 80% 게이트 통과**, 인지복잡도 위반 96 → 3건

## 3. E2E

- 별도 저장소로 **gRPC 테스트 콘솔**을 만들고 Kubernetes에 배포
- **가짜 데몬 + 시나리오 러너**(상태 확인 → Stop → STOPPED 대기 → Delete)로 실제 데몬 없이도 흐름을 재현
- 헤드리스 Chrome + CDP로 콘솔 화면 스모크 테스트
- CI 파이프라인에서 자동 빌드·검증
