# 機材構成図作成支援システム 要求定義書

## 1. 背景・目的

### 1.1 現状の課題

バーチャルプロダクション案件では、機材構成図を draw.io で手作業で作成している。参考: `G:\マイドライブ\public\252603_SuiseiBirthday\2026_03_15_TOKYO-NODE-HALL`（3ページ、機材50台以上、接続80本以上）。

現状のペイン:
- 機材が多く、draw.io 上でゼロから配置・接続するのに時間がかかる
- 機材リストと構成図が別管理のため、変更時に二重メンテが発生する
- ケーブル種別・長さ・データ内容など接続の属性情報が図面上でしか管理されない
- 会場の地理的条件（フロア構成、部屋の配置、ケーブル敷設経路）を構造化して管理する手段がない
- 過去案件の機材構成を再利用しにくい

### 1.2 目的

機材台帳（Google Sheets）から GUI で配線を定義し、draw.io の構成図を自動生成するワークフローを構築する。これにより:

- 機材リストの一元管理（スプシが Single Source of Truth）
- 配線作業の GUI 化による効率化
- draw.io XML の自動生成による手作業削減
- 会場の空間情報を構造化し、LLM（Claude Code）が図面レイアウトを支援できる状態にする

### 1.3 想定利用者

- 自分自身（1名）。個人ツールとして開発・運用する

### 1.4 利用頻度

- 不定期。案件ごとに機材構成図を作成する際に使用

## 2. 全体ワークフロー

```
┌─────────────────────────────────────────────────────────┐
│ 1. 会場情報セットアップ（案件初回のみ）                  │
│    会場図面(画像/PDF) → Claude Code が読み取り           │
│    → 会場定義YAML を生成（ゾーン・隣接関係・距離）       │
│    → 人間が補足・修正                                    │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│ 2. 機材登録（Google Sheets）                             │
│    案件で使用する機材をスプレッドシートに登録              │
│    - 機材名、種別、数量、仕様、配置ゾーン                 │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│ 3. 配線定義（GUI ツール）                                │
│    スプシから機材を読み込み → キャンバスに配置            │
│    → 機材間をクリックで接続                               │
│    → 接続プロパティ（ケーブル種別・データ内容・長さ）入力  │
│    → 案件定義 YAML をエクスポート                         │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│ 4. 構成図生成（Claude Code + draw.io MCP）               │
│    案件定義YAML + 会場定義YAML                           │
│    → Claude Code が draw.io XML を生成                   │
│    → draw.io MCP でプレビュー                            │
│    → 修正指示 → 再生成（繰り返し）                       │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│ 5. 仕上げ（draw.io エディタ）                            │
│    生成された図を draw.io で開き、空間配置を微調整         │
│    → Google Drive に保存                                 │
└─────────────────────────────────────────────────────────┘
```

## 3. システム構成

```
 Google Sheets  ──(API)──▶  ┌──────────────┐
  (機材台帳)                │   配線GUI     │
                            │  (Web App)    │──▶  案件定義YAML
 会場定義YAML  ───────────▶ │              │
                            └──────────────┘
                                                      │
                                                      ▼
                            ┌──────────────┐   ┌──────────────┐
                            │  Claude Code  │──▶│  draw.io MCP  │
                            │ (XML生成)     │   │ (プレビュー)  │
                            └──────────────┘   └──────────────┘
```

### コンポーネント一覧

| # | コンポーネント | 役割 | 形態 |
|---|---------------|------|------|
| A | **機材台帳** | 機材マスタデータの管理 | Google Sheets |
| B | **配線GUI** | 機材の配置と接続の定義 | Web アプリケーション |
| C | **会場定義YAML** | 会場の空間情報を構造化 | YAML ファイル |
| D | **構成図ジェネレータ** | YAML から draw.io XML を生成 | Claude Code のプロンプト/スクリプト |

## 4. 機能要件

### 4.1 コンポーネントA: 機材台帳（Google Sheets）

#### 4.1.1 シート構成

**機材マスタシート**

| 列 | 説明 | 例 | 必須 |
|----|------|-----|------|
| id | 機材の一意識別子 | `cam_a` | Yes |
| name | 表示名 | `カメラA` | Yes |
| category | 機材カテゴリ | `camera` | Yes |
| subcategory | 細分類 | `tripod`, `fixed`, `crane` | No |
| description | 補足説明（改行可） | `三脚+VR雲台` | No |
| manufacturer | メーカー | `Blackmagic` | No |
| model | 型番 | `VideoHub 10x10 12G` | No |
| quantity | 台数 | `1` | Yes |
| zone | 配置ゾーン | `hall` | No |
| style_hint | 表示スタイルヒント | `large`, `standard`, `small` | No |
| color | 色指定（任意） | `#647687` | No |
| notes | 備考 | `使用未定` | No |

**カテゴリ定義** (参考図から抽出):

| category | 説明 | 代表的な subcategory |
|----------|------|---------------------|
| `camera` | カメラ | `tripod`, `fixed`, `crane`, `ptz` |
| `pc` | PC / ワークステーション | `desktop`, `notebook` |
| `monitor` | モニタ / ディスプレイ | `operator`, `performer_return`, `direction` |
| `switcher` | 映像スイッチャー / ルーター | `videohub`, `converter` |
| `network` | ネットワーク機器 | `l2_switch`, `wifi_router`, `usb_lan` |
| `audio` | 音響機器 | `mixer`, `monitor`, `speaker`, `amp` |
| `recorder` | 収録機器 | `hyperdeck` |
| `capture` | モーションキャプチャ | `optitrack`, `facial` |
| `lighting` | 照明 | `console`, `led` |
| `person` | 人物 | `operator`, `actor`, `director` |

#### 4.1.2 将来拡張

- CSV / Excel ファイルからのインポートにも対応できるよう、Google Sheets 固有の機能（数式・条件付き書式等）には依存しない列設計とする

### 4.2 コンポーネントB: 配線GUI（Web アプリケーション）

#### 4.2.1 機材読み込み

- Google Sheets API 経由で機材台帳を読み込む
- 読み込んだ機材をカテゴリ別にパレット表示する
- リロード/再読み込みボタンでスプシの最新状態を反映する

#### 4.2.2 キャンバス

- ゾーン（会場定義YAML から読み込み）を背景として表示する
- 機材パレットからキャンバスにドラッグ&ドロップで配置する
- 配置済み機材を移動できる
- 機材をゾーン内に配置すると、所属ゾーンが自動設定される

#### 4.2.3 接続定義

- 機材Aの出力ポートから機材Bの入力ポートへドラッグで接続線を引く
- 接続線をクリックすると、接続プロパティの編集パネルが開く

**接続プロパティ**:

| プロパティ | 説明 | 例 | 必須 |
|-----------|------|-----|------|
| cable_type | ケーブル種別 | `3G-SDI`, `12G-SDI`, `HDMI2.0`, `DisplayPort`, `Ethernet_CAT6A`, `XLR`, `WiFi` | Yes |
| data_content | 伝送データの内容 | `カメラA,B,D,E 合成後映像` | No |
| data_format | データフォーマット | `田の字`, `ArtNet`, `LTC` | No |
| cable_length | ケーブル長 | `50～100m` | No |
| direction | 方向 | `unidirectional` (default), `bidirectional` | No |
| notes | 備考 | `会場機材で対応していれば` | No |

- ケーブル種別はプリセットリストから選択（カスタム追加可）
- 複数の接続を一括選択してプロパティを一括編集できる

#### 4.2.4 表示

- 接続線はケーブル種別に応じて色分けする（凡例表示付き）
- 機材はカテゴリに応じたアイコンまたは色で表示する
- ゾーンの背景色を会場定義YAML から反映する
- ズーム・パン操作に対応する

#### 4.2.5 エクスポート

- キャンバスの状態を「案件定義YAML」としてエクスポートする
- エクスポートされる YAML には以下を含む:
  - 機材リスト（ID、名前、ゾーン、キャンバス上の座標）
  - 接続リスト（始点、終点、接続プロパティ全項目）
- インポート機能: 既存の案件定義YAML を読み込んで編集を再開できる

#### 4.2.6 NOT スコープ（配線GUI がやらないこと）

- draw.io XML の直接生成や編集
- 最終的な見た目（フォント、線の太さ等）の調整
- 会場定義YAML の編集（別途テキストエディタまたは Claude Code で行う）

### 4.3 コンポーネントC: 会場定義YAML

#### 4.3.1 作成フロー

1. 会場の図面画像（平面図PDF、写真等）を Claude Code に渡す
2. Claude Code がマルチモーダル入力から空間構造を読み取り、YAML の雛形を生成する
3. 人間がケーブル敷設距離、アクセス経路等を補足・修正する
4. 同一会場の別案件では、この YAML を再利用する

#### 4.3.2 YAML スキーマ

```yaml
venue:
  name: "TOKYO NODE HALL"
  address: "東京都港区虎ノ門2丁目6-2"

floors:
  - id: "46f"
    label: "46F"
    areas:
      - id: "lobby_46f"
        label: "46F ロビー（HALL客席側）"
        color: "#b0e3e6"
        stroke_color: "#0e8088"
        contains:
          - id: "mocap_area"
            label: "MoCapエリア"
        adjacent_to: ["hall"]
        cable_runs:
          - to: "hall"
            distance: "20m"
            route: "同フロア直接"
          - to: "lobby_47f"
            distance: "50-100m"
            route: "階段/EV経由"
            notes: "CAT6A推奨"

  - id: "47f"
    label: "47F"
    areas:
      - id: "lobby_47f"
        label: "47F ロビー（HALL客席側）"
        color: "#a0522d"
        stroke_color: "#6D1F00"
        adjacent_to: ["hall"]
        cable_runs:
          - to: "hall"
            distance: "10-20m"
            route: "同フロア直接"

  - id: "46f_47f"
    label: "46F～47F"
    areas:
      - id: "hall"
        label: "TOKYO NODE HALL"
        color: "#fad9d5"
        stroke_color: "#ae4132"
        sub_areas:
          - id: "stage"
            label: "ステージ"
          - id: "audience"
            label: "客席"
          - id: "control_booth"
            label: "コントロールブース"
```

#### 4.3.3 空間情報で表現する内容

| 情報 | 表現方法 | 目的 |
|------|---------|------|
| フロア構成 | `floors` の階層 | 垂直方向の関係 |
| エリアの隣接関係 | `adjacent_to` | 水平方向の関係 |
| エリアの包含関係 | `contains`, `sub_areas` | 入れ子構造 |
| ケーブル敷設距離 | `cable_runs.distance` | ケーブル長の見積もり |
| 敷設経路 | `cable_runs.route` | 経路の制約 |
| ゾーンの色 | `color`, `stroke_color` | draw.io での表示色 |

#### 4.3.4 空間情報のLLM活用方針

- Claude Code はこの YAML を読むことで、会場の地理的関係を理解できる
- draw.io XML 生成時に、ゾーンの相対位置（上下左右）やサイズ比率を推論して配置する
- ケーブル敷設距離の情報から、接続線のラベルにケーブル長を自動補完できる
- 初回は会場図面画像からの生成を前提とするが、テキスト記述のみでも作成可能とする

### 4.4 コンポーネントD: 構成図ジェネレータ（Claude Code）

#### 4.4.1 入力

- 案件定義YAML（配線GUI からエクスポート）
- 会場定義YAML

#### 4.4.2 出力

- draw.io XML ファイル（mxGraph 形式）

#### 4.4.3 生成ルール

**ゾーン描画**:
- 会場定義YAML のフロア・エリア構造に基づき、背景矩形を配置する
- 色・ラベルは YAML の値をそのまま使用する
- ゾーンのサイズは、内包する機材数に応じて自動調整する

**機材描画**:
- カテゴリごとにスタイルテンプレートを適用する（下記参照）
- 同一ゾーン内の機材はグリッド配置で並べる（初期レイアウト）
- `person` カテゴリは UML Actor シェイプを使用する

**スタイルテンプレート**:

| カテゴリ | draw.io スタイル | サイズ目安 |
|---------|----------------|----------|
| `camera` | `whiteSpace=wrap;html=1;rounded=0;` | 120x60 |
| `pc` | `whiteSpace=wrap;html=1;rounded=0;` | 200x395 (large), 120x60 (standard) |
| `monitor` | `whiteSpace=wrap;html=1;rounded=0;` | 120x60 |
| `switcher` (videohub) | `whiteSpace=wrap;html=1;rounded=0;fillColor=#647687;fontColor=#ffffff;` | 120x930 |
| `network` (l2_switch) | `whiteSpace=wrap;html=1;rounded=0;fillColor={color};fontColor=#ffffff;` | 75x60 |
| `network` (wifi_router) | `whiteSpace=wrap;html=1;rounded=0;fillColor=#76608a;fontColor=#ffffff;` | 80x60 |
| `audio` (mixer) | `whiteSpace=wrap;html=1;rounded=0;` | 160x140 |
| `person` | `shape=umlActor;verticalLabelPosition=bottom;verticalAlign=top;html=1;` | 30x60 |

**接続線描画**:
- `edgeStyle=orthogonalEdgeStyle;rounded=0;` を基本スタイルとする
- エッジラベルには cable_type、data_content、cable_length を改行区切りで表示する
- 表示例: `"Ethernet\nCAT6A 50～100m\n\nFacialモーション\nBodyモーション"`

**複数ページ対応**:
- 案件定義YAML に `pages` を定義可能とする
- 各ページは異なるビュー（映像系統、音響系統、タイムコード経路等）を表現する
- 同一機材が複数ページに出現してもよい

#### 4.4.4 生成後の人間の作業

構成図ジェネレータが出力する XML は「論理的に正しいが、空間配置は仮」の状態。以下は人間が draw.io エディタで調整する:

- 機材の最終的な位置（会場の地理を反映した配置）
- 接続線の経路（交差を避ける調整）
- ラベルの位置の微調整
- ゾーンのサイズの微調整

## 5. 案件定義YAML スキーマ

配線GUI がエクスポートする YAML の完全スキーマ。

```yaml
project:
  name: "2026_03_15_TOKYO-NODE-HALL"
  venue_ref: "./venues/tokyo_node_hall.yaml"  # 会場定義YAMLへの参照
  created: "2026-02-21"

pages:
  - id: "virtual_video"
    label: "バーチャル映像"
  - id: "timecode"
    label: "タイムコード経路"
  - id: "work_assignment"
    label: "Unity作業分担"

equipment:
  - id: "unity_a"
    name: "Unity A"
    category: "pc"
    subcategory: "desktop"
    description: "カメラA,B,D,E\n担当"
    zone: "lobby_47f"
    style_hint: "large"
    pages: ["virtual_video"]  # 表示するページ

  - id: "cam_a"
    name: "カメラA"
    category: "camera"
    subcategory: "tripod"
    description: "三脚+VR雲台"
    zone: "hall"
    pages: ["virtual_video"]

  - id: "l2_switch_motion_1"
    name: "L2 Switch"
    category: "network"
    subcategory: "l2_switch"
    description: "Motion"
    zone: "lobby_47f"
    color: "#60a917"
    pages: ["virtual_video"]

  - id: "operator_cam_a"
    name: "カメラA\n（三脚）\nオペレータ"
    category: "person"
    subcategory: "operator"
    zone: "hall"
    pages: ["virtual_video"]

connections:
  - id: "conn_001"
    from: "cam_a"
    to: "videohub_hall"
    cable_type: "3G-SDI"
    pages: ["virtual_video"]

  - id: "conn_002"
    from: "cam_a"
    to: "l2_switch_ar_1"
    cable_type: "Ethernet"
    data_content: "カメラ情報"
    pages: ["virtual_video"]

  - id: "conn_003"
    from: "l2_switch_motion_mocap"
    to: "unity_a"
    cable_type: "Ethernet"
    data_content: "Facialモーション\nBodyモーション"
    cable_length: "50～100m"
    notes: "CAT6A"
    pages: ["virtual_video"]

  - id: "conn_004"
    from: "unity_a"
    to: "hdmi_to_sdi_a"
    cable_type: "HDMI2.0"
    data_content: "カメラA,B,D,E\n合成後映像"
    data_format: "田の字"
    pages: ["virtual_video"]
```

## 6. ケーブル種別プリセット

参考図から抽出した、配線GUI のケーブル種別プリセット一覧。

| ID | 表示名 | 表示色（案） | 備考 |
|----|-------|-------------|------|
| `3G-SDI` | 3G-SDI | 緑 | SD/HD映像 |
| `12G-SDI` | 12G-SDI | 青 | 4K映像 |
| `HDMI2.0` | HDMI 2.0 | 赤 | 映像 |
| `DisplayPort` | DisplayPort (DP) | オレンジ | 映像 |
| `Ethernet` | Ethernet | 灰 | 汎用ネットワーク |
| `Ethernet_CAT6A` | Ethernet CAT6A | 灰（太） | 長距離ネットワーク |
| `XLR` | XLR | 紫 | 音声 |
| `WiFi` | Wi-Fi | 点線 | 無線 |
| `DMX` | DMX | 黄 | 照明制御 |
| `custom` | カスタム | ユーザー指定 | プリセット以外 |

## 7. 技術スタック（推奨）

| 要素 | 推奨技術 | 理由 |
|------|---------|------|
| 配線GUI フレームワーク | React | エコシステムが広く、Claude Code との相性が良い |
| キャンバスライブラリ | React Flow | ノード＆エッジのGUIに特化。D&D、ズーム、パン標準対応 |
| Google Sheets 連携 | Google Sheets API v4 | 直接読み込み。将来の CSV 対応はファイル入力で代替可能 |
| YAML 処理 | js-yaml | YAML パース/シリアライズ |
| ビルド/開発 | Vite | 高速。設定が少ない |
| デプロイ | ローカル実行 (npm run dev) | 個人ツールのため。将来的にVercel等も可 |

## 8. 制約・前提条件

### 前提

- Claude Code + draw.io MCP が設定済みの環境で使用する
- Google Drive デスクトップアプリがインストール済み
- Node.js / npm が利用可能
- npm レジストリは `--registry https://registry.npmjs.org` の指定が必要

### 制約

- 配線GUI は機材の論理的な接続のみを管理する。draw.io 上の最終的な見た目は管理しない
- 会場定義YAML は手動（+ Claude Code 支援）で作成する。配線GUI からの編集はスコープ外
- draw.io XML の生成は Claude Code のプロンプトとして実行する。専用の変換スクリプトは初期スコープに含まない（必要になれば追加）
- 個人利用のため、認証・権限管理は不要

### 将来拡張の余地

- CSV / Excel からの機材インポート
- 配線GUI 上での会場定義YAML 編集
- 案件テンプレート機能（過去案件の機材構成を雛形として再利用）
- draw.io XML 生成の自動化スクリプト（Claude Code を介さず直接変換）
- 機材台帳と構成図の差分検出（使われていない機材、台帳にない機材の警告）

## 9. 検証方法

### 9.1 配線GUI

- Google Sheets から機材を読み込み、キャンバスに配置できること
- 機材間に接続線を引き、接続プロパティを設定できること
- 案件定義YAML をエクスポートし、再インポートして状態が復元されること

### 9.2 会場定義YAML

- 参考案件（TOKYO NODE HALL）の会場構造を YAML で表現できること
- 配線GUI でゾーン背景として表示されること

### 9.3 構成図ジェネレータ

- 案件定義YAML + 会場定義YAML から draw.io XML が生成されること
- 生成された XML を draw.io MCP (`open_drawio_xml`) でプレビューできること
- 参考図と同等の情報量（機材・接続・ゾーン）が含まれていること
