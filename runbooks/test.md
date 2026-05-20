# AWS ↔ 온프레미스 하이브리드 인프라 테스트 체크리스트

---

# 1. 네트워크 연결

## AWS EC2 → 온프레미스

### Ping 테스트

```bash
ping 172.17.128.21
```

### 확인 항목

- 패킷 손실 여부
- Latency 확인
- VPN 경유 여부

---

## 온프레미스 → AWS EC2

### Ping 테스트

```bash
ping <ec2-private-ip>
```

### 확인 항목

- AWS Security Group 허용 여부
- NACL 허용 여부
- VPN Phase2 CIDR 정상 여부

---

# 2. 라우팅 / VPN

## AWS VPN 상태 확인

### 확인 항목

- Tunnel UP 여부
- BGP 상태
- Bytes In/Out 증가 여부

### AWS 콘솔 경로

```text
VPC
└── Site-to-Site VPN
    └── Tunnel Details
```

---

# 3. DNS

## AWS → 온프레미스 DNS 조회

```bash
nslookup 172.17.128.240
```

### 확인 항목

- 온프레미스 DNS 응답 여부
- Reverse DNS 정상 여부

---

## 온프레미스 → Route53 조회

```bash
nslookup fhwang.cloud
```

### 확인 항목

- Route53 정상 응답 여부
- VPN 경유 DNS 통신 여부

---

# 4. Kubernetes / VM 통신

## kube-vip 정상 여부 확인

### kube-apiserver 확인

```bash
kubectl get pods -n kube-system -o wide | grep kube-apiserver
```

### etcd 확인

```bash
kubectl get pods -n kube-system -l component=etcd -o wide
```

---

## Leader Node 변경 테스트

### 테스트 절차

1. cp1 종료
2. VIP 이동 여부 확인

### 확인 항목

- kube-vip failover 정상 여부
- VIP 재할당 여부

---

## API HA 테스트

```bash
kubectl get nodes
```

### 확인 항목

- kubectl 정상 동작 여부
- VIP failover 정상 여부
- API Server 연결 유지 여부

---

## Pod 간 통신 테스트

### 테스트 Pod 생성

```bash
kubectl run test --rm -it --image=busybox -- sh
```

### Pod 내부 테스트

```bash
ping <pod-ip>
wget <service-name>
```

### 확인 항목

- CNI 정상 동작 여부
- Service DNS 정상 여부

---

## Service / Ingress 테스트

```bash
curl http://<ingress-ip>
```

### 확인 항목

- ingress-nginx 정상 여부
- MetalLB External IP 정상 여부
- Service 라우팅 정상 여부

---

# 5. 스토리지

## Ceph 상태 확인

```bash
ceph -s
```

### 확인 항목

- HEALTH_OK 여부
- OSD up/in 상태
- Degraded 여부

---

## RBD 마운트 테스트

```bash
kubectl get pvc,pv
```

### 확인 항목

- PVC Bound 상태
- PV 정상 연결 여부
- Pod 마운트 정상 여부

---

## S3 백업 테스트

### 업로드 테스트

```bash
aws s3 cp test.txt s3://bucket/
```

---

### 다운로드 테스트

```bash
aws s3 cp s3://bucket/test.txt .
```

---

### 대용량 업로드 테스트

#### 테스트 파일 생성

```bash
dd if=/dev/zero of=1g.img bs=1M count=1024
```

#### 업로드 테스트

```bash
aws s3 cp 1g.img s3://bucket/
```

### 확인 항목

- Multipart Upload 여부
- Timeout 발생 여부
- 업로드 속도 확인

---

# 6. 장애 복구 (HA)

## Control Plane 장애 테스트

### 테스트 절차

```bash
shutdown -h now
```

### 확인 항목

- API 정상 동작 여부
- kube-vip failover 여부
- Remaining Control Plane 정상 여부

---

## Worker 장애 테스트

### 테스트 절차

```bash
shutdown -h now
```

### Pod 재배치 확인

```bash
kubectl get pods -o wide -w
```

### 확인 항목

- Pod reschedule 정상 여부
- Service 영향 여부

---

## VPN 장애 테스트

### 테스트 절차

- Tunnel 1 비활성화

### 확인 항목

- Tunnel failover 여부
- BGP 재수렴 시간
- 서비스 영향 여부

---

# 7. 보안

## Security Group 최소 허용 확인

### 포트 스캔 테스트

```bash
nmap <target-ip>
```

### etcd 외부 접근 차단 확인

```bash
nc -zv <cp-ip> 2379
```

### 확인 항목

- 불필요한 포트 노출 여부
- etcd 외부 접근 차단 여부

---

## Kubernetes Secret 확인

```bash
kubectl get secrets -A
```

### 확인 항목

- 불필요한 Secret 존재 여부
- Namespace 별 Secret 관리 여부

---

# 8. 모니터링

## Prometheus Target 확인

### Port Forward

```bash
kubectl port-forward svc/prometheus 9090:9090
```

### 확인 항목

- Target UP 상태
- Node Exporter 정상 여부
- kube-state-metrics 정상 여부

---

## Alert 테스트

### 테스트 절차

- node-exporter 중지

### 확인 항목

- Alertmanager Alert 발생 여부
- Slack / Discord / Email 알림 여부

---

## 로그 수집 확인

### Fluentbit / Loki / ELK 확인

```bash
kubectl logs <pod-name>
```

### 확인 항목

- 로그 수집 여부
- 로그 검색 가능 여부
- 로그 유실 여부

---

# 9. 성능 테스트

## 네트워크 성능 테스트

### Server

```bash
iperf3 -s
```

### Client

```bash
iperf3 -c <ip>
```

### 확인 항목

- 10G 처리량 여부
- Packet Loss 여부
- MTU 영향 여부

---

## Kubernetes 부하 테스트

### Deployment Scale 테스트

```bash
kubectl scale deployment nginx --replicas=100
```

### 확인 항목

- Scheduler 정상 동작 여부
- Resource 부족 여부
- Ceph / Network 부하 상태
- Pod 생성 속도

---

# 최종 점검 항목

| 항목 | 확인 여부 |
|------|------|
| AWS ↔ 온프레미스 통신 | ☐ |
| VPN Tunnel 이중화 | ☐ |
| DNS 정상 응답 | ☐ |
| Kubernetes HA | ☐ |
| kube-vip Failover | ☐ |
| Ceph 상태 정상 | ☐ |
| S3 백업 정상 | ☐ |
| Worker 장애 복구 | ☐ |
| Prometheus 모니터링 | ☐ |
| Alert 정상 동작 | ☐ |
| 10G 네트워크 성능 | ☐ |

---