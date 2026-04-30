# ビルドとテスト (Build and Test)

**目的**: すべてのユニットをビルドし、包括的なテスト戦略を実行する

## 前提条件
- すべてのユニットのコード生成 (Code Generation) が完了していること
- すべてのコード成果物が生成されていること
- プロジェクトがビルドとテストの準備ができていること

---

## ステップ1: テスト要件を分析する

プロジェクトを分析して適切なテスト戦略を決定する:
- **ユニットテスト**: コード生成 (Code Generation) 中に各ユニットで既に生成済み
- **統合テスト**: ユニット/サービス間のインタラクションをテストする
- **パフォーマンステスト**: 負荷テスト、ストレステスト、スケーラビリティテスト
- **エンドツーエンドテスト**: 完全なユーザーワークフロー
- **コントラクトテスト**: サービス間のAPI コントラクト検証
- **セキュリティテスト**: 脆弱性スキャン、ペネトレーションテスト

---

## ステップ2: ビルド手順を生成する

`aidlc-docs/construction/build-and-test/build-instructions.md` を作成する:

```markdown
# Build Instructions

## Prerequisites
- **Build Tool**: [Tool name and version]
- **Dependencies**: [List all required dependencies]
- **Environment Variables**: [List required env vars]
- **System Requirements**: [OS, memory, disk space]

## Build Steps

### 1. Install Dependencies
\`\`\`bash
[Command to install dependencies]
# Example: npm install, mvn dependency:resolve, pip install -r requirements.txt
\`\`\`

### 2. Configure Environment
\`\`\`bash
[Commands to set up environment]
# Example: export variables, configure credentials
\`\`\`

### 3. Build All Units
\`\`\`bash
[Command to build all units]
# Example: mvn clean install, npm run build, brazil-build
\`\`\`

### 4. Verify Build Success
- **Expected Output**: [Describe successful build output]
- **Build Artifacts**: [List generated artifacts and locations]
- **Common Warnings**: [Note any acceptable warnings]

## Troubleshooting

### Build Fails with Dependency Errors
- **Cause**: [Common causes]
- **Solution**: [Step-by-step fix]

### Build Fails with Compilation Errors
- **Cause**: [Common causes]
- **Solution**: [Step-by-step fix]
```

---

## ステップ3: ユニットテスト実行手順を生成する

`aidlc-docs/construction/build-and-test/unit-test-instructions.md` を作成する:

```markdown
# Unit Test Execution

## Run Unit Tests

### 1. Execute All Unit Tests
\`\`\`bash
[Command to run all unit tests]
# Example: mvn test, npm test, pytest tests/unit
\`\`\`

### 2. Review Test Results
- **Expected**: [X] tests pass, 0 failures
- **Test Coverage**: [Expected coverage percentage]
- **Test Report Location**: [Path to test reports]

### 3. Fix Failing Tests
If tests fail:
1. Review test output in [location]
2. Identify failing test cases
3. Fix code issues
4. Rerun tests until all pass
```

---

## ステップ4: 統合テスト手順を生成する

`aidlc-docs/construction/build-and-test/integration-test-instructions.md` を作成する:

```markdown
# Integration Test Instructions

## Purpose
Test interactions between units/services to ensure they work together correctly.

## Test Scenarios

### Scenario 1: [Unit A] → [Unit B] Integration
- **Description**: [What is being tested]
- **Setup**: [Required test environment setup]
- **Test Steps**: [Step-by-step test execution]
- **Expected Results**: [What should happen]
- **Cleanup**: [How to clean up after test]

### Scenario 2: [Unit B] → [Unit C] Integration
[Similar structure]

## Setup Integration Test Environment

### 1. Start Required Services
\`\`\`bash
[Commands to start services]
# Example: docker-compose up, start test database
\`\`\`

### 2. Configure Service Endpoints
\`\`\`bash
[Commands to configure endpoints]
# Example: export API_URL=http://localhost:8080
\`\`\`

## Run Integration Tests

### 1. Execute Integration Test Suite
\`\`\`bash
[Command to run integration tests]
# Example: mvn integration-test, npm run test:integration
\`\`\`

### 2. Verify Service Interactions
- **Test Scenarios**: [List key integration test scenarios]
- **Expected Results**: [Describe expected outcomes]
- **Logs Location**: [Where to check logs]

### 3. Cleanup
\`\`\`bash
[Commands to clean up test environment]
# Example: docker-compose down, stop test services
\`\`\`
```

---

## ステップ5: パフォーマンステスト手順を生成する（該当する場合）

`aidlc-docs/construction/build-and-test/performance-test-instructions.md` を作成する:

```markdown
# Performance Test Instructions

## Purpose
Validate system performance under load to ensure it meets requirements.

## Performance Requirements
- **Response Time**: < [X]ms for [Y]% of requests
- **Throughput**: [X] requests/second
- **Concurrent Users**: Support [X] concurrent users
- **Error Rate**: < [X]%

## Setup Performance Test Environment

### 1. Prepare Test Environment
\`\`\`bash
[Commands to set up performance testing]
# Example: scale services, configure load balancers
\`\`\`

### 2. Configure Test Parameters
- **Test Duration**: [X] minutes
- **Ramp-up Time**: [X] seconds
- **Virtual Users**: [X] users

## Run Performance Tests

### 1. Execute Load Tests
\`\`\`bash
[Command to run load tests]
# Example: jmeter -n -t test.jmx, k6 run script.js
\`\`\`

### 2. Execute Stress Tests
\`\`\`bash
[Command to run stress tests]
# Example: gradually increase load until failure
\`\`\`

### 3. Analyze Performance Results
- **Response Time**: [Actual vs Expected]
- **Throughput**: [Actual vs Expected]
- **Error Rate**: [Actual vs Expected]
- **Bottlenecks**: [Identified bottlenecks]
- **Results Location**: [Path to performance reports]

## Performance Optimization

If performance doesn't meet requirements:
1. Identify bottlenecks from test results
2. Optimize code/queries/configurations
3. Rerun tests to validate improvements
```

---

## ステップ6: 追加テスト手順を生成する（必要に応じて）

プロジェクト要件に基づき、追加のテスト手順ファイルを生成する:

### コントラクトテスト（マイクロサービス向け）
`aidlc-docs/construction/build-and-test/contract-test-instructions.md` を作成する:
- サービス間のAPI コントラクト検証
- コンシューマー駆動コントラクトテスト
- スキーマ検証

### セキュリティテスト
`aidlc-docs/construction/build-and-test/security-test-instructions.md` を作成する:
- 脆弱性スキャン
- 依存関係のセキュリティチェック
- 認証/認可テスト
- 入力バリデーションテスト

### エンドツーエンドテスト
`aidlc-docs/construction/build-and-test/e2e-test-instructions.md` を作成する:
- 完全なユーザーワークフローテスト
- クロスサービスシナリオ
- UI テスト（該当する場合）

---

## ステップ7: テスト要約を生成する

`aidlc-docs/construction/build-and-test/build-and-test-summary.md` を作成する:

```markdown
# Build and Test Summary

## Build Status
- **Build Tool**: [Tool name]
- **Build Status**: [Success/Failed]
- **Build Artifacts**: [List artifacts]
- **Build Time**: [Duration]

## Test Execution Summary

### Unit Tests
- **Total Tests**: [X]
- **Passed**: [X]
- **Failed**: [X]
- **Coverage**: [X]%
- **Status**: [Pass/Fail]

### Integration Tests
- **Test Scenarios**: [X]
- **Passed**: [X]
- **Failed**: [X]
- **Status**: [Pass/Fail]

### Performance Tests
- **Response Time**: [Actual] (Target: [Expected])
- **Throughput**: [Actual] (Target: [Expected])
- **Error Rate**: [Actual] (Target: [Expected])
- **Status**: [Pass/Fail]

### Additional Tests
- **Contract Tests**: [Pass/Fail/N/A]
- **Security Tests**: [Pass/Fail/N/A]
- **E2E Tests**: [Pass/Fail/N/A]

## Overall Status
- **Build**: [Success/Failed]
- **All Tests**: [Pass/Fail]
- **Ready for Operations**: [Yes/No]

## Next Steps
[If all pass]: Ready to proceed to Operations phase for deployment planning
[If failures]: Address failing tests and rebuild
```

---

## ステップ8: 状態追跡を更新する

`aidlc-docs/aidlc-state.md` を更新する:
- ビルドとテスト (Build and Test) ステージを完了としてマークする
- 現在のステータスを更新する

---

## ステップ9: 結果をユーザーに提示する

完了メッセージを以下の構成で提示する:
     1. **完了アナウンス** (必須): 常に以下で始めること:

```markdown
# 🔨 Build and Test Complete
```

     2. **AI要約** (任意): ビルドとテスト結果の構造化された箇条書き要約を提供する
        - 形式: 「Build and test has completed with the following results:」
        - ビルドステータスと成果物を列挙する
        - カテゴリ別のテスト結果を列挙する（ユニット、統合、パフォーマンスなど）
        - 生成された手順ファイルを列挙する
        - ワークフロー指示は含めない（「please review」「let me know」「proceed to next phase」「before we proceed」など）
        - 事実に基づき、コンテンツに焦点を当てること
     3. **フォーマットされたワークフローメッセージ** (必須): 常に以下の正確な形式で終わること:

```markdown
> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the build and test summary at: `aidlc-docs/construction/build-and-test/build-and-test-summary.md`



> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** - Ask for modifications to the build and test instructions based on your review
> ✅ **Approve & Continue** - Approve build and test results and proceed to **Operations**

---
```

---

## ステップ10: インタラクションを記録する

**必須 (MANDATORY)**: `aidlc-docs/audit.md` にステージ完了を記録する:

```markdown
## Build and Test Stage
**Timestamp**: [ISO timestamp]
**Build Status**: [Success/Failed]
**Test Status**: [Pass/Fail]
**Files Generated**:
- build-instructions.md
- unit-test-instructions.md
- integration-test-instructions.md
- performance-test-instructions.md
- build-and-test-summary.md

---
```
