# LeRobot の変遷

LeRobot は、シミュレーション上の模倣学習・強化学習ライブラリから、実機ロボット、共有データセット、VLA（Vision-Language-Action）モデル、評価ベンチマークまでを扱うロボット学習基盤へ発展してきた。

## リリース一覧

| バージョン | リリース日 | 位置づけ |
| --- | --- | --- |
| v0.1.0 | 2024年3月9日 | PyPI で確認できる最初の公開版 |
| v0.2.0 | 公式リリースなし（内部バージョン変更：2025年7月21日） | パッケージ構成変更に伴う開発版 |
| v0.3.0 | 公式リリースなし（内部バージョン変更：2025年8月1日） | v0.3 系の開発版。最初の正式な GitHub リリースは v0.3.2 |
| v0.3.2 | 2025年8月1日 | v0.3 系として公開された最初の GitHub リリース |
| v0.4.0 | 2025年10月23日 | Dataset v3 とシミュレーション・VLA 対応を大きく拡張 |
| v0.5.0 | 2026年3月9日 | ハードウェア、VLA、データ処理、EnvHub を拡張 |
| v0.6.0 | 2026年7月6日 | 世界モデル、報酬モデル、評価、デプロイを統合 |

日付は、GitHub にリリースが存在する版は GitHub のリリース日、GitHub リリースがない v0.1.0 は PyPI の公開日を基準にした。PyPI のリリース履歴には v0.2.0 と v0.3.0 が掲載されていないため、これらは正式リリースとして扱わず、開発上のバージョン変更として記載している。

内部バージョンの変更日は、v0.2.0 が [PR #1515](https://github.com/huggingface/lerobot/pull/1515)、v0.3.0 が [PR #1649](https://github.com/huggingface/lerobot/pull/1649) の変更日である。

## バージョンごとの主な追加機能

### v0.1.0 — 基本的な学習基盤の成立

- PushT、Aloha、XArm などのシミュレーション環境とデータセットを扱えるようになった。
- ACT と Diffusion Policy による模倣学習の学習・評価フローが整備された。
- データセットの可視化、再生、統計量の計算、Weights & Biases へのログ出力に対応した。
- `LeRobotDataset` を中心に、Hugging Face Hub からデータセットや学習済みモデルを取得・共有できる基盤が整った。
- CPU、CUDA、Apple Silicon の MPS を含む複数の実行環境、CI、エンドツーエンドテストが整備された。
- 後半には ALOHA などの実機ロボット、シリアル通信、テレオペレーション、実機用のデータ収録・再生も加わった。

参照：[PyPI v0.1.0](https://pypi.org/project/lerobot/0.1.0/)、[v0.3.2 の変更履歴](https://github.com/huggingface/lerobot/releases/tag/v0.3.2)

### v0.2.0 — パッケージ構成とデータ形式の転換

v0.2.0 は個別の公式リリースではないが、2025年7月にパッケージ構成を大きく整理する際の内部バージョンとして使われた。この時期の主な変更は次のとおりである。

- ソースコードを `src/lerobot/` 配下へ移し、従来の `lerobot.common` を廃止して公開 API を整理した。
- `LeRobotDataset v2.1` を含む、実機データを扱うためのデータセット基盤を拡張した。
- SO-100 のモバイル構成、Aloha 系の実機データ、Pi0 などの実機向けポリシーを取り込んだ。
- データセットの可視化、Hub へのアップロード、実機制御の CLI とドキュメントを整理した。

参照：[パッケージ構成変更 PR #1417](https://github.com/huggingface/lerobot/pull/1417)、[PyPI のリリース履歴](https://pypi.org/project/lerobot/#history)

### v0.3.0 / v0.3.2 — 共通 API と実機利用の強化

v0.3.0 も個別の公式リリースにはならず、v0.3.2 として公開された。v0.3 系では、既存の研究用コードを共通ライブラリとして使いやすくする変更が進んだ。

- `LeRobotDataset`、ポリシー、データローダー、正規化、チェックポイント保存の API を統一した。
- 実機ロボットを制御するデバイス抽象化、モーター設定、キャリブレーション、テレオペレーション、データ収録を追加・整理した。
- ALOHA、Koch、Koch bimanual などの実機に対応し、実機上で ACT を動かせるようになった。
- 複数データセットを組み合わせる `MultiLeRobotDataset`、エピソード単位のサンプリング、動画デコード、Dataset Card、Rerun による可視化を追加した。
- VQ-BeT、TD-MPC、オンライン学習、データ拡張、OpenX/RLDS からの変換など、学習対象とデータ連携を広げた。

参照：[GitHub v0.3.2 リリース](https://github.com/huggingface/lerobot/releases/tag/v0.3.2)

### v0.4.0 — Dataset v3 と VLA への拡張

- `LeRobotDataset v3.0` を導入した。複数エピソードをまとめたファイル構成、Parquet によるメタデータ管理、大規模データセット向けのストリーミングに対応した。
- データセットの変換・削除・分割・結合などを行う編集ツールを追加した。
- LIBERO と Meta-World のシミュレーション環境、複数 GPU 学習を追加した。
- PI0、PI0.5、GR00T N1.5 などの VLA ポリシーを追加した。
- データ処理を共通化する `DataProcessorPipeline` を導入し、電話を使ったテレオペレーションにも対応した。
- Reachy 2 などのロボットを外部プラグインとして統合しやすくした。
- Hugging Face Robot Learning Course を公開した。

参照：[LeRobot v0.4.0 リリースブログ](https://huggingface.co/blog/lerobot-release-v040)、[GitHub v0.4.0 リリース](https://github.com/huggingface/lerobot/releases/tag/v0.4.0)

### v0.5.0 — ハードウェア、VLA、データ処理のスケールアップ

- Unitree G1 の全身制御、OpenArm / OpenArm Mini、CAN バスモーターなど、対応ハードウェアを拡張した。
- Pi0-FAST の自己回帰 VLA、Real-Time Chunking、Wall-X、X-VLA、SARM、PEFT / LoRA を追加した。
- ストリーミング動画エンコード、画像学習の高速化、データセット編集ツールを追加した。
- EnvHub により、Hugging Face Hub 上のシミュレーション環境を直接読み込めるようにした。
- NVIDIA IsaacLab-Arena と連携した。
- Python 3.12 以上、Transformers v5 を前提とするモダンなコードベースへ移行した。
- サードパーティ製ポリシーをプラグインとして追加できる仕組みを整備した。

参照：[LeRobot v0.5.0 リリースブログ](https://huggingface.co/blog/lerobot-release-v050)、[GitHub v0.5.0 リリース](https://github.com/huggingface/lerobot/releases/tag/v0.5.0)

### v0.6.0 — 想像・評価・改善のループを統合

- 将来の状態を予測してから行動する世界モデル系ポリシーとして、VLA-JEPA、LingBot-VA、FastWAM を追加した。
- GR00T N1.7、MolmoAct2、EO-1、Multitask DiT、EVO1 などの VLA を追加した。
- `lerobot.rewards` に報酬モデル API を導入し、Robometer と TOPReward による成功判定・進捗評価に対応した。
- 深度カメラの記録・デコード・学習、動画コーデックの指定、VLM による言語アノテーションを追加した。
- 並列動画デコードなどにより、動画データの読み込みを最大約2倍に高速化した。
- `lerobot-eval` から LIBERO-plus、RoboTwin 2.0、RoboCasa365、RoboCerebra、RoboMME、VLABench を統一的に評価できるようにした。
- デプロイ専用の `lerobot-rollout` CLI と DAgger 形式の人間介入データ収集を追加した。
- FSDP によるマルチ GPU 学習と、Hugging Face Jobs を利用したクラウド学習に対応した。
- 基本インストールの依存関係を機能別の extras に分離し、最小インストールを軽量化した。

なお、v0.6.0 では `pip install lerobot` の依存関係、インポートパス、GR00T のバージョン、PyTorch の最低バージョンなどに破壊的変更がある。既存環境を更新する場合は公式リリースノートを確認する必要がある。

参照：[LeRobot v0.6.0 リリースブログ](https://huggingface.co/blog/lerobot-release-v060)、[GitHub v0.6.0 リリース](https://github.com/huggingface/lerobot/releases/tag/v0.6.0)

## 全体の流れ

LeRobot の変化は、次のように整理できる。

1. **v0.1 系**：シミュレーション、ACT / Diffusion、データセット、学習・評価の基本機能を整備した。
2. **v0.2〜v0.3 系**：実機制御、Hub 連携、データセット API、ポリシー API、パッケージ構成を整理した。
3. **v0.4 系**：Dataset v3、ストリーミング、VLA、シミュレーション、共通プロセッサを追加した。
4. **v0.5 系**：対応ロボット、VLA、EnvHub、動画処理、プラグインを拡張した。
5. **v0.6 系**：世界モデル、報酬モデル、深度・言語データ、評価ベンチマーク、デプロイ、クラウド学習を一つのループに統合した。

### 参考資料

- [LeRobot GitHub リポジトリ](https://github.com/huggingface/lerobot)
- [GitHub タグ一覧](https://github.com/huggingface/lerobot/tags)
- [PyPI のリリース履歴](https://pypi.org/project/lerobot/#history)
