# chatbot-docs

장금상선 그룹 챗봇의 **공개 기술 문서 사이트**. mkdocs Material 로 빌드되어 GitHub Pages 에 발행됩니다.

👉 사용자 매뉴얼(운영): https://sinokorofficial.github.io/chatbot-docs/ 
👉 개발자 문서(전 기능): https://sinokorofficial.github.io/chatbot-docs/dev/

## 구조

```
chatbot-docs/
├── docs/                # 발행 문서 (Markdown · Mermaid)
│   ├── index.md
│   ├── architecture.md · modules.md · services.md · ...
│   ├── code-walkthrough/    # 코드 워크스루 16편
│   └── screenshots/         # 화면 캡처
├── mkdocs.yml           # 사이트 설정 (nav · theme · 제외 문서)
├── requirements.txt     # mkdocs / material / pymdown-extensions
└── .github/workflows/docs.yml   # Pages 자동 빌드/배포
```

> 사내 전용 문서(`admin.md`, `operations.md`, `deployment.md`, `README.md`)는 본 공개 리포에 푸시되지 않습니다.

## 로컬 미리보기

```bash
pip install -r requirements.txt
mkdocs serve            # http://127.0.0.1:8000
mkdocs build --strict   # site/ 정적 산출물
```

## 발행

`main` 에 푸시되면 [`docs.yml`](./.github/workflows/docs.yml) 워크플로가 자동으로 빌드 → GitHub Pages 배포.

## 원본 리포

문서 본문은 비공개 소스 리포에서 관리하고, 공개판은 본 리포로 동기화됩니다. 갱신 절차/제외 규칙은 비공개 측 가이드를 따릅니다.
