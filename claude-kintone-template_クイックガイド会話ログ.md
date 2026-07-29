# Claude活用ログ：kintone開発テンプレートのセットアップマニュアル作成

## 概要

社内の `claude-kintone-template`（kintoneカスタマイズ / プラグイン開発用テンプレート）について、初心者メンバーでも迷わず環境構築できるよう、Claudeにマニュアル作成を依頼しました。

---

## 投げたプロンプト

リポジトリ内の `README.md` / `docs/setup.md` / `docs/mcp.md` / `CLAUDE.md` / `.claude/settings.json` / `docs/claude-tips.md` をアップロードした上で、以下のように依頼しました。

```
このkintone-devkitフォルダ内のmdファイルを参考に、
社内メンバー向けの環境構築マニュアルをMarkdown形式で作成してください。

対象読者：Git/Node.jsにまだ慣れていないメンバー
含めてほしい章：
1. kintone dev kitとは何か
2. 環境構築手順

環境構築の章は、このマニュアル通りに設定すれば
実際に開発・ビルドができる状態になるところまで、
必要な設定を一通り含めてください。

他に参考にした方がよいファイルがあれば教えてください。
```

---

## Claudeの動き（ポイント）

- アップロードされた6つのmdファイルを実際に読み込んだ上でマニュアルを作成（記憶や推測で書いていない）
- ドキュメント間で記載が食い違っていた箇所（プラグインの一括コマンド名が`README.md`と`docs/setup.md`で異なっていた）を発見し、どちらを採用するか確認を挟んでから作成
- 「設定を一通り含めて」という指示に対して、実際に手元で`npm run build`まで通る状態になるよう、単なる`npm install`だけでなく、リポジトリのclone・`customize-manifest.json`の作成など抜けがちな手順も補って追加
- 最後にチェックリスト形式で「これで完了」と分かるようにまとめた

---

## 生成された結果（成果物）

`claude-kintone-template_クイックガイド.md` として、以下の構成で出力されました。

```markdown
# claude-kintone-template クイックガイド

## 目次
1. claude-kintone-templateとは
2. 環境構築手順

## claude-kintone-templateとは
- 概要
- できること（CLAUDE.md / .claude/settings.json / esbuild+TypeScript / 自動アップロード / secrets隔離）
- 利用するメリット
- kintone開発における役割

## 環境構築手順
- 事前に必要なもの
- ステップ0: リポジトリを取得する（git clone）
- ステップ1: 依存パッケージのインストール（npm install）
- ステップ2: 認証情報のセットアップ（.env / secrets隔離 or 直接記述）
- ステップ3: アプリ設定ファイル（manifest）の作成
- ステップ4: 動作確認（npm run build）
- （任意）ステップ5: プラグイン開発をする場合
- （任意）ステップ6: MCP連携をする場合
- 確認済みの設定ファイル一覧
- 環境構築チェックリスト
```

> 📎 実際のファイル本体は別途 `claude-kintone-template_クイックガイド.md` として出力済みです。

---

## 所感・気づき

- ソースとなるmdファイルを渡すだけで、内容の矛盾点まで拾って質問してくれた
- 「これ通りに設定すれば使える状態に」という条件だけで、実際に動く状態まで持っていくために必要な手順（manifest作成など）を漏らさず含めてくれた
