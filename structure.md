# LeRobot ディレクトリ構成と SO-ARM 101 ワークショップ参照ガイド

## この文書について

この文書は、手元の `lerobot` リポジトリ（v0.6.0 系）のディレクトリ構成を調べ、SO-ARM 101 を使ったワークショップで参照すべき場所をまとめたものです。

ワークショップでは、すべての実装を読む必要はありません。まず `docs/source/` の手順を上から進め、挙動を理解したいときだけ `src/lerobot/` の対応実装や `examples/` のコードを参照するのが効率的です。

## 最初に見るべきファイル

| 優先度 | パス | 用途 |
| --- | --- | --- |
| 1 | [`docs/source/so101.mdx`](docs/source/so101.mdx) | SO-101 の組立、USB ポート確認、モーター ID・baudrate 設定、キャリブレーション |
| 1 | [`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) | 実機でのテレオペレーション、カメラ、データ収録、再生、学習、推論の一連の流れ |
| 1 | [`docs/source/installation.mdx`](docs/source/installation.mdx) | Python 環境と LeRobot のインストール |
| 1 | [`docs/source/cameras.mdx`](docs/source/cameras.mdx) | カメラの検出・設定・トラブルシューティング |
| 2 | [`docs/source/act.mdx`](docs/source/act.mdx) | 初回の模倣学習に向く ACT の説明・学習・評価例 |
| 2 | [`docs/source/feetech.mdx`](docs/source/feetech.mdx) | Feetech モーターを使う場合の補足 |
| 2 | [`docs/source/lelab.mdx`](docs/source/lelab.mdx) | CLI の代わりに GUI で SO-ARM101 を設定・操作・収録・学習する方法 |
| 3 | [`README.md`](README.md) | プロジェクト全体の目的、対応ハードウェア、関連リンク |

## リポジトリ全体の構成

```text
lerobot/
├── docs/                 # 利用者向けドキュメント。ワークショップの主な入口
│   └── source/           # Markdown/MDX の原稿
├── examples/             # API や特定用途の実行例
├── src/lerobot/          # LeRobot 本体の Python パッケージ
│   ├── robots/           # フォロワーロボットの実装
│   ├── teleoperators/   # リーダーや入力デバイスの実装
│   ├── motors/           # モーター・シリアルバスの抽象化
│   ├── cameras/          # カメラドライバ
│   ├── datasets/         # LeRobotDataset の保存・読み込み・Hub 連携
│   ├── policies/         # ACT、Diffusion、SmolVLA などの学習ポリシー
│   ├── configs/          # CLI 設定とサブクラス登録
│   ├── processor/        # 観測・行動の変換パイプライン
│   ├── rollout/          # 学習済みポリシーの実機実行
│   └── scripts/          # lerobot-* コマンドのエントリポイント
├── tests/                # テスト
├── utils/                # ドキュメント・設定・開発用ユーティリティ
├── docker/               # Docker 用設定
├── media/                # README・ドキュメント用画像や動画
├── pyproject.toml        # 依存関係、extras、CLI エントリポイント
└── Makefile              # 開発・ドキュメント関連のタスク
```

## SO-ARM 101 の実装マップ

### 物理構成と通信

- [`src/lerobot/robots/so_follower/`](src/lerobot/robots/so_follower/)  
  SO-101 フォロワーアームを制御します。6 個の `sts3215` モーター（`shoulder_pan`、`shoulder_lift`、`elbow_flex`、`wrist_flex`、`wrist_roll`、`gripper`）、キャリブレーション、カメラ接続、位置制御を扱います。
- [`src/lerobot/teleoperators/so_leader/`](src/lerobot/teleoperators/so_leader/)  
  SO-101 リーダーアームから関節位置を読み取り、フォロワーへフィードバックするテレオペレーターです。
- [`src/lerobot/motors/feetech/`](src/lerobot/motors/feetech/)  
  Feetech SDK を通じたシリアル通信、モーター ID、baudrate、制御テーブル、読み書きを担当します。

設定名は CLI では `so101_follower` と `so101_leader` を使います。コード上では互換性のため `SO100Follower` / `SO101Follower`、`SO100Leader` / `SO101Leader` が同じ基底実装を参照する箇所があります。ワークショップではコマンドの機種名を混同しないよう、SO-101 用には `so101_*` を使ってください。

### カメラ、データ、学習済みポリシー

- [`src/lerobot/cameras/`](src/lerobot/cameras/)  
  OpenCV、RealSense などのカメラ実装。カメラ ID や解像度、FPS は [`docs/source/cameras.mdx`](docs/source/cameras.mdx) と併読します。
- [`src/lerobot/datasets/`](src/lerobot/datasets/)  
  関節状態・行動・画像・動画を LeRobotDataset 形式で保存し、Hugging Face Hub と連携します。
- [`src/lerobot/policies/act/`](src/lerobot/policies/act/)  
  初回ワークショップの学習対象として扱いやすい ACT の設定とモデル実装です。
- [`src/lerobot/policies/smolvla/`](src/lerobot/policies/smolvla/)  
  言語指示を含む VLA を扱う場合の実装です。ACT より計算資源と依存関係の要求が大きいため、発展編向けです。
- [`src/lerobot/rollout/`](src/lerobot/rollout/)  
  学習済みポリシーを実機で実行する rollout の制御ロジックです。

## ワークショップの推奨参照順

### 1. 事前準備・インストール

1. ハードウェアの部品表・3D プリント・組立は [`docs/source/so101.mdx`](docs/source/so101.mdx) の冒頭にある SO-ARM100 リポジトリへのリンクを確認します。機械部品の一次資料はこのリポジトリの外部にあります。
2. ソフトウェア環境は [`docs/source/installation.mdx`](docs/source/installation.mdx) を読みます。
3. Feetech 用追加依存関係が必要なため、SO-101 の手順にある `pip install -e ".[feetech]"` を確認します。

### 2. モーター設定とキャリブレーション

参照先は [`docs/source/so101.mdx`](docs/source/so101.mdx) です。

- `lerobot-find-port`：リーダーとフォロワーの USB ポートを特定
- `lerobot-setup-motors --robot.type=so101_follower`：フォロワーの ID・baudrate 設定
- `lerobot-setup-motors --teleop.type=so101_leader`：リーダーの ID・baudrate 設定
- `lerobot-calibrate`：各関節の可動範囲とホーム位置を登録

対応する CLI 実装は [`src/lerobot/scripts/`](src/lerobot/scripts/) の `lerobot_find_port.py`、`lerobot_setup_motors.py`、`lerobot_calibrate.py` です。機械を壊す可能性がある工程なので、受講者にはまずドキュメントの接続順と電源・ケーブル確認を徹底してもらいます。

### 3. テレオペレーション確認

まずカメラなしで動作確認し、その後カメラを追加します。

- 手順と SO-101 のコマンド例：[`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Teleoperate」
- 実行スクリプト：[`src/lerobot/scripts/lerobot_teleoperate.py`](src/lerobot/scripts/lerobot_teleoperate.py)
- リーダー実装：[`src/lerobot/teleoperators/so_leader/`](src/lerobot/teleoperators/so_leader/)
- フォロワー実装：[`src/lerobot/robots/so_follower/`](src/lerobot/robots/so_follower/)

### 4. カメラ確認とデータセット収録

- カメラの番号・映像・解像度は [`docs/source/cameras.mdx`](docs/source/cameras.mdx) を参照します。
- 収録手順、エピソード操作、Hub へのアップロードは [`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Record a dataset」を参照します。
- 実行スクリプトは [`src/lerobot/scripts/lerobot_record.py`](src/lerobot/scripts/lerobot_record.py) です。
- API の最小例を見たい場合は [`examples/dataset/`](examples/dataset/) を参照します。

最初の実演では少数エピソードで接続とデータ形式を確認し、本番データでは同じタスクを複数の物体位置・条件で繰り返します。収録後は [`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Visualize a dataset」で映像・関節状態・行動を確認します。

### 5. 再生とデータ検証

- [`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Replay an episode」
- [`src/lerobot/scripts/lerobot_replay.py`](src/lerobot/scripts/lerobot_replay.py)
- [`examples/backward_compatibility/replay.py`](examples/backward_compatibility/replay.py)
- [`docs/source/using_dataset_tools.mdx`](docs/source/using_dataset_tools.mdx)

学習前に、収録データのエピソード数、動画の視認性、タスク説明、FPS、関節の動きが揃っているかを確認します。

### 6. 初回の模倣学習

最初は ACT を推奨します。

- 概念とコマンド：[`docs/source/act.mdx`](docs/source/act.mdx)
- 実機ワークフロー内の学習手順：[`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Train a policy」
- CLI 実装：[`src/lerobot/scripts/lerobot_train.py`](src/lerobot/scripts/lerobot_train.py)
- ポリシー実装：[`src/lerobot/policies/act/`](src/lerobot/policies/act/)
- 学習用コード例：[`examples/training/`](examples/training/)

学習結果は通常 `outputs/train/<job_name>/checkpoints/` に保存されます。GPU が不足する場合は [`docs/source/hardware_guide.mdx`](docs/source/hardware_guide.mdx) を見て、バッチサイズやポリシーの選択を調整します。

### 7. 実機 rollout・評価

- 手順：[`docs/source/il_robots.mdx`](docs/source/il_robots.mdx) の「Run inference and evaluate your policy」
- CLI 実装：[`src/lerobot/scripts/lerobot_rollout.py`](src/lerobot/scripts/lerobot_rollout.py)
- 実行基盤：[`src/lerobot/rollout/`](src/lerobot/rollout/)
- ACT の評価例：[`docs/source/act.mdx`](docs/source/act.mdx)

初回は低速・短時間・小さな対象物で評価し、非常停止できる状態で実行します。学習時と同じカメラ名、解像度、タスク説明、ロボットのキャリブレーションを使うことが重要です。

## コマンドと実装の対応

| ワークショップ操作 | CLI | 主な実装 |
| --- | --- | --- |
| USB ポート検出 | `lerobot-find-port` | `src/lerobot/scripts/lerobot_find_port.py` |
| モーター設定 | `lerobot-setup-motors` | `src/lerobot/scripts/lerobot_setup_motors.py`、`robots/so_follower`、`teleoperators/so_leader` |
| キャリブレーション | `lerobot-calibrate` | `src/lerobot/scripts/lerobot_calibrate.py`、`robots/so_follower`、`teleoperators/so_leader` |
| リーダーで操作 | `lerobot-teleoperate` | `src/lerobot/scripts/lerobot_teleoperate.py` |
| データ収録 | `lerobot-record` | `src/lerobot/scripts/lerobot_record.py`、`datasets/` |
| エピソード再生 | `lerobot-replay` | `src/lerobot/scripts/lerobot_replay.py` |
| ポリシー学習 | `lerobot-train` | `src/lerobot/scripts/lerobot_train.py`、`policies/` |
| 学習済みモデル実行 | `lerobot-rollout` | `src/lerobot/scripts/lerobot_rollout.py`、`rollout/` |

これらのコマンドは [`pyproject.toml`](pyproject.toml) の `[project.scripts]` で `src/lerobot/scripts/` の関数に登録されています。CLI の引数を確認するときは、まずドキュメントの例を使い、足りない場合に各スクリプトの `--help` と設定クラスを確認します。

## ワークショップでは後回しにしてよい場所

以下は LeRobot 全体では重要ですが、SO-ARM 101 の基本ワークショップでは通常後回しにできます。

- `src/lerobot/rl/`：強化学習
- `src/lerobot/envs/`：シミュレーション環境・ベンチマーク
- `src/lerobot/distributed/`：分散学習
- `src/lerobot/annotations/`、`src/lerobot/rewards/`：アノテーション・報酬モデル
- `src/lerobot/policies/` の ACT 以外の各ディレクトリ：別モデルを扱う発展編
- `docker/`：環境を Docker に統一する場合のみ
- `tests/`、`utils/`：開発者・コントリビューター向け

## 参照の基本方針

1. 実際に操作する人は `docs/source/so101.mdx` と `docs/source/il_robots.mdx` を中心に読む。
2. コマンドが何をしているか確認したいときは `src/lerobot/scripts/` を読む。
3. SO-101 固有の動作を確認したいときは `robots/so_follower/`、`teleoperators/so_leader/`、`motors/feetech/` の順に読む。
4. データの中身を理解するときは `datasets/` と `docs/source/lerobot-dataset-v3.mdx` を読む。
5. 初回学習は ACT、言語指示や大規模モデルは SmolVLA などの発展編に分ける。

