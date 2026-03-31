# 🧠 Claude TIL Vault

Claude와의 기술 Q&A를 체계적으로 아카이빙하는 Obsidian vault.

## 폴더 구조

```
claude-til-vault/
├── README.md
├── _templates/          # Obsidian 템플릿
│   └── til-template.md
├── _widgets/            # 인터랙티브 HTML 위젯 보관
│   └── .gitkeep
├── db-internals/        # B+Tree, InnoDB, Index, MySQL
├── kafka-distributed/   # Kafka, Spark, 분산시스템
├── jvm-kotlin/          # JVM internals, Kotlin, GraalVM
├── spring-jpa/          # Spring Boot, JPA/Hibernate
├── infra/               # K8s, Valkey/Redis, Docker
├── architecture/        # 설계 패턴, 시스템 아키텍처
├── ai-tooling/          # Claude, MCP, AI IDE, 프롬프팅
└── misc/                # 분류 전 임시 보관
```

## 사용법

### 1. TIL 작성 (Claude 대화 후)
Claude에게 `"이거 TIL로 정리해줘"` 라고 말하면:
- 마크다운 문서 → 해당 카테고리 폴더에 저장
- 인터랙티브 위젯 → `_widgets/` 폴더에 .html로 저장

### 2. 위젯 연결
마크다운에서 위젯을 참조할 때:
```markdown
> 🎮 [인터랙티브 위젯 열기](../_widgets/btree-insert-visualizer.html)
```

### 3. 태그 컨벤션
```
#til #db #kafka #jvm #kotlin #spring #infra #k8s #architecture #ai
#deep-dive    ← 심층 학습
#quick-ref    ← 빠른 참조용
#troubleshoot ← 트러블슈팅/디버깅
```

### 4. GitHub 연동
```bash
cd claude-til-vault
git init
git remote add origin git@github.com:<username>/claude-til-vault.git
git add -A && git commit -m "init vault"
git push -u origin main
```

GitHub Pages를 켜두면 `_widgets/` 안의 HTML을 브라우저에서 바로 실행 가능:
`https://<username>.github.io/claude-til-vault/_widgets/btree-insert-visualizer.html`
