# Ceph 스토리지 용량 계획
| 구분            |       사용량 |     여유 공간 |      총 할당 |
| ------------- | --------: | --------: | --------: |
| RBD           |     1.8TB |     0.2TB |     2.0TB |
| S3            |     0.3TB |     0.5TB |     0.8TB |
| **총 사용 계획**   | **2.1TB** | **0.7TB** | **2.8TB** |
| 추가 여유 공간      |         - |     0.7TB |     0.7TB |
| **Ceph 총 용량** |           |           | **3.5TB** |


# VM 별 저장공간 할당
| VM      | 용도            |       디스크 |
| ------- | ------------- | --------: |
| cp1     | Control Plane |     150GB |
| cp2     | Control Plane |     150GB |
| cp3     | Control Plane |     150GB |
| worker1 | 서비스           |     200GB |
| worker2 | 서비스           |     200GB |
| worker3 | 서비스 + Galera  |     250GB |
| worker4 | 서비스 + Galera  |     250GB |
| worker5 | 서비스 + Galera  |     250GB |
| worker6 | 서비스           |     200GB |
| **합계**  |               | **1.8TB** |
