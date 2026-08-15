# LeRobot でできること

この文書は、LeRobot で「何ができるか」を機能単位で列挙したものである。粒度は機能名レベルに揃え、詳細な手順やコマンド引数は各リンク先に委ねる。

対象は本リポジトリのサブモジュール `lerobot/`（`v0.6.1-24-g6adf5151`）であり、記載内容はそのチェックアウトの実装・ドキュメントに基づく。

## 実機ロボットの操作

- USB ポートの検出、モーター ID・baudrate の設定、キャリブレーションができる（`lerobot-find-port` / `lerobot-setup-motors` / `lerobot-calibrate`）
- リーダーアームによるテレオペレーションができる（`lerobot-teleoperate`）
- キーボード、ゲームパッド、スマートフォンからロボットを操作できる
- 関節の可動限界を実測して取得できる（`lerobot-find-joint-limits`）
- CAN バス接続のモーターを設定できる（`lerobot-setup-can`）
- SO-100/101、Koch、LeKiwi、HopeJR、OMX、OpenARM、Reachy 2、Unitree G1、reBot B601、EarthRover Mini+ などを共通の `Robot` インターフェースで扱える
- Feetech、Dynamixel、Damiao、Robstride のモーターバスを共通 API で扱える
- OpenCV、Intel RealSense、ZMQ 経由のカメラを接続できる（`lerobot-find-cameras`）
- RealSense などの深度カメラから深度ストリームを取得・記録できる
- 双腕構成（bi_so_follower など）を扱える
- 独自ロボット・テレオペ機器・カメラをプラグイン（`lerobot_robot_*` 等）として追加できる
- Web ブラウザ GUI「LeLab」から CLI を使わずに一連の操作ができる（[lelab.mdx](./lerobot/docs/source/lelab.mdx)）

## データセット

- 実機の観測・行動・映像を LeRobotDataset 形式で収録できる（`lerobot-record`）
- 収録したエピソードを実機で再生して検証できる（`lerobot-replay`）
- データセットを Hugging Face Hub へアップロード・ダウンロードして共有できる
- 大規模データセットをローカルへ全展開せずストリーミングで学習に使える
- 複数のデータセットを結合して 1 つとして扱える（`MultiLeRobotDataset`、`aggregate`）
- エピソード削除、分割、結合、特徴量の削除、タスク文字列の書き換えができる（`lerobot-edit-dataset`）
- 動画コーデック・品質を指定して再エンコードできる
- 旧バージョンのデータセットを v3 形式へ変換できる
- VLM に映像を見せてサブタスク・説明文・音声発話などの言語アノテーションを自動付与できる（`lerobot-annotate`）
- ポリシーからのツール呼び出し（`say(...)` など）をデータセット側の定義として扱える（[tools.mdx](./lerobot/docs/source/tools.mdx)）
- データセットの内容・統計を確認し、Web またはローカルで可視化できる（`lerobot-info` / `lerobot-dataset-viz`）
- 画像変換（データ拡張）の効果を可視化して確認できる（`lerobot-imgtransform-viz`）

## 学習（模倣学習）

- 収録データからポリシーを学習できる（`lerobot-train`）
- ACT、Diffusion Policy、VQ-BeT、TD-MPC といった標準的なポリシーを学習できる
- SmolVLA、π0 / π0.5 / π0-FAST、GR00T N1.7、MolmoAct2、EO-1、EVO1、X-VLA、Wall-X、Multitask DiT などの VLA を学習・ファインチューニングできる
- VLA-JEPA、LingBot-VA、FastWAM といった世界モデル系ポリシー（将来の状態を予測してから行動する系統）を学習できる
- 事前学習済みモデルを Hub から取得してファインチューニングできる
- LoRA など PEFT による省メモリのファインチューニングができる（[peft_training.mdx](./lerobot/docs/source/peft_training.mdx)）
- 学習経過を Weights & Biases に記録できる
- チェックポイントから学習を再開できる
- 独自ポリシーを追加して同じ学習パイプラインに載せられる（[bring_your_own_policies.mdx](./lerobot/docs/source/bring_your_own_policies.mdx)）

## 学習インフラ

- マルチ GPU 学習ができる（DDP：モデルを全 GPU に複製）
- FSDP により、単一 GPU の VRAM に収まらない大規模モデルを分散して学習できる
- HSDP により、GPU グループ内でシャード・グループ間で複製するハイブリッド構成が組める
- `torchrun` と `accelerate launch` のどちらからでも分散学習を起動できる（[multi_gpu_training.mdx](./lerobot/docs/source/multi_gpu_training.mdx)）
- Hugging Face Jobs でクラウド GPU 上に学習を投げ、ログを追跡できる
- CPU、CUDA、Apple Silicon（MPS）、Intel GPU（XPU）で学習・推論を実行できる（[torch_accelerators.mdx](./lerobot/docs/source/torch_accelerators.mdx)）
- 分散チェックポイント（DCP）を通常の形式へ変換できる（`lerobot-convert-dcp`）

## 強化学習・人の介入

- HIL-SERL（人の介入を伴う SAC/RLPD によるオンライン強化学習）ができる。actor と learner を別プロセスで動かし gRPC で遷移と重みを交換する
- `gym-hil`（Franka Panda）のシミュレーション上で HIL-SERL を先に検証できる
- 画像から成功／失敗を判定する報酬分類器を学習できる
- DAgger 形式で、ポリシー実行中に人が介入して修正データを収集し、そのデータで追加学習できる（`lerobot-rollout --strategy.type=dagger`）
- 注意：SO-ARM 101 実機での HIL-SERL は、現行チェックアウトでは起動しない（`SOLeader` に `get_teleop_events()` がなく、修正 PR も未マージ）。SO-101 で人の介入を使うなら DAgger を選ぶ。詳細は [HI-SERL.md](./HI-SERL.md) を参照する

## 推論・デプロイ

- 学習済みポリシーを実機で実行できる（`lerobot-rollout`）
- 行動予測と行動実行を分離した非同期推論により、推論待ちの停止をなくせる（`PolicyServer` と `RobotClient`、ネットワーク越しの実行も可能）
- Real-Time Chunking（RTC）により、行動チャンクを滑らかにつなげて実行できる
- ロールアウトの動かし方を目的別に選べる（`episodic`：`lerobot-record` 相当、`dagger`：人の介入、`sentry`：連続自律収録と自動アップロード、`highlight`：リングバッファによる必要箇所だけの記録）

## 評価・シミュレーション

- 学習済みポリシーをシミュレーション環境で評価できる（`lerobot-eval`）
- ALOHA、PushT、LIBERO、LIBERO-plus、Meta-World、RoboCasa365、RoboTwin 2.0、VLABench、RoboMME、RoboCerebra といったベンチマークを統一コマンドで実行できる
- Hugging Face Hub 上に置かれたシミュレーション環境を、パッケージ導入なしに読み込んで使える（EnvHub）
- NVIDIA IsaacLab Arena や LeIsaac と連携し、SO-101 を含むロボットをシミュレーション上で動かせる（[envhub.md](./envhub.md)）
- 自作のシミュレーション環境を Hub 経由で第三者へ配布できる
- 学習中に一定間隔で評価ロールアウトを挟める（Hub 環境は不可。`--env_eval_freq=0` にして `lerobot-eval` で別途評価する）

## 報酬モデル・タスク評価

- 統一 API（`lerobot.rewards`）でタスクの成功判定・進捗評価ができる
- Robometer により、100 万以上の軌跡で事前学習された汎用モデルで進行度・成功率をスコア化できる
- TOPReward により、事前学習済み VLM を使って追加学習なし（zero-shot）で成功判定ができる
- 自前の画像ベース報酬分類器を学習して使える

## 処理パイプラインの構築

- 観測・行動の変換を `DataProcessorPipeline` として組み立て、学習と実機実行で同じ処理を共有できる
- 正規化、画像変換、座標変換、末端エフェクタ空間への変換などをステップとして差し替えられる
- パイプラインの中身を段階ごとに確認してデバッグできる（[debug_processor_pipeline.mdx](./lerobot/docs/source/debug_processor_pipeline.mdx)）
- 独自の処理ステップを追加できる（[implement_your_own_processor.mdx](./lerobot/docs/source/implement_your_own_processor.mdx)）
- URDF による逆運動学（IK）を用いて、関節空間ではなく末端エフェクタ空間で操作できる

## 関連文書

- ディレクトリ構成と SO-101 ワークショップの参照順：[structure.md](./structure.md)
- v0.6.0 の新機能：[v0.6.0.md](./v0.6.0.md)
- バージョンごとの変遷：[release.md](./release.md)
- HIL-SERL と SO-ARM 101 の対応状況：[HI-SERL.md](./HI-SERL.md)
- EnvHub の詳細：[envhub.md](./envhub.md)
