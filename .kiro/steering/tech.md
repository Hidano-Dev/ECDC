# 技術スタック

## アーキテクチャ

クライアントサイド中心の SPA 構成。配線GUI が主要な Web アプリケーションとして機能し、外部データソース（Google Sheets）からの読み込みと YAML エクスポートを行う。draw.io XML の生成は Claude Code + MCP 経由で別プロセスとして実行する。

```
Google Sheets API -> [配線GUI (React SPA)] -> 案件定義YAML
会場定義YAML ------> [配線GUI (React SPA)]
案件定義YAML + 会場定義YAML -> [Claude Code + draw.io MCP] -> draw.io XML
```

## コア技術

- **言語**: TypeScript
- **フレームワーク**: React
- **キャンバスライブラリ**: React Flow（ノード&エッジ GUI に特化、D&D/ズーム/パン標準対応）
- **ビルドツール**: Vite
- **ランタイム**: Node.js

## 主要ライブラリ

開発パターンに影響を与えるもののみ記載:

- **React Flow** -- キャンバス上のノード（機材）とエッジ（接続線）の管理。配線GUI の中核
- **js-yaml** -- YAML のパース/シリアライズ。案件定義・会場定義 YAML の読み書き
- **Google Sheets API v4** -- 機材台帳の読み込み。将来は CSV 入力でも代替可能

## 開発基準

### 型安全性

TypeScript を使用する。機材カテゴリ、ケーブル種別、接続プロパティなど、ドメイン固有の値は型として定義する。

### コード品質

ESLint + Prettier を使用する。設定は Vite のデフォルト推奨構成をベースにする。

### テスト

ユニットテストは主要なデータ変換ロジック（YAML パース、エクスポート、Google Sheets データの正規化）を対象とする。GUI のインタラクションテストは初期スコープでは優先度低。

## 開発環境

### 前提ツール

- Node.js / npm
- Claude Code + draw.io MCP（構成図生成に使用）
- Google Drive デスクトップアプリ

### npm レジストリ設定

```bash
# .npmrc で設定済み
registry=https://registry.npmjs.org/
```

### 主要コマンド

```bash
# 開発サーバー: npm run dev
# ビルド: npm run build
# テスト: npm test
```

## 重要な技術的判断

1. **React Flow の採用**: 機材配置と接続定義に特化した GUI を効率的に構築するため。汎用キャンバスライブラリではなくノード&エッジ専用を選択
2. **YAML を中間データ形式に**: 人間が読み書きでき、Claude Code が解釈しやすい形式として採用。JSON ではなく YAML を選んだのは可読性重視
3. **draw.io XML 生成は Claude Code に委譲**: 専用変換スクリプトではなく Claude Code + MCP でインタラクティブに生成・修正する方針。将来的に自動化スクリプト化も視野に入れる
4. **ローカル実行優先**: 個人ツールのため、認証やサーバーサイドの複雑さを排除。Vite dev server でローカル実行が基本

---
_基準とパターンを記述し、全依存関係のリストは避ける_
