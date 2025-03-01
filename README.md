# ros2_gazebo_ws

##目次

[スケジュール](docs/schedule.md)


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



了解しました。ROS2 Humble と Gazebo Harmonic を使用し、Hokuyo LiDAR を搭載した差動二輪自律移動ロボットのシミュレーション環境の構築に必要なソースコードや設定ファイルを調査します。オドメトリはデッドレコニング、経路計画には A*、そして Nav2 を活用する形で進めます。

調査完了後に、ROS2パッケージのディレクトリ構成、C++ のソースコード、launch ファイル、URDF、起動手順などをまとめてお知らせします。少々お待ちください。

**概要:** ROS2 Humble と Gazebo (Ignition) Harmonic を用いて、Hokuyo LiDAR 搭載の差動二輪ロボットのシミュレーション環境を構築します。ロボットは左右2輪の差動駆動で、エンコーダからのオドメトリによるデッドレコニングで自己位置推定し、A*アルゴリズムで経路計画、Nav2スタックで経路追従と障害物回避を行います。開発は C++ で行い、URDF でロボットモデルを定義します。以下にシステム構成と実装ポイントをまとめます。

## ロボットモデルとシミュレーション環境 (URDF + Gazebo)  
ロボットは URDF(Xacro) 形式で定義し、ベースとなるボディリンク（`base_link`）、左右の車輪リンク、Hokuyo LiDARセンサ用のリンクを含みます。URDF上で差動二輪のジョイント（左・右車輪の回転ジョイント）とLiDARセンサの取り付け位置（固定ジョイント）を定義します。また Gazebo 上で物理シミュレーションするため、URDF内に Gazebo プラグインを記述します。差動駆動には Gazeboのdiff-driveプラグイン（ROS2対応版）を利用し、`<gazebo>`タグ内でプラグインを指定します。このプラグインにより、`/cmd_vel`トピックからの速度指令を車輪ジョイントの駆動に変換し、オドメトリも自動計算・発行されます。例えばURDF内では次のように設定します（概略）: 左右ジョイント名、車輪間隔 (`wheel_separation`)、車輪径 (`wheel_diameter`)、指令トピック名 (`<command_topic>cmd_vel</command_topic>`)、オドメトリの出力有効化 (`<publish_odom>true</publish_odom>`)やフレームID (`<odometry_frame>odom</odometry_frame>`, `<robot_base_frame>base_footprint</robot_base_frame>`) 等です。Hokuyo LiDARセンサはURDF上でリンクを追加し、そのリンクに対してGazeboのレーザセンサプラグインを設定します。Gazebo Classicの場合 `<plugin name="gazebo_ros_laser"` 等を用いてLaserScanを発行できますし、Gazebo Ignition (Harmonic)の場合はSDFの`<sensor type="gpu_lidar">`等でセンサを定義し、ROS2側へはブリッジでLaserScanトピックを繋ぎます。例えばIgnitionではDiffDriveやLiDARなどのプラグインの出力をROS2トピック（`/odom`, `/scan`など）へブリッジする設定をYAMLで行います。URDFモデルは `description` パッケージとしてまとめ、Gazebo 起動時にSpawnします。Gazebo Harmonicでは`ros_gz_sim`パッケージの `create` ノードでURDF/SDFモデルをスポーンし、`ros_gz_bridge`の parameter_bridge 機能で必要なトピック（`/cmd_vel`, `/odom`, `/scan` 等）をROS2側と接続します。  

## 差動二輪の制御とオドメトリ (コントローラノード)  
コントローラノードはロボットの差動駆動を制御し、オドメトリを算出するC++ノードです。ROS2 Navigation2から送られる速度指令（`geometry_msgs/Twist`型の/cmd_vel）を購読し、左右車輪の目標角速度に変換します。シミュレータとのインタフェースは、上記のdiff-driveプラグインにより実現できます。すなわちコントローラノードはNav2からの/cmd_velをそのままGazebo（Ignitionの場合はブリッジ経由）に流し、車輪ジョイントへ適用します。オドメトリ算出は差動二輪の逆運動学に基づき、左右輪の回転量（エンコーダ値）からロボットの並進速度と角速度を推定します。例えば左右輪速度を $V_l, V_r$ とすると、ロボットの角速度 $\omega$ は $\omega = (V_r - V_l)/L$（Lは車輪間距離）で与えられます。同様に並進速度は $(V_r+V_l)/2$ で求まります。コントローラノードでは一定周期でエンコーダ（車輪ジョイント角）の増分を読み取り（Gazeboでは`/joint_states`トピックから取得可能）、上述の式でロボットの速度と進行方向を計算し、これを時間積分して現在の姿勢（x, y, θ）を更新します。算出したオドメトリは`nav_msgs/Odometry`としてトピック出力し、TF（`odom->base_link`変換）もブロードキャストします。Gazeboのdiff-driveプラグインを使用する場合、自動でオドメトリトピック`/odom`やTFを発行する設定も可能ですが、自前のノードで計算することで実機ロボット移行時のコード共通化や理解を深めることができます。  

 差動二輪ロボットの運動学。左右輪速度 V<sub>l</sub>, V<sub>r</sub> の差によりロボットはICC（瞬時旋回中心）を中心に角速度 ω で旋回する。この原理に基づきエンコーダからオドメトリを算出する。

## センサ処理ノード (エンコーダ・LiDARデータ)  
センサ処理ノードはGazebo上のセンサデータを受け取り、必要なROS2トピックとして配信・前処理するC++ノードです。まずエンコーダ（車輪角度）データは前述の通り`/joint_states`トピックから取得できますが、このノードでは必要に応じてエンコーダ生データのフィルタ処理やデッドレコニングのための前計算を行います（例えば左右車輪の回転数から移動距離Δsと方位変化Δθを計算する処理を実装可能です）。もっとも、シンプルな構成ではコントローラノード内で直接/joint_statesを購読しオドメトリ計算してもよいでしょう。Hokuyo LiDARのレーザスキャンについては、Gazeboプラグインが直接`/scan`トピックとして `sensor_msgs/LaserScan` 型データを発行できます。Gazebo Classicなら`gazebo_ros`パッケージのレーザプラグイン（例: `libgazebo_ros_ray_sensor.so`）をURDFに仕込むことで自動で`/scan`が得られます。Gazebo HarmonicではIgnitionのSensorプラグインでGazebo内トピック`/scan`を発行し、`ros_gz_bridge`でROS2側`/scan`に橋渡しする設定を行います。センサ処理ノード自体は通常、受け取ったLaserScanをNav2のコストマップが利用できるようそのまま転送します。必要に応じ、LiDARデータの前処理（ノイズ低減やダウンサンプリング）を実装できますが、本シミュレーションではGazeboの理想センサを想定するためそのまま使用します。なお、LiDARの座標系（`laser_frame`など）はURDFでロボットにマウントしたリンクとして定義し、`robot_state_publisher`ノードでTFツリーに含めます。以上により、エンコーダ由来のオドメトリ`/odom`とLiDARの`/scan`がROS2上で利用可能となり、Nav2による自己位置推定・環境認識に供されます。

## 経路計画ノード (A*アルゴリズム)  
経路計画ノードはグローバル経路を生成するためのC++ノードで、A*アルゴリズムを用いています。あらかじめ与えられた地図（Occupancy Gridマップ）上で、現在位置から目標位置までの最短経路をグリッド探索します。Nav2には標準でグローバルプランナ（`nav2_planner`サーバ）が含まれており、NavFnやSmacPlannerといったプラグインが利用可能です。NavFnプランナは wavefront法に基づくNavFnアルゴリズムで、設定によりA* またはダイクストラ法を用いてコストマップ上の経路を計算します。本構成では、自作のA*ノードを独立に実装するか、Nav2のグローバルプランナをA*モードで用いるかの選択があります。自作する場合、経路計画ノードは地図トピック`/map`を購読してコストマップや経路グラフを構築し、サービスやトピック経由で経路要求を受け取ってA*探索を行い、一連の経由点列（`nav_msgs/Path`など）を出力します。Nav2と統合する場合、グローバルプランナプラグインとして組み込むことも可能ですが、ここでは簡単のためNav2標準プランナを用いる想定とします。Nav2のNavFnプランナを使う場合、パラメータで`use_astar: True`と設定すれば、ヒューリスティック付きのA*探索で経路生成が行われます。A*アルゴリズムにより静的地図上の最短経路が得られれば、それを後述のNav2制御器に渡して追従制御させます。

## Nav2による経路追従・障害物回避 (統合ノード)  
Nav2 (Navigation2) はROS2における自律移動ナビゲーションのフレームワークであり、本システムの経路追従や局所的な障害物回避を担います。Nav2は複数のノード（もしくはライフサイクルノード）で構成されていますが、ここでは「統合ノード」としてまとめ、Nav2の基本機能を起動します。この中にはAMCLによる自己位置推定（マップ座標系に対するロボット位置）、グローバルコストマップ・ローカルコストマップ、プランナ（グローバルプラン）とコントローラ（ローカルプラン）などが含まれます。グローバルプランナは前述のA*経路を計算し、ローカルコントローラが実時間で速度指令を生成してロボットを経路に沿わせます。デフォルトのローカルコントローラはDWB (Dynamic Window Approach-based) コントローラで、動的ウィンドウアプローチにより障害物を回避しつつ軌道を生成します。DWBコントローラはROS1のDWAローカルプランナをROS2に移植・拡張したもので、Nav2の既定の局所制御器です。「統合ノード」ではNav2の各ノード（`controller_server`, `planner_server`, `bt_navigator`, `amcl`, `map_server` 等）を起動し、Behavior Treeに従って目標位置までのナビゲーションを実行します。グローバルコストマップには静的地図+動的障害物レイヤ、ローカルコストマップにはロボット周辺の障害物レイヤが含まれ、LaserScanから障害物をマーキングします。Nav2のデフォルトでは global_costmap に「static_layer」「obstacle_layer」「inflation_layer」プラグインが使われ、local_costmapには「obstacle_layer（あるいはvoxel_layer）」「inflation_layer」等が使われます。静的レイヤーが地図情報を読み込み、オブスタクルレイヤーがLiDARの検知する障害物をコストマップに投影し、インフレーションレイヤーがロボット半径に応じて障害物を膨張させます。Nav2の設定ファイルで各種パラメータ（コストマップ解像度やロボットの半径、インフレーション半径、プランナ・コントローラのアルゴリズム選択や速度制限等）を調整します。例えばDWBコントローラでは最大速度や加速度、各種軌道評価クライテリアを設定できます。障害物回避はローカルコストマップ上でDWBの軌道評価により実現され、グローバルプランから逸脱しない範囲でロボットが障害物を迂回します。以上により、Nav2が全体経路に沿った自律走行を実現します。

## ROS2パッケージ構成  
開発するROS2パッケージは、複数のサブコンポーネントを含む統合パッケージとして構成できます（名前例: `my_robot_sim` とします）。ディレクトリ構成は以下のようになります。

- **package.xml**, **CMakeLists.txt** – パッケージ定義とビルド設定。必要な依存としてROS2の標準パッケージ（rclcpp, sensor_msgs, nav_msgs, tf2_ros など）やnav2、gazebo_ros、ros_gz_sim 等を記載します。CMakeListsでは各ノードのソースをビルドし、実行可能ファイルを生成します。  
- **urdf/** – ロボットモデルのURDF/Xacroファイル（例: `my_robot.urdf.xacro`）。ベースリンク・車輪・LiDARのリンクとジョイント、およびGazeboプラグインの記述を含みます。  
- **launch/** – 起動用のLaunchファイル。Python launchでGazeboシミュレータ、ロボットモデルのスポーン、各ノード（コントローラ、センサ処理、Nav2）を起動します。例えば`sim_launch.py`を用意し、Ignition Gazebo をHeadlessモードで起動後、`ros_gz_sim create`によりURDFをスポーンし、`ros_gz_bridge`のブリッジノード、さらにNav2のbringupや自作ノードを順次起動します。必要に応じてRVizも起動し、ロボットの地図や経路を可視化できるようにします。  
- **src/** – C++ソースコード。要求された各ノード（controller, sensor, planner, integration）に対応するソースを実装します。例: `diff_drive_controller_node.cpp`, `sensor_processing_node.cpp`, `global_planner_node.cpp` など。それぞれ`rclcpp::Node`を継承したクラスで構成し、main関数で`rclcpp::spin`する実行ファイルにします。コード内には十分なコメントを付し、差動輪の速度計算式やA*の処理手順などを解説します。特にデッドレコニング部分は計算式（上記ωや位置更新式）をコメントで示し、経路計画部分はオープンリスト/クローズドリストの管理などアルゴリズムの要点をコメントします。  
- **include/** – ヘッダファイル（必要なら）。ノードクラスの宣言や共通定数の定義。  
- **config/** – Nav2や各種パラメータ用の設定ファイル。Nav2用に`nav2_params.yaml`を用意し、`amcl`, `map_server`, `planner_server`, `controller_server`, `bt_navigator` などのパラメータを記述します。グローバルコストマップ・ローカルコストマップの解像度やサイズ、使用レイヤー、各レイヤーのパラメータ（例: static_layerはマップトピック名`map`、obstacle_layerはセンサトピック`scan`やセンサ範囲、inflation_layerは膨張半径など）を設定します。グローバルプランナとしてNavFn(A*)を使うなら`planner_server`のパラメータで`planner_plugin: nav2_navfn_planner/NavfnPlanner`かつ`use_astar: True`を設定します。ローカルコントローラはデフォルトのDWBを使う場合`controller_server`のパラメータで`controller_plugin: dwb_core::DWBLocalPlanner`等を指定し、DWBの各Criticのパラメータ（障害物距離評価や目標への角度合わせ重みなど）を調整できます。さらに`yaml`内で`use_sim_time: True`を有効にし、シミュレータ時間を利用するよう全ノードに適用します。

## Launchファイルと起動手順  
システム全体を起動するLaunchファイル（例: `sim_launch.py`）では、以下の手順でノードを立ち上げます。

1. **Gazebo (Ignition) の起動:** `ros2 launch ros_gz_sim gazebo.launch.py headless:=false` などでGazeboを起動し、空のワールド（地面平面と原点にロボットを置く）を読み込みます。もしくはLaunchファイル内でNodeとして`ros_gz_sim`のgazeboサーバ・クライアントをincludeします。Gazeboワールドは必要に応じて`worlds/`フォルダにSDFを用意できますが、シンプルな床面でよい場合はデフォルトを使用します。  
2. **ロボットモデルのスポーン:** 次に、URDFファイルをGazebo上にスポーンします。Ignition Gazeboでは `ros2 run ros_gz_sim create -name my_robot -file my_robot.urdf` に相当するNodeをLaunchで呼び出します。これによりGazebo内にロボットが生成され、`/model/my_robot` 等のエンティティとして登場します。  
3. **ブリッジノード起動:** Gazebo Harmonic環境ではROS2トピックとIgnitionトピックを繋ぐブリッジが必要です。Launchファイルで`ros_gz_bridge`のparameter_bridgeノードを起動し、あらかじめ用意したブリッジ設定YAML（例: `bridge.yaml`）を渡します。このYAMLには`cmd_vel` (ROS2→Gazebo), `odom`, `tf`, `scan` (Gazebo→ROS2) 等のブリッジ項目が列挙されています。これによりGazebo内のロボットの動作とROS2側のNav2ノード群が通信できるようになります。  
4. **ロボット状態パブリッシャー:** URDFで定義したロボットのTFツリー（base_linkや各リンク関係）を配信するため、`robot_state_publisher`ノードを起動します。URDFパスを与え、`use_sim_time`をtrueに設定します。これでTFツリーに`base_link`や`laser_link`などが公開されます。  
5. **自作ノードの起動:** コントローラノード、センサ処理ノード、（必要なら）経路計画ノードをそれぞれ起動します。コントローラノードは`/cmd_vel`を購読しGazeboブリッジ経由で車輪を駆動、`/odom`を発行します。センサノードは`/scan`やエンコーダ情報を購読し必要処理後にNav2側へ提供します。経路計画ノードはNav2を使う場合省略も可能ですが、独自に実装した場合はサービス待機等のため起動します。  
6. **Nav2の起動:** `nav2_bringup`パッケージのLaunchをインクルードするか、各Nav2ノードを構成して起動します。簡単には `ros2 launch nav2_bringup navigation_launch.py use_sim_time:=True params_file:=path/to/nav2_params.yaml` のようにlaunchファイル内から呼び出せます。Nav2起動時にAMCLがマップとLaserScanで自己位置推定を開始し、plannerとcontrollerが動作待機状態になります。RVizも起動し、マップ表示やロボットの現在位置・経路を可視化できるようにします。  

**起動後の動作:** シミュレーションを起動すると、Gazebo上でロボットがスポーンされ静止しています。まずRViz上で初期位置のセット（AMCLへの初期姿勢送信）を行います（デッドレコニングのみで運用する場合は地図座標系とodom座標系を一致させたまま進めます）。次にナビゲーション目標を2D Nav Goalツール等で設定すると、Nav2のプランナがグローバル経路を計算し（デフォルトはA*による最短経路）、緑色の経路がRVizに表示されます。続いてDWBローカルプランナが速度指令を生成し、コントローラノード経由でロボットが動き出します。Hokuyo LiDARによって障害物が検出されれば局所コストマップに反映され、必要に応じてローカル経路を迂回させます。ゴールまで到達するとNav2のBTが完了状態になり、ロボットは停止します。シミュレーション環境ではログやRVizで各データを確認できます。例えば`/odom`トピックを監視すると、ロボットが移動するにつれオドメトリが更新されていることがわかります。また`/scan`トピックは周囲の障害物距離を配列データで提供しており、Nav2のコストマップがこれを利用していることが確認できます。終了する際はCtrl+CでLaunchを停止すればGazeboも終了します。

**参考文献:** ROS2 HumbleおよびGazebo/Ignitionの公式ドキュメントやNav2のチュートリアルが、本シミュレーション環境の構築に役立ちます。などを参照しつつ、各コンポーネントを組み合わせて実装してください。以上が、Hokuyo LiDAR搭載差動二輪ロボットのROS2シミュレーション環境構築におけるソフトウェア実装の概要です。
