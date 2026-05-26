# ADR-004: MariaDB Galera on Ceph RBD

- 상태: 채택(Accepted)

## Context

온프렘에서 관계형 DB(직원/프로필 메타데이터)를 **고가용**으로 운영해야 한다.
단일 MariaDB는 SPOF이고, 영속 스토리지도 노드 종속이면 안 된다.

## Decision

- **MariaDB Galera 3-replica multi-primary** StatefulSet(`data` ns).
- 각 Pod는 **독립 RWO Ceph RBD PVC**(`ceph-rbd` StorageClass, reclaimPolicy `Retain`).
- 앱 접속점: `mysql.data.svc.cluster.local`.
- Galera 노드는 전용 노드(label `db-role=galera`, worker3~5)에 배치.

## Consequences

- (+) 노드 1대 장애에도 쓰기/읽기 지속(동기 복제 multi-primary).
- (+) RBD로 스토리지가 노드와 분리 → 재스케줄 시 볼륨 재연결.
- (+) `Retain`으로 PVC 삭제 시 데이터 즉시 소실 방지.
- (−) Galera 동기 복제 특성상 쓰기 지연·인증서(certification) 충돌 가능 → 핫스팟 쓰기 주의.
- (−) `Retain`은 수동 회수 필요(고아 RBD 이미지 정리).
- 참고: AWS RDS Multi-AZ 대비 **온프렘 자체 HA**를 택한 비교는 [ADR-002](ADR-002-network-separation.md) 및 비용 분석(발표 자료) 참조.
