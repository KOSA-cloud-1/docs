# 장애 시나리오 / 가용성

각 계층의 이중화 방식과, 단일 구성요소 장애 시 동작을 정리한다.

## 1. 계층별 HA 동작

| 계층 | 방식 | 1개 장애 시 |
|---|---|---|
| AWS HAProxy | active/active (NLB) | NLB가 health check로 제외, 나머지 1대가 처리 |
| AWS VPN(StrongSwan) | active/backup | Lambda(1분)가 EIP·route를 정상 인스턴스로 이전 |
| 온프렘 HAProxy | active/backup (VRRP) | PRIMARY 다운 → BACKUP이 VIP 인수(우선순위 110/100) |
| ingress-nginx | active/active (2 replica) | 다른 replica가 처리. `externalTrafficPolicy: Cluster`로 노드 단위 장애에도 포워딩 |
| K8s control-plane | kube-vip + CP×3 | API VIP가 정상 CP로 페일오버 |
| MariaDB Galera | 3-replica multi-primary | 나머지 노드로 쓰기/읽기 지속 |
| 앱(gateway/photo 등) | HPA + PDB + topologySpread | replica 분산, drain 시 최소 가용 보장 |

## 2. 알려진 장애 이력 / 교훈

### 2.1 ingress-nginx 단일 replica SPOF
- 증상: ingress pod가 단일이라 죽으면 MetalLB VIP announce 중단 → 앞단 HAProxy backend DOWN → 502.
- 조치: **2 replica + topologySpread + PDB(minAvailable 1) + externalTrafficPolicy Cluster**로 HA화.

### 2.2 worker2 calico-node 간헐 Unknown (재발성)
- 증상: worker2의 calico-node가 주기적으로 Unknown → 해당 노드 파드 네트워킹 영향. 한 번은 ingress가 그 노드에 몰려 영향.
- 조치: calico-node 파드 강제 재생성으로 복구. ingress가 한 노드에 고정되지 않도록 분산.

### 2.3 kube-vip 페일오버 (전원 차단)
- 증상: CP 노드 **행(hang)이 아니라 전원 차단** 시 VIP 페일오버.
- 조치: CP에 softdog/systemd watchdog(playbook 11) 적용으로 페일오버 보장.

### 2.4 ArgoCD 루트앱 revision 드리프트
- 증상: `kosa-platform` 루트앱이 수동으로 옛 feature 브랜치를 가리켜, `kosa-apps`가 OutOfSync(옛 정의로 restructured `apps/`를 raw apply 시도). prod는 마지막 정상본으로 가동 중.
- 조치: 루트앱 `targetRevision: dev` 복구 + **self-managed(`00-kosa-platform.yaml`)로 재발 방지.**
- 교훈: **ArgoCD가 이상하면 live 루트앱 revision부터 확인.**

### 2.5 Proxmox 메모리 ballooning OOM 위험
- k8s VM에 ballooning ON이면 OOM 위험 → `qm set --balloon 0`. tfvars 드리프트로 일부 worker가 축소될 수 있어 k8s-vm은 신중히 apply.

## 3. 장애 점검 우선순위(체크 순서)

1. 외부 접속 불가: Route53 → NLB target health → AWS HAProxy → **VPN 터널 up?** → 온프렘 HAProxy VIP → ingress
2. 일부 서비스만 오류: 해당 Deployment/HPA/파드 상태 + 로그(Loki) + ServiceMonitor 타깃
3. 전반적 sync 이상: ArgoCD App 상태 + **루트앱 revision** + sync error
4. DB 오류: Galera 멤버십(`wsrep_ready`)·RBD PVC Bound 여부

구체 명령은 [troubleshooting.md](troubleshooting.md) 참고.
