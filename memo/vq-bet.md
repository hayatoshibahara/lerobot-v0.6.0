# VQ-BeT 入門：ACT との違い、セットアップ、注意点

VQ-BeT（Vector-Quantized Behavior Transformer）は、実演データからロボットの行動を学ぶ**模倣学習**の手法である。ACT と同じく画像と関節状態からアームの関節操作を出力するが、「あり得る動作の選択肢」を表す方法が異なる。

この資料では、SO-ARM 101 で収録したデータを Google Colab で学習するケースを扱う。最初のモデルは引き続き ACT を推奨する。VQ-BeT は、ACT の結果と比較して「複数の自然な手順があるタスク」で試す 2 本目の候補である。

## 先に結論

- **最初の一周**: ACT を使う。複数カメラにも対応し、設定が少ない。
- **VQ-BeT を試す場面**: 同じ開始状態でも「右からつかむ／左からつかむ」のように、複数の成功手順がデータに含まれる場合。
- **Colab では GPU を使う**: VQ-BeT は CPU 学習には向かず、Apple Silicon の MPS も LeRobot v0.6 系では未対応である。
- **カメラは 1 台にする**: 現在の LeRobot 実装の VQ-BeT は、入力画像を 1 台分だけ受け付ける。

## ACT と VQ-BeT の違い

| 観点 | ACT | VQ-BeT |
| --- | --- | --- |
| 行動の作り方 | 観測から連続値の行動チャンクを直接回帰する | 行動チャンクを「離散的な行動トークン＋細かな補正」として生成する |
| 行動の曖昧さへの対応 | VAE の潜在変数を使うが、基本は一つの滑らかな予測へ寄りやすい | 複数の行動モードをコードとして選ぶため、異なる自然な手順を表しやすい |
| 時系列の入力 | 現在の観測 1 ステップ | 直近 5 ステップの観測（デフォルト） |
| カメラ | 同じ解像度なら複数台を使える | **1 台のみ** |
| 初期設定 | 少ない。LeRobot の最初の推奨モデル | 2 段階学習、コードブック、履歴長などを理解する必要がある |
| 計算資源の目安 | 軽量 | 軽量。ACT と同じく概ね 2–6 GB VRAM が目安 |

ACT は、画像・関節状態・（学習時は）正解の行動チャンクから VAE の潜在変数を学び、Transformer が連続値の行動チャンクを出力する。VQ-BeT は、連続値の行動チャンクをいったん学習済みの「行動辞書」の組合せへ変換してから、GPT 型 Transformer でその組合せを選ぶ。

```text
ACT
画像・関節状態 ──> Transformer ──> 連続値の行動チャンク

VQ-BeT
行動チャンク ──> Residual VQ-VAE ──> 行動コード（学習の第 1 段階）
画像・関節状態の履歴 ──> GPT ──> 行動コード + 連続値の補正 ──> 行動チャンク
                                         （学習の第 2 段階）
```

### VQ-BeT のアーキテクチャ

VQ-BeT は次の 2 段階で学習する。

1. **行動を圧縮する**。Residual VQ-VAE が、数ステップ分の連続値の行動をコードブックのベクトルの組合せに置き換え、元の行動を復元できるように学習する。第 1 コードは大まかな動作、第 2 コードは細かな違いを表す。
2. **コードを予測する**。直近の画像と関節状態を入力した GPT 型 Transformer が、どの行動コードを選ぶかと、そのコードからの連続値のずれ（offset）を予測する。

コードだけでは量子化による粗さが残るため、offset ヘッドで連続値の補正も出力する。この「離散的な選択」と「連続的な補正」の組合せが、複数の動作パターンを扱いつつロボットを細かく制御するための要点である。論文では 2 層の Residual VQ、コード予測、offset の各要素が重要だと報告している。[VQ-BeT 論文（arXiv）](https://arxiv.org/abs/2403.03181)

> [!NOTE]
> 論文の比較結果は、別のロボット・タスク・データセットで得られたものである。SO-ARM 101 で ACT より必ず成功率が上がることを意味しない。同じ収録データ、同じ評価回数で比較する。

## VQ-BeT が向くタスク・向かないタスク

### 向く可能性がある

- 物体への近づき方や把持の向きに複数の成功パターンがある。
- 複数工程のタスクで、途中から異なる手順へ分岐し得る。
- Diffusion Policy のような反復的な生成より、速い推論を試したい。

### 最初は ACT のままにする

- 物体位置も手順も固定した、初回の単一タスク。
- 2 台以上のカメラを使いたい。
- まず収録、学習、実機評価の問題を一つずつ切り分けたい。
- デモの質が不安定で、失敗や迷いが多い。モデル変更より先にデータを改善する。

## データ収録の条件

VQ-BeT 用に新しいデータ形式は不要である。LeRobotDataset 形式で、次を満たすデータセットを使う。

- `observation.state`: SO-ARM 101 の関節状態
- `action`: 各時刻の関節操作
- `observation.image.*`: **ちょうど 1 台分**のカメラ画像

VQ-BeT のデフォルトは過去 5 ステップの観測を使う。録画中にフレームが大きく欠けたり、FPS がエピソードごとに違ったりすると、履歴から動きを読むことが難しくなる。まずは 640×480・30 FPS・固定カメラ 1 台で、明るさとカメラ位置を固定する。

カメラを 2 台で収録済みの場合も、VQ-BeT の学習時は 1 台だけを入力に選ぶ必要がある。VQ-BeT の制限であり、データが壊れているわけではない。複数視点を使う比較では ACT を使う。

## Google Colab のセットアップ

ローカル PC の `install.md` にあるインストールへ `[training]` を追加する必要はない。学習環境である Colab にだけ入れる。

### 1. GPU ランタイムを選ぶ

Colab のメニューで「ランタイム」→「ランタイムのタイプを変更」→「ハードウェア アクセラレータ」を **GPU** に設定する。その後、次を実行する。

```python
import torch
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else "GPU が見つからない")
```

`True` と GPU 名が表示されることを確認する。`False` のまま学習を始めない。

### 2. LeRobot をインストールする

次を Colab のセルで実行する。VQ-BeT 自体に専用の追加パッケージは不要で、`training` extra にデータセットと学習に必要な依存関係が含まれる。

```python
!pip install "lerobot[training]"
```

ワークショップで使用している LeRobot と Colab の LeRobot のバージョンを合わせたい場合は、ワークショップで使うリリース番号を指定する。

```python
!pip install "lerobot[training]==<ワークショップで使うバージョン>"
```

このリポジトリで確認する場合は、次のコマンドでバージョンを確認できる。

```python
!lerobot-info
```

### 3. Hugging Face にログインする

収録済みデータセットの取得と学習済みモデルの保存に Hugging Face を使う。Hugging Face の Settings → Access Tokens で **Write** 権限のトークンを作成し、Colab のセルで実行する。

```python
!hf auth login
```

トークンはノートブックに直接書かない。入力を求められたときだけ貼り付ける。

### 4. 学習を実行する

`<HFユーザー名>`、`<データセット名>`、`<モデル名>` は置き換える。最初はデフォルトの 100,000 step を使う。VQ-BeT は最初の 20,000 step を行動コードブックの学習に使うため、これより短い試行では本体の Transformer 学習が始まらない、またはほとんど進まない。

```python
!lerobot-train \
  --dataset.repo_id=<HFユーザー名>/<データセット名> \
  --policy.type=vqbet \
  --policy.device=cuda \
  --output_dir=outputs/train/vqbet_<モデル名> \
  --job_name=vqbet_<モデル名> \
  --batch_size=8 \
  --steps=100000 \
  --wandb.enable=false \
  --save_checkpoint_to_hub=true \
  --policy.repo_id=<HFユーザー名>/vqbet_<モデル名>
```

GPU メモリ不足が起きたら、まず `batch_size` を 4、次に 2 へ下げる。モデルの構造やコードブックの大きさは、初回比較では変更しない。

### 5. 学習済みモデルを確認・保存する

学習の出力は Colab の一時ストレージに保存される。上のコマンドでは `--policy.repo_id` により、終了時にモデルと前後処理の設定が Hugging Face Hub へ保存される。`--save_checkpoint_to_hub=true` により、定期チェックポイントも Hub へ保存されるため、Colab が切断しても再開しやすい。

学習が終わったら、Hub 上のモデルに `config.json`、`model.safetensors`、前処理・後処理の設定があることを確認する。

## 実機で評価する

評価前に、ローカル PC の LeRobot が Colab と同じ主要バージョンであることを確認する。評価は、アームを低速で動かせる状態にして、周囲に手を入れず、すぐ電源を切れる状態で始める。

ACT の評価時と同じように `lerobot-record` でポリシーを指定する。カメラ名・解像度・FPS は学習データのものと合わせる。また VQ-BeT は 1 台分のカメラだけを指定する。

```bash
lerobot-record \
  --robot.type=so101_follower --robot.port=<フォロワーのポート> --robot.id=<フォロワーID> \
  --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
  --dataset.repo_id=<HFユーザー名>/eval_vqbet_<モデル名> \
  --dataset.single_task="<学習時と同じタスク説明>" \
  --dataset.num_episodes=10 \
  --policy.path=<HFユーザー名>/vqbet_<モデル名>
```

学習に使ったカメラが `front` 以外なら、同じ名前と設定を使う。ACT と VQ-BeT を比較する際は、同じ初期位置・物体配置・評価回数で成功数を記録する。

## 主要な懸念事項と対処

| 懸念事項 | 原因・影響 | 最初の対処 |
| --- | --- | --- |
| Apple Silicon の Mac で学習できない | LeRobot v0.6 系の VQ-BeT は MPS の `unique_dim` 演算をサポートしない | Colab の CUDA GPU を使う |
| カメラを 2 台指定するとエラーになる | VQ-BeT は LeRobot 実装で単一画像入力のみ対応 | 1 台だけを収録・学習・評価に使う。複数視点なら ACT を使う |
| 学習を短く切り上げても動かない | 初期 20,000 step は Residual VQ-VAE の行動コードブックを学習する段階 | まず 100,000 step を完走する |
| 出力が不自然・ばらつく | コードブックが行動を十分に復元できない、またはデモに迷い・失敗が混ざる | 収録映像を見直し、タスクを単純化して ACT と同じデータで比較する |
| GPU メモリ不足 | Colab の GPU 容量や他プロセスの使用量による | `--batch_size=4`、次に `--batch_size=2` を試す |
| 実行ごとに少し違う動作になる | コードの選択に確率サンプリングを使う | 同じ条件で複数回評価する。初回は温度などを変更しない |

学習ログでは、初期段階に `recon_l1_error`（行動チャンクの復元誤差）と使用されたコード数が出る。復元誤差が大きい、または使われるコードが極端に少ない場合は、コードブックがデータの動きを表現できていない可能性がある。この場合も、最初に変更すべきなのは複雑なハイパーパラメータではなく、デモの一貫性とタスクの難易度である。

## 初回比較の進め方

1. 固定カメラ 1 台・固定物体位置の単純なタスクを 50 エピソード程度収録する。
2. 同じデータセットで ACT を学習・評価し、成功率を基準値にする。
3. 同じデータセット、同じ 10 回の実機評価で VQ-BeT を試す。
4. VQ-BeT が改善しない場合は、まずデータ品質を直す。複数の手順を意図して収録したか、迷いのある操作を混ぜていないかを確認する。
5. 明確な複数モードがあるタスクでのみ、VQ-BeT の採用を検討する。

## 参考資料

- [VQ-BeT 論文: Behavior Generation with Latent Actions](https://arxiv.org/abs/2403.03181)
- [LeRobot の VQ-BeT 設定](./lerobot/src/lerobot/policies/vqbet/configuration_vqbet.py)
- [LeRobot の VQ-BeT 実装](./lerobot/src/lerobot/policies/vqbet/modeling_vqbet.py)
- [LeRobot のハードウェア目安](./lerobot/docs/source/hardware_guide.mdx)
- [SO-ARM 101 のワークフロー](./structure.md)
