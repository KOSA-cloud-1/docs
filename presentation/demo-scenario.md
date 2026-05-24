# 데모 시나리오

라이브 시연은 **prod(`apps` 네임스페이스)** 에서 한다. HPA가 prod에만 있어 스케일아웃을 보여줄 수 있고,
dev는 on-demand(평소 0 replica)라 시연 대상이 아니다(분리 "증빙"으로만 보여줌).

## 시연 흐름

```text
1. 사용자가 AI 프로필 이미지 변환 요청 (https://fhwang.cloud)
2. Ingress request rate 증가          → Grafana: Ingress NGINX 대시보드
3. photo-service Pod CPU/메모리 증가   → Grafana: AI Profile / Workload 대시보드
4. 이미지 변환 처리 시간 증가          → P95 latency 패널
5. HPA가 replica 확장 (photo min2→max4) → Workload 대시보드 replica 그래프
6. 부하 안정화 후 replica/사용률 안정 확인
7. 일부 실패 요청은 Loki에서 ERROR 로그로 추적
8. 변환 결과 저장으로 Ceph(RGW/RBD) 사용량 변화 확인
```

## 사전 준비

```bash
# prod 상태 확인 (모두 Running, HPA 존재)
kubectl -n apps get deploy,hpa,pods
# Grafana 접속 (172.17.128.242) — 7개 대시보드 로드 확인
# dev 분리 "증빙"용 (켤 필요 없음)
kubectl get ns apps apps-dev
kubectl -n argocd get applications | grep kosa-apps   # kosa-apps(운영)/kosa-apps-dev(개발)
```

## 부하 생성 (예시)

```bash
# 외부 도메인으로 변환 요청을 반복 전송 (별도 부하 도구/스크립트)
# 도메인 미사용 시 Host 헤더로:  curl -H "Host: fhwang.cloud" ...
```

## 관전 포인트 (발표 매핑)

| 보여줄 것 | 대시보드/도구 | 발표 섹션 |
|---|---|---|
| 요청 급증 | Ingress NGINX | #7 트래픽 흐름 |
| CPU/메모리 상승 | AI Profile / Memory | #16 모니터링 |
| HPA 스케일아웃 | Workload(replica) | #5 확장성, #17 HA |
| 실패 로그 추적 | Loki | #18 장애 대응 |
| 스토리지 사용량 | (Ceph/PVC) | #12,13 Ceph |

## 장애 주입(선택, #18 보강)

- ingress pod 1개 삭제 → 다른 replica가 처리(무중단) 확인.
- photo-service pod 삭제 → 재스케줄 + HPA 동작.
- (주의) 운영 데이터 공유형이므로 DB 파괴적 조작은 금지.

## 폴백

- 라이브 부하가 어려우면: 사전 캡처한 Grafana 그래프 + `failure-scenarios.md`의 실제 이력으로 설명 대체.
