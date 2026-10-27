# 🏗 아키텍처

> 실제 제품 코드와 내부 명칭은 제외하고, 구조와 설계 판단을 일반화해 정리했습니다.

## 1. 계층 구조

```
gRPC API  ─▶  Service  ─▶  Strategy (복제 타입별)  ─▶  Storage / Agent 연동
                 │
                 ├─▶ Repository (CockroachDB)
                 └─▶ Background Workers (폴러 · 워처 · 컷오버 · 스냅샷 스케줄)
```

| 계층 | 책임 |
|---|---|
| **gRPC API** | 복제 시작·중지·동기화·상태 조회·재시작·플랜 업데이트, 스냅샷 생성·조회 |
| **Service** | 플랜 단위 상태 전이, 동시 실행 직렬화, 트랜잭션 경계 |
| **Strategy** | 클라우드 간 / 물리 서버 / VM 복제의 서로 다른 절차를 같은 인터페이스로 분리 |
| **Repository** | 플랜·볼륨·호스트·스냅샷 메타데이터 저장, 스키마 마이그레이션 |
| **Workers** | 복제 상태 폴링, 미러 완료 감시, 컷오버, 주기 스냅샷 실행 |

### 왜 Strategy 패턴인가
복제 타입마다 "시작 → 초기 동기화 → 상태 조회 → 정지·삭제"의 큰 흐름은 같지만 세부 절차가 완전히 다릅니다.
흐름은 Service가, 세부 절차는 Strategy가 맡게 해서 새 복제 타입(예: RHV → OpenStack)을 추가할 때 기존 경로를 건드리지 않도록 했습니다.

```go
// 설명용 예시 (실제 코드 아님)
type ReplicationStrategy interface {
    Start(ctx context.Context, plan Plan) error
    Status(ctx context.Context, plan Plan) (PlanStatus, error)
    Stop(ctx context.Context, plan Plan) error
}
```

## 2. OpenStack 간 복제 경로

| 경로 | 방식 | 고려한 점 |
|---|---|---|
| **Ceph → Ceph** | rbd-mirror 기반 이미지 복제 (pool peer 등록·해제, image mirror enable/disable) | 미러 완료를 비동기로 감시하는 워처, 완료 시각을 CAS로 한 번만 기록 |
| **Ceph → Device/File** | 소스 스냅샷 기반 동기화 회차, 스케줄러·락 | 실패 시 소스 스냅샷 롤백, restart 지원 |
| **Device/File → Device/File** | 리플리케이터 기반 멀티 스토리지 복제 | 인스턴스별 다중 RPC → 플랜 단위 단일 배치 요청으로 처리량 개선 |

## 3. HA (고가용성)

- 여러 데몬 노드 중 **리더만** 쓰기 작업·스케줄 실행·워커를 담당
- 리더가 아닌 노드로 온 쓰기 요청은 **리더로 포워딩**, 리더 확정 전에는 서빙 게이트로 요청을 막음
- 리더 교체 시 진행 중이던 작업이 중복 실행되거나 끝까지 실행되지 않도록 컨텍스트 취소와 상태 확인을 함께 설계

## 4. 실패와 재기동에 강한 설계

| 상황 | 처리 |
|---|---|
| 복제 시작 중 일부 볼륨 실패 | 이미 만든 리소스(peer, 타겟 볼륨, 스냅샷)를 되돌리는 **보상 처리**, defer 기반 자동 정리 |
| 데몬 재시작 | 진행 중 작업(스냅샷·컷오버·미러 완료 감시)을 DB 플래그로 찾아 **자동 재개(resume)** |
| 상태 조회 중 외부 시스템 지연 | 컷오버 중에는 DB 기준으로 응답해 불필요한 외부 호출 감소 |
| 시간 값 이상(0·음수 경과 시간) | 원인 분석 후 완료 시각 기록 시점과 계산식을 보정 |

## 5. 기술 스택

Go · gRPC · CockroachDB · OpenStack(Nova, Cinder, Neutron) · Ceph RBD / rbd-mirror · Kubernetes · Operator SDK · SonarQube
