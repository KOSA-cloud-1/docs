# 백업 / 복구

## 1. 대상과 방식

| 대상 | 방식 | 위치 |
|---|---|---|
| 사진 객체(이미지) | Ceph RGW(S3) 버킷에 저장 + **백업 CronJob → AWS S3** | `backup` ns CronJob |
| DB(employees) | MariaDB Galera 3-replica(자체 다중화) + 논리 백업 | `data` ns |
| Grafana / Alertmanager 설정 | PVC + ExternalSecret(`prod/monitoring/*`) | `monitoring` ns |
| 시크릿 원본 | **AWS Secrets Manager**(`prod/*`)가 원본 | AWS |

## 2. 사진 백업 CronJob (`backup` ns)

- `backup` 네임스페이스의 CronJob이 주기적으로 실행되어, Ceph RGW의 사진 객체를 **AWS S3 목적지 버킷**으로 복제한다.
- 자격증명은 ExternalSecret로 주입: `ceph-secret`(원본 RGW), `aws-secret`(`prod/aws`, 목적지 S3).
- 완료된 Job/Pod는 CronJob history limit만큼만 유지된다(정상 동작이며 "죽은 파드"가 아님).

확인:
```bash
kubectl -n backup get cronjob,job,pod
kubectl -n backup logs job/<photo-backup-...>
```

## 3. DB (Galera)

- Galera는 `data` ns에서 **3-replica multi-primary**로 동작 → 노드 1대 장애에도 가용.
- 각 Pod는 독립 RWO RBD 볼륨(`ceph-rbd`, reclaimPolicy `Retain`).
- 앱 접속점: `mysql.data.svc.cluster.local`.
- 논리 백업(mysqldump 등)을 별도 주기로 권장(스키마/데이터 시점 복구용). 현 구성은 Galera 복제 + RBD `Retain`이 1차 보호.

## 4. 시크릿 / 설정

- 모든 시크릿 값의 **원본은 AWS Secrets Manager**. 클러스터 Secret은 ESO가 재생성하므로,
  네임스페이스/Secret이 사라져도 `external-secrets/`를 재적용(또는 ArgoCD sync)하면 복구된다.
- `external-secrets/aws-secretsmanager-credentials`(ESO bootstrap 자격증명)만 Git 밖에서 생성.

## 5. 복구 시나리오(요약)

| 상황 | 복구 |
|---|---|
| 앱 파드/Deployment 손상 | ArgoCD re-sync (Git이 desired state) |
| 네임스페이스 Secret 소실 | ESO refresh / `external-secrets/` 재적용 → AWS Secrets Manager에서 재생성 |
| Galera Pod 1대 손실 | 재스케줄 → 나머지 노드에서 SST/IST 재동기화 |
| 사진 객체 손실 | AWS S3 백업 버킷에서 RGW로 복원 |
| 클러스터 재구축 | `infra-proxmox`(Terraform+Ansible) → `k8s-manifests/deploy.sh` 부트스트랩 → ArgoCD가 전체 복원 |

> RBD StorageClass는 `Retain`이라 PVC 삭제 시 데이터가 즉시 삭제되지 않는다. 의도적 정리 시 수동 회수 필요.
