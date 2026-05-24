# 배포 흐름 (GitOps)

운영 배포의 **기준(source of truth)은 ArgoCD**다. GitHub Actions는 이미지 빌드와 매니페스트
이미지 태그 갱신만 담당하고, 실제 클러스터 반영은 ArgoCD가 한다. `deploy.sh`는 최초 부트스트랩 전용이다.

## 1. dev / prod 환경 분리

한 클러스터 안에서 **네임스페이스 + Kustomize overlay**로 환경을 가른다(브랜치가 아니라 overlay 경로로 구분).

| 축 | dev | prod |
|---|---|---|
| 네임스페이스 | `apps-dev` | `apps` |
| ArgoCD App | `kosa-apps-dev` | `kosa-apps` |
| overlay 경로 | `apps/overlays/dev` | `apps/overlays/prod` |
| 트리거(app repo) | `dev` push | `main` push(릴리즈) |
| replica | 평소 0 (on-demand) | 2~4 (HPA) |

## 2. 파이프라인

```text
[app repo]
  dev  push ──CI(build+push kosa1team/*:<sha>)──▶ k8s-manifests apps/overlays/dev  의 images[].newName/newTag 갱신
  main push ──CI────────────────────────────────▶ k8s-manifests apps/overlays/prod 의 images[].newName/newTag 갱신
                                                          │ (두 갱신 모두 k8s-manifests dev 브랜치로 commit)
                                                          ▼
[ArgoCD]  kosa-platform(app-of-apps, targetRevision: dev)
            ├─ kosa-apps      → apps/overlays/prod → apps(PROD)
            └─ kosa-apps-dev  → apps/overlays/dev  → apps-dev(DEV)
```

- **이미지 레지스트리/태그**는 매니페스트에 하드코딩하지 않고, CI가 `DOCKERHUB_USERNAME`(newName)·커밋 SHA(newTag)를 overlay `images[]`에 주입한다.
- **ArgoCD source 브랜치는 `dev`**. main은 릴리즈 스냅샷일 뿐 ArgoCD가 보지 않는다. (근거: [../decisions/ADR-003-gitops-with-argocd.md](../decisions/ADR-003-gitops-with-argocd.md))

## 3. 운영 승격 (dev → prod)

Git Flow를 따른다(`main`/`dev`/`feature/*`/`release/*`/`hotfix/*`).

1. 기능: `feature/*` → `dev` 머지 → app `dev` push → dev 이미지 빌드 → `apps-dev` 반영
2. 검증: 필요 시 dev 환경을 on-demand로 켜서(`kubectl -n apps-dev scale deploy --all --replicas=1`) 확인
3. 승격: `release/*` → `main` 머지 → app `main` push → prod 이미지 빌드 → `overlays/prod` 태그 갱신 → ArgoCD가 `apps`(prod) 롤링 업데이트

## 4. 부트스트랩(최초 1회)

```bash
cd k8s-manifests
cp external-secrets/secrets.env.example external-secrets/secrets.env  # 실제 값
bash deploy.sh
```

`deploy.sh`는 namespace/ArgoCD/ESO 설치, AWS Secrets bootstrap, Ceph CSI 설치, root Application 적용까지만 한다.
이후 모든 변경은 Git commit/push + ArgoCD sync로 진행한다.

## 5. ArgoCD app-of-apps

- root `kosa-platform`은 `argocd/applications/`를 recurse하며, **자기 자신도 `00-kosa-platform.yaml`로 self-managed**(targetRevision: dev self-heal).
- 자식 App은 sync-wave 순서로 배포(namespaces → metallb → ESO → ingress → storage → data → apps → infra → backup → monitoring).
- 전체 인벤토리는 `k8s-manifests/README.md`의 child Application 표 참고.
