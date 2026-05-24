# 트러블슈팅 (명령 모음)

> 온프렘은 로컬 kubectl이 없으면 cp1(관리망)에 SSH 후 실행한다.

## 1. 외부 접속 불가 (502 등)

```bash
# AWS: NLB target health
aws elbv2 describe-target-health --target-group-arn <tg-arn>
# VPN 터널 / 라우팅 (AWS route table이 active VPN ENI를 가리키는지)
# 온프렘 HAProxy
ssh <haproxy-vm>; systemctl status haproxy keepalived; ip -br addr | grep 172.17.32.20
haproxy -c -f /etc/haproxy/haproxy.cfg
# ingress
kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx get svc ingress-nginx-controller   # EXTERNAL-IP=172.17.128.240?
```

## 2. ArgoCD / GitOps

```bash
# 앱 전체 상태
kubectl -n argocd get applications -o custom-columns=NAME:.metadata.name,REV:.spec.source.targetRevision,SYNC:.status.sync.status,HEALTH:.status.health.status
# ★ 루트앱 revision 확인 (이상 시 1순위) — dev 여야 함
kubectl -n argocd get application kosa-platform -o jsonpath='{.spec.source.targetRevision}{"\n"}'
# 드리프트 복구
kubectl -n argocd patch application kosa-platform --type merge -p '{"spec":{"source":{"targetRevision":"dev"}}}'
# 막힌 앱 강제 재동기화: 리소스 직접 삭제보다 App 삭제→app-of-apps 재생성이 안정적
kubectl -n argocd delete application <name>
```

## 3. 노드 / CNI

```bash
kubectl get nodes -o wide
# worker2 calico-node Unknown → 강제 재생성
kubectl -n kube-system delete pod -l k8s-app=calico-node --field-selector spec.nodeName=worker2
# kube-vip API VIP 점유 CP 확인
kubectl -n kube-system get pods -l app.kubernetes.io/name=kube-vip -o wide
```

## 4. dev 환경 on-demand 기동/종료

```bash
kubectl -n apps-dev scale deploy --all --replicas=1   # 검증 시 켜기
kubectl -n apps-dev rollout status deploy/photo-service
kubectl -n apps-dev scale deploy --all --replicas=0   # 끄기
```

## 5. 시크릿 (ExternalSecret)

```bash
kubectl get externalsecret -A
kubectl -n apps get secret mariadb-secret ceph-secret
# 강제 동기화
kubectl -n <ns> annotate externalsecret <name> force-sync=$(date +%s) --overwrite
```

## 6. 모니터링 / 메트릭

```bash
# Prometheus 타깃에 메트릭 들어오는지 (ingress 대시보드 빈 값 디버깅 등)
# 트래픽 무관 메트릭이 있으면 스크레이프는 정상, request rate 패널이 비면 트래픽 부족
kubectl -n monitoring get servicemonitor
kubectl -n ingress-nginx get svc ingress-nginx-controller-metrics   # :10254
```

## 7. 스토리지 (Ceph)

```bash
kubectl get pvc -A | grep -i pending          # Bound 안 되면 CSI/권한 확인
kubectl -n ceph-csi-rbd get pods               # CSI driver Running?
kubectl -n data get pods                        # Galera, wsrep_ready
```

## 8. 빈 ReplicaSet 누적 (ArgoCD UI 클러터)

잦은 롤아웃으로 `apps`에 0-replica RS가 쌓이면 → Deployment `revisionHistoryLimit: 3` 적용(자동 GC).
(파드가 아니라 0개짜리 RS이며 리소스는 안 먹음)
