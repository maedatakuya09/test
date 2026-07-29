# claude-kintone-template クイックガイド

対象読者:
- kintone開発者
- Git / Node.js などの開発環境にまだ慣れていないメンバー

このガイドの手順をすべて実施すると、**カスタマイズの開発・ビルド・アップロードができる状態**になります。

---

## 目次
1. [claude-kintone-templateとは](#claude-kintone-templateとは)
2. [環境構築手順](#環境構築手順)

---

## claude-kintone-templateとは

### 概要

`claude-kintone-template` は、**kintoneのカスタマイズ / プラグイン開発**に特化した、**Claude Code専用**の開発テンプレートです。TypeScript + esbuildをベースに構築されています。

> 💡 補足: 「Claude Code」はAnthropicのAIコーディングツールです。このテンプレートは、AIにkintone開発のルールやガードレールを注入した状態で開発を進められるように設計されています。

### できること

| 機能 | 内容 |
|---|---|
| **CLAUDE.md** | kintone開発の規約・作法をAIに注入する設定ファイル |
| **`.claude/settings.json`** | AI操作のガードレール（認証情報のRead拒否、副作用のある操作はask化） |
| **esbuild + TypeScript** | 高速なバンドルと型安全な開発 |
| **自動アップロード** | `@kintone/customize-uploader` と `@kintone/plugin-uploader` によるアップロード自動化 |
| **secretsのリポジトリ外配置** | `.env`のリダイレクト機能により、誤コミット・情報流出を防止 |

### 利用するメリット

- ✅ **AIに安全に開発を任せられる**: ガードレールにより、認証情報の読み取りや危険な操作をAIが勝手に実行しない
- ✅ **セキュリティリスクの低減**: secretsをリポジトリ外に隔離できる仕組みが標準で用意されている
- ✅ **開発の高速化**: esbuildによる高速ビルドと自動アップロード
- ✅ **AIなしでも使える**: `CLAUDE.md`や`.claude/`を無視・削除すれば、通常のTypeScript + esbuildテンプレートとしてそのまま使用可能

### kintone開発における役割

このテンプレートは、**アプリ単位のカスタマイズ（JS/CSS）** と **複数アプリで再利用できるプラグイン** の両方の開発を1つのプロジェクトでカバーします。Claude Codeと組み合わせて使うことを前提に設計されている点が最大の特徴です。

---

## 環境構築手順

以下を上から順番に実施してください。**ステップ4まで完了すれば、カスタマイズの開発・アップロードができる状態**になります。プラグイン開発やMCP連携をする場合は、追加のステップも実施してください。

### 事前に必要なもの

- Node.js / npm
- Git
- kintoneのログイン情報（またはAPIトークン）
- 開発用のkintone環境（開発用スペース・アプリ）

### ステップ0: リポジトリを取得する

社内のGitリポジトリからテンプレートを取得します（リポジトリURLは社内の共有先を確認してください）。

```bash
git clone <社内リポジトリのURL>
cd <クローンしたフォルダ名>
```

### ステップ1: 依存パッケージのインストール

```bash
npm install
```

### ステップ2: 認証情報のセットアップ（必須）

`npm run upload` / `plugin:upload` / `gen:types` などが kintone に接続する際に使う認証情報を設定します。

#### 仕組み

`scripts/with-env.mjs` は、ワークスペース直下の `.env` **のみ**を参照します。`.env`内に `ENV_FILE_REDIRECT=<外部ファイルへのパス>` と記述すると、そのパス先のファイルから値を読み込みます（リダイレクト機能）。

#### 方法A（推奨）: secretsをリポジトリ外に隔離する

社内では基本的に**こちらの方法を推奨**します。

1. ワークスペースの**外**の任意の場所（例: `C:\Users\<name>\secrets\` や `~/secrets/`）にsecrets用ファイルを作成します。

   ```env
   KINTONE_BASE_URL=https://your-subdomain.cybozu.com
   KINTONE_USERNAME=your-username
   KINTONE_PASSWORD=your-password
   ```

2. ワークスペース直下に `.env` を作成し、上記ファイルへの**パスだけ**を書きます（値は書きません）。

   ```env
   # <workspace>/.env
   ENV_FILE_REDIRECT=C:\Users\<name>\secrets\kintone-<project>.env
   ```

> 📝 相対パスは `.env` のあるディレクトリ基準で解決されます。`.env`自体も`.gitignore`済みなので、万一secretsを書いてしまっても保険になります。

#### 方法B（簡易）: `.env` に直接記述する

個人検証などでsecrets隔離が不要な場合は、ワークスペース直下の `.env` に直接値を書いても構いません。

```env
KINTONE_BASE_URL=https://your-subdomain.cybozu.com
KINTONE_USERNAME=your-username
KINTONE_PASSWORD=your-password
```

> ⚠️ 社内の共有プロジェクトや本番相当の情報を扱う場合は、方法A（隔離）を優先してください。

#### 動作確認

```bash
npm run check:env
```

`✓ 認証情報の読み込みに成功しました` と表示されればOKです。リダイレクト構成の場合は、実際に読み込まれた外部ファイルのパスが「ソース」として表示されます。

### ステップ3: アプリ設定ファイル（manifest）の作成（カスタマイズ開発の場合・必須）

カスタマイズを特定のkintoneアプリにアップロードするために、対象アプリのIDを設定します。**これを設定しないと、アップロードができません。**

```bash
cp customize-manifest.example.json customize-manifest.json
```

作成した `customize-manifest.json` を開き、以下を編集します。

- `"app"`: アップロード先のkintoneアプリID（数値）に書き換える
- `scope`: `"ALL"` / `"ADMIN"` / `"NONE"` から選択する
- モバイル対応が必要な場合は `mobile.js` / `mobile.css` にdistパスを追加する

> 📝 `customize-manifest.json` はアプリIDを含むため `.gitignore` 済みです（コミットされるのは `.example.json` のみ）。メンバーごとに開発対象アプリが異なる場合は、各自でこのファイルを作成してください。

### ステップ4: 動作確認（ビルドしてみる）

ここまで設定できていれば、以下のコマンドが正常に動くはずです。

```bash
npm run build
```

`dist/customize/` にファイルが生成されればOKです。ここまでで、**カスタマイズの開発・ビルド・アップロードができる状態**になっています。

> 🚫 実際に `npm run upload` / `npm run build:upload` を実行する際は、必ず**開発環境のアプリ**であることを確認してから行ってください（本番環境への直接アップロードは禁止です）。

---

### （任意）ステップ5: プラグイン開発をする場合

プラグイン開発を行う場合、秘密鍵は初回ビルド時に自動生成されるため、事前設定は基本的に不要です。

- 新規プラグインの場合: `npm run plugin:build` を初めて実行した際に `src/plugin/private.ppk` が自動生成されます
- 既存プラグインを更新する場合: 既存の `.ppk` ファイルを `src/plugin/private.ppk` に配置してから作業を始めてください

  ```bash
  cp /path/to/your-existing.ppk src/plugin/private.ppk
  ```

### （任意）ステップ6: Claude CodeからkintoneやブラウザをMCP連携で操作する場合

Claude Codeからkintoneのレコード操作やブラウザでの動作確認まで自動化したい場合のみ設定します。**必須ではありません。**

```bash
cp .mcp.example.json .mcp.json
```

- 認証情報は`.env`（ステップ2で設定済みのもの）から自動的に読み込まれるため、`.mcp.json`に直接書く必要はありません
- 🚨 kintone MCPサーバーは書き込み操作も可能なため、**接続先は必ず開発環境にしてください**

### 確認済みの設定ファイル（クローン時点ですでに用意されているもの）

以下はリポジトリに最初から含まれており、**個別の設定作業は不要**です。存在の確認だけしておきましょう。

| ファイル | 役割 |
|---|---|
| `CLAUDE.md` | Claude Codeにkintone開発の規約を注入する設定（公式ドキュメント参照ルールなど） |
| `.claude/settings.json` | Claude Codeの操作許可設定（認証情報の読み取り拒否、アップロード系はask確認など） |

### 環境構築チェックリスト

- [ ] リポジトリをクローンした（ステップ0）
- [ ] `npm install` を実行した（ステップ1）
- [ ] `.env`（または`ENV_FILE_REDIRECT`先）にkintone認証情報を設定した（ステップ2）
- [ ] `npm run check:env` が成功した（ステップ2）
- [ ] `customize-manifest.json` を作成し、対象アプリIDを設定した（ステップ3）
- [ ] `npm run build` が成功した（ステップ4）
- [ ] （プラグイン開発する場合）秘密鍵の扱いを確認した（ステップ5）
- [ ] （MCP連携する場合）`.mcp.json` を作成した（ステップ6）

すべてチェックできれば、環境構築は完了です。
