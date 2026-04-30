# セキュリティベースライン — オプトイン (Security Baseline — Opt-In)

**拡張機能**: セキュリティベースライン (Security Baseline)

## オプトイン (Opt-In) プロンプト

このエクステンションがロードされた際、以下の質問が要件分析 (Requirements Analysis) の確認事項として自動的に含まれます：

```markdown
## Question: Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)
B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)
X) Other (please describe after [Answer]: tag below)

[Answer]: 
```
