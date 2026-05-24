# 네트워크 설계

## 1. 네트워크 대역(요약)

| 영역 | 대역 / 값 | 용도 |
|---|---|---|
| Proxmox 노드망 | `192.168.36.0/24` | Proxmox API, 노드 SSH |
| 관리망 | `172.17.128.0/22` | Ansible SSH, Kubernetes API VIP, MetalLB pool |
| K8s 노드망 | `10.10.10.0/24` | kubelet node IP, control-plane advertise, Calico, Ceph mon/RGW |
| DMZ (VLAN20) | `172.17.32.0/24` | 온프렘 HAProxy VM IP + HAProxy VIP |
| AWS VPC | `10.1.0.0/16` | AWS Public/Private Subnet (ap-northeast-2a/2b) |

## 2. 주요 VIP / 고정 IP

| 항목 | 값 | 제공 |
|---|---|---|
| Kubernetes API VIP | 관리망(`172.17.128.0/22`) | kube-vip |
| ingress-nginx | `172.17.128.240` | MetalLB |
| argocd-server | `172.17.128.241` | MetalLB |
| grafana | `172.17.128.242` | MetalLB |
| 온프렘 HAProxy VIP | `172.17.32.20` | keepalived (DMZ) |
| DMZ gateway | `172.17.32.1` | 온프렘 게이트웨이 |
| Ceph mon / RGW(S3) | `10.10.10.11:6789` / `:7480` | Ceph |

> MetalLB pool: `172.17.128.240-172.17.128.250`. 부트스트랩 초기엔 MetalLB 미준비 가능성이 있어
> ArgoCD/ingress/Grafana는 NodePort로 시작 후 LoadBalancer로 전환한다.

## 3. 트래픽 경로 (외부 → 파드)

```text
client(https)
  → Route53(fhwang.cloud) → AWS NLB :443(TLS 종료) / :80
  → AWS HAProxy EC2 (:80=301 redirect, :8080=복호화 HTTP)
  → [IPsec VPN tunnel]  (사설망 도달)
  → 온프렘 HAProxy VIP 172.17.32.20:80
  → ingress-nginx VIP 172.17.128.240:80 (MetalLB)
  → gateway:5000 → service → pod
```

- **TLS는 AWS NLB(443)에서 종료**하고, 이후 전 구간은 평문 HTTP `80`만 사용한다(온프렘 HAProxy도 80 전용).
- AWS HAProxy가 온프렘 사설 IP(`172.17.32.20`)에 닿는 것은 **VPN 터널**을 통해서다.

## 4. VLAN / 분리 원칙

- **VLAN20 = DMZ**: 외부에서 들어온 트래픽이 처음 만나는 온프렘 경계(HAProxy). 외부 진입은 DMZ까지만 직접 라우팅하고, 내부 K8s 노드망/관리망은 직접 노출하지 않는다.
- **K8s 노드망(`10.10.10.0/24`)**: kubelet/Calico/Ceph 등 클러스터 내부 통신 전용.
- **관리망(`172.17.128.0/22`)**: 운영자 SSH·API·MetalLB. VPN on-prem CIDR 기본값은 DMZ(`172.17.32.0/24`)로 한정해 AWS에서 내부망으로 직접 라우팅하지 않는다.
- 분리 근거는 [../decisions/ADR-002-network-separation.md](../decisions/ADR-002-network-separation.md) 참고.

## 5. 클러스터 내부 네트워킹

- **Calico CNI**: Pod 네트워크. (worker2 calico-node가 간헐 Unknown → 파드 강제 재생성으로 복구한 이력 있음 — `../operations/troubleshooting.md`)
- **kube-vip**: control-plane API VIP HA. 전원 차단 시 페일오버되도록 CP에 softdog/systemd watchdog 적용.
- **MetalLB(L2)**: LoadBalancer Service에 관리망 VIP 할당.
- **ingress-nginx**: 클러스터 진입 게이트. 단일 replica는 SPOF였어서 2 replica + topologySpread + PDB + `externalTrafficPolicy: Cluster`로 HA화.
- **NetworkPolicy**: `apps` 네임스페이스는 ingress-nginx·monitoring 네임스페이스에서의 인입만 허용.
