# Requirements Document

## Introduction

ECDC (Equipment Configuration Diagram Creator) は、バーチャルプロダクション案件における機材構成図の作成を支援する個人ツールである。機材台帳（Google Sheets）と配線GUI（React Flow ベースの Web アプリケーション）を組み合わせ、案件定義 YAML をエクスポートし、Claude Code + draw.io MCP で構成図を自動生成するワークフローを提供する。

本ドキュメントでは、配線GUI を中心としたシステムの機能要件を EARS 形式で定義する。

## Requirements

### Requirement 1: 機材台帳の読み込み

**Objective:** 利用者として、Google Sheets の機材台帳から機材データを配線GUI に読み込みたい。それにより、スプレッドシートを Single Source of Truth として一元管理し、二重メンテを排除できる。

#### Acceptance Criteria

1. When 利用者が機材読み込み操作を実行した時, the 配線GUI shall Google Sheets API v4 経由で指定されたスプレッドシートから機材マスタデータ（id, name, category, subcategory, description, manufacturer, model, quantity, zone, style_hint, color, notes）を取得して内部データに変換する
2. When 機材データの読み込みが完了した時, the 配線GUI shall 読み込んだ機材をカテゴリ別（camera, pc, monitor, switcher, network, audio, recorder, capture, lighting, person）にグループ化してパレットに表示する
3. When 利用者がリロードボタンをクリックした時, the 配線GUI shall スプレッドシートの最新状態を再取得してパレットの機材リストを更新する
4. If Google Sheets API からのデータ取得に失敗した場合, the 配線GUI shall エラーメッセージを表示し、前回取得したデータがあればそれを保持する
5. The 配線GUI shall 機材データの必須フィールド（id, name, category, quantity）が欠落している場合にバリデーションエラーを表示する

---

### Requirement 2: キャンバスへの機材配置

**Objective:** 利用者として、読み込んだ機材をキャンバス上にドラッグ&ドロップで配置したい。それにより、機材の空間的な配置関係を視覚的に定義できる。

#### Acceptance Criteria

1. When 利用者が機材パレットから機材をキャンバスにドラッグ&ドロップした時, the 配線GUI shall 機材をキャンバス上のドロップ位置に React Flow ノードとして配置する
2. When 配置済みの機材ノードをドラッグした時, the 配線GUI shall 機材ノードをキャンバス上の新しい位置に移動する
3. When 機材がゾーン領域の内部に配置された時, the 配線GUI shall その機材の所属ゾーンを該当ゾーンに自動設定する
4. The 配線GUI shall 機材ノードをカテゴリに応じたアイコンまたは色で表示する（person カテゴリは UML Actor 形状を使用する）
5. The 配線GUI shall キャンバス上でズーム・パン操作に対応する

---

### Requirement 3: 会場ゾーンの表示

**Objective:** 利用者として、会場定義 YAML から読み込んだゾーン情報をキャンバスの背景として表示したい。それにより、機材の配置先ゾーンを視覚的に把握しながら作業できる。

#### Acceptance Criteria

1. When 会場定義 YAML がロードされた時, the 配線GUI shall YAML に定義されたフロア・エリア構造（floors, areas, sub_areas, contains）を階層的なゾーン背景としてキャンバスに描画する
2. The 配線GUI shall 各ゾーンの背景色（color）と枠線色（stroke_color）を会場定義 YAML の値に従って表示する
3. The 配線GUI shall 各ゾーンのラベル（label）をゾーン領域上に表示する
4. If 会場定義 YAML のパースに失敗した場合, the 配線GUI shall パースエラーの内容を表示し、ゾーンなしの状態でキャンバスを表示する

---

### Requirement 4: 機材間の接続定義

**Objective:** 利用者として、キャンバス上で機材間の接続線をドラッグ操作で定義し、接続プロパティを編集したい。それにより、機材間の論理的な配線関係を効率的に構築できる。

#### Acceptance Criteria

1. When 利用者が機材ノードの出力ポートから別の機材ノードの入力ポートへドラッグした時, the 配線GUI shall 2つの機材間に接続線（React Flow エッジ）を作成する
2. When 利用者が接続線をクリックした時, the 配線GUI shall 接続プロパティの編集パネルを表示する（cable_type, data_content, data_format, cable_length, direction, notes）
3. The 配線GUI shall ケーブル種別（cable_type）をプリセットリスト（3G-SDI, 12G-SDI, HDMI2.0, DisplayPort, Ethernet, Ethernet_CAT6A, XLR, WiFi, DMX, custom）から選択可能にする
4. When 利用者がケーブル種別としてカスタムを選択した時, the 配線GUI shall 任意のケーブル種別名を入力できるフィールドを表示する
5. When 複数の接続線が選択された時, the 配線GUI shall 選択された接続のプロパティを一括編集できる機能を提供する
6. The 配線GUI shall 接続線をケーブル種別に応じた色で表示し、凡例を表示する
7. The 配線GUI shall 接続の方向（direction）をデフォルト値 unidirectional で作成し、bidirectional への変更を可能にする

---

### Requirement 5: 案件定義 YAML のエクスポート

**Objective:** 利用者として、キャンバスの状態を案件定義 YAML としてエクスポートしたい。それにより、Claude Code による draw.io 構成図生成の入力データを作成できる。

#### Acceptance Criteria

1. When 利用者がエクスポート操作を実行した時, the 配線GUI shall キャンバスの状態を案件定義 YAML スキーマに準拠した YAML ファイルとして出力する
2. The 配線GUI shall エクスポートされる YAML に以下を含める: プロジェクト情報（project）、ページ定義（pages）、機材リスト（equipment: id, name, category, subcategory, description, zone, style_hint, color, pages, キャンバス座標）、接続リスト（connections: id, from, to, cable_type, data_content, data_format, cable_length, direction, notes, pages）
3. When 利用者がエクスポートされた YAML をインポートした時, the 配線GUI shall YAML の内容を読み込みキャンバスの状態（機材配置、接続、プロパティ）を復元する
4. If インポートされた YAML が案件定義スキーマに準拠していない場合, the 配線GUI shall バリデーションエラーの詳細を表示する

---

### Requirement 6: ページ管理

**Objective:** 利用者として、案件の構成図を複数ページ（映像系統、音響系統、タイムコード経路等）に分割して管理したい。それにより、複雑な構成を系統別に整理できる。

#### Acceptance Criteria

1. The 配線GUI shall 案件内に複数のページ（ビュー）を定義する機能を提供する
2. When 利用者が新しいページを作成した時, the 配線GUI shall ページ ID とラベルを設定できるフォームを表示する
3. The 配線GUI shall 各機材および接続に対して、表示対象ページを指定する機能を提供する
4. When 利用者がページを切り替えた時, the 配線GUI shall 選択されたページに割り当てられた機材と接続のみをキャンバスに表示する
5. The 配線GUI shall 同一機材が複数ページに出現することを許容する

---

### Requirement 7: データ型とドメインモデル

**Objective:** 利用者として、機材カテゴリ、ケーブル種別、接続プロパティなどのドメイン固有の値が型安全に管理されていることを期待する。それにより、データの整合性が保たれ、不正な値の混入を防止できる。

#### Acceptance Criteria

1. The 配線GUI shall 機材カテゴリ（camera, pc, monitor, switcher, network, audio, recorder, capture, lighting, person）を TypeScript の型として定義する
2. The 配線GUI shall ケーブル種別プリセット（3G-SDI, 12G-SDI, HDMI2.0, DisplayPort, Ethernet, Ethernet_CAT6A, XLR, WiFi, DMX, custom）を TypeScript の型として定義する
3. The 配線GUI shall 接続プロパティ（cable_type, data_content, data_format, cable_length, direction, notes）を TypeScript の型として定義する
4. The 配線GUI shall 会場定義 YAML のスキーマ（venue, floors, areas, sub_areas, contains, adjacent_to, cable_runs）に対応する TypeScript の型を定義する
5. The 配線GUI shall 案件定義 YAML のスキーマ（project, pages, equipment, connections）に対応する TypeScript の型を定義する

---

### Requirement 8: YAML データ変換

**Objective:** 利用者として、YAML ファイルの読み書きが正確に行われることを期待する。それにより、配線GUI と外部ツール（Claude Code、テキストエディタ）間でデータの受け渡しが確実にできる。

#### Acceptance Criteria

1. The 配線GUI shall js-yaml ライブラリを使用して会場定義 YAML のパースおよびシリアライズを行う
2. The 配線GUI shall js-yaml ライブラリを使用して案件定義 YAML のパースおよびシリアライズを行う
3. When YAML データを TypeScript オブジェクトに変換する時, the 配線GUI shall ドメイン型定義に基づいたバリデーションを実施する
4. If YAML ファイルに構文エラーが含まれている場合, the 配線GUI shall エラー行番号と内容を含むエラーメッセージを表示する

---

### Requirement 9: 会場定義 YAML の空間情報活用

**Objective:** 利用者として、会場定義 YAML に記述された空間情報（隣接関係、ケーブル敷設距離）がキャンバス上で参照できることを期待する。それにより、物理的な制約を考慮した配線設計が可能になる。

#### Acceptance Criteria

1. The 配線GUI shall 会場定義 YAML のエリア間隣接関係（adjacent_to）をキャンバスのゾーンレイアウトに反映する
2. While 接続プロパティの編集中, when 接続元と接続先が異なるゾーンに配置されている時, the 配線GUI shall 会場定義 YAML の cable_runs から該当区間のケーブル敷設距離と経路情報を参考情報として表示する
3. The 配線GUI shall 会場定義 YAML のフロア階層構造（floors）を反映したゾーンの垂直方向の配置を行う
4. The 配線GUI shall 会場定義 YAML の包含関係（contains, sub_areas）を反映したゾーンの入れ子表示を行う
