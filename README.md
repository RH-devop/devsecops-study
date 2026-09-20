# DevSecOps Study — Trivy Security Scan

## 概要

Next.js アプリケーションを対象に、Trivy と GitHub Actions を使ったセキュリティチェックを実装した。

脆弱性や秘密情報を意図的に含めた状態で CI を失敗させ、問題を修正すると CI が成功することを確認した。

**学習の目的：セキュリティ上の問題を CI/CD の段階で検出し、問題があるコードの見逃しを防ぐ仕組みを理解すること。**

## 使用技術

- Next.js
- Trivy
- GitHub Actions
- Git / GitHub

## 実装したセキュリティチェック

| 項目 | 内容 |
|---|---|
| SCA | 依存パッケージの既知の脆弱性を検出 |
| Secret Scan | ファイル内の秘密情報に該当する文字列を検出 |
| CI | `main` ブランチへの push で自動実行 |
| 失敗条件 | HIGH または CRITICAL の問題を検出した場合、終了コード `1` で失敗 |

## 1. SCA：依存関係の脆弱性スキャン

意図的に脆弱性を含む `lodash@4.17.20` を導入し、Trivy で検出できることを確認した。

実施した流れ：

1. 脆弱性を含むバージョンの `lodash` を導入
2. Trivy による脆弱性検出を確認
3. GitHub Actions の Security Scan が失敗することを確認
4. `lodash` を修正
5. Security Scan が成功することを確認

**学んだこと：** アプリケーション自身のコードだけでなく、利用する依存パッケージもセキュリティチェックの対象になる。

## 2. Secret Scan：秘密情報の検出

学習用のダミー文字列を使い、Trivy が秘密情報に該当する文字列を検出できるか検証した。

※ 実際のパスワードや API キーは使用していない。

### カスタム検出ルール

リポジトリ直下に `trivy-secret.yaml` を作成した。

```yaml
rules:
  - id: study-dummy-password
    category: general
    title: Study Dummy Password
    severity: HIGH
    regex: 'MyTestPassword123!'

disable-allow-rules:
  - tests
```

`study-dummy-password` は、学習用のダミー文字列を HIGH として検出する独自ルール。

### 検出できなかった原因と解決

当初は、ファイル名・拡張子・スキャン対象パスを変更しても、ダミー文字列を検出できなかった。

その後、Trivy の組み込み許可ルール `tests` を無効化すると、**同じダミーファイルと同じ正規表現のまま HIGH 1件を検出**できた。

```yaml
disable-allow-rules:
  - tests
```

**学んだこと：** Secret Scan は、正規表現に一致した文字列をすべて報告するとは限らない。誤検知を減らすための許可ルールによって、検出結果が除外される場合がある。

今回は学習用のダミー文字列を検出するため、`tests` の許可ルールを無効化した。

### ローカルでの検証

```powershell
trivy fs --scanners secret --secret-config .\trivy-secret.yaml --exit-code 1 .\secret-scan-test
```

ダミー文字列を含む状態では、次の結果を確認した。

```text
secret-test.txt (secrets)
Total: 1 (HIGH: 1)

HIGH: general (study-dummy-password)

ExitCode=1
```

## 3. GitHub Actions との連携

`.github/workflows/security.yml` に、脆弱性スキャンと Secret Scan を明示的に指定した。

```yaml
name: Security Scan

on:
  push:
    branches:
      - main

jobs:
  trivy-scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          scanners: vuln,secret
          severity: HIGH,CRITICAL
          exit-code: 1
```

`scanners: vuln,secret` により、既存の SCA を残しつつ Secret Scan も実行する構成にした。

## 4. CI での検証結果

| 実行 | 検証内容 | 結果 |
|---|---|---|
| Security Scan #1 | 脆弱性を含む依存パッケージ | 失敗 |
| Security Scan #2 | 依存パッケージを修正 | 成功 |
| Security Scan #3 | ダミーの秘密情報を含むファイル | 失敗 |
| Security Scan #4 | ダミー文字列を削除 | 成功 |

Secret Scan の失敗時には、GitHub Actions のログで次の内容を確認した。

```text
secret-scan-test/secret-test.txt (secrets)
Total: 1 (HIGH: 1, CRITICAL: 0)

HIGH: general (study-dummy-password)

Error: Process completed with exit code 1.
```

修正後のコミット `0eb03d7` では、**Security Scan #4 が成功**した。

## まとめ

今回の学習では、SCA と Secret Scan の両方で、次の一連の動作を実践した。

**問題を混入させる → Trivy が検出する → CI が失敗する → 問題を修正する → CI が成功する**

これにより、セキュリティチェックを開発フローに組み込み、問題を自動的に検出する DevSecOps の基本的な仕組みを理解した。
