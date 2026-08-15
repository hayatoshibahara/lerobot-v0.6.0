# Google Colab で ACT を訓練し、SO-101 で評価する

## 目次

- [概要](#概要)
- [この資料で行うこと](#この資料で行うこと)
- [はじめる前に](#はじめる前に)
- [用語と設定値](#用語と設定値)
- [ステップ 1: Colab で GPU を有効にする](#ステップ-1-colab-で-gpu-を有効にする)
- [ステップ 2: LeRobot v0.6.0 をインストールする](#ステップ-2-lerobot-v060-をインストールする)
- [ステップ 3: Hugging Face と W&B にログインする](#ステップ-3-hugging-face-と-wb-にログインする)
- [ステップ 4: Hub のデータセットを確認する](#ステップ-4-hub-のデータセットを確認する)
- [ステップ 5: 初回の訓練設定を決める](#ステップ-5-初回の訓練設定を決める)
- [ステップ 6: ACT を訓練する](#ステップ-6-act-を訓練する)
- [ステップ 7: W&B で訓練進捗を確認する](#ステップ-7-wb-で訓練進捗を確認する)
- [早期終了の判断方法](#早期終了の判断方法)
- [モデルを Hugging Face Hub へアップロードする](#モデルを-hugging-face-hub-へアップロードする)
- [途中の訓練を再開する](#途中の訓練を再開する)
- [訓練パラメータの適切な値を探す](#訓練パラメータの適切な値を探す)
- [ローカル PC の SO-101 で評価する](#ローカル-pc-の-so-101-で評価する)
- [完了チェックリスト](#完了チェックリスト)
- [トラブルシューティング](#トラブルシューティング)
- [参考にしたファイル・URL](#参考にしたファイルurl)

## 概要

この資料では、[`dataset.md`](./dataset.md) で Hugging Face Hub に保存した SO-101 のデータセットを使い、Google Colab で ACT（Action Chunking with Transformers）を訓練する。

進捗は Weights & Biases（以下 W&B）で確認し、チェックポイントと完成モデルは Hub に保存する。

Colab の実行環境は、切断されると削除される。そこで、5,000 step ごとにチェックポイントを Hub へ保存し、新しいランタイムから訓練を再開できるようにする。

訓練後は、Hub 上のモデルをローカル PC へダウンロードして SO-101 で評価する。チェックポイントの選別は各 10 回、最終評価は 25 回以上行い、実機成功率で採用モデルを決める。

## この資料で行うこと

1. Colab の GPU ランタイムを有効にする。
2. ワークショップと同じ LeRobot v0.6.0 を Colab にインストールする。
3. Hugging Face と W&B の認証を準備する。W&B は未登録の状態から説明する。
4. Hub からデータセットのメタデータを取得し、エピソード数、フレーム数、FPS、カメラ名を確認する。
5. ACT を訓練し、W&B で訓練損失と検証損失を確認する。
6. 途中のチェックポイントと完成モデルを Hugging Face Hub へ保存する。
7. Colab の切断後や訓練を延長するときの再開方法を確認する。
8. パラメータを 1 項目ずつ比較し、適切な値を探す。
9. ローカル PC の SO-101 で候補モデルを 10 回以上、最終候補を 25 回以上評価し、成功率と工程別到達率を記録する。

## はじめる前に

- [`dataset.md`](./dataset.md) の手順を終え、本番データセットを Hugging Face Hub にアップロード済みであること
- Hub の Visualizer ですべてのエピソードを確認し、失敗、白画面、動画の乱れ、長い静止を含むデータを除外済みであること
- データセットの `所有者名/リポジトリ名` が分かっていること
- [`dataset.md`](./dataset.md) で作成した Hugging Face の Write 権限トークンを、Colab Secrets の `HF_TOKEN` に登録済みであること
- Google アカウントで Google Colab を開けること
- 評価用 PC で [`install.md`](./install.md) と [`setup.md`](./setup.md) の手順を終えていること
- 評価時に、収録時と同じ SO-101、キャリブレーション、カメラ名、解像度、FPS、作業台、照明を再現できること

Colab のコードは、上から 1 セルずつ実行する。`!` で始まる行は Colab 内の Linux コマンドであり、ローカル PC では実行しない。

Hugging Face トークンと W&B API キーは、パスワードと同様に扱う。コード、出力、スクリーンショット、Git リポジトリへ書かず、`print()` でも表示しない。ノートブックを共有する前に、セル出力へ認証情報が含まれていないことを確認する。

## 用語と設定値

| 項目 | 意味 | この資料の例 |
| --- | --- | --- |
| `HF_TOKEN` | Colab Secrets に保存する Hugging Face の Write 権限トークン名 | `HF_TOKEN`（大文字・小文字を区別する） |
| `WANDB_API_KEY` | Colab Secrets に保存する W&B の個人用 API キー名 | `WANDB_API_KEY`（大文字・小文字を区別する） |
| `HF_USER` | Hugging Face のユーザー名または組織名 | `my-hf-name` |
| `DATASET_REPO_ID` | [`dataset.md`](./dataset.md) で作った本番データセット | `my-hf-name/so101-pick-red-cube-production-20260815` |
| `MODEL_REPO_ID` | 訓練済み ACT を保存する Hub のモデルリポジトリ | `my-hf-name/act-so101-pick-red-cube-v1` |
| `RUN_NAME` | Colab の出力と W&B の実験を識別する名前 | `act-so101-pick-red-cube-v1` |
| `OUTPUT_DIR` | Colab 内のチェックポイント保存先 | `/content/outputs/train/act-so101-pick-red-cube-v1` |
| `WANDB_PROJECT` | W&B で複数の実験をまとめるプロジェクト名 | `lerobot-so101-act` |
| `batch_size` | 1 step で処理するサンプル数 | まず `8` |
| `steps` | 訓練ループを繰り返す合計回数 | まず `30000` |
| epoch | 訓練データを 1 周分処理した量 | データ量とバッチサイズから換算する |
| `save_freq` | チェックポイントを保存する間隔 | `5000` step |
| `eval_split` | 検証専用に取り分けるエピソードの割合 | `0.1` |
| `eval_steps` | 検証損失を計算する間隔 | `1000` step |

`MODEL_REPO_ID` と `RUN_NAME` には、タスク名と実験番号を含める。異なる実験で同じ `OUTPUT_DIR` を使うと既存データと衝突するため、`v1`、`v2` または変更した設定値を名前に含める。

## ステップ 1: Colab で GPU を有効にする

1. [Google Colab](https://colab.research.google.com/) を開き、「ノートブックを新規作成」を選ぶ。
2. メニューの「ランタイム」→「ランタイムのタイプを変更」を開く。
3. ハードウェア アクセラレータを **GPU** にして保存する。
4. 次のセルを実行する。

```python
import torch

assert torch.cuda.is_available(), "GPU が有効ではない。Colab のランタイム設定を確認する"
print("PyTorch:", torch.__version__)
print("GPU:", torch.cuda.get_device_name(0))
print("VRAM (GB):", round(torch.cuda.get_device_properties(0).total_memory / 1024**3, 1))
```

GPU 名と VRAM が表示されれば準備完了である。割り当てられた GPU 名は、訓練条件として記録しておく。

### T4 と A100 のどちらを使うか

この資料の初回設定（ACT、バッチサイズ 8、30,000 step）は、まず **T4** で試す。LeRobot の hardware guide では、ACT・バッチサイズ 8 の使用量を約 2〜6 GB VRAM と見積もっている。A100 は高速だが、初回訓練には必須ではない。

| GPU | この資料での位置付け | 費用と注意 |
| --- | --- | --- |
| T4 | 初回の推奨。バッチサイズ 8 で開始する | 無料枠で割り当てられる場合があるが、利用可否や連続利用時間は保証されない |
| L4 / V100 | T4 より速い有料候補 | 有料プランでも在庫と compute unit によって利用可否が変わる |
| A100 | 多数の実験を短時間で行う場合の候補 | 通常は有料プランまたは従量課金が必要。初回訓練には必須ではない |

T4 が割り当てられたら、そのままバッチサイズ 8 で始める。メモリ不足になった場合だけ、バッチサイズ 4 と勾配蓄積へ変更する。1,000 step 程度進んだら、W&B の `train/step_s` から残り時間を概算できる。

```text
残り時間（時間） ≈ train/step_s × 残り step 数 ÷ 3600
```

無料枠でも完了する可能性はあるが、GPU の種類と利用上限は固定されていない。途中で切断された場合は、Hub に保存したチェックポイントから再開する。

## ステップ 2: LeRobot v0.6.0 をインストールする

公式 GitHub から v0.6.0 を取得し、データセット、訓練、W&B に必要な `training` 追加依存関係をインストールする。

```python
%cd /content
!git clone --depth 1 --branch v0.6.0 https://github.com/huggingface/lerobot.git
%pip install -e "/content/lerobot[training]"
```

続けてバージョンとコマンドを確認する。

```python
!cd /content/lerobot && git describe --tags --always
!lerobot-train --help > /dev/null && echo "lerobot-train: OK"
!hf --help > /dev/null && echo "hf: OK"
```

`v0.6.0`、`lerobot-train: OK`、`hf: OK` が表示されればよい。

`destination path 'lerobot' already exists` と表示された場合は、同じランタイムで clone セルを 2 回実行している。既にインストールが終わっていれば次へ進む。

環境が不明な場合は「ランタイム」→「接続を解除してランタイムを削除」を実行し、ステップ 1 からやり直す。

## ステップ 3: Hugging Face と W&B にログインする

### 3-1. Hugging Face にログインする

[`dataset.md`](./dataset.md) で作成した Write 権限のトークンを、Colab Secrets から読み込む。未登録の場合は、次の操作を行う。

1. Colab の左サイドバーにある鍵アイコンの **シークレット** を開く。
2. **新しいシークレットを追加**し、名前を `HF_TOKEN`、値を `hf_` から始まるトークンにする。名前は大文字・小文字を区別する。
3. `HF_TOKEN` の **ノートブックからのアクセス**を有効にする。

シークレットはランタイムが切断されても残る。別のノートブックで使う場合は、そのノートブックからのアクセスを改めて許可する。

次のセルを実行する。

```python
from google.colab import userdata
from huggingface_hub import login, whoami

login(
    token=userdata.get("HF_TOKEN"),
    add_to_git_credential=False,
)
print("Hugging Face user:", whoami()["name"])
```

ユーザー名が表示されればログイン成功である。新しいランタイムで再開するときも、このセルを実行する。トークンを再発行した場合は、シークレットの値も更新する。

### 3-2. W&B アカウントを作成する

W&B に未登録である前提で、次の順に準備する。

1. [W&B の登録ページ](https://wandb.ai/site) を開き、アカウントを作成する。
2. 必要な場合はメールアドレスを確認し、W&B にログインする。
3. W&B の User Settings にある **API keys** を開く。
4. 用途が分かる名前で新しい個人用 API キーを作成し、作成時に一度だけ表示される値をコピーする。
5. Colab の左サイドバーにある鍵アイコンの **シークレット**を開く。
6. **新しいシークレットを追加**し、名前を `WANDB_API_KEY`、値をコピーした API キーにする。名前は大文字・小文字を区別する。
7. `WANDB_API_KEY` の **ノートブックからのアクセス**を有効にする。

次のセルを実行し、API キーが有効か確認する。

```python
from google.colab import userdata
import wandb

logged_in = wandb.login(
    key=userdata.get("WANDB_API_KEY"),
    verify=True,
)
assert logged_in, "W&B へログインできなかった。シークレットとネットワークを確認する"
```

エラーなく完了すればログイン成功である。新しいランタイムでは、このセルも再実行する。API キーを再発行した場合は、シークレットの値も更新する。

## ステップ 4: Hub のデータセットを確認する

まず、実際の値へ置き換えて設定セルを実行する。`DATASET_REPO_ID` は Hub のデータセットページに表示される `所有者名/リポジトリ名` をそのまま使う。

```python
HF_USER = "my-hf-name"
DATASET_REPO_ID = f"{HF_USER}/so101-pick-red-cube-production-20260815"

RUN_NAME = "act-so101-pick-red-cube-v1"
MODEL_REPO_ID = f"{HF_USER}/{RUN_NAME}"
OUTPUT_DIR = f"/content/outputs/train/{RUN_NAME}"
WANDB_PROJECT = "lerobot-so101-act"

print("dataset:", DATASET_REPO_ID)
print("model:", MODEL_REPO_ID)
print("output:", OUTPUT_DIR)
```

次のセルは Hub からメタデータを取得する。非公開データセットでも、ステップ 3 の認証が正しければ読み込める。

```python
from lerobot.datasets import LeRobotDatasetMetadata

metadata = LeRobotDatasetMetadata(DATASET_REPO_ID)
print("episodes:", metadata.total_episodes)
print("frames:", metadata.total_frames)
print("fps:", metadata.fps)
print("cameras:", metadata.camera_keys)
print("robot:", metadata.robot_type)
```

次を確認する。

- `episodes` が本番収録した数と一致する
- `frames` が 0 より大きい
- `fps` が収録時の値と一致する
- `cameras` の名前が `observation.images.front` など、収録時のカメラ名と一致する
- `robot` が想定した SO-101 系の値になっている

`lerobot-train` はこの後、指定した `DATASET_REPO_ID` から Parquet と動画を Colab の Hugging Face キャッシュへ自動ダウンロードする。データセットを Google Drive へ手動コピーする必要はない。

## ステップ 5: 初回の訓練設定を決める

初回は次の値を使う。30,000 step は訓練の完了値ではなく、最初に実機評価するための区切りである。

```python
BATCH_SIZE = 8
STEPS = 30_000
SAVE_FREQ = 5_000
EVAL_SPLIT = 0.1
EVAL_STEPS = 1_000

steps_per_epoch = -(-metadata.total_frames // BATCH_SIZE)
print("steps per epoch (全データでの概算):", steps_per_epoch)
print("planned epochs (概算):", round(STEPS / steps_per_epoch, 1))
```

[`dataset.md`](./dataset.md) の例（50 エピソード、各 30 秒、30 FPS）は、約 45,000 フレームである。バッチサイズ 8 の場合、全データを 1 周するには約 5,625 step かかる。

```text
steps_per_epoch = ceil(total_frames / batch_size)
```

したがって、30,000 step は全データ換算で約 5.3 epoch になる。`eval_split=0.1` で検証データを除くと、訓練データ換算では約 5.9 epoch である。ただし、エピソードごとの長さによって実際の値は変わる。

`eval_split=0.1` は、各タスクの末尾 10% を検証用に取り分ける設定である。50 エピソードなら約 5 エピソードが対象になる。末尾のエピソードが特定の開始位置に偏っていないか、Hub の映像で確認する。検証損失は実験間の比較に使えるが、実機成功率の代わりにはならない。

### これらの初期値を設定した根拠

LeRobot は epoch 数ではなく、更新回数を `steps` で指定する。W&B の `train/epochs` は、処理したサンプル数から計算される換算値である。データ量やバッチサイズを変えた場合は、同じ `steps` をそのまま使わず、epoch を計算し直す。

| 設定 | 初期値 | 根拠と注意 |
| --- | ---: | --- |
| `batch_size` | `8` | LeRobot のデフォルトであり、ACT 公式例でも使われている。T4 でメモリ不足になった場合は 4 に下げる |
| `steps` | `30000` | LeRobot ガイドの「まず 5 epoch で評価する」という目安を、この資料のデータ量に換算した値。最終値ではない |
| `save_freq` | `5000` | 6 個のチェックポイントを比較でき、切断時に失う進捗を約 5,000 step 以内に抑えられる |
| `eval_split` | `0.1` | 約 5 エピソードを検証へ回し、約 45 エピソードを訓練へ残す |
| `eval_steps` | `1000` | 30,000 step 中に約 30 回、検証損失を確認できる。遅い場合は 2,000〜5,000 に広げる |
| `log_freq` | `100` | 異常を早めに見つけつつ、ログ送信を増やしすぎない間隔である |
| 学習率 | `1e-5` | LeRobot ACT と ACT 公式実装の基準値である |
| `chunk_size` / `n_action_steps` | `100` / `100` | LeRobot ACT のデフォルトであり、ACT 公式例の `chunk_size` も 100 である |
| `kl_weight` | `10` | LeRobot ACT と ACT 公式例の基準値である |
| `seed` | `1000` | LeRobot のデフォルト。採用前に別の seed でも結果を確認する |

ACT 公式実装の epoch は、LeRobot と定義が異なる。50 エピソードを 80:20 に分け、バッチサイズ 8 で学習する場合、ACT 公式実装の 1 epoch は約 5 step である。そのため、公式 README の実機向け 5,000 epoch は約 25,000 step に相当する。

これは 30,000 step に近いが、実装やデータが異なるため、参考値としてのみ扱う。

根拠の確認先は次のとおりである。

- epoch と step の目安: [`hardware_guide.mdx`](./lerobot/docs/source/hardware_guide.mdx)、[`AGENT_GUIDE.md`](./lerobot/AGENT_GUIDE.md)
- LeRobot のデフォルト値と計算方法: [`train.py`](./lerobot/src/lerobot/configs/train.py)、[`logging_utils.py`](./lerobot/src/lerobot/utils/logging_utils.py)
- ACT 公式実装との比較: [ACT 公式 README](https://github.com/tonyzhaozh/act#example-usages)、[ACT 公式データローダー](https://github.com/tonyzhaozh/act/blob/main/utils.py)

## ステップ 6: ACT を訓練する

次のセルを実行する。初回はデータセットの動画ダウンロードとキャッシュ作成に時間がかかる。

```python
!lerobot-train \
  --dataset.repo_id={DATASET_REPO_ID} \
  --dataset.eval_split={EVAL_SPLIT} \
  --policy.type=act \
  --policy.device=cuda \
  --output_dir={OUTPUT_DIR} \
  --job_name={RUN_NAME} \
  --batch_size={BATCH_SIZE} \
  --steps={STEPS} \
  --log_freq=100 \
  --eval_steps={EVAL_STEPS} \
  --save_freq={SAVE_FREQ} \
  --wandb.enable=true \
  --wandb.project={WANDB_PROJECT} \
  --wandb.disable_artifact=true \
  --policy.repo_id={MODEL_REPO_ID} \
  --policy.private=true \
  --save_checkpoint_to_hub=true
```

関連する引数をまとめると、次のようになる。

| 引数 | 目的 |
| --- | --- |
| `--dataset.repo_id` | Hub から訓練用データセットを取得する |
| `--dataset.eval_split`、`--eval_steps` | 検証データの割合と検証間隔を指定する |
| `--policy.type`、`--policy.device` | ACT を Colab の GPU で訓練する |
| `--steps`、`--save_freq` | 合計 step とチェックポイントの保存間隔を指定する |
| `--wandb.*` | W&B にグラフを送り、モデルファイルの重複保存は無効にする |
| `--policy.repo_id`、`--policy.private` | 保存先の非公開モデルリポジトリを指定する |
| `--save_checkpoint_to_hub=true` | チェックポイントを Hub に保存し、切断後の再開に使えるようにする |

訓練が始まると、`loss`、`grdn`、`lr`、`data_s`、`updt_s` などが定期的に表示される。5,000 step に達したら、Hub に `checkpoints/005000` が作成されたことを確認する。

## ステップ 7: W&B で訓練進捗を確認する

訓練開始時の Colab ログに `Track this run -->` で始まる URL が表示される。URL を開くか、[W&B](https://wandb.ai/home) のプロジェクト一覧から `lerobot-so101-act` を開く。

最初に確認するグラフは次のとおりである。

| W&B の項目 | 意味 | 見方 |
| --- | --- | --- |
| `train/loss` | ACT の訓練用合計損失 | 初期より全体として下がっているかを見る。細かな上下は正常である |
| `train/l1_loss` | 予測行動とデモ行動の L1 誤差 | 基準実験と同じ尺度で比較する |
| `train/kld_loss` | ACT の VAE 潜在変数に対する正則化損失 | 単独の最小化を目的にしない |
| `eval/eval_loss` | 取り分けたエピソードでの合計損失 | 同じデータ分割の実験同士で比較する |
| `train/grad_norm` | 勾配の大きさ | 急増が続く、`NaN` になる場合は不安定化を疑う |
| `train/gpu_mem_gb` | 使用した GPU メモリ | 上限近くならバッチサイズを増やさない |
| `train/step_s` | 1 step の所要時間 | 極端に遅い場合は動画読み込みや Colab の負荷を確認する |
| `train/epochs` | データセットを何周分読んだか | バッチサイズが異なる実験を比較するときに使う |

`eval/eval_loss` が見当たらない場合は、1,000 step に達しているか確認する。記録済みなのにグラフがない場合は、W&B の **Add panels** → **Quick panel builder** から `eval/eval_loss` を追加する。

平滑化は傾向を見るときに使い、実験を比較するときは平滑化前の値と step も確認する。損失の大きさはロボットやカメラ構成で変わるため、他人の実験値を合格基準にしない。

### 正常・要注意・中止の目安

| 状況 | 判断 |
| --- | --- |
| `train/loss` と `eval/eval_loss` が全体として低下する | 正常。予定 step まで継続する |
| 訓練損失は下がるが検証損失が継続して上がる | 過学習の候補。上昇前のチェックポイントも実機評価する |
| 損失が上下しながら緩やかに下がる | ミニバッチ訓練では正常である |
| `loss=NaN`、`grad_norm=NaN`、損失が急増し続ける | 中止し、データと学習率を確認する |
| `data_s` が `updt_s` より大幅に長い | 動画デコードや DataLoader がボトルネックである可能性がある |
| 損失は低いが実機が動かない | カメラ名・配置、キャリブレーション、静止フレーム、行動データを先に確認する |

## 早期終了の判断方法

LeRobot v0.6.0 の `lerobot-train` には、一定期間改善しなければ自動停止する機能がない。ACT は損失が横ばいになった後も実機動作が改善する場合があるため、`train/loss` だけを見て停止しない。

次の順で判断する。

1. まず 30,000 step を上限として、5,000 step ごとにチェックポイントを残す。
2. `eval/eval_loss` が最も低い付近、その前、その後の 3 チェックポイントを候補にする。
3. 各候補を同じ初期条件で 10 回ずつ実機評価する。
4. 成功率が同じなら、衝突、振動、停止が少なく、完了時間が安定した方を選ぶ。
5. 最後のチェックポイントまで成功率が改善している場合だけ、50,000〜80,000 step へ延長する。

次の場合は、30,000 step より前でも停止してよい。

- `NaN` や勾配の発散が発生した
- 間違ったデータセット、カメラ、タスクを指定した
- 動画破損や大量の静止フレームなど、訓練前に直すべきデータ問題が分かった
- 検証損失が複数の保存間隔にわたって悪化し、以前のチェックポイントの実機成功率が高いことも確認できた
- 既に目標成功率を満たすチェックポイントがあり、追加訓練で改善しないことを実機比較できた

手動で早期終了するときは、Colab ログにチェックポイント保存が表示され、Hub の `checkpoints/NNNNNN` を確認してからセルの停止ボタンを押す。保存中に停止すると、そのチェックポイントが不完全になる可能性がある。最後に正常保存された番号から再開する。

## モデルを Hugging Face Hub へアップロードする

前の訓練コマンドでは、`--policy.repo_id` と `--save_checkpoint_to_hub=true` を指定している。そのため次の 2 種類が自動アップロードされる。

- 訓練中: `checkpoints/005000`、`checkpoints/010000` のような再開用チェックポイント
- 正常終了時: `MODEL_REPO_ID` のルートに、推論用の最終モデル、processor、`train_config.json`、モデルカード

ブラウザで次を開き、`config.json`、`model.safetensors`、`train_config.json` と `checkpoints/` があることを確認する。

```text
https://huggingface.co/<MODEL_REPO_ID>
```

正常終了したのに推論用モデルがルートへ見当たらない場合は、Colab の最後のローカルチェックポイントを手動でアップロードする。

```python
!hf upload {MODEL_REPO_ID} {OUTPUT_DIR}/checkpoints/last/pretrained_model
```

このコマンドは `last/pretrained_model` の推論用ファイルをモデルリポジトリのルートへ送る。`training_state` だけをアップロードしても実機推論には使えない。

モデルカードには、訓練データセット、タスク、カメラ、step、バッチサイズ、採用したチェックポイント、実機成功率、既知の失敗条件を記録する。非公開モデルでも省略しない。

## 途中の訓練を再開する

`--policy.path` は重みだけを読み込み、新しい訓練を始めるときに使う。中断地点から続けるには、重みに加えて、重みの更新に使う内部状態（optimizer）、学習率、step も戻す必要がある。そのため、`--config_path` と `--resume=true` を使う。

### 同じ Colab ランタイムで再開する

ローカルの `last` チェックポイントが残っている場合は、次を実行する。

```python
!lerobot-train \
  --config_path={OUTPUT_DIR}/checkpoints/last/pretrained_model/train_config.json \
  --resume=true
```

中断前の設定にある合計 `steps` まで再開する。例えば 30,000 step の設定を 18,000 step 付近で中断し、最後の保存が 15,000 step なら、15,000 step から 30,000 step まで進む。

### 新しい Colab ランタイムで Hub から再開する

Colab が切断された場合は、ステップ 1〜3 を実行して LeRobot v0.6.0、Hugging Face、W&B を準備し、設定セルで同じ `MODEL_REPO_ID` を設定する。その後、次を実行する。

```python
!lerobot-train \
  --config_path={MODEL_REPO_ID} \
  --resume=true
```

Hub の `checkpoints/` から最新のチェックポイントが取得される。W&B のログも、保存済みの実験へ続けて記録される。

### 合計 step を延長して再開する

30,000 step 完了後に 50,000 step まで延長する例は次のとおりである。

```python
!lerobot-train \
  --config_path={MODEL_REPO_ID} \
  --resume=true \
  --steps=50000
```

`--steps=50000` は「さらに 50,000 step」ではなく、開始時から数えた **合計 50,000 step** である。30,000 step から再開した場合は、残り 20,000 step が実行される。

再開時は、バッチサイズ、データセット、モデル構造、`chunk_size` を変えない。変更すると、保存済みの訓練状態と整合しなくなる。設定を変える場合は、新しい `RUN_NAME` と `MODEL_REPO_ID` で訓練し直す。

## 訓練パラメータの適切な値を探す

### 基本方針

適切な値はタスク、データ量、動作時間、カメラ、GPU に依存する。最初から大規模な探索をせず、次の順で 1 項目ずつ比較する。

1. デフォルトに近い基準実験を作る。
2. 同じ訓練・検証分割、同じ seed、同じ評価初期位置を使う。
3. 5,000 step ごとの候補を実機評価し、必要な訓練時間を決める。
4. GPU メモリに余裕がある場合だけバッチサイズを比較する。
5. それでも学習が不安定な場合だけ学習率を比較する。
6. タスクの時間構造に明確な理由がある場合だけ `chunk_size` を比較する。
7. 最後に別 seed でも結果が再現するか確認する。

すべての実験を同じ `WANDB_PROJECT` に入れ、`RUN_NAME` に変更値を含める。例えば `act-pick-v1-bs8-lr1e-5` とする。W&B Sweeps でも自動探索できるが、実機評価には時間がかかる。まずは 2〜3 候補を手動で比較する。

### 初回の推奨範囲

| パラメータ | 最初の値 | 次に試す候補 | 判断方法 |
| --- | ---: | ---: | --- |
| `steps` | `30000` | `50000`、`80000` | 5,000 step ごとの実機成功率がまだ改善する場合だけ延長する |
| `batch_size` | `8` | `16`、メモリ不足時は `4` | 同じ epoch 数で検証損失と実機成功率を比較する |
| `policy.optimizer_lr` | `1e-5` | `5e-6`、`2e-5` | 発散なら下げ、学習が極端に遅い場合だけ上げる |
| `policy.chunk_size` | `100` | `50` | タスクのまとまった動作時間と実機の反応性を比較する |
| `policy.n_action_steps` | `100` | chunk と同じ `50` | 必ず `chunk_size` 以下にする |
| `policy.kl_weight` | `10` | 原則変更しない | ACT の基準値であり、他を検証してから扱う |
| `seed` | `1000` | `1001`、`1002` | 同じ設定を 2〜3 回行い、偶然の良い結果でないか確認する |

30 FPS では `chunk_size=100` は約 3.3 秒、`50` は約 1.7 秒分の将来行動に相当する。ACT の元研究は 50 Hz、chunk 90 で実機タスクを評価しているが、SO-101 の最適値を保証するものではない。

まず LeRobot のデフォルト 100 を使う。長い停止や環境変化への反応の遅さが見られた場合だけ、50 と比較する。

学習率を変更する実験の追加引数:

```text
--policy.optimizer_lr=5e-6
```

chunk を 50 にする実験の追加引数:

```text
--policy.chunk_size=50 --policy.n_action_steps=50
```

### GPU メモリが足りない場合

`CUDA out of memory` が出た場合は、バッチサイズを 8 から 4 に下げる。4 サンプル分の勾配を 2 回蓄積すれば、実効バッチサイズを 8 に保てる。

```text
--batch_size=4 --accelerator.gradient_accumulation.steps=2
```

この場合も `steps` はデータを読む micro-batch の回数である。epoch 数を合わせるには、`steps_per_epoch = ceil(total_frames / 4)` で計算し直す。

### 公平な実験表を作る

各実験について次を記録する。

| 実験 | batch | LR | chunk | steps | seed | 最良の検証 step | 実機成功数 / 10 | 備考 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| baseline | 8 | 1e-5 | 100 | 30000 | 1000 | 20000 | 7 / 10 | 右奥で失敗 2 回 |
| longer | 8 | 1e-5 | 100 | 50000 | 1000 | 40000 | 8 / 10 | 動作が滑らか |

比較する 10 回は、[`dataset.md`](./dataset.md) で決めた 5 つの開始位置から各 2 回など、毎回同じ条件にする。訓練データとまったく同じ固定位置だけでなく、想定する利用範囲内の未収録位置も分けて記録する。

## ローカル PC の SO-101 で評価する

### 評価前の準備

1. ローカル PC で `conda activate lerobot` を実行する。
2. `hf auth login` を実行し、非公開モデルを読める Hugging Face アカウントでログインする。
3. フォロワー、電源、USB、カメラを接続する。
4. [`dataset.md`](./dataset.md) と同じフォロワー ID、カメラ名、解像度、FPS を使う。
5. 対象物以外を作業台から片付け、非常停止できる位置にいる。
6. 最初は対象物を置かず、短時間の動作で安全を確認する。

Hub のモデルは `--policy.path` を指定すると自動ダウンロードされる。ローカルへモデルフォルダを手作業でコピーする必要はない。

以下のコマンドは、[`dataset.md`](./dataset.md) と同じ `front` カメラ 1 台用である。複数のカメラで収録した場合は、同じ名前と台数ですべて指定する。1 台でも欠けると、モデルの入力と一致しない。

macOS の例:

```bash
conda activate lerobot
hf auth login

export MODEL_REPO_ID="my-hf-name/act-so101-pick-red-cube-v1"
export HF_USER="my-hf-name"
export POLICY_DEVICE="mps"
export CAMERA_INDEX=0
export CAMERA_WIDTH=640
export CAMERA_HEIGHT=480
export DATASET_FPS=30
export TASK_DESCRIPTION="赤い立方体を右側の箱へ入れる"
```

Apple Silicon では `POLICY_DEVICE=mps`、Intel Mac または MPS で問題が出る場合は `cpu` を使う。

Windows（Miniforge Prompt）の例:

```bat
conda activate lerobot
hf auth login

set MODEL_REPO_ID=my-hf-name/act-so101-pick-red-cube-v1
set HF_USER=my-hf-name
set POLICY_DEVICE=cpu
set CAMERA_INDEX=0
set CAMERA_WIDTH=640
set CAMERA_HEIGHT=480
set DATASET_FPS=30
set TASK_DESCRIPTION=赤い立方体を右側の箱へ入れる
```

NVIDIA GPU と CUDA 対応 PyTorch をローカル環境へ別途導入済みの場合だけ `POLICY_DEVICE=cuda` を使う。

### 短時間の安全確認をする

対象物を置かず、低リスクな初期姿勢で 10 秒だけ実行する。次は macOS の例である。

```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path="$MODEL_REPO_ID" \
  --device="$POLICY_DEVICE" \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id=my_follower \
  --robot.cameras="{front: {type: opencv, index_or_path: $CAMERA_INDEX, width: $CAMERA_WIDTH, height: $CAMERA_HEIGHT, fps: $DATASET_FPS}}" \
  --task="$TASK_DESCRIPTION" \
  --fps="$DATASET_FPS" \
  --duration=10 \
  --display_data=true
```

異常な方向へ動く、カメラ不一致エラーが出る、激しく振動する場合は直ちに `Ctrl-C` で止める。モデルの再訓練より先に、カメラ名、アーム ID、キャリブレーション、収録時との差を確認する。

### 10 エピソードの選別評価を記録する

短時間の確認に問題がなければ、評価用データセットを作りながら 10 回実行する。評価リポジトリには `eval_` 接頭辞を付け、訓練データと混ぜない。

macOS:

```bash
export EVAL_REPO_ID="$HF_USER/eval_act-so101-pick-red-cube-v1"

lerobot-rollout \
  --strategy.type=episodic \
  --policy.path="$MODEL_REPO_ID" \
  --device="$POLICY_DEVICE" \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id=my_follower \
  --robot.cameras="{front: {type: opencv, index_or_path: $CAMERA_INDEX, width: $CAMERA_WIDTH, height: $CAMERA_HEIGHT, fps: $DATASET_FPS}}" \
  --dataset.repo_id="$EVAL_REPO_ID" \
  --dataset.no_stamp=true \
  --dataset.private=true \
  --dataset.num_episodes=10 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=10 \
  --dataset.single_task="$TASK_DESCRIPTION" \
  --fps="$DATASET_FPS" \
  --display_data=true
```

Windows（Miniforge Prompt）:

```bat
set EVAL_REPO_ID=%HF_USER%/eval_act-so101-pick-red-cube-v1

lerobot-rollout ^
  --strategy.type=episodic ^
  --policy.path="%MODEL_REPO_ID%" ^
  --device="%POLICY_DEVICE%" ^
  --robot.type=so101_follower ^
  --robot.port="%FOLLOWER_PORT%" ^
  --robot.id=my_follower ^
  --robot.cameras="{front: {type: opencv, index_or_path: %CAMERA_INDEX%, width: %CAMERA_WIDTH%, height: %CAMERA_HEIGHT%, fps: %DATASET_FPS%}}" ^
  --dataset.repo_id="%EVAL_REPO_ID%" ^
  --dataset.no_stamp=true ^
  --dataset.private=true ^
  --dataset.num_episodes=10 ^
  --dataset.episode_time_s=30 ^
  --dataset.reset_time_s=10 ^
  --dataset.single_task="%TASK_DESCRIPTION%" ^
  --fps="%DATASET_FPS%" ^
  --display_data=true
```

評価中のキー操作は次のとおりである。

| キー | 操作 |
| --- | --- |
| `→` | 現在のエピソードまたはリセットを早く終了する |
| `←` | 現在の評価エピソードを破棄してやり直す |
| `ESC` | 評価セッションを終了する |

### 成功条件と工程別到達率を記録する

実機評価では、最終的なタスク成功を定量評価の主指標にし、途中まで進んだ試行も工程別到達率として定量化する。「あと少し」「かすめた」のような自由記述だけでモデルを比較すると、評価者や試行ごとに判定が変わるためである。

まず、制限時間、物体の最終状態、安定している時間を含む成功条件を評価前に固定する。この資料の例では次の条件をすべて満たした場合だけ成功とする。

```text
30 秒以内に立方体全体が箱の中へ入り、
グリッパーが立方体を離し、その状態で 2 秒以上とどまる
```

立方体を掴めても箱へ入れられなければ、最終成功は `0` である。その代わり、各エピソードで到達した工程を `1`、到達しなかった工程を `0` として記録する。距離や保持時間のしきい値は例なので、使用する物体とタスクに合わせて評価開始前に決め、途中で変更しない。

| 工程 | `1` と判定する例 |
| --- | --- |
| 接触 | グリッパーが立方体に触れた |
| 把持 | 立方体を落とさず 1 秒以上保持した |
| 持ち上げ | 立方体の底面が作業台から 2 cm 以上離れ、その状態を 1 秒以上保った |
| 運搬 | 把持した立方体を箱の上まで運んだ |
| 配置 | 立方体全体が箱の中へ入った |
| 最終成功 | リリース後も立方体が箱内に 2 秒以上とどまった |

「かすめた」は接触 `1`、把持 `0` として扱う。「箱の直前まで運んだ」のような「あと少し」は運搬 `1`、配置 `0` として扱う。これにより、どの工程がボトルネックかをモデル間で比較できる。

各エピソードについて次の表を記録する。`lerobot-rollout` の episodic 方式は映像とロボット状態を保存するが、成功・失敗は自動判定しない。実行中または保存した動画を見て採点する。

| episode | 開始位置 | 接触 | 把持 | 持ち上げ | 運搬 | 配置 | 最終成功 | 完了秒 | 失敗分類・備考 |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 0 | 中央 | 1 | 1 | 1 | 1 | 1 | 1 | 18.4 | なし |
| 1 | 右奥 | 1 | 1 | 1 | 1 | 0 | 0 | - | 箱の右へ外した |
| 2 | 左手前 | 1 | 0 | 0 | 0 | 0 | 0 | - | 接触後に空振りした |

集計には次を使う。

```text
成功率 (%) = 成功エピソード数 / 全評価エピソード数 × 100
工程別到達率 (%) = その工程へ到達したエピソード数 / 全評価エピソード数 × 100
```

失敗分類には、`把持前の空振り`、`把持後の落下`、`配置位置のずれ`、`リリース失敗`、`途中停止`、`衝突・振動`、`時間切れ` など、観察できる事実を記録する。自由記述は、失敗データを追加収録するときの診断に使い、最終成功率の代わりにはしない。

人が途中で補助した試行、安全のために停止した試行、時間切れは失敗として分母に含める。カメラ切断、USB 切断、評価開始前の配置ミスなど、モデルと無関係な異常が起きた試行だけを無効としてやり直す。`←` キーはこの無効試行にだけ使い、モデル自身の失敗を破棄してはならない。

10 回は途中チェックポイントを絞るための選別評価である。10 回中 8 回なら観測成功率は 80% だが、1 回の差が 10 ポイントになる。8/10 と 7/10 だけで優劣を確定しない。

最終候補は、ACT 原論文の実機評価と同じく 25 回以上実行する。25 回なら 1 回が 4 ポイントになる。最終評価では `--dataset.num_episodes=25` と新しい `EVAL_REPO_ID` を使い、開始位置別の成功率も記録する。

ACT 原論文でも、最終成功率に加えて `Grasp`、`Place`、`Insert` などの工程別成功率を報告し、実機タスクは各モデル 25 回評価している。この資料も同じ考え方を採用する。ただし、工程と成功条件は SO-101 で行うタスクに合わせて定義する。

### 途中チェックポイントをローカルで比較する

Hub の `checkpoints/020000/pretrained_model` を明示的に取得する例は次のとおりである。`020000` は実在する番号へ置き換える。

macOS:

```bash
hf download "$MODEL_REPO_ID" \
  --include "checkpoints/020000/pretrained_model/*" \
  --local-dir "$HOME/lerobot-models/act-checkpoints"
```

Windows（Miniforge Prompt）:

```bat
hf download "%MODEL_REPO_ID%" ^
  --include "checkpoints/020000/pretrained_model/*" ^
  --local-dir "%USERPROFILE%\lerobot-models\act-checkpoints"
```

評価時の `--policy.path` は、macOS では次を使う。

```text
--policy.path=$HOME/lerobot-models/act-checkpoints/checkpoints/020000/pretrained_model
```

Windows では次を使う。

```text
--policy.path=%USERPROFILE%\lerobot-models\act-checkpoints\checkpoints\020000\pretrained_model
```

各チェックポイントで異なる `EVAL_REPO_ID` を使い、同じ 10 通りの初期条件で比較する。

## 完了チェックリスト

- [ ] Colab の GPU 名と VRAM を確認した
- [ ] LeRobot v0.6.0 を `training` 追加依存関係付きでインストールした
- [ ] Hugging Face の Write トークンを `HF_TOKEN` として Colab Secrets に保存し、コードへ書かずにログインした
- [ ] W&B アカウントを作成し、API キーを `WANDB_API_KEY` として Colab Secrets に保存してログインした
- [ ] Hub からデータセットのエピソード数、フレーム数、FPS、カメラ名を確認した
- [ ] 初回はバッチサイズ 8、30,000 step、ACT のその他のデフォルトを基準にした
- [ ] 検証エピソードを取り分け、W&B で `train/loss` と `eval/eval_loss` を確認した
- [ ] 5,000 step ごとのチェックポイントが Hub にある
- [ ] 完成モデルのルートに `config.json`、`model.safetensors`、processor、`train_config.json` がある
- [ ] Hub からの再開コマンドを確認した
- [ ] 収録時と同じカメラ名、解像度、FPS、キャリブレーションで評価した
- [ ] 候補ごとに 10 回の選別評価を行い、最終候補は 25 回以上の成功数、工程別到達率、失敗分類、開始位置を記録した
- [ ] 訓練損失だけでなく実機成功率で採用チェックポイントを決めた
- [ ] モデルカードへ訓練条件と評価結果を記載した

## トラブルシューティング

| 状況 | 確認・対処 |
| --- | --- |
| Colab で `torch.cuda.is_available()` が `False` | 「ランタイムのタイプを変更」で GPU を選び、ランタイムへ再接続する。|
| `destination path 'lerobot' already exists` | clone セルを再実行している。既にインストール済みなら次へ進み、環境が壊れている場合はランタイムを削除してやり直す。|
| `userdata.get("HF_TOKEN")` が失敗する | シークレット名が大文字の `HF_TOKEN` か確認し、Colab のシークレット画面で **ノートブックからのアクセス**を有効にする。|
| Hub のデータセットが `401` または `403` | Colab Secrets の `HF_TOKEN` が有効な Write 権限トークンか確認し、ステップ 3-1 のログインセルを再実行する。`DATASET_REPO_ID` の所有者名も確認する。|
| `Repository Not Found` | データセット用 URL とモデル用 URL を混同していないか、`所有者名/リポジトリ名` の綴りと非公開リポジトリへの権限を確認する。|
| `userdata.get("WANDB_API_KEY")` が失敗する | シークレット名が大文字の `WANDB_API_KEY` か確認し、Colab のシークレット画面で **ノートブックからのアクセス**を有効にする。|
| W&B に実験が作られない | ステップ 3-2 のログインセルを再実行し、訓練コマンドの `--wandb.enable=true` と `--wandb.project` を確認する。|
| `CUDA out of memory` | `--batch_size=4 --accelerator.gradient_accumulation.steps=2` にする。ランタイム内の不要な Python セッションがあれば再起動する。|
| `eval_steps > 0 requires dataset.eval_split > 0.0` | `--dataset.eval_split=0.1` を付ける。検証を行わない場合だけ `--eval_steps=0` にする。|
| 訓練が Colab 切断で止まった | 新しいランタイムでインストールと両サービスへのログインを行い、`--config_path=$MODEL_REPO_ID --resume=true` で Hub の最新チェックポイントから再開する。|
| 再開してもすぐ終了する | 保存済み step が設定の合計 `steps` に達している。延長するなら `--steps=50000` のように、より大きい合計値を指定する。|
| Hub に `checkpoints/` はあるがルートにモデルがない | 訓練が正常終了前に止まっている。再開して完了させるか、最後のローカル `pretrained_model` を `hf upload` する。|
| `Output directory ... already exists` | 新しい実験なら `RUN_NAME` と `OUTPUT_DIR` を変える。続きなら新規訓練コマンドではなく `--resume=true` を使う。|
| ローカル評価でカメラの入力項目が一致しない | 収録時と同じカメラ名（例: `front`）を指定する。カメラ番号が同じでも、名前が異なると別の入力として扱われる。|
| ローカル評価でモデルが取得できない | `hf auth whoami` を確認する。非公開モデルには同じアカウントの認証が必要である。|
| 損失は下がったが実機で動かない | 静止フレームの割合、`observation.state` と `action` の変化、カメラ位置、キャリブレーション、タスクの一貫性を確認する。step やモデルサイズを先に増やさない。|
| 実機動作が途中で止まる、ぎこちない | 複数チェックポイントを実機比較する。ACT 公式の指摘どおり、損失が横ばいでも長く訓練すると改善する場合がある。ただしデータに長い静止がないかも確認する。|

## 参考にしたファイル・URL

### このリポジトリ

- [`dataset.md`](./dataset.md): SO-101 の訓練データ収録、Hub 保存、品質確認
- [`install.md`](./install.md): ローカル PC の LeRobot 環境
- [`setup.md`](./setup.md): SO-101、カメラ、ポート、キャリブレーション
- [`lerobot/docs/source/act.mdx`](./lerobot/docs/source/act.mdx): ACT の概要、訓練コマンド、初期値
- [`lerobot/docs/source/il_robots.mdx`](./lerobot/docs/source/il_robots.mdx): 訓練、チェックポイント再開、Hub アップロード、実機評価
- [`lerobot/docs/source/notebooks.mdx`](./lerobot/docs/source/notebooks.mdx): LeRobot 公式 ACT Colab ノートブック
- [`lerobot/docs/source/hardware_guide.mdx`](./lerobot/docs/source/hardware_guide.mdx): GPU メモリ、epoch と step、訓練時間の目安
- [`lerobot/docs/source/inference.mdx`](./lerobot/docs/source/inference.mdx): `lerobot-rollout` の base と episodic 評価
- [`lerobot/src/lerobot/configs/train.py`](./lerobot/src/lerobot/configs/train.py): `steps`、検証、保存、再開の設定
- [`lerobot/src/lerobot/datasets/factory.py`](./lerobot/src/lerobot/datasets/factory.py): 訓練・検証エピソードの分割方法
- [`lerobot/src/lerobot/policies/act/configuration_act.py`](./lerobot/src/lerobot/policies/act/configuration_act.py): ACT のデフォルト構造、学習率、chunk、KL weight
- [`lerobot/src/lerobot/scripts/lerobot_train.py`](./lerobot/src/lerobot/scripts/lerobot_train.py): W&B の記録、検証損失、チェックポイント、最終アップロードの処理

### 公式・一次資料

- [LeRobot: ACT](https://huggingface.co/docs/lerobot/v0.6.0/act): ACT の公式説明と訓練例
- [LeRobot: Imitation Learning on Real-World Robots](https://huggingface.co/docs/lerobot/v0.6.0/il_robots): データセット訓練、W&B、再開、アップロード、実機評価
- [LeRobot 公式 ACT Colab ノートブック](https://github.com/huggingface/notebooks/blob/main/lerobot/training-act.ipynb): Colab での公式訓練例
- [Hugging Face Hub: Quickstart](https://huggingface.co/docs/huggingface_hub/quick-start): Colab Secrets の `HF_TOKEN` と安全な認証方法
- [Hugging Face Hub: Authentication](https://huggingface.co/docs/huggingface_hub/guides/cli#hf-auth-login): Hub のログイン
- [Hugging Face Hub: Download files](https://huggingface.co/docs/huggingface_hub/guides/download): `hf download` と `--include`、`--local-dir`
- [Hugging Face Hub: Upload files](https://huggingface.co/docs/huggingface_hub/guides/cli#hf-upload): `hf upload`
- [W&B Quickstart](https://docs.wandb.ai/models/quickstart): アカウント、API キー、ログイン、実験の確認
- [W&B Python SDK: `wandb.login()`](https://docs.wandb.ai/models/ref/python/functions/login): `key` と `verify` を使ったプログラムからの認証
- [W&B Sweeps](https://docs.wandb.ai/models/sweeps): ハイパーパラメータ探索の公式機能
- [Google Colab FAQ](https://research.google.com/colaboratory/faq.html): GPU・利用上限・仮想マシンの寿命と切断
- [Learning Fine-Grained Bimanual Manipulation with Low-Cost Hardware](https://arxiv.org/html/2304.13705#S5): ACT の原論文。最終成功率、工程別成功率、実機 25 試行の評価方法
- [ACT 著者の公式 GitHub](https://github.com/tonyzhaozh/act): 基準パラメータと、損失の plateau 後も訓練を続ける場合の tuning tips
- [ALOHA / ACT 公式プロジェクトページ](https://tonyzhaozh.github.io/aloha/): 実機デモ数、制御周波数、chunk size、成功率の一次情報
