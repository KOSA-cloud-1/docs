# AWS 통합 (NLB / HAProxy / VPN)

AWS 측 리소스는 `infra-aws` Terraform으로 관리한다. 기존 `terraform/` root module이
`haproxy`, `vpn` 하위 모듈을 조합한다. **EKS는 사용하지 않는다.**

## 1. 공개 진입 (Route53 + NLB + HAProxy)

```text
Route53(fhwang.cloud, www) ─alias─▶ NLB(cross-zone)
  ├─ :443 TLS listener (ACM 인증서로 TLS 종료) ─▶ HAProxy:8080 (복호화 HTTP)
  └─ :80  TCP listener ───────────────────────▶ HAProxy:80 (HTTP→HTTPS 301 redirect)
HAProxy EC2 × 2 (haproxy-a / haproxy-b, Public Subnet) → backend: 온프렘 HAProxy VIP 172.17.32.20:80
```

- **이중화 = active/active**: NLB가 cross-zone LB로 두 EC2에 동시 분산. health check(TCP)로 비정상 인스턴스 제외.
- TLS는 NLB에서 종료, ACM 인증서(`ap-northeast-2`) 사용.
- backend(`haproxy_backends`)는 온프렘 HAProxy VIP의 HTTP 80.

## 2. 사이트 간 VPN (StrongSwan)

```text
StrongSwan EC2 × 2 (vpn-a:prio100 / vpn-b:prio90)  +  서비스 EIP 1개
  ↔ IPsec ↔ 온프렘 ER605 (initiator)
온프렘 CIDR(172.17.32.0/24, 192.168.36.0/24) → active VPN ENI 로 route
```

- **이중화 = active/backup**: EIP·route를 active 인스턴스에만 연결.
- **페일오버**: EventBridge(`rate(1 minute)`) → Lambda(`vpn_failover`)가 인스턴스 헬스 확인 후, 정상·고우선순위 인스턴스로 **EIP 재연결 + route 교체**.
- ER605 Remote Gateway에는 Terraform output `vpn_server_public_ip`(서비스 EIP)를 설정.

## 3. 왜 HAProxy는 active/active, VPN은 active/backup인가

- HAProxy 앞단은 NLB가 L4로 **상태 없이 분산**할 수 있어 active/active가 자연스럽다.
- IPsec 터널은 peer IP가 고정된 **stateful 연결**이라 두 엔드포인트를 동시에 두기 어렵다 → **단일 EIP + 페일오버**로 active/backup.
- 근거: [../decisions/ADR-002-network-separation.md](../decisions/ADR-002-network-separation.md), `infra-aws/README.md`.

## 4. 보안 그룹 / 인증서

- NLB SG: 클라이언트로부터 80/443 허용.
- HAProxy SG: NLB로부터 80/8080만 허용(+ 필요 시 SSH CIDR).
- VPN SG: IKE(500/udp), NAT-T(4500/udp), ESP(proto 50) — peer(ER605) CIDR 한정 + 온프렘/VPC 내부 트래픽.
- ACM 인증서 ARN, VPN PSK는 `terraform.tfvars`에서 실제 값으로 설정(커밋 금지).

## 5. 적용

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # 실제 값 입력
terraform init && terraform plan && terraform apply
# 또는: ./terraform-execute.sh
```

설정 필수 값: `haproxy_backends.address`(온프렘 HAProxy VIP), `nlb_tls_certificate_arn`(ACM),
`vpn_preshared_key`(ER605 PSK).
