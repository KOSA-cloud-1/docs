# 전제 / 가정 (Assumptions)

설계·구축 시 **참(true)으로 가정한 것들**. 틀리면 재검토가 필요한 항목.

## 운영

- AWS Edge(NLB/HAProxy/VPN)만 공인 노출하고, 온프렘은 공인 IP 없이 VPN으로만 유입된다.
- TLS는 AWS NLB(443)에서 종료하고, 이후 전 구간은 평문 HTTP 80으로 충분하다(내부망/VPN 신뢰).
- ArgoCD는 `dev` 브랜치만 동기화한다. `main`은 릴리즈 스냅샷(배포 트리거 아님).
- `DOCKERHUB_USERNAME` = `kosa1team` (overlay `images[].newName`과 일치).

## 데이터 / 시크릿

- 모든 시크릿의 원본은 AWS Secrets Manager(`prod/*`)이며, 클러스터 Secret은 ESO가 재생성 가능하다.
- dev 환경은 운영 DB·Ceph·시크릿을 **공유**한다(공유형). 따라서 dev에서의 파괴적 테스트는 운영 데이터에 영향을 줄 수 있다 → 데모 한정.
- Galera + RBD `Retain`이 1차 데이터 보호이며, 시점 복구가 필요하면 별도 논리 백업을 운영한다.

## 자원

- 모니터링 스택은 전용 노드(worker7, `dedicated=monitoring`)에 격리한다.
- photo-service는 메모리(4Gi/pod)가 커서 worker1/worker2에만 스케줄된다(노드 affinity).
- dev는 평소 0 replica(on-demand)라 추가 자원 점유가 거의 없다.

## 검증 필요(틀릴 수 있는 가정)

- VPN(active/backup) 페일오버 시 짧은 단절은 허용 범위로 본다(1분 주기 감지).
- 시연 시점에 prod에 실제 부하를 주어 HPA 스케일아웃을 보여줄 수 있다.
- MetalLB가 준비된 후 ArgoCD/ingress/Grafana를 LoadBalancer로 전환한다(초기엔 NodePort).
