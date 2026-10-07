# npm と Go の比較：テスト・Lint・CI/CD まとめ

## 目次

1. [コマンド対応表](#1-コマンド対応表)
2. [Go のテスト（go test）](#2-go-のテストgo-test)
3. [フロントエンドのテスト（npm test）](#3-フロントエンドのテストnpm-test)
4. [テストのルール比較まとめ](#4-テストのルール比較まとめ)
5. [go build と go run の違い](#5-go-build-と-go-run-の違い)
6. [GitHub Actions による CI/CD](#6-github-actions-による-cicd)

---

## 1. コマンド対応表

| 目的 | npm (TypeScript / JavaScript) | Go |
| --- | --- | --- |
| Lint（コードの書き方ルールチェック） | `npm run lint` | `gofmt`（標準整形）+ `go vet` + `golangci-lint`（総合 Lint） |
| 型チェック | `npm run typecheck` | `go build`（コンパイル時に型チェックされる・標準） |
| テスト | `npm test` | `go test ./...` |

---

## 2. Go のテスト（go test）

Go には標準で強力なテスト機能が組み込まれています。言語仕様や標準ツールチェーンに公式ルールが厳密に組み込まれており、非常にシンプルです。

### 実行コマンド

```bash
go test ./...
```

### テストコードの例（`main_test.go`）

```go
package main

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    if result != expected {
        t.Errorf("期待値は %d ですが、実際は %d でした", expected, result)
    }
}
```

### ① ファイル名の約束事

- テスト対象と同じパッケージ内に **`*_test.go`** という名前で保存する
- 例：`main.go` → `main_test.go`、`user.go` → `user_test.go`

> ⚠️ `*_test.go` 以外のファイル名にすると、`go test` がテストファイルとして認識しません。

### ② 配置（ディレクトリ）の約束事

**基本ルール：** テスト対象のコードと同じディレクトリ（同じパッケージ内）に置く。

```plaintext
my-go-project/
├── math.go       # 本体のコード (package math)
└── math_test.go  # テストコード (package math)
```

**例外（外部テスト）：** パッケージの公開 API だけを外側からテストしたい場合は、ファイル名は `math_test.go` のまま、パッケージ宣言を `package math_test` に変える慣習もあります。

### ③ 関数名の約束事

- 必ず **`Test`** から始め、続きも大文字始まり（キャメルケース）
- 引数に **`*testing.T`** を受け取る

```go
func TestCalculateTotal(t *testing.T) {
    // テスト処理
}
```

### ④ 設定ファイルへの登録

**不要。** `package.json` のような事前登録は必要ありません。`go test ./...` と打つだけで、全ディレクトリの `*_test.go` を自動検出して実行します。

---

## 3. フロントエンドのテスト（npm test）

JavaScript / TypeScript では、**Vitest** や **Jest** などのライブラリを使って実行します。テストランナーによって柔軟ですが、標準的なディレクトリルールと `package.json` への登録ルールがあります。

### 実行コマンド

```bash
npm test
# 内部的には "vitest" や "jest" が動く
```

### テストコードの例（`math.test.ts`）

```typescript
import { describe, it, expect } from 'vitest';
import { add } from './math';

describe('add関数', () => {
  it('2 と 3 を足すと 5 になること', () => {
    expect(add(2, 3)).toBe(5);
  });
});
```

### ① ファイル名の約束事

以下のいずれかで命名するのが一般的です（テストランナーが自動検知します）。

- **パターン A：** `[ファイル名].test.ts`（`.test.js` / `.test.tsx`）
- **パターン B：** `[ファイル名].spec.ts`（`.spec.js` / `.spec.tsx`）

### ② 配置（ディレクトリ）の約束事

**パターン 1：同所配置（Colocation）** ★フロントエンドで人気

```plaintext
src/
├── components/
│   ├── Button.tsx
│   └── Button.test.tsx  # 隣に置く
```

**パターン 2：`__tests__` / `tests` ディレクトリに集約**

```plaintext
src/
├── components/
│   └── Button.tsx
└── __tests__/           # または root に tests/ を作成
    └── Button.test.tsx
```

### ③ package.json への登録

`npm test` で実行するには、`package.json` の `scripts` に登録が必要です。

```json
{
  "name": "my-app",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "test": "vitest"
  }
}
```

> 💡 `"test": "vitest"` や `"test": "jest"` と定義しておくと、`npm test`（または `pnpm test` / `bun test`）実行時に裏でテストツールが起動し、`*.test.ts` を探しに行きます。

---

## 4. テストのルール比較まとめ

| 比較項目 | Go | npm (TypeScript / JavaScript) |
| --- | --- | --- |
| ファイル名 | `*_test.go`（必須） | `*.test.ts` または `*.spec.ts` |
| 配置場所 | 実装ファイルと同じディレクトリ | 実装の隣、または `__tests__` / `tests` |
| 設定ファイルへの登録 | 不要（言語機能として標準搭載） | 必要（`package.json` に `"test"` を登録） |
| 実行コマンド | `go test ./...` | `npm test` |

---

## 5. go build と go run の違い

| 項目 | `go build` | `go run .` |
| --- | --- | --- |
| 主な目的 | 配布・デプロイ用の実行ファイル作成 | 開発中の動作確認・スクリプト的実行 |
| 生成物の出力 | 現在のディレクトリにバイナリを出力 | 一時ディレクトリにビルドし、実行後に消去 |
| 実行タイミング | ビルドのみ（実行は別途 `./アプリ名`） | ビルドしてそのまま即実行 |
| 実行スピード | バイナリを直接動かすため最速 | 毎回コンパイルが入るため少し遅い |

---

## 6. GitHub Actions による CI/CD

### 6-1. 基本的な仕組みと配置方法

リポジトリ内に YAML ファイルを置くだけで自動的に認識されます。

- **配置場所：** リポジトリ直下の `.github/workflows/` に `.yml` ファイルを置く
- **例：** `.github/workflows/ci.yml`

### 6-2. ワークフローの構成要素

| 要素 | 役割 |
| --- | --- |
| `name` | ワークフローの名前 |
| `on` | トリガー条件（PR 作成時、`main` への Push 時など） |
| `jobs` | 実行する処理の単位 |
| `runs-on` | 実行環境（`ubuntu-latest` が一般的でコスト面でもおすすめ） |
| `steps` | 実行手順（コード取得 → 環境セットアップ → コマンド実行） |

### 6-3. パターン A：フロントエンド（Node.js / React / Next.js）の CI

PR 作成時や `main` 更新時に **Lint・型チェック・テスト** を自動実行します。

**`.github/workflows/frontend-ci.yml`**

```yaml
name: Frontend CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest

    steps:
      # 1. リポジトリのコードをチェックアウト
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Node.js 環境のセットアップ + キャッシュ設定
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm' # package-lock.json を基にキャッシュを自動有効化

      # 3. 依存関係のインストール
      - name: Install dependencies
        run: npm ci

      # 4. Lint の実行
      - name: Run Linter
        run: npm run lint

      # 5. 型チェックの実行（TypeScript プロジェクトの場合）
      - name: Run Type Check
        run: npm run typecheck
        continue-on-error: false

      # 6. テストの実行
      - name: Run Tests
        run: npm test
```

### 6-4. パターン B：バックエンド（Go）の CI

**フォーマットチェック（gofmt）・go vet・golangci-lint・単体テスト** を自動実行します。

**`.github/workflows/backend-ci.yml`**

```yaml
name: Go Backend CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test-and-lint:
    runs-on: ubuntu-latest

    steps:
      # 1. コードのチェックアウト
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Go 環境のセットアップ
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true # go.sum をもとにモジュールキャッシュを有効化

      # 3. コードスタイルの検証 (gofmt)
      - name: Check formatting
        run: |
          if [ -n "$(gofmt -l .)" ]; then
            echo "The following files are not formatted properly:"
            gofmt -l .
            exit 1
          fi

      # 4. 標準静的解析 (go vet)
      - name: Run go vet
        run: go vet ./...

      # 5. golangci-lint の実行（公式 Action を使用）
      - name: Run golangci-lint
        uses: golangci-lint/golangci-lint-action@v6
        with:
          version: v1.58.0

      # 6. テストの実行（レース検出 + カバレッジ）
      - name: Run Unit Tests
        run: go test -v -race -cover ./...
```

### 6-5. CD（自動デプロイ）を追加する

CI が成功した後に、Docker Hub などへ自動ビルド & プッシュする例です。上記 `jobs:` の下に追加します。

```yaml
  deploy:
    needs: test-and-lint # CI ジョブが成功した後にのみ実行
    if: github.ref == 'refs/heads/main' && github.event_name == 'push' # main への Push 時のみ
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: my-docker-hub-user/my-app:latest
```

### 6-6. 機密情報（API キー・パスワード）の扱い

> 🔐 `DOCKER_USERNAME` や `DOCKER_PASSWORD` などは、GitHub のリポジトリ設定
> **Settings > Secrets and variables > Actions** の **Repository secrets** に保存し、
> `${{ secrets.SECRET_NAME }}` で安全に参照します。
