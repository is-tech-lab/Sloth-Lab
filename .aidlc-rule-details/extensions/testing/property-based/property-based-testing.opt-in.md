# プロパティベーステスト — オプトイン (Property-Based Testing — Opt-In)

**拡張機能**: プロパティベーステスト (Property-Based Testing)

## オプトイン (Opt-In) プロンプト

このエクステンションがロードされた際、以下の質問が要件分析 (Requirements Analysis) の確認事項として自動的に含まれます：

```markdown
## Question: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)
B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)
C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)
X) Other (please describe after [Answer]: tag below)

[Answer]: 
```
