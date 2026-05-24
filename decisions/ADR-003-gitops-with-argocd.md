# ADR-003: GitOps with ArgoCD

- 상태: 채택(Accepted)

## Context

여러 컴포넌트(앱·DB·스토리지·모니터링·인프라 리소스)를 재현 가능하게 배포·운영해야 한다.
GUI 수동 조작은 재현성·감사·롤백이 약하다.

## Decision

- **ArgoCD app-of-apps**(`kosa-platform`)가 클러스터 steady-state를 소유한다.
- **GitOps source = `k8s-manifests`의 `dev` 브랜치.** 환경 구분은 **브랜치가 아니라 Kustomize overlay 경로**로 한다(`overlays/prod`→`apps`, `overlays/dev`→`apps-dev`).
- **GitHub Actions는 이미지 빌드 + 매니페스트 image 태그 갱신만** 한다(steady-state 변경 금지).
- root Application은 **self-managed**(`argocd/applications/00-kosa-platform.yaml`) → `targetRevision: dev` self-heal로 수동 드리프트 방지.
- 자식 App은 sync-wave로 순서 보장, `automated: {prune, selfHeal}`.

## Consequences

- (+) 선언적·재현 가능·자동 복구. Git이 단일 진실원.
- (+) 단일 브랜치(dev) + overlay → 브랜치 난립 없이 dev/prod 분리.
- (−) `main`은 릴리즈 스냅샷일 뿐 **ArgoCD가 보지 않음** → "main에 올리면 배포된다"는 오해 주의(배포는 overlay 태그 갱신으로).
- (−) squash 머지 시 feature 브랜치가 "미머지"처럼 보임(ancestry 미포함) — 내용은 반영됨.
- (−) 루트앱 수동 revision 변경이 사고를 낸 적 있음 → self-managed로 완화([../operations/failure-scenarios.md](../operations/failure-scenarios.md) 2.4).
