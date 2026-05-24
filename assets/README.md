# assets

다이어그램·이미지 리소스를 보관한다.

```text
assets/
├─ diagrams/   # 아키텍처/네트워크/트래픽 흐름 다이어그램 (drawio, svg, png)
└─ images/     # 스크린샷(Grafana 대시보드 등), 발표용 이미지
```

## 원칙

- 문서에서 상대 경로로 참조한다. 예: `![아키텍처](../assets/diagrams/overview.png)`
- 원본 편집 파일(drawio 등)과 내보낸 이미지(png/svg)를 함께 둔다.
- 발표용 Grafana 캡처는 시연 폴백 자료로도 사용(`../presentation/demo-scenario.md`).
