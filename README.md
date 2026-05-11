# KOSA Cloud Project - Documentation

## 개요

이 레포지토리는 하이브리드 클라우드 인프라 프로젝트의 **설계, 구축, 운영, 의사결정 기록**을 관리하기 위한 문서 저장소입니다.

프로젝트는 온프레미스 환경과 AWS 클라우드를 연결한 **Hybrid Infrastructure**를 기반으로 구성되며, Kubernetes 및 GitOps 기반 배포 구조를 포함합니다.

---

## 전체 시스템 구성 요약

* **On-Premise**

  * Proxmox Cluster
  * Kubernetes Cluster
  * Ceph Storage
  * pfSense 기반 네트워크

* **Cloud (AWS)**

  * VPC
  * EKS
  * Site-to-Site VPN

* **배포 방식**

  * GitHub Actions → Docker Image Build
  * ArgoCD → Kubernetes 배포 (GitOps)

---

## 문서 구조

```
docs/
├─ README.md
├─ architecture/
├─ requirements/
├─ runbooks/
├─ operations/
├─ decisions/
├─ schedules/
├─ presentation/
└─ assets/
```

---

## 디렉토리 설명

### architecture/

시스템의 전체 구조 및 설계를 정의합니다.

* `overview.md` : 전체 아키텍처 개요
* `network-design.md` : VLAN, DMZ, 네트워크 구조
* `hybrid-architecture.md` : 온프레미스 + AWS 연동 구조
* `aws-integration.md` : VPN, VPC, EKS 구성

---

### requirements/

프로젝트의 요구사항 및 전제 조건을 정의합니다.

* `project-scope.md`
* `fixed-conditions.md`
* `assumptions.md`

---

### runbooks/

인프라 구축 절차를 단계별로 정리합니다.

* `proxmox-cluster.md`
* `pfsense-network.md`
* `kubernetes-cluster.md`
* `argocd-deployment.md`
* `mariadb-galera.md`
* `monitoring.md`

---

### operations/

운영 및 장애 대응 관련 문서입니다.

* `deployment-flow.md`
* `backup-restore.md`
* `failure-scenarios.md`
* `troubleshooting.md`

---

### decisions/

아키텍처 및 기술 선택에 대한 의사결정을 기록합니다. (ADR)

* `ADR-001-repository-structure.md`
* `ADR-002-network-separation.md`
* `ADR-003-gitops-with-argocd.md`
* `ADR-004-mariadb-galera-on-ceph.md`

---

### schedules/

프로젝트 일정 및 역할 분담을 관리합니다.

* `2-week-plan.md`
* `role-assignment.md`
* `daily-log.md`

---

### presentation/

발표 및 데모 준비를 위한 문서입니다.

* `final-presentation-outline.md`
* `demo-scenario.md`

---

### assets/

다이어그램 및 이미지 리소스를 저장합니다.

* `diagrams/`
* `images/`

---

## 문서 작성 원칙

* 모든 설계 변경은 반드시 `decisions/`에 기록 (ADR)
* 구축 절차는 `runbooks/`에 단계별로 작성
* 장애 대응 및 운영 관련 내용은 `operations/`에 정리
* 다이어그램은 `assets/`에 저장 후 문서에서 참조

---

## 활용 목적

이 레포지토리는 단순 문서 저장이 아니라 다음을 목표로 합니다:

* 인프라 설계 근거 명확화
* 구축 과정 표준화
* 장애 대응 가능성 확보
* 팀 간 지식 공유
* 발표 및 평가 대응 자료 확보

---

## 참고 흐름

```
설계 (architecture)
→ 구축 (runbooks)
→ 운영 (operations)
→ 개선 및 의사결정 기록 (decisions)
```

---

## 비고

본 프로젝트는 제한된 기간(약 2주) 내 수행되므로
문서는 **핵심 내용 위주로 명확하게 작성**하는 것을 우선합니다.
