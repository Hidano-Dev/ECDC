# プロジェクト構造

## 構成方針

機能ベース（feature-first）の構成を採用する。配線GUI が主要アプリケーションであり、そのドメイン（機材、接続、会場、YAML エクスポート）に沿ってコードを整理する。

## ディレクトリパターン

### プロジェクトルート
**場所**: `/`
**目的**: 設定ファイル、ドキュメント、仕様管理
**パターン**: `package.json`, `vite.config.ts`, `tsconfig.json` などのプロジェクト設定。要求定義書は `/docs/` に配置。仕様は `.kiro/specs/` で管理。

### ソースコード
**場所**: `/src/`
**目的**: アプリケーションの全ソースコード
**パターン**: エントリポイント (`main.tsx`, `App.tsx`) をルートに、機能単位でサブディレクトリに分割

### 機能モジュール
**場所**: `/src/features/`
**目的**: ドメイン機能ごとの独立したモジュール
**例**:
- `/src/features/canvas/` -- React Flow キャンバス関連（ノード、エッジ、インタラクション）
- `/src/features/equipment/` -- 機材管理（パレット、カテゴリ、Google Sheets 連携）
- `/src/features/connections/` -- 接続定義（プロパティ編集、ケーブル種別）
- `/src/features/venue/` -- 会場定義 YAML の読み込み・ゾーン表示
- `/src/features/export/` -- 案件定義 YAML のインポート/エクスポート

### 共通コンポーネント
**場所**: `/src/components/`
**目的**: 機能横断的な再利用可能 UI コンポーネント
**パターン**: PascalCase ファイル名。ビジネスロジックを持たない

### 型定義
**場所**: `/src/types/`
**目的**: ドメインモデルの TypeScript 型定義
**パターン**: `equipment.ts`, `connection.ts`, `venue.ts` など、ドメイン概念ごとにファイルを分割

### ユーティリティ
**場所**: `/src/utils/`
**目的**: 汎用ヘルパー関数
**パターン**: YAML パース、データ変換、バリデーションなど

### 静的アセット
**場所**: `/public/` または `/src/assets/`
**目的**: アイコン、画像、静的ファイル

## 命名規約

- **ファイル（コンポーネント）**: PascalCase -- `EquipmentPalette.tsx`, `ConnectionEditor.tsx`
- **ファイル（ユーティリティ/型）**: camelCase -- `yamlExporter.ts`, `equipment.ts`
- **コンポーネント**: PascalCase -- `EquipmentNode`, `CableTypeSelector`
- **関数/変数**: camelCase -- `parseVenueYaml`, `equipmentList`
- **型/インターフェース**: PascalCase -- `Equipment`, `Connection`, `VenueDefinition`
- **定数**: UPPER_SNAKE_CASE -- `CABLE_TYPE_PRESETS`, `DEFAULT_NODE_SIZE`

## インポート構成

```typescript
// 1. 外部ライブラリ
import { useCallback } from 'react'
import { Node, Edge } from 'reactflow'

// 2. 内部絶対パス（パスエイリアス使用）
import { Equipment } from '@/types/equipment'
import { parseVenueYaml } from '@/utils/yamlParser'

// 3. 同一機能内の相対パス
import { EquipmentNode } from './EquipmentNode'
```

**パスエイリアス**:
- `@/` -> `src/`

## コード構成の原則

1. **機能の独立性**: 各 feature モジュールは自身のコンポーネント、フック、ユーティリティを内包する。他の feature への直接依存は避け、共通の型定義やイベントを介して連携する
2. **データフローの明確化**: Google Sheets -> 機材データ -> キャンバス配置 -> YAML エクスポートという単方向のデータフローを意識する
3. **ドメイン型の活用**: 機材カテゴリ、ケーブル種別、接続プロパティなど、ドメイン固有の概念は全て TypeScript の型として定義する
4. **YAML スキーマとの対応**: TypeScript の型定義は案件定義 YAML / 会場定義 YAML のスキーマと 1:1 で対応させる

---
_パターンを記述し、ファイルツリーの網羅的なリストは避ける。パターンに従う新規ファイルの追加時にこのドキュメントの更新は不要_
