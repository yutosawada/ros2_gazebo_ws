# ros2_gazebo_ws

## 生成AI入力用プロンプト例

あなたはROS2とGazeboの専門知識を持つエンジニアです。以下の要件に基づいて、差動二輪自律移動ロボットのシミュレーション環境用ソフトウェアのソースコードと設定ファイル（ROS2パッケージ、ノード、launchファイル、URDFで定義したロボットモデルなど）を生成してください。

## 要件

### ミドルウェアと環境
- 使用するミドルウェア: ROS2 Humble
- シミュレーション環境: Gazeboシミュレータ

### ロボット仕様
- 差動二輪駆動の自律移動ロボット
- センサ: エンコーダーと2D LiDARを搭載
- 自己位置推定や目標地点への移動が可能なシンプルな自律機能を持つ
- シミュレーション上で、エンコーダーと2D LiDARのセンサデータを取得・処理する

### システム構成
- **コントローラノード**: 差動ドライブの動作制御（速度指令、オドメトリ計算など）
- **センサ処理ノード**: Gazeboからのエンコーダーおよび2D LiDARのデータ受信と処理
- **統合ノード**: 自律移動アルゴリズム（目標地点設定、障害物回避などのシンプルな経路計画）の実装

各ノード間はROS2のトピックやサービスを用いて通信する

### 実装言語
- C++で実装する

### ロボットモデル定義
- ロボットの定義ファイルはURDFを使用する

### その他の要求事項
- コードには十分なコメントを記述し、各部の動作や役割を明確に説明すること
- ROS2パッケージのディレクトリ構造、ビルド方法（colconを利用）、依存関係の管理についても記述する
- launchファイルを作成し、Gazeboシミュレータと各ROS2ノードが自動で起動するようにする
- URDFファイルを用いてロボットモデルの定義とそのサンプルを含める

## 出力例の構成
- ROS2パッケージ全体のディレクトリ構造の説明
- 各ROS2ノード（C++実装）のソースコード（十分なコメント付き）
- launchファイルの例（ROS2 launchファイルを使用してGazeboと各ノードを起動）
- ロボットモデルの定義ファイル（URDF）
- 簡単な動作説明と起動手順

