# claude-kintone-template 社内利用マニュアル

このマニュアルは、`claude-kintone-template` を使って kintone のカスタマイズ／プラグイン開発を行う社内メンバー向けの手引きです。README / CLAUDE.md / setup.md / mcp.md / claude-tips.md / settings.json の内容をベースにまとめています。

---

## 目次

1. [このテンプレートについて](#1-このテンプレートについて)
2. [事前準備](#2-事前準備)
3. [セットアップ手順](#3-セットアップ手順)
4. [ディレクトリ構成](#4-ディレクトリ構成)
5. [カスタマイズ開発フロー](#5-カスタマイズ開発フロー-srccustomize)
6. [プラグイン開発フロー](#6-プラグイン開発フロー-srcplugin)
7. [コマンドリファレンス](#7-コマンドリファレンス)
8. [Claude Code との付き合い方](#8-claude-code-との付き合い方)
9. [MCP連携(任意)](#9-mcp連携任意)
10. [セキュリティ・運用ルール(必読)](#10-セキュリティ運用ルール必読)
11. [よくあるトラブルと対処法](#11-よくあるトラブルと対処法)
12. [参考リンク](#12-参考リンク)

---

## 1. このテンプレートについて

`claude-kintone-template` は、kintone のカスタマイズ・プラグイン開発に特化した **Claude Code 用テンプレートリポジトリ**です。

主な特徴は次の4点です。

| 特徴 | 内容 |
|---|---|
| AIへの規約注入 | `CLAUDE.md` に kintone 開発の作法を記述し、Claude Code がそれを踏まえて動く |
| ガードレール | `.claude/settings.json` で、認証情報の読み取り拒否やアップロード系操作の確認(ask)を設定 |
| 高速・型安全な開発 | esbuild + TypeScript によるビルド |
| 自動アップロード | `@kintone/customize-uploader` / `@kintone/plugin-uploader` を利用 |

> 💡 **AIを使わない場合**: `CLAUDE.md` と `.claude/` を無視・削除すれば、通常の TypeScript + esbuild + customize-uploader のテンプレートとしてそのまま使えます。


---

## 2. 事前準備

開発を始める前に、以下を用意してください。

- **Node.js / npm** — ビルド・アップロードスクリプトの実行に必要
- **kintone 開発環境のアカウント**(サブドメイン・ユーザー名・パスワード、またはAPIトークン)
- **Claude Code**(AI支援を使う場合。使わない場合は不要)
- Git(バージョン管理・チーム共有用)

---

## 3. セットアップ手順

### 3.1 依存パッケージのインストール

```bash
npm install
```

### 3.2 認証情報のセットアップ

`npm run upload` / `plugin:upload` / `gen:types` などが kintone に接続する際に使われます。

仕組み: `scripts/with-env.mjs` は、ワークスペース直下の `.env` のみを参照します。

**推奨方法: secrets をリポジトリ外に隔離する**

1. リポジトリ外の任意の場所(例 `~/secrets/`)に認証情報ファイルを作成:

    ```env
    KINTONE_BASE_URL=https://your-subdomain.cybozu.com
    KINTONE_USERNAME=your-username
    KINTONE_PASSWORD=your-password
    ```

2. ワークスペース直下の `.env` には、そのファイルへのパスだけを書く:

    ```env
    # <workspace>/.env
    ENV_FILE_REDIRECT=C:\Users\<name>\secrets\kintone-<project>.env
    ```

   相対パスは `.env` のあるディレクトリ基準で解決されます。

**簡易方法**: 個人検証など隔離が不要な場合は、`.env` に直接 `KINTONE_BASE_URL` 等を書いても構いません。

**動作確認:**

```bash
npm run check:env
```

`✓ 認証情報の読み込みに成功しました` と表示されればOKです(リダイレクト構成の場合は最終的に読み込まれた外部ファイルのパスも表示されます)。

> ⚠️ `.env` / `.env.*` / `.mcp.json` は `.gitignore` 済みです。誤って認証情報をリポジトリにコミットしないよう、これらのファイル名は変更しないでください。

*(挿入位置: ここに `.env` リダイレクトの仕組みを示す図 — workspace/.env → ENV_FILE_REDIRECT → 外部secretsファイル、という参照の流れ)*

---

## 4. ディレクトリ構成

```
.
├── CLAUDE.md                    # Claude Code コンテキスト定義
├── .claude/
│   └── settings.json            # AI操作のガードレール(権限ポリシー)
├── docs/                        # ドキュメント(本マニュアルの元ネタ)
├── src/
│   ├── customize/               # カスタマイズ(アプリ単位)
│   │   ├── sample.ts
│   │   └── sample.css
│   └── plugin/                  # プラグイン開発
│       ├── manifest.json
│       ├── src/                 #   TSソース
│       ├── css/ html/ image/    #   静的アセット
│       └── private.ppk          #   秘密鍵(自動生成・.gitignore済み)
├── scripts/
│   ├── build.mjs                # カスタマイズ用ビルド(esbuild)
│   ├── build-plugin.mjs         # プラグイン用ビルド(esbuild + GCC + plugin-packer)
│   ├── deploy-plugin.mjs        # プラグイン用デプロイ(build → upload)
│   └── with-env.mjs             # 認証情報の外部読み込みヘルパー
├── dist/                        # ビルド成果物(Git管理外)
├── types/                       # フィールド型定義(自動生成・Git管理外)
├── customize-manifest.example.json
├── .mcp.example.json
├── cspell.json
├── package.json
└── .env.example
```

---

## 5. カスタマイズ開発フロー (`src/customize/`)

kintone の**アプリ単位のカスタマイズ**(JavaScript / CSS)を開発する際のワークフローです。

### 5.1 ファイルの作成

`src/customize/` にファイルを追加します。ファイル名はアプリを識別しやすい任意の名前でOKです(例: `受注管理.ts`)。

### 5.2 アプリ設定(manifest)の準備

```bash
cp customize-manifest.example.json customize-manifest.json
```

`customize-manifest.json` の `"app"` を、実際のアプリIDに書き換えます。

- `scope`: `"ALL"` / `"ADMIN"` / `"NONE"` から選択
- モバイル用JS/CSSを有効化する場合は `mobile.js` / `mobile.css` に dist パスを追加

> `customize-manifest.json` はアプリIDを含むため `.gitignore` 済みです。コミットするのは `.example.json` のみにしてください。

### 5.3 開発の実践フロー

*(挿入位置: ここに「編集 → watch → 目視確認 → build:upload」のサイクル図)*

1. `src/customize/` のファイルを編集
2. 開発中は `npm run watch` でウォッチビルドしながら確認
3. 完成したら以下でビルド

    ```bash
    npm run build          # src/customize/*.{ts,js,css} → dist/customize/*
    ```

4. 開発環境で動作確認後、アップロード

    ```bash
    npm run upload         # dist/ を kintone にアップロード
    # または一括で
    npm run build:upload   # ビルド → アップロードを一括実行
    ```

> ⚠️ `upload` / `build:upload` は `.claude/settings.json` で **ask(実行前確認)** に設定されています。Claude Code経由で実行する場合、意図しないアップロードを防ぐため必ず確認プロンプトが挟まります。

### 5.4 TypeScriptの型サポート

`kintone.*` グローバルの型は `@kintone/dts-gen` の ambient 型定義で自動解決されますが、フィールドごとの厳密な型が欲しい場合は以下を実行します。

```bash
npm run gen:types -- --app-id 123   # types/ に型定義を出力
```

---

## 6. プラグイン開発フロー (`src/plugin/`)

`src/customize/` がアプリ単位のカスタマイズなのに対し、`src/plugin/` は**複数アプリに配布可能なプラグイン**用です。

### 6.1 ディレクトリ

```
src/plugin/
├── manifest.json   # プラグインメタデータ
├── src/            # TypeScriptソース(*.ts → js/ にバンドルされる)
│   ├── desktop.ts
│   └── config.ts
├── css/            # CSS(そのままコピー)
├── html/           # 設定画面HTML
└── image/          # アイコン等
```

`manifest.json` の `desktop.js` / `config.js` には、**バンドル後**のパス(例: `js/desktop.js`)を記述する点に注意してください。

### 6.2 実践フロー(サンプルプロジェクトの例)

*(挿入位置: ここに「manifest編集 → src実装 → build → 秘密鍵生成/署名 → 開発環境で動作確認 → upload」の一連の流れを示すフローチャート)*

1. `src/plugin/manifest.json` にプラグイン名・説明・アイコン等を設定
2. `src/plugin/src/desktop.ts`(画面用)、`config.ts`(設定画面用)を実装
3. ビルド

    ```bash
    npm run plugin:build          # src/plugin/ → dist/plugin.zip(署名込み)
    ```

   ビルド時の内部処理は次の3ステップです。
   1. `src/plugin/src/*.ts` を esbuild で `dist/plugin/js/*.js` にバンドル
   2. `manifest.json` / `css` / `html` / `image` を `dist/plugin/` にコピー
   3. `@kintone/plugin-packer` で署名・ZIP化 → `dist/plugin.zip`

4. 必要に応じて最適化レベルを指定してビルド

    ```bash
    npm run plugin:build --opt-level=SIMPLE
    ```

5. 開発環境の kintone にアップロードして動作確認

    ```bash
    npm run plugin:upload         # dist/plugin.zip を kintone にアップロード
    # または一括で
    npm run plugin:deploy         # ビルド → アップロード
    npm run plugin:deploy --opt-level=SIMPLE   # 最適化あり → アップロード
    ```

6. kintone管理画面でプラグインが反映されているか確認し、対象アプリに適用してテスト

> ⚠️ `plugin:upload` / `plugin:deploy` も settings.json で **ask** に設定されています。

### 6.3 コンパイル最適化レベル(`--opt-level`)

| 値 | 内容 |
|---|---|
| `WHITESPACE_ONLY` | 空白・コメント除去のみ。最も安全 |
| `SIMPLE`(推奨) | ローカル変数のリネーム |
| `ADVANCED` | 全シンボルリネーム・デッドコード除去。**kintoneグローバルAPIの呼び出しが壊れる場合があるため要検証** |

### 6.4 秘密鍵 `src/plugin/private.ppk` の取り扱い(重要)

この鍵はプラグインの一意性を保証するものです。**紛失するとプラグインの更新ができなくなります**(=既存プラグインの上書き更新が不可能になり、実質的に別プラグイン扱いになります)。`.gitignore` 済みのため、**別途バックアップを推奨**します。

- **新規プラグインの場合**: 初回 `plugin:build` 実行時に自動生成され、`src/plugin/private.ppk` に保存されます。以降のビルドではこの鍵が使い回されます。
- **既存の鍵を持ち込む場合**(既存プラグインの更新など): 以下のようにファイルを配置するだけで自動的に使われます。

    ```bash
    cp /path/to/your-existing.ppk src/plugin/private.ppk
    ```

  - ファイル名は必ず `private.ppk` にしてください(スクリプトがこのパス固定で参照)
  - kintoneはこの鍵から導出されるPlugin IDで「同一プラグインか」を判定するため、対応する鍵で署名すれば自動的に「更新」扱いになります
  - 鍵を変えると別プラグイン扱いになるので注意

---

## 7. コマンドリファレンス

### カスタマイズ系

| コマンド | 内容 | 実務で使うタイミング |
|---|---|---|
| `npm run build` | `src/customize/` をビルド → `dist/customize/` | コード変更後、アップロード前の確認 |
| `npm run watch` | ウォッチモードでビルド | 実装中、変更のたびに手動ビルドしたくない時 |
| `npm run upload` | `dist/customize/` をアップロード(ask確認あり) | 開発環境で動作確認できた後 |
| `npm run build:upload` | ビルド → アップロードを一括実行(ask確認あり) | ビルドとアップロードを毎回セットで行いたい時 |

### プラグイン系

| コマンド | 内容 | 実務で使うタイミング |
|---|---|---|
| `npm run plugin:build` | ビルド → `dist/plugin.zip` | プラグインのコード変更後 |
| `npm run plugin:build --opt-level=SIMPLE` | GCCでコンパイル・最適化してからビルド | 配布用に軽量化したい時 |
| `npm run plugin:upload` | `dist/plugin.zip` をアップロード(ask確認あり) | 開発環境での動作確認後 |
| `npm run plugin:deploy` | ビルド → アップロード一括(ask確認あり) | 一連の作業をまとめて行いたい時 |
| `npm run plugin:deploy --opt-level=SIMPLE` | 最適化ビルドあり → アップロード(ask確認あり) | 最適化込みで一括デプロイしたい時 |

### 共通・その他

| コマンド | 内容 |
|---|---|
| `npm run gen:types` | kintoneフィールド型定義を自動生成 |
| `npm run check:env` | 認証情報の読み込み確認 |
| `npm run type-check` | TypeScript型チェック |
| `npm run check` | Biomeでリント+フォーマット一括 |
| `npm run lint` | Biomeでリント |
| `npm run format` | Biomeでフォーマット |
| `npm run spell` | cspellでスペルチェック |

> `cspell.json` の `words` に kintone 固有の単語を登録済みです。未知語エラーが出た場合は `words` に追加するか、コード上に `// cspell:ignore <word>` を付けてください。

**注意点(全コマンド共通)**:
- アップロード系コマンド(`upload` / `build:upload` / `plugin:upload` / `plugin:deploy` 系)は、Claude Code経由の場合 `ask` 設定により実行前に必ず確認が入ります。誤操作防止のためであり、正常な動作です。
- `npm run gen:types` や `npm run check:env` は kintoneへの接続(読み取り)が発生するため、認証情報が正しくセットアップされていないと失敗します。

---

## 8. Claude Code との付き合い方

### 8.1 CLAUDE.md の役割

`CLAUDE.md` には、kintone開発における必須ルールが定義されています。

- **kintoneの仕様・APIに関するコードを書く前に、必ず該当する公式ドキュメントをWebFetchする**(記憶や推測で書かない、というルール)
- 参照すべき公式ドキュメント一覧(JS APIリファレンス、イベント一覧、REST APIリファレンス、フィールドタイプ仕様、プラグイン開発、`@kintone/rest-api-client`、`cli-kintone`)
- **`@kintone/kintone-js-sdk` は非推奨** → 現行の `@kintone/rest-api-client` を使う
- 2020年以前のQiita/Zenn記事のサンプルコードは動かない可能性が高いという注意
- 不明な仕様に遭遇したら、公式docsをWebFetchし、それでも解決しなければ推測で進めず人間に判断を仰ぐ

つまり、**Claude Codeは毎回最新の公式ドキュメントを確認してから実装する**ように設計されています。古い記憶ベースの実装で仕様がズレることを防ぐためです。

### 8.2 自然言語での指示例

`CLAUDE.md` の文脈を踏まえた上で、Claude Codeには自然言語でそのまま指示できます。

```
「一覧画面にカテゴリフィルターを追加して」
「submit イベントでタイトル必須バリデーションを追加して」
「新しいカスタマイズファイルを『受注管理』という名前で作って」
「フィールドコードをマジックストリングで書いてる箇所を定数に置き換えて」
「現在のソースを kintone 開発環境にアップロードして」
「アプリID 3 のフィールド型定義を生成して」
```

### 8.3 ガードレール(`.claude/settings.json`)

Claude Codeが実行できる操作は、権限ポリシーで制御されています。

| 区分 | 内容 |
|---|---|
| **deny(拒否)** | `.env`系ファイルの読み取り、`.mcp.json`、秘密鍵(`.ppk`)の読み取り、`rm -rf`、`git push --force`、`git reset --hard` など破壊的操作 |
| **ask(実行前確認)** | アップロード系コマンド全般、`git push` / `commit` / `merge` / `rebase` / `tag`、`gh pr merge` / `release`、manifest系ファイルへの書き込みなど |
| **allow(自動許可)** | `npm install` / `build` / `watch` / `lint` / `format` / `type-check` など読み取り・ビルド系、`git status` / `diff` / `log` などの参照系 |

これにより、**認証情報の流出**と**意図しない本番反映・破壊的操作**を防いでいます。ガードレールがあるからといって内容を確認せず許可しないよう注意してください。

---

## 9. MCP連携(任意)

Claude CodeからMCP(Model Context Protocol)経由で外部サービスを直接操作することもできます。

### セットアップ

```bash
cp .mcp.example.json .mcp.json
```

`.mcp.example.json` を参考にプロジェクトルートに `.mcp.json` を作成します(`.gitignore` 済み)。

### 含まれるMCPサーバー

| サーバー | 用途 |
|---|---|
| `@kintone/mcp-server`(公式) | レコードの参照・追加・更新・削除を自然言語で操作 |
| `@playwright/mcp`(Microsoft公式) | ブラウザ自動操作。カスタマイズ/プラグイン反映後のUI動作確認に利用 |

> ⚠️ `@kintone/mcp-server` は**書き込み系操作も可能**です。**接続先は必ず開発環境にしてください。** 本番テナントに接続した状態でAIに操作を任せると、意図しないデータ変更が起きるリスクがあります。

使用例:

```
「kintone のレコード一覧画面を開いて、追加したカスタマイズが動いているか確認して」
「プラグイン設定画面のフォームに値を入力して保存できるかテストして」
「ヘッドレスでログイン画面を開いてスクリーンショットを撮って」
```

認証情報は `.env` から `scripts/with-env.mjs` 経由で読み込まれるため、`.mcp.json` に直接書く必要はありません。

---

## 10. セキュリティ・運用ルール(必読)

- **本番環境への直接アップロードは禁止**です。必ず開発環境で確認後に手動デプロイしてください。
- MCP経由でkintoneに接続する場合も、**接続先は開発環境限定**です。

### Gitにコミットしてはいけないファイル(`.gitignore` 済み)

| ファイル | 理由 |
|---|---|
| `.env` / `.env.*` | 認証情報 |
| `.mcp.json` | MCP接続設定(認証情報を含む可能性) |
| `dist/` | ビルド成果物 |
| `src/plugin/private.ppk` | プラグイン秘密鍵(紛失注意) |
| `customize-manifest.json` | アプリIDを含む(`.example.json`のみコミット) |
| `types/` | 自動生成された型定義 |

これらのファイルが誤ってコミットされていないか、`git status` / `git diff` で都度確認する習慣をつけてください。

---

## 11. よくあるトラブルと対処法

| 症状 | 原因・対処法 |
|---|---|
| `npm run check:env` が失敗する | `.env` の記述ミス、または `ENV_FILE_REDIRECT` のパスが誤っている可能性。相対パスは `.env` のあるディレクトリ基準で解決される点に注意 |
| プラグインが「更新」ではなく「新規」として認識される | `private.ppk` が変わっている(または紛失した)可能性。バックアップした鍵を `src/plugin/private.ppk` に配置し直す |
| `--opt-level=ADVANCED` でビルド後、プラグインが動かない | ADVANCED最適化は全シンボルをリネームするため、kintoneグローバルAPIの参照が壊れることがある。`SIMPLE` に落とす |
| アップロードコマンドを実行しても何も起きない(Claude Code経由) | `settings.json` の `ask` 設定により確認プロンプトが出ているはず。見落としていないか確認 |
| cspellで見慣れない単語がエラーになる | `cspell.json` の `words` に追加するか、該当行に `// cspell:ignore <word>` を付ける |
| kintoneのAPI仕様について実装中に迷った | `CLAUDE.md` のルールに従い、公式ドキュメントをまず確認する。それでも不明なら推測で進めず、チーム内で確認する |

---

## 12. 参考リンク

- [MCP設定](mcp.md) — Claude Codeからkintone APIを直接操作する場合
- [Claude Codeとの対話例](claude-tips.md)
- ディレクトリ構成の詳細はルート `README.md` を参照
- kintone JS API: https://cybozu.dev/ja/kintone/docs/js-api/
- kintone REST API: https://cybozu.dev/ja/kintone/docs/rest-api/
- プラグイン開発: https://cybozu.dev/ja/kintone/docs/plug-in/
- `@kintone/rest-api-client`: https://github.com/kintone/js-sdk/tree/main/packages/rest-api-client
- `cli-kintone`: https://cybozu.dev/ja/kintone/docs/cli-kintone/
