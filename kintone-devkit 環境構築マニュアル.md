# kintone dev kit 初心者向け環境構築マニュアル

対象読者:
- kintone開発者
- Git / Node.js などの開発環境にまだ慣れていないメンバー

このマニュアルを読み終えると、以下ができるようになります。
- kintone-devkitが「何をするツールか」を理解できる
- 自分のPCにローカル開発環境を構築し、kintoneカスタマイズ開発を開始できる

---

## 目次
1. [kintone dev kitとは](#kintone-dev-kitとは)
2. [必要な環境](#必要な環境)
3. [環境構築手順](#環境構築手順)
4. [基本的な使い方](#基本的な使い方)
5. [トラブルシューティング](#トラブルシューティング)
6. [まとめ](#まとめ)

---

## kintone dev kitとは

### 概要

`kintone-devkit`は、**TypeScript + Vite** をベースにしたkintone開発用のツールキット（OSS：オープンソースソフトウェア）です。

- 開発元: [https://github.com/oga114/kintone-devkit](https://github.com/oga114/kintone-devkit)
- ライセンス: MIT（社内利用・改変も可能）
- 一言でいうと: **「kintoneのカスタマイズ・プラグイン開発を、モダンなフロントエンド開発と同じ感覚で行えるようにするツール」** です。

> 💡 補足: これまでkintoneのカスタマイズJS/CSSは、管理画面から手動でファイルをアップロードするのが一般的でした。kintone-devkitを使うと、ローカルのエディタでコードを書いて保存するだけで、自動的にkintone環境へ反映されるようになります。

### できること

kintone-devkitは、大きく分けて4つの機能を持っています。

| 機能 | 内容 |
|---|---|
| 🔥 ホットリロード | ファイルを保存すると自動でビルド＆kintoneへアップロード |
| 🔌 プラグイン開発 | プラグインの署名・パッケージング・アップロードを自動化 |
| 📋 スキーマ管理 | 開発環境と本番環境のアプリ設計（フィールド・ビュー等）の差分検出・デプロイ |
| 💾 バックアップ / 復元 | レコードデータや添付ファイルのバックアップ・復元 |

その他にも、以下のような機能があります。

- TypeScriptによる型安全な開発（kintoneの型定義が同梱）
- 1つのプロジェクトで複数アプリ・複数プラグインをまとめて管理
- 対話形式（`npm run create`）でのアプリ・プラグインの新規作成

### 利用するメリット

- ✅ **開発効率の向上**: 保存するだけで反映されるため、手動アップロードの手間がなくなる
- ✅ **環境間のスキーマ同期が簡単**: 開発環境で作った項目を、差分確認しながら本番環境へ反映できる
- ✅ **バックアップ忘れによるデータ消失を防止**: デプロイ前の自動バックアップに対応
- ✅ **プラグイン開発のハードルを下げる**: 署名やZIP化などの面倒な作業を自動化

### kintone開発における役割

kintone-devkitは、kintoneの「カスタマイズ開発」「プラグイン開発」「アプリ設計の管理」を1つのプロジェクトの中で完結させるための **開発支援ツール** という位置づけです。kintone本体の機能を置き換えるものではなく、あくまで「kintoneを開発しやすくするための足回り（開発基盤）」と理解してください。

---

## 必要な環境

### 必要なツール一覧

| ツール | 用途 | 必須度 |
|---|---|---|
| Node.js（v20以上） | kintone-devkitの実行に必要なJavaScript実行環境 | 必須 |
| npm（Node.jsに同梱） | パッケージのインストール・コマンド実行に使用 | 必須 |
| Git | GitHubからソースコードを取得（クローン）するために使用 | 必須 |
| コードエディタ（VS Code推奨） | ソースコードの編集 | 推奨 |
| kintoneのログイン情報 or APIトークン | kintone環境への接続に使用 | 必須 |

### 対応OS

| 環境 | カスタマイズ開発 | プラグイン開発 | 備考 |
|---|---|---|---|
| Windows | ✅ | ✅ | 追加セットアップ不要 |
| macOS | ✅ | ✅ | 追加セットアップ不要 |
| Linux | ✅ | ⚠️ | Chrome関連の依存関係が必要な場合あり |
| WSL | ✅ | ⚠️ | `npm run setup:wsl` の実行が必要 |

> ⚠️ 注意: プラグイン開発時の「自動アップロード機能」はPuppeteer（Chromeの自動操作ツール）を使用します。手動でアップロードする場合は、どの環境でも問題なく動作します。

### その他必要なもの

- **kintoneの管理者権限を持つアカウント**（またはAPIトークン）
  - 開発対象アプリの「アプリ管理」権限が必要です
- **開発用のkintone環境**（本番環境を直接触らないことを強く推奨します）
- （任意）GitHubアカウント：社内でソースコードを管理・共有する場合

---

## 環境構築手順

### ステップ1: Node.jsをインストールする

[Node.js公式サイト](https://nodejs.org/)から、**LTS版（20系以上）** をダウンロードしてインストールします。

インストール後、ターミナル（Windowsの場合はコマンドプロンプトやPowerShell）で以下を実行し、バージョンが表示されればOKです。

```bash
node -v
npm -v
```

### ステップ2: Gitをインストールする

[Git公式サイト](https://git-scm.com/)からインストールします。以下のコマンドでバージョンが表示されれば完了です。

```bash
git -v
```

### ステップ3: kintone-devkitをクローンする

任意の作業フォルダで、以下のコマンドを実行してリポジトリを取得します。

```bash
git clone https://github.com/oga114/kintone-devkit.git
cd kintone-devkit
```

### ステップ4: 依存パッケージをインストールする

```bash
npm install
```

> 🕒 初回はパッケージのダウンロードに数分かかることがあります。

### ステップ5: 環境設定ファイルを作成する

`.env.example` をコピーして `.env` を作成します。

```bash
cp .env.example .env
```

`.env` ファイルをエディタで開き、以下のようにkintoneの接続情報を設定します。

```env
# kintone環境設定
KINTONE_BASE_URL=https://your-domain.cybozu.com
KINTONE_USERNAME=your-username
KINTONE_PASSWORD=your-password

# または、ユーザー名・パスワードの代わりにAPIトークンでも接続可能
# KINTONE_API_TOKEN=your-api-token

# 環境識別子（スキーマ取得コマンドなどで使用）
KINTONE_ENV=dev
```

> 🔒 セキュリティ上の注意: `.env`ファイルには接続情報が平文で保存されます。Gitの管理対象外（`.gitignore`済み）ですが、社内での共有時もパスワードやトークンの取り扱いには十分注意してください。

### ステップ6（WSL利用者のみ）: WSL用セットアップ

WSL（Windows Subsystem for Linux）環境でプラグインの自動アップロード機能を使う場合は、以下を追加で実行します。

```bash
npm run setup:wsl
```

これにより、Puppeteer/Chrome用の追加ライブラリがインストールされます。

### ここまでで完了していること

- ✅ Node.js / Git のインストール
- ✅ kintone-devkitのソースコード取得
- ✅ 依存パッケージのインストール
- ✅ kintone接続情報の設定

これで、ローカル環境からkintoneにアクセスしてカスタマイズ開発を行う準備が整いました。

---

## 基本的な使い方

### プロジェクト構成

kintone-devkitのフォルダ構成は以下の通りです。開発するのは主に `src/apps/` 配下です。

```
kintone-devkit/
├── src/
│   ├── apps/                    # カスタマイズのソースコード
│   │   ├── my-app/
│   │   │   ├── index.ts         # エントリーポイント（メインの処理を書く場所）
│   │   │   └── style.css        # スタイル
│   │   └── another-app/
│   │       ├── index.ts
│   │       └── style.css
│   └── types/
│       └── kintone.d.ts         # kintoneのTypeScript型定義
├── dist/                        # ビルド後のファイルが出力される場所
│   └── my-app/
│       ├── my-app.js
│       └── my-app.css
├── .kintone/                    # kintoneから同期した既存ファイルなど
├── scripts/                     # ビルド・管理用スクリプト
├── kintone.config.ts            # アプリごとの設定ファイル
├── .env                         # 環境変数（自分で作成）
└── .env.example                 # 環境変数のテンプレート
```

### 開発から反映までの流れ

kintone-devkitでの開発は、大まかに以下の流れで進めます。

```mermaid
flowchart LR
    A[1. アプリを新規作成] --> B[2. コードを編集]
    B --> C[3. 保存すると自動ビルド&アップロード]
    C --> D[4. kintone画面で動作確認]
    D --> E[5. 本番用ビルド]
    E --> F[6. 本番環境へアップロード]
```

1. `npm run create` で新しいアプリの雛形を作成
2. `src/apps/<アプリ名>/index.ts` にカスタマイズのコードを記述
3. `npm run dev` を実行しておくと、保存するたびに自動でビルド＆kintoneへアップロードされる
4. kintone画面をブラウザで開き、実際の動作を確認
5. 動作確認が済んだら、本番用ビルド（`npm run build`）を行う
6. 本番アプリへアップロード（`npm run upload`）

### コマンド例

#### 1. 新規アプリの作成

```bash
npm run create
```

対話形式で「アプリ名」や「環境パターン（単一環境／ソースコード分離／スキーマ同期用）」を選択すると、雛形が自動生成されます。

#### 2. 開発モード（ホットリロード）

```bash
# すべてのアプリを対象に開発モードで起動
npm run dev

# 特定のアプリだけを対象にする場合
npm run dev -- my-app
```

ファイルを保存するたびに、自動的にビルド＆kintoneへアップロードされます。

#### 3. 本番用ビルド

```bash
npm run build -- my-app
```

#### 4. 既存ファイルの同期（すでにkintoneにあるカスタマイズを取り込む）

```bash
npm run sync -- my-app
```

#### 5. アプリ設計（スキーマ）の取得・差分確認・デプロイ

```bash
# 開発環境のスキーマを取得
npm run schema -- my-app

# 本番環境のスキーマを取得
KINTONE_ENV=prod npm run schema -- my-app

# 差分を確認
npm run schema:diff -- my-app

# 本番へデプロイ（まずはドライランで確認）
npm run schema:deploy -- my-app
npm run schema:deploy -- my-app --execute
```

> ⚠️ 本番環境への反映前は、必ずドライラン（`--execute`をつけない状態）で内容を確認してください。

#### 6. レコードのバックアップ・復元

```bash
# バックアップ
npm run backup -- my-app

# 復元
npm run backup:restore -- my-app
```

主要コマンドの一覧は以下の通りです。

| コマンド | 説明 |
|---|---|
| `npm run create` | 新しいアプリを対話的に作成 |
| `npm run dev -- <app>` | 開発モード（ホットリロード） |
| `npm run build -- <app>` | 本番用ビルド |
| `npm run sync -- <app>` | 既存ファイルの同期 |
| `npm run schema -- <app>` | アプリスキーマの取得 |
| `npm run schema:diff` | 環境間のスキーマ差分検出 |
| `npm run schema:deploy -- --execute` | スキーマのデプロイ実行 |
| `npm run backup -- <app>` | レコードのバックアップ |
| `npm run backup:restore -- <app>` | バックアップからの復元 |
| `npm run typecheck` | TypeScriptの型チェック |

---

## トラブルシューティング

### よくあるエラー

#### ① アップロードエラーが出る

- `.env`のkintone接続情報（URL・ユーザー名・パスワード・APIトークン）が正しいか確認する
- 対象のアプリIDが正しく設定されているか確認する
- ログインユーザーに「アプリ管理」権限があるか確認する

#### ② ビルドエラーが出る

依存パッケージやキャッシュの不整合が原因のことが多いです。以下を試してください。

```bash
# node_modulesを再インストール
rm -rf node_modules && npm install

# Viteのキャッシュをクリア
rm -rf node_modules/.vite
```

> 💡 Windowsの場合は `rm -rf` の代わりにエクスプローラーでフォルダを削除するか、PowerShellで `Remove-Item -Recurse -Force node_modules` を使用してください。

#### ③ 型エラーが出る

```bash
npm run typecheck
```

を実行し、エラー内容（ファイル名・行番号）を確認して修正します。

#### ④（WSL環境）プラグインの自動アップロードが動かない

WSL用のセットアップが未実施の可能性があります。

```bash
npm run setup:wsl
```

を実行し、Chrome関連の依存ライブラリをインストールしてください。

### 解決方法（共通の切り分け手順）

エラーが解決しない場合は、以下の順番で切り分けを行うと原因を特定しやすくなります。

1. `node -v` / `npm -v` でNode.jsのバージョンが20以上か確認する
2. `.env`の設定内容にタイプミスがないか再確認する
3. `node_modules`を削除して再インストールする
4. エラーメッセージをそのままGoogle検索、またはチーム内で共有する
5. それでも解決しない場合は、[GitHubリポジトリのIssue](https://github.com/oga114/kintone-devkit/issues)を確認・投稿する

---

## まとめ

- kintone-devkitは、**TypeScript + Vite** をベースにした、kintoneのカスタマイズ・プラグイン開発を効率化するOSSツールです。
- **ホットリロード・プラグイン開発支援・スキーマ管理・バックアップ機能** を備えており、手動アップロードや環境間の差分管理の手間を大幅に削減できます。
- 環境構築は、**Node.js / Git のインストール → クローン → `npm install` → `.env`設定** という流れで完了します。
- 開発は `npm run create` でアプリを作成し、`npm run dev` でホットリロード開発を行うのが基本の流れです。
- **本番環境へのスキーマデプロイやレコード操作は必ずドライラン・バックアップを併用**し、慎重に行ってください。

まずは開発環境（本番とは別のアプリ・スペース）で `npm run create` から `npm run dev` までの一連の流れを実際に試してみることをおすすめします。