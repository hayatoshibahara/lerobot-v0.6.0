# SO-101 の訓練用データセットを収録して Hugging Face Hub に保存する

## 目次

- [概要](#概要)
- [この資料で行うこと](#この資料で行うこと)
- [はじめる前に](#はじめる前に)
- [用語と設定値](#用語と設定値)
- [ステップ 1: 作業環境とロボットを確認する](#ステップ-1-作業環境とロボットを確認する)
- [ステップ 2: Hugging Face アカウントを準備してログインする](#ステップ-2-hugging-face-アカウントを準備してログインする)
- [ステップ 3: データセット名と保存先を決める](#ステップ-3-データセット名と保存先を決める)
- [ステップ 4: 少数エピソードで試験収録する](#ステップ-4-少数エピソードで試験収録する)
- [ステップ 5: Hub 上のデータを確認する](#ステップ-5-hub-上のデータを確認する)
- [ステップ 6: 本番データを収録する](#ステップ-6-本番データを収録する)
- [追加収録する](#追加収録する)
- [データセットカードを書く](#データセットカードを書く)
- [収録品質のチェックリスト](#収録品質のチェックリスト)
- [トラブルシューティング](#トラブルシューティング)
- [次に行うこと](#次に行うこと)
- [参考にしたファイル・URL](#参考にしたファイルurl)

## 概要

この資料では、SO-101 のテレオペレーションを記録し、ACT の訓練に使う LeRobotDataset を作る。関節状態、指令、カメラ映像は Hugging Face Hub のデータセットリポジトリへ保存する。

`lerobot-record` は収録終了時にデータセットを Hub へアップロードし、存在しないデータセットリポジトリも自動作成する。そのため、事前に必要なのは Hugging Face アカウントと、そのアカウントに書き込めるトークンである。Hub の画面で空のリポジトリを先に作る方法も使えるが、必須ではない。

最初に 3 エピソードを試験収録し、保存後の映像とアップロードを確認する。問題がなければ、1 つのタスクについて成功デモを 50 エピソード収録する。

## この資料で行うこと

1. 収録に使う SO-101、カメラ、作業環境を確認する。
2. Hugging Face アカウントを作成し、書き込み権限のトークンでログインする。
3. データセットの名前、非公開設定、ローカル保存先、タスク説明を決める。
4. 3 エピソードを収録して Hub への自動アップロードを確認する。
5. Hub 上で映像と記録内容を確認する。
6. 成功デモを 50 エピソード収録し、データセットカードを記入する。

## はじめる前に

- [`install.md`](./install.md) の手順を終え、`lerobot-info` が動くこと
- [`setup.md`](./setup.md) の手順を終え、SO-101 のテレオペレーションとカメラ映像を確認済みであること
- フォロワーとリーダーのポート名、アーム ID、2 台のカメラ番号が分かっていること
- 作業台、対象物、照明、カメラ位置を収録中に固定できること
- インターネット接続と Hugging Face アカウントを用意できること
- フォロワーの周囲に手指や障害物がなく、すぐに停止できること

収録中もフォロワーはリーダーに追従する。開始前に対象物以外を作業台から片付け、アームの可動範囲に手を入れない。異常な動作をした場合は、すぐに `Ctrl-C` で停止する。

公開リポジトリでは映像を含むデータが誰でも閲覧・ダウンロードできる。人物、顔、個人情報、社外秘の物、著作権上共有できない物が映る可能性がある間は、必ず非公開で収録・確認する。

## 用語と設定値

| 項目 | 意味 | この資料の例 |
| --- | --- | --- |
| `HF_USER` | Hugging Face のユーザー名または書き込み権限がある組織名 | `my-hf-name` |
| `DATASET_NAME` | データセットリポジトリの名前。小文字・数字・ハイフン中心で付ける | `so101-pick-red-cube-20260815` |
| `DATASET_REPO_ID` | Hub 上のリポジトリ ID | `my-hf-name/so101-pick-red-cube-20260815` |
| `DATASET_ROOT` | PC 上でデータを一時保存・追加収録するフォルダ | `~/lerobot-datasets/so101-pick-red-cube-20260815` |
| `TASK_DESCRIPTION` | 全エピソードで共通にする、短く具体的なタスク説明 | `赤い立方体を右側の箱へ入れる` |
| エピソード | 初期状態からタスク完了までの 1 回のデモ | 30 秒以内 |
| リセット時間 | エピソード間に物とアームを初期状態へ戻す時間 | 10 秒 |

`DATASET_REPO_ID` は `所有者名/リポジトリ名` の形式で指定する。`eval_` で始まるリポジトリ名は評価用として予約されているため、`lerobot-record` の収録データには使わない。

この資料では `--dataset.no_stamp=true` を指定し、決めた名前をそのままリポジトリ名に使う。同じ名前とローカル保存先で収録をやり直すと、新規収録は失敗する。別のデータセットを作るときは、日付などを含む新しい名前を使う。

## ステップ 1: 作業環境とロボットを確認する

新しいターミナル、または Windows の Miniforge Prompt を開き、LeRobot 環境を有効にする。

```bash
conda activate lerobot
```

SO-101 の収録コマンドと Hugging Face CLI が利用できることを確認する。

```bash
lerobot-record --help
hf --help
```

`lerobot-record` が見つからない場合は [`install.md`](./install.md) を確認する。`hf` が見つからない場合は LeRobot の環境が有効になっているか確認し、`huggingface_hub` を含む LeRobot のインストールをやり直す。

続けて、フォロワーの電源、リーダー、USB ケーブル、カメラを接続する。ポート名、アーム ID、カメラ設定は [`setup.md`](./setup.md) で確認したものを使う。異なるポートや異なるアーム ID を指定すると、キャリブレーション済みの設定を使えない。

この資料のコマンドでは次の値を使う。

```text
フォロワー: port=$FOLLOWER_PORT, id=my_follower
リーダー:   port=$LEADER_PORT,   id=my_leader
カメラ:     front=0, wrist=1, 640x480, 30 FPS
```

`$FOLLOWER_PORT` と `$LEADER_PORT` は、[`setup.md`](./setup.md) の「ステップ 2」で設定したシェル変数である。新しいターミナルを開いた場合は、実際のポート名を使ってもう一度設定する。

macOS:

```bash
export FOLLOWER_PORT="/dev/tty.usbmodemXXXXXXXX"
export LEADER_PORT="/dev/tty.usbmodemYYYYYYYY"
```

Windows（Miniforge Prompt）:

```bat
set FOLLOWER_PORT=COM3
set LEADER_PORT=COM4
```

## ステップ 2: Hugging Face アカウントを準備してログインする

### 2-1. アカウントと書き込みトークンを作成する

Hugging Face アカウントがない場合は、[Hugging Face の登録ページ](https://huggingface.co/join) で作成し、メールアドレスを確認する。次に [Access Tokens の設定ページ](https://huggingface.co/settings/tokens) を開き、用途が分かる名前（例: `so101-dataset-upload`）で **Write** 権限の User Access Token を作成する。

トークンは作成時にだけ表示される。ローカルでのデータセット収録と、Colab での訓練・モデル保存に同じトークンを利用できる。パスワードと同じように扱い、チャット、スクリーンショット、ソースコード、Git リポジトリへ貼り付けない。PC を共有する場合は、収録後に `hf auth logout` でログアウトする。

### 2-2. CLI へログインする

トークンをコマンドに直接書くとシェル履歴へ残るため、次を実行し、表示された入力欄へ貼り付ける。入力した文字は画面に表示されない。

```bash
hf auth login
```

ログイン中のユーザー名を確認する。

```bash
hf auth whoami
```

ユーザー名または組織名が表示されれば準備完了である。`401`、`403`、または書き込み権限のエラーが出る場合は、Write 権限のトークンでログインし直す。

### 2-3. 後続の Colab 訓練用にトークンを保存する

後の ACT 訓練でも使えるように、同じトークンを Colab Secrets に保存する。

1. [Google Colab](https://colab.research.google.com/) で訓練用ノートブックを開く。
2. 左サイドバーにある鍵アイコンの **シークレット** を開く。
3. **新しいシークレットを追加**し、名前を `HF_TOKEN`、値を `hf_` から始まるトークンにする。名前は大文字・小文字を区別する。
4. `HF_TOKEN` の **ノートブックからのアクセス**を有効にする。

トークンを `print()` で表示してはならない。訓練時の読み込み方は [`act.md` の「Hugging Face にログインする」](./act.md#3-1-hugging-face-にログインする) を参照する。トークンを再発行した場合は、Colab Secrets の値も更新する。

## ステップ 3: データセット名と保存先を決める

まず、収録するタスクと条件を決める。

| 決める項目 | 例 |
| --- | --- |
| タスク | 赤い立方体を右側の箱へ入れる |
| 成功条件 | 立方体が箱の中に入り、グリッパーが離れている |
| 初期配置 | 立方体は印を付けた 5 位置のいずれか、箱は右側に固定 |
| 固定する条件 | カメラ、箱、作業台、照明、操作手順 |

成功を映像だけで判断できる内容にする。途中で「持ち上げるだけ」「別の箱へ入れる」など、異なるタスクを混ぜない。

以下の値は例なので、`my-hf-name`、日付、タスク説明を実際の内容へ置き換える。`HF_USER` には、前のステップの `hf auth whoami` に表示されたユーザー名を設定する。

macOS:

```bash
export HF_USER="my-hf-name"
export DATASET_NAME="so101-pick-red-cube-20260815"
export DATASET_REPO_ID="$HF_USER/$DATASET_NAME"
export DATASET_ROOT="$HOME/lerobot-datasets/$DATASET_NAME"
export TASK_DESCRIPTION="赤い立方体を右側の箱へ入れる"
export FRONT_CAMERA_INDEX=0
export WRIST_CAMERA_INDEX=1
export CAMERA_WIDTH=640
export CAMERA_HEIGHT=480
export DATASET_FPS=30
```

Windows（Miniforge Prompt）:

```bat
set HF_USER=my-hf-name
set DATASET_NAME=so101-pick-red-cube-20260815
set DATASET_REPO_ID=%HF_USER%/%DATASET_NAME%
set DATASET_ROOT=%USERPROFILE%\lerobot-datasets\%DATASET_NAME%
set TASK_DESCRIPTION=赤い立方体を右側の箱へ入れる
set FRONT_CAMERA_INDEX=0
set WRIST_CAMERA_INDEX=1
set CAMERA_WIDTH=640
set CAMERA_HEIGHT=480
set DATASET_FPS=30
```

設定値を確認する。

macOS:

```bash
echo "$DATASET_REPO_ID"
echo "$DATASET_ROOT"
```

Windows（Miniforge Prompt）:

```bat
echo %DATASET_REPO_ID%
echo %DATASET_ROOT%
```

### Hub リポジトリについて

この後の `lerobot-record` に `--dataset.repo_id="$DATASET_REPO_ID"` を渡すと、アップロード時に同名の **データセットリポジトリ** が自動作成される。通常は Hub の画面で事前作成する必要はない。本資料ではこの自動作成を使う。

収録前に公開設定を確認したい場合だけ、空のリポジトリを手動作成する。

1. Hub のプロフィールメニューから **New Dataset** を選び、`DATASET_NAME` と同じ名前を指定する。
2. 最初は **Private** を選び、作成する。

CLI で作成する場合は次を実行する。

```bash
hf repos create "$DATASET_REPO_ID" --repo-type dataset --private
```

どちらの方法でも、後述する `lerobot-record` は同じリポジトリへアップロードする。手動作成しない場合は、`--dataset.private=true` によって非公開リポジトリが自動作成される。

## ステップ 4: 少数エピソードで試験収録する

本番と同じロボット、カメラ、照明で 3 エピソードを収録する。`front` には作業台全体、`wrist` にはグリッパーと対象物が映るようにする。

macOS では試験用のエピソード数と時間を設定する。

```bash
export NUM_EPISODES=3
export EPISODE_TIME_S=30
export RESET_TIME_S=10
```

Windows（Miniforge Prompt）:

```bat
set NUM_EPISODES=3
set EPISODE_TIME_S=30
set RESET_TIME_S=10
```

フォロワーの周囲が安全であることを確認してから、次を実行する。アーム ID とカメラ番号は [`setup.md`](./setup.md) で使った値に合わせる。

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id=my_follower \
  --robot.cameras="{front: {type: opencv, index_or_path: $FRONT_CAMERA_INDEX, width: $CAMERA_WIDTH, height: $CAMERA_HEIGHT, fps: $DATASET_FPS}, wrist: {type: opencv, index_or_path: $WRIST_CAMERA_INDEX, width: $CAMERA_WIDTH, height: $CAMERA_HEIGHT, fps: $DATASET_FPS}}" \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id=my_leader \
  --display_data=true \
  --dataset.repo_id="$DATASET_REPO_ID" \
  --dataset.root="$DATASET_ROOT" \
  --dataset.fps="$DATASET_FPS" \
  --dataset.num_episodes="$NUM_EPISODES" \
  --dataset.episode_time_s="$EPISODE_TIME_S" \
  --dataset.reset_time_s="$RESET_TIME_S" \
  --dataset.single_task="$TASK_DESCRIPTION" \
  --dataset.private=true \
  --dataset.no_stamp=true
```

Windows では、上のコマンド中の `$FOLLOWER_PORT` のような変数を `%FOLLOWER_PORT%` の形式に読み替える。行継続文字 `\` は使わず、1 行にまとめて実行してもよい。

### 収録中の操作

収録開始後、リーダーをゆっくり動かしてタスクを完了させる。各エピソードの後は、表示されるリセット時間中に対象物とアームを同じ初期状態へ戻す。次のキーで収録を制御する。

| キー | 操作 |
| --- | --- |
| `→` または `n` | 現在の収録またはリセットを早めに終え、次へ進む |
| `←` または `r` | 現在のエピソードを破棄して、同じエピソードを録り直す |
| `Esc` または `q` | 収録セッションを終了する。保存済みエピソードを動画化して Hub へアップロードする |

正常終了すると、LeRobot が保存済みエピソードを動画化し、`$DATASET_REPO_ID` を作成または更新する。エンコードとアップロードが終わるまでターミナルを閉じない。

実効 FPS が指定した 30 FPS を大きく下回る場合は、ほかのカメラ利用アプリを閉じて試す。それでも改善しなければ解像度を下げ、試験収録からやり直す。

## ステップ 5: Hub 上のデータを確認する

アップロード後、次の URL をブラウザで開く。

```text
https://huggingface.co/datasets/<HF_USER>/<DATASET_NAME>
```

今回の例では次のようになる。

```bash
echo "https://huggingface.co/datasets/$DATASET_REPO_ID"
```

Hub のデータセットページでエピソード数、ファイル、README が作成されていることを確認する。LeRobot は収録形式に合わせたデータセットカードを作成するため、通常の画像・動画ファイルをブラウザから追加する必要はない。

続けて、[LeRobot Dataset Visualizer](https://huggingface.co/spaces/lerobot/visualize_dataset) を開き、`DATASET_REPO_ID` を入力してすべての試験エピソードを確認する。非公開リポジトリを確認する場合は、同じ Hugging Face アカウントでログインする。

次を確認する。

- 対象物、グリッパー、ゴールが最初から最後まで画像に見える
- `front` と `wrist` の両方に、白画面、乱れ、停止がない
- リーダーの操作とフォロワーの動きが一致し、急な衝突・停止・誤操作がない
- タスク開始時の配置と完了条件が全エピソードでそろっている
- 関節状態と行動が記録され、映像が途中で止まっていない
- エピソード数が意図した数であり、タスク説明が実際の作業と一致している
- ターミナルの `Cadence summary` で、実効 FPS が 30 Hz に近い

過去の LeRobot GitHub Issues では、2 台同時の収録失敗、収録画面の白表示、保存後の映像の乱れ、訓練時に判明する時刻同期エラーが報告されている。いずれも終了済みの個別事例だが、ライブ映像だけで判断せず、保存後のデータを確認する理由になる。

試験データに問題がある場合は、そのデータを訓練に使わない。カメラやキャリブレーションを直した後、`DATASET_NAME` と `DATASET_ROOT` を新しい値に変えて、新しい試験データセットを作る。

## ステップ 6: 本番データを収録する

試験収録に問題がなければ、本番用に新しいデータセット名を設定する。試験データと本番データを混ぜないためである。タスク、カメラ、照明、作業台、ロボットのキャリブレーションは試験時と同じ条件に保つ。

macOS:

```bash
export DATASET_NAME="so101-pick-red-cube-production-20260815"
export DATASET_REPO_ID="$HF_USER/$DATASET_NAME"
export DATASET_ROOT="$HOME/lerobot-datasets/$DATASET_NAME"
export NUM_EPISODES=50
export EPISODE_TIME_S=30
export RESET_TIME_S=10
```

Windows（Miniforge Prompt）:

```bat
set DATASET_NAME=so101-pick-red-cube-production-20260815
set DATASET_REPO_ID=%HF_USER%/%DATASET_NAME%
set DATASET_ROOT=%USERPROFILE%\lerobot-datasets\%DATASET_NAME%
set NUM_EPISODES=50
set EPISODE_TIME_S=30
set RESET_TIME_S=10
```

収録前に、記録せず 5〜10 回練習する。接近方向、つかむ位置、持ち上げる高さ、離す位置をそろえ、迷いや持ち直しを減らす。把持失敗、衝突、見切れが起きたエピソードは `r` で録り直す。

ステップ 4 と同じ `lerobot-record` コマンドを実行する。対象物の開始位置を変える場合は、位置をテープなどで示し、次のように均等に収録する。

| 開始位置 | エピソード数 |
| --- | ---: |
| 左手前 | 10 |
| 左奥 | 10 |
| 中央 | 10 |
| 右手前 | 10 |
| 右奥 | 10 |

収録中は次の条件を守る。

- タスク、成功条件、カメラ、照明、箱の位置を変えない。
- 「接近 → 把持 → 持ち上げ → 移動 → 解放」の順序をそろえる。
- 成功したら `n` を押し、不要な静止時間を含めない。
- 開始位置ごとの本数を記録し、収録後にすべての映像を確認する。

最初から物体、背景、照明、操作方法を同時に変えない。初回の ACT を評価した後、失敗が集中した開始位置や工程の成功デモを 10〜20 エピソード追加する。

## 追加収録する

50 エピソードを一度に収録する必要はない。同じデータセットへ追加する場合は、最初の収録時と **同じ** `DATASET_REPO_ID`、`DATASET_ROOT`、ロボット設定、カメラ設定、タスク説明を使う。次の追加分のエピソード数を設定する。ここでの `20` は最終的な合計数ではなく、今回追加する数である。

macOS:

```bash
export NUM_EPISODES=20
```

Windows（Miniforge Prompt）:

```bat
set NUM_EPISODES=20
```

ステップ 4 の収録コマンドで、最後の `--dataset.no_stamp=true` を次の行へ置き換えて実行する。

```bash
  --resume=true
```

`--dataset.root` を省略せず、初回と同じ保存先を指定する。リポジトリ ID、ロボット、カメラ、FPS、タスク説明も変更しない。

## データセットカードを書く

収録後、Hub のデータセットページで `README.md` を編集する。次は初心者向け ACT データセットの記載例である。

```markdown
## 概要

SO-101 で赤い立方体を右側の箱へ入れるデモを収録した、ACT の初回訓練用データセットである。

## 収録条件

- 成功条件: 立方体が箱の中に入り、グリッパーが離れている
- ロボット: SO-101 follower と SO-101 leader
- カメラ: USB カメラ 2 台。front は作業台正面、wrist はグリッパー上部。各 640x480、30 FPS
- 環境: 白い作業台、室内照明を使用。影や映り込みが変わらないように固定
- 初期状態: 赤い立方体は印を付けた 5 位置のいずれか、箱は右側に固定
- データ: 成功したデモ 50 エピソード、各 30 秒以内
- 除外したデータ: 衝突、見切れ、把持失敗を含むエピソードは収録中に破棄し、未収録

## 利用条件

- ライセンス: CC BY 4.0
- 公開してよい物だけを撮影し、人物、個人情報、共有できない物が映る映像は含まない
```

ライセンスは例である。データの権利関係に合わせて選び、判断できない場合は公開しない。公開前に全映像を見直し、人物、名札、画面、書類、社外秘の物、第三者の著作物が映っていないことを確認する。

## 収録品質のチェックリスト

本番データを訓練へ渡す前に、次を満たしているか確認する。

- [ ] すべてのエピソードが同じタスク説明と成功条件に対応している
- [ ] 本番と同じ 2 台のカメラで試験収録を完了している
- [ ] カメラ位置、視野、解像度、FPS、照明が収録中に変わっていない
- [ ] 対象物、グリッパー、ゴールが映像で確認できる
- [ ] 開始姿勢とリセット後の配置が一貫している
- [ ] 成功デモだけを使い、衝突、見切れ、大きな操作ミス、失敗を含むエピソードは除外している
- [ ] 5 か所の開始位置を各 10 エピソード収録している
- [ ] 実効 FPS が目標に近く、保存後の動画に白画面、乱れ、停止がない
- [ ] Hub の Visualizer ですべてのエピソードを確認した
- [ ] リポジトリの公開範囲とデータセットカードの内容を確認した

## トラブルシューティング

| 状況 | 確認・対処 |
| --- | --- |
| `hf auth whoami` が失敗する | `hf auth login` を実行し、Write 権限のある Hugging Face アカウントで認証する。|
| アップロード時に `401` または `403` が出る | `DATASET_REPO_ID` の所有者がログイン中のユーザーまたは書き込み権限を持つ組織か確認する。Write 権限のトークンで再ログインする。|
| Colab から `HF_TOKEN` を取得できない | Secret 名が大文字の `HF_TOKEN` か確認し、Colab のシークレット画面で **ノートブックからのアクセス**を有効にする。|
| 新規収録時に保存先が既に存在すると表示される | 同じ名前で新規作成しようとしている。追加収録なら初回と同じ値で `--resume=true` を使い、別データセットなら新しい `DATASET_NAME` と `DATASET_ROOT` を使う。|
| カメラ映像が表示されない | Zoom、Teams、ブラウザなどカメラを占有するアプリを閉じる。`lerobot-find-cameras opencv` でカメラ番号を再確認する。|
| 実効 FPS が低い | カメラ解像度や FPS を下げ、他のアプリを閉じる。設定を変えたら本番前に試験収録をやり直す。|
| 収録後のアップロードに失敗した | 収録済みデータは `DATASET_ROOT` に残る。ネットワークと `hf auth whoami` を確認後、確定済みのフォルダを次でアップロードする。`DATASET_REPO_ID` と `DATASET_ROOT` は実際の値を使う。|

アップロードだけをやり直す場合:

```bash
hf upload "$DATASET_REPO_ID" "$DATASET_ROOT" --repo-type dataset
```

このコマンドはローカルのデータセットフォルダを Hub へ送る。収録の途中で実行せず、`lerobot-record` が終了して動画エンコードまで完了した後にだけ使う。

## 次に行うこと

データセットの全エピソードを確認したら、`lerobot-train` で ACT の模倣学習を行う。訓練時には、ここで作成した `DATASET_REPO_ID` を `--dataset.repo_id` に指定する。

```bash
lerobot-train \
  --policy.type=act \
  --dataset.repo_id="$DATASET_REPO_ID"
```

訓練用データセットと実機評価では、収録時と同じカメラ構成、解像度、FPS、アーム ID、キャリブレーションを使う。

## 参考にしたファイル・URL

- [`setup.md`](./setup.md): このワークショップ用の SO-101 セットアップとカメラ確認手順
- [`lerobot/AGENT_GUIDE.md`](./lerobot/AGENT_GUIDE.md): SO-101 向けの収録と ACT の推奨事項
- [`lerobot/docs/source/il_robots.mdx`](./lerobot/docs/source/il_robots.mdx): LeRobot の収録、アップロード、可視化、再生の公式手順
- [`lerobot/src/lerobot/scripts/lerobot_record.py`](./lerobot/src/lerobot/scripts/lerobot_record.py): `lerobot-record` の収録・再開・アップロード処理
- [`lerobot/docs/source/streaming_video_encoding.mdx`](./lerobot/docs/source/streaming_video_encoding.mdx): 動画エンコード負荷、欠損フレーム、設定調整
- [Hugging Face Hub: Quickstart](https://huggingface.co/docs/huggingface_hub/quick-start): アカウント、認証、Write トークン
- [Hugging Face Hub: Uploading datasets](https://huggingface.co/docs/hub/datasets-adding): データセットリポジトリの作成と公開設定
- [Hugging Face Hub: Create and manage a repository](https://huggingface.co/docs/huggingface_hub/en/guides/repository): `hf repos create` によるデータセットリポジトリ作成
- [LeRobot Community Datasets: The “ImageNet” of Robotics — When and How?](https://huggingface.co/blog/lerobot-datasets): 画像品質、カメラ名、タスク記述の公式チェックリスト
- LeRobot GitHub Issues: [2 台同時の収録失敗](https://github.com/huggingface/lerobot/issues/1743)、[収録画面の白表示](https://github.com/huggingface/lerobot/issues/3218)、[保存後の映像の乱れ](https://github.com/huggingface/lerobot/issues/1606)、[時刻同期エラー](https://github.com/huggingface/lerobot/issues/1875)の事例。いずれも Closed
- [Learning Fine-Grained Bimanual Manipulation with Low-Cost Hardware](https://arxiv.org/abs/2304.13705): ACT の原論文
