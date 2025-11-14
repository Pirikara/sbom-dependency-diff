# sbom-dependency-diff 設計書

## 1. 概要

### 1.1 目的

GitHub Pull Request 上で、

* **base ref と head ref のソースコードから SBOM を生成**し
* **依存関係（パッケージ）の差分**を取り

  * 追加された依存（added）
  * 削除された依存（removed）
  * バージョンが変更された依存（changed）
* を **JSON 形式の outputs として返す GitHub Action**

を提供する。

この Action 自体は **「差分を検出して返す」ことに責務を限定**する。
脆弱性スキャン・リリース日チェック・ライセンスチェックなどは、
**後続の workflow ステップ側で、この Action の outputs を利用して実装する**前提。

### 1.2 特徴

* **SBOM 生成に Syft** を利用（エコシステムごとの個別対応をなくす）
* SBOM フォーマットは **CycloneDX JSON をデフォルト**とする
* **Composite Action** として実装（Dockerfile 不要）
* outputs は JSON 文字列なので、後続ステップで `jq` やスクリプトから扱いやすい

---

## 2. スコープ / 非スコープ

### 2.1 スコープ（この Action がやること）

* Git リポジトリの base ref と head ref に対して：

  * Syft を使って SBOM を生成
  * CycloneDX CLI で SBOM を diff
  * diff 結果を整理して、

    * `added`（追加された依存の配列）
    * `removed`（削除された依存の配列）
    * `changed`（バージョン変更された依存の配列）
  * を JSON にまとめて outputs として返す
* 依存差分の有無を `has_diff`（boolean）として返す
* オプションとして、差分があれば Job を fail させる `fail-on-diff` を提供

### 2.2 非スコープ（この Action がやらないこと）

* 脆弱性のスキャン（OSV など）
* リリース日の取得・ポリシーチェック
* ライセンスポリシーの評価
* 依存のツリー構造を直接出力すること（あくまでコンポーネント単位）

これらはすべて**後続のステップ**で、
`added` / `removed` / `changed` の JSON を利用して自由に実装する。

---

## 3. 命名 / リポジトリ構成

### 3.1 リポジトリ名 / Action 名

* リポジトリ名: `sbom-dependency-diff`
* `uses` での指定例:

```yaml
uses: your-org/sbom-dependency-diff@v1
```

### 3.2 想定ディレクトリ構成

```text
sbom-dependency-diff/
  ├─ action.yml
  ├─ scripts/
  │    └─ sbom-diff.sh           # SBOM生成とdiff実行メイン処理
  ├─ README.md
  └─ LICENSE
```

※ シンプルさ優先で、JSON の正規化も `jq` と `bash` で `sbom-diff.sh` 内にまとめる想定。
（必要なら将来的に `scripts/normalize-diff.js` や `.py` に分割可能）

---

## 4. 利用イメージ

### 4.1 最小利用例

```yaml
name: dependency-diff

on:
  pull_request:

jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: SBOM dependency diff
        id: diff
        uses: your-org/sbom-dependency-diff@v1
        with:
          path: .
          sbom-format: cyclonedx-json

      - name: Show added dependencies
        run: |
          echo '${{ steps.diff.outputs.added }}' | jq .
```

### 4.2 差分があれば fail させる例

```yaml
      - name: SBOM dependency diff
        id: diff
        uses: your-org/sbom-dependency-diff@v1
        with:
          path: .
          sbom-format: cyclonedx-json
          fail-on-diff: "true"
```

---

## 5. Inputs / Outputs 仕様

### 5.1 Inputs

| 名称                  | 型                           | 必須  | デフォルト                              | 説明                                         |
| ------------------- | --------------------------- | --- | ---------------------------------- | ------------------------------------------ |
| `base-ref`          | string                      | いいえ | `GITHUB_BASE_REF` or `origin/main` | 比較元の Git ref（PR の base ブランチ想定）             |
| `head-ref`          | string                      | いいえ | `GITHUB_HEAD_REF` or `HEAD`        | 比較先の Git ref（PR の head ブランチ想定）             |
| `path`              | string                      | いいえ | `"."`                              | SBOM 生成対象のディレクトリパス（モノレポ対応用）                |
| `sbom-format`       | string                      | いいえ | `"cyclonedx-json"`                 | Syft の SBOM 出力フォーマット（現状 CycloneDX JSON 前提） |
| `syft-args`         | string                      | いいえ | `""`                               | Syft 実行時に追加で渡す引数（イメージスキャン等をしたくなったとき用）      |
| `working-directory` | string                      | いいえ | `"."`                              | Git操作や Syft 実行を行うワーキングディレクトリ               |
| `fail-on-diff`      | string (`"true"`/`"false"`) | いいえ | `"false"`                          | 差分がある場合に Action をエラー終了させるかどうか              |

**デフォルト挙動（ref 周り）**

* `base-ref` 未指定:

  * `GITHUB_BASE_REF` 環境変数があればそれを使用（PR コンテキスト）
  * なければ `origin/main` を使用
* `head-ref` 未指定:

  * `GITHUB_HEAD_REF` 環境変数があればそれを使用（PR コンテキスト）
  * なければ `HEAD` を使用

※ README に「PR ベースで使う想定」「`actions/checkout` は `fetch-depth: 0` を推奨」と明記する。

### 5.2 Outputs

すべて **文字列（JSON 文字列）** として出力される。

| 名称         | 型（論理）               | 内容                                   |
| ---------- | ------------------- | ------------------------------------ |
| `has_diff` | boolean（文字列）        | `true` / `false`。依存差分が1つでもあれば `true` |
| `added`    | JSON string (array) | 追加された依存コンポーネントの配列                    |
| `removed`  | JSON string (array) | 削除された依存コンポーネントの配列                    |
| `changed`  | JSON string (array) | バージョン変更された依存コンポーネントの配列               |

#### 5.2.1 `added` / `removed` 要素例

```jsonc
[
  {
    "name": "lodash",
    "version": "4.17.21",
    "purl": "pkg:npm/lodash@4.17.21",
    "bom_ref": "pkg:npm/lodash@4.17.21",
    "type": "library"
  }
]
```

#### 5.2.2 `changed` 要素例

```jsonc
[
  {
    "name": "lodash",
    "from_version": "4.17.20",
    "to_version": "4.17.21",
    "from_purl": "pkg:npm/lodash@4.17.20",
    "to_purl": "pkg:npm/lodash@4.17.21",
    "bom_ref": "pkg:npm/lodash@4.17.21"
  }
]
```

※ 実際のキー名は CycloneDX CLI の diff JSON の仕様に合わせて調整する。
ここでは **ターゲットとしてほしい形**を仕様として定義している。

---

## 6. 内部処理フロー

Composite Action 内での処理フローは以下。

1. **環境準備**

   * `working-directory` に `cd`
   * `curl` で Syft をインストール
   * `curl` で CycloneDX CLI をインストール
   * `jq` を `apt-get` でインストール

2. **base / head ref の解決**

   * `base-ref` / `head-ref` input を読み取り
   * 未指定の場合は `GITHUB_BASE_REF` / `GITHUB_HEAD_REF` / `origin/main` / `HEAD` の順で決定

3. **base 用 SBOM 生成**

   * `git checkout <base-ref>`
   * `syft dir:<path> -o <sbom-format> <syft-args> > /tmp/sbom-base.json`

4. **head 用 SBOM 生成**

   * `git checkout <head-ref>`
   * `syft dir:<path> -o <sbom-format> <syft-args> > /tmp/sbom-head.json`

5. **SBOM diff の取得**

   * `cyclonedx-cli diff /tmp/sbom-base.json /tmp/sbom-head.json --component-versions --output-format json > /tmp/sbom-diff-raw.json`
   * （オプションやキー名は実装時に CycloneDX CLI の仕様を確認して調整）

6. **diff JSON の正規化**

   * `jq` を使い、`/tmp/sbom-diff-raw.json` から

     * 追加されたコンポーネントの配列 → `added`
     * 削除されたコンポーネントの配列 → `removed`
     * バージョン変更されたコンポーネントの配列 → `changed`
   * を抽出し、前述の仕様に合う JSON 形式に整形

7. **GitHub Outputs へ書き込み**

   * `added` / `removed` / `changed` の JSON 文字列化
   * 3つの配列のいずれかが非空であれば `has_diff=true`、それ以外は `false`
   * `GITHUB_OUTPUT` に `has_diff` / `added` / `removed` / `changed` を書き込む

8. **fail-on-diff の処理**

   * `fail-on-diff == "true"` かつ `has_diff == "true"` の場合は `exit 1` でジョブを fail させる

---

## 7. action.yml（Composite Action）案

```yaml
name: "SBOM Dependency Diff"
description: "Generate SBOMs for base and head refs using Syft and output dependency diff (added/removed/changed) as JSON."
author: "your-name-or-org"

branding:
  icon: "layers"
  color: "blue"

inputs:
  base-ref:
    description: "Base git ref to compare (e.g. PR base branch). Defaults to GITHUB_BASE_REF or origin/main."
    required: false
  head-ref:
    description: "Head git ref to compare (e.g. PR head branch). Defaults to GITHUB_HEAD_REF or HEAD."
    required: false
  path:
    description: "Target path to scan for SBOM generation (e.g. '.' or 'subdir')."
    required: false
    default: "."
  sbom-format:
    description: "SBOM output format for Syft (e.g. cyclonedx-json)."
    required: false
    default: "cyclonedx-json"
  syft-args:
    description: "Additional arguments passed to Syft when generating SBOM."
    required: false
    default: ""
  working-directory:
    description: "Working directory to run git and syft commands in."
    required: false
    default: "."
  fail-on-diff:
    description: "If 'true', action exits with non-zero code when any diff is found."
    required: false
    default: "false"

outputs:
  has_diff:
    description: "true if there is any dependency difference between base and head."
    value: ${{ steps.normalize.outputs.has_diff }}
  added:
    description: "JSON string array of added components."
    value: ${{ steps.normalize.outputs.added }}
  removed:
    description: "JSON string array of removed components."
    value: ${{ steps.normalize.outputs.removed }}
  changed:
    description: "JSON string array of changed components (with old/new versions)."
    value: ${{ steps.normalize.outputs.changed }}

runs:
  using: "composite"
  steps:
    - name: Prepare tools (Syft, CycloneDX CLI, jq)
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        set -euo pipefail
        echo "Installing Syft..."
        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh \
          | sh -s -- -b /usr/local/bin

        echo "Installing CycloneDX CLI..."
        curl -sSfL https://github.com/CycloneDX/cyclonedx-cli/releases/latest/download/cyclonedx-linux-x64 \
          -o /usr/local/bin/cyclonedx-cli
        chmod +x /usr/local/bin/cyclonedx-cli

        echo "Installing jq..."
        sudo apt-get update -y
        sudo apt-get install -y jq

    - name: Generate SBOMs and raw diff
      id: diff
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        set -euo pipefail

        BASE_REF_INPUT="${{ inputs.base-ref }}"
        HEAD_REF_INPUT="${{ inputs.head-ref }}"
        TARGET_PATH="${{ inputs.path }}"
        SBOM_FORMAT="${{ inputs.sbom-format }}"
        SYFT_ARGS="${{ inputs.syft-args }}"

        # resolve base ref
        if [ -n "$BASE_REF_INPUT" ]; then
          BASE_REF="$BASE_REF_INPUT"
        elif [ -n "${GITHUB_BASE_REF:-}" ]; then
          BASE_REF="$GITHUB_BASE_REF"
        else
          BASE_REF="origin/main"
        fi

        # resolve head ref
        if [ -n "$HEAD_REF_INPUT" ]; then
          HEAD_REF="$HEAD_REF_INPUT"
        elif [ -n "${GITHUB_HEAD_REF:-}" ]; then
          HEAD_REF="$GITHUB_HEAD_REF"
        else
          HEAD_REF="HEAD"
        fi

        echo "Base ref: $BASE_REF"
        echo "Head ref: $HEAD_REF"
        echo "Target path: $TARGET_PATH"
        echo "SBOM format: $SBOM_FORMAT"

        echo "::group::Generate SBOM for base ref"
        git checkout "$BASE_REF"
        syft "dir:${TARGET_PATH}" -o "$SBOM_FORMAT" $SYFT_ARGS > /tmp/sbom-base.json
        echo "::endgroup::"

        echo "::group::Generate SBOM for head ref"
        git checkout "$HEAD_REF"
        syft "dir:${TARGET_PATH}" -o "$SBOM_FORMAT" $SYFT_ARGS > /tmp/sbom-head.json
        echo "::endgroup::"

        echo "::group::Diff SBOMs"
        # NOTE: CycloneDX CLI のオプション/出力は実装時に確認して調整すること
        cyclonedx-cli diff /tmp/sbom-base.json /tmp/sbom-head.json \
          --component-versions \
          --output-format json > /tmp/sbom-diff-raw.json
        echo "::endgroup::"

        echo "sbom_diff_raw=/tmp/sbom-diff-raw.json" >> "$GITHUB_OUTPUT"

    - name: Normalize diff JSON
      id: normalize
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        set -euo pipefail

        RAW_DIFF_PATH="${{ steps.diff.outputs.sbom_diff_raw }}"
        if [ ! -f "$RAW_DIFF_PATH" ]; then
          echo "Raw diff file not found: $RAW_DIFF_PATH"
          echo "has_diff=false" >> "$GITHUB_OUTPUT"
          echo "added=[]" >> "$GITHUB_OUTPUT"
          echo "removed=[]" >> "$GITHUB_OUTPUT"
          echo "changed=[]" >> "$GITHUB_OUTPUT"
          exit 0
        fi

        # ここから先は CycloneDX CLI の diff JSON の構造に依存するため、
        # 実際のフィールド名に合わせて jq の処理を書いてください。
        #
        # 以下は擬似コードイメージ：
        #
        # - .components.added[] から name/version/purl を抜き出す
        # - .components.removed[] から name/version/purl を抜き出す
        # - .components.changed[] から old/new の version/purl を抜き出す
        #
        # 実際には cyclonedx-cli diff の出力を確認して書き換えること。

        ADDED_JSON=$(jq -c '
          .components.added // [] |
          map({
            name: (.name // .bomRef // ""),
            version: (.version // ""),
            purl: (.purl // ""),
            bom_ref: (.bomRef // ""),
            type: (.type // "")
          })
        ' "$RAW_DIFF_PATH")

        REMOVED_JSON=$(jq -c '
          .components.removed // [] |
          map({
            name: (.name // .bomRef // ""),
            version: (.version // ""),
            purl: (.purl // ""),
            bom_ref: (.bomRef // ""),
            type: (.type // "")
          })
        ' "$RAW_DIFF_PATH")

        CHANGED_JSON=$(jq -c '
          .components.changed // [] |
          map({
            name: (.name // .new.bomRef // .old.bomRef // ""),
            from_version: (.old.version // ""),
            to_version: (.new.version // ""),
            from_purl: (.old.purl // ""),
            to_purl: (.new.purl // ""),
            bom_ref: (.new.bomRef // .old.bomRef // "")
          })
        ' "$RAW_DIFF_PATH")

        HAS_DIFF="false"
        if [ "$ADDED_JSON" != "[]" ] || [ "$REMOVED_JSON" != "[]" ] || [ "$CHANGED_JSON" != "[]" ]; then
          HAS_DIFF="true"
        fi

        echo "has_diff=$HAS_DIFF" >> "$GITHUB_OUTPUT"
        echo "added=$ADDED_JSON" >> "$GITHUB_OUTPUT"
        echo "removed=$REMOVED_JSON" >> "$GITHUB_OUTPUT"
        echo "changed=$CHANGED_JSON" >> "$GITHUB_OUTPUT"

        echo "has_diff: $HAS_DIFF"
        echo "added count: $(echo "$ADDED_JSON" | jq 'length')"
        echo "removed count: $(echo "$REMOVED_JSON" | jq 'length')"
        echo "changed count: $(echo "$CHANGED_JSON" | jq 'length')"

        # fail-on-diff 処理
        FAIL_ON_DIFF="${{ inputs.fail-on-diff }}"
        if [ "$FAIL_ON_DIFF" = "true" ] && [ "$HAS_DIFF" = "true" ]; then
          echo "Dependency diff detected and fail-on-diff=true. Failing job."
          exit 1
        fi
```

※ `jq` 部分は CycloneDX CLI の実際の diff JSON 構造を確認して調整必須。
　そこだけ TODO コメントを入れておくと実装者が迷わないはずです。

---

## 8. 実装者向け TODO メモ

この設計を元に実装する人向けに、やることリスト：

1. リポジトリ `sbom-dependency-diff` を作成
2. `action.yml` を上記案ベースで作成
3. `README.md` に

   * 概要
   * Inputs / Outputs
   * 使い方サンプル
   * `actions/checkout@v4` の `fetch-depth: 0` 推奨
   * 注意点（CycloneDX CLI の JSON 仕様に依存すること）
     を記載
4. 実際に `cyclonedx-cli diff --output-format json` の出力を確認し、
   `Normalize diff JSON` ステップの `jq` を実データに合わせて修正
5. 実際のリポジトリ（簡単な npm / go / python プロジェクト）で PR を作り、

   * 依存追加
   * 依存削除
   * バージョン変更
     の3パターンをテストして、`added` / `removed` / `changed` / `has_diff` outputs を確認

---

「jq での正規化部分は CycloneDX CLI の出力見ながら埋める」という前提で書いてあるので、そこだけ現物合わせで詰めてもらう形になります。
