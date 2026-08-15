# SO-101 のセットアップとテレオペレーション

## 目次

- [概要](#概要)
- [この資料で行うこと](#この資料で行うこと)
- [はじめる前に](#はじめる前に)
- [用語と設定値](#用語と設定値)
- [ステップ 1: 作業環境を開始する](#ステップ-1-作業環境を開始する)
- [ステップ 2: USB ポートを調べる](#ステップ-2-usb-ポートを調べる)
- [ステップ 3: モーター ID と通信速度を設定する](#ステップ-3-モーター-id-と通信速度を設定する)
- [ステップ 4: キャリブレーションする](#ステップ-4-キャリブレーションする)
- [ステップ 5: テレオペレーションを確認する](#ステップ-5-テレオペレーションを確認する)
- [ステップ 6: 2 台のカメラを確認する](#ステップ-6-2-台のカメラを確認する)
- [次に行うこと](#次に行うこと)
- [参考にしたファイル・URL](#参考にしたファイルurl)

## 概要

この資料では、SO-101 のリーダーアームを動かしてフォロワーアームを操作できる状態にする。最初に USB ポートを調べ、未設定のアームだけモーター ID と通信速度を設定する。続けて両方のアームをキャリブレーションし、テレオペレーションとカメラを確認する。

ここで作る設定は、この後に行うデモデータの収録、ACT による模倣学習、Google Colab で訓練したポリシーの実機評価でも使う。特にアーム ID は後のコマンドでも同じ値を使う。

## この資料で行うこと

1. リーダー・フォロワーの USB ポートを確認する。
2. 初回だけ、各モーターの ID と通信速度を設定する。
3. リーダー・フォロワーをキャリブレーションする。
4. カメラなしでテレオペレーションを確認する。
5. カメラを追加し、収録に使う視野を確認する。

データ収録と ACT の訓練は、次の資料で扱う。

## はじめる前に

- [`install.md`](./install.md) の手順を終え、`lerobot-info` が動くこと
- 組み立て済みの SO-101 リーダーアームとフォロワーアーム
- 各アーム用の USB ケーブル、フォロワー用電源、作業用 PC
- `front` 用・`wrist` 用の USB カメラ 2 台
- アームの周囲に、動作中にぶつかる物や手指がないこと

モーターの配線を抜き差しするときは、必ずアームの電源を切る。テレオペレーション中はフォロワーがすぐに動くため、最初は低い位置で、周囲に人や物を置かずに確認する。

## 用語と設定値

| 項目 | 意味 | この資料の例 |
| --- | --- | --- |
| リーダー | 手で動かして指示を与えるアーム | `my_leader` |
| フォロワー | リーダーに追従して動くアーム | `my_follower` |
| ポート | PC が USB 接続したアームを識別する名前 | macOS: `/dev/tty.usbmodem...`、Windows: `COM3` など |
| アーム ID | キャリブレーション結果を保存する名前 | `my_leader`、`my_follower` |

ポート名は、ステップ 2 でシェル変数に保存する。`my_follower` と `my_leader` は分かりやすい任意の名前でよいが、キャリブレーション後は収録・評価を含むすべてのコマンドで同じ名前を使う。

## ステップ 1: 作業環境を開始する

新しいターミナル、または Windows の Miniforge Prompt を開き、LeRobot の環境を有効にする。

```bash
conda activate lerobot
```

行頭に `(lerobot)` と表示されたら準備完了である。続けて、SO-101 用のコマンドが使えることを確認する。

```bash
lerobot-calibrate --help
```

`command not found` と表示される場合は、[`install.md`](./install.md) の「ステップ 5: LeRobot をインストールする」を確認する。

## ステップ 2: USB ポートを調べる

リーダーとフォロワーを同時に USB 接続する。フォロワーは電源も接続する。次を実行する。

```bash
lerobot-find-port
```

表示に従い、片方のアームの USB ケーブルを抜いて `Enter` を押す。表示されたポート名を、どちらのアームか分かるように記録する。もう一方のアームについても同じ操作を行う。

macOS では `/dev/tty.usbmodem...`、Windows では `COM3` のような名前が表示される。確認したポート名を、次のコマンドでこのターミナルだけで有効な環境変数へ保存する。

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

設定した値を確認する。

macOS:

```bash
echo "$FOLLOWER_PORT"
echo "$LEADER_PORT"
```

Windows（Miniforge Prompt）:

```bat
echo %FOLLOWER_PORT%
echo %LEADER_PORT%
```

ポート名は USB の差し替え先によって変わることがある。

以降のコマンドでは macOS で `$FOLLOWER_PORT`、`$LEADER_PORT` を参照する。Windows の Miniforge Prompt では、それぞれ `%FOLLOWER_PORT%`、`%LEADER_PORT%` に読み替える。ターミナルまたは Miniforge Prompt を閉じると変数は消えるため、次回はこのステップの変数設定だけをもう一度実行する。

### `lerobot-find-port` がアームを見つけない場合

アームの電源、USB ケーブル、USB ハブを確認する。macOS では通常 `/dev/tty.usbmodem...`、Windows ではデバイス マネージャーの「ポート (COM と LPT)」に表示される `COM` 番号を確認する。別の USB ポートやケーブルへの交換も試す。

## ステップ 3: モーター ID と通信速度を設定する

このステップは、新品のモーターを初めて使うとき、またはモーターを交換したときだけ行う。一度正常に設定したアームでは繰り返さない。モーター内部の不揮発性メモリへ設定を書き込むためである。

アームごとに、画面の指示に従って **コントローラーボードへモーターを 1 個だけ接続** する。複数のモーターをデイジーチェーン接続したまま実行しない。各モーターに異なる ID を割り当てるためである。

### 3-1. フォロワーを設定する

フォロワーのコントローラーボードへ USB と電源を接続し、次を実行する。

```bash
lerobot-setup-motors \
  --robot.type=so101_follower \
  --robot.port=$FOLLOWER_PORT
```

画面に表示された順に、指定されたモーターだけを接続して `Enter` を押す。完了後は、フォロワーの配線図どおりにモーターをデイジーチェーン接続し直す。

### 3-2. リーダーを設定する

同じ手順でリーダーを設定する。

```bash
lerobot-setup-motors \
  --teleop.type=so101_leader \
  --teleop.port=$LEADER_PORT
```

完了後、リーダーも配線図どおりに接続し直す。エラーが出た場合は、電源、USB ケーブル、3 ピンのモーターケーブルが確実に接続されているかを確認する。

### モーター設定中に通信エラーが出る場合

電源、PC とコントローラーボードをつなぐ USB ケーブル、コントローラーボードとモーターをつなぐ 3 ピンケーブルを確認する。設定中は、画面で指定されたモーター以外をコントローラーボードに接続しない。

## ステップ 4: キャリブレーションする

キャリブレーションは、各関節の中間位置と可動範囲を登録する操作である。リーダーとフォロワーを同じ基準で扱えるようにするため、必ず両方に実行する。

開始前に、アームが自由に動かせる場所へ置く。実行後は、まず全関節を可動範囲の中間付近へ動かす。画面の案内で `Enter` を押した後、各関節を無理な力を掛けずに端から端までゆっくり動かす。

### 4-1. フォロワーをキャリブレーションする

```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=$FOLLOWER_PORT \
  --robot.id=my_follower
```

### 4-2. リーダーをキャリブレーションする

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=$LEADER_PORT \
  --teleop.id=my_leader
```

### キャリブレーションの進め方

フォロワー、リーダーともに同じ流れで行う。まず全関節を可動範囲の中間位置へ動かしてから、画面の案内に従って `Enter` を押す。その後、各関節を無理な力を掛けずに可動域の端から端までゆっくり動かす。グリッパーも開いた位置と閉じた位置まで動かす。

公式の実演動画では、中間位置でのリセットから各関節の可動域確認までを確認できる。

<video controls width="600">
  <source src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/lerobot/calibrate_so101_2.mp4" type="video/mp4" />
  お使いの環境では動画を再生できません。
</video>

動画が表示されない場合は、[SO-101 キャリブレーションの公式動画](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/lerobot/calibrate_so101_2.mp4) を開く。

キャリブレーションが終わったら、アーム ID とポート名を記録する。

```text
フォロワー: port=$FOLLOWER_PORT, id=my_follower
リーダー:   port=$LEADER_PORT, id=my_leader
```

関節を交換した、組み立てを変更した、またはリーダーとフォロワーの同じ姿勢で大きくずれる場合は、両方を再キャリブレーションする。

## ステップ 5: テレオペレーションを確認する

最初はカメラを付けずに動作を確認する。フォロワーの周囲が安全であることを確認してから、次を実行する。

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=$FOLLOWER_PORT \
  --robot.id=my_follower \
  --teleop.type=so101_leader \
  --teleop.port=$LEADER_PORT \
  --teleop.id=my_leader
```

リーダーをゆっくり動かし、フォロワーが対応する関節を追従することを確認する。グリッパーも開閉することを確認する。止めるときはターミナルで `Ctrl-C` を押す。

フォロワーが動かない、または姿勢が大きくずれる場合は、無理に動かし続けずに停止する。ポート、アーム ID、電源、キャリブレーションを確認する。

### フォロワーが動かない、または異常に動く場合

すぐに `Ctrl-C` で停止する。フォロワーの電源と配線、リーダー・フォロワーのポート指定、アーム ID を確認する。その後、両方のアームを再キャリブレーションする。

## ステップ 6: 2 台のカメラを確認する

ACT 用のデータには、タスク対象が十分に見えるカメラ画像が重要である。このワークショップでは、正面を撮影する `front` カメラと、手元を撮影する `wrist` カメラの 2 台を使う。まず 2 台のカメラ番号を調べる。

```bash
lerobot-find-cameras opencv
```

表示された 2 台のカメラ番号を控える。以下の `0` と `1` は例なので、実際の番号に置き換える。`front` には作業台全体が見えるカメラ、`wrist` にはグリッパーと対象物が見えるカメラを指定する。

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=$FOLLOWER_PORT \
  --robot.id=my_follower \
  --robot.cameras="{front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}, wrist: {type: opencv, index_or_path: 1, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port=$LEADER_PORT \
  --teleop.id=my_leader \
  --display_data=true
```

両方の映像で、作業台、対象物、グリッパーがタスクの最初から最後まで見えることを確認する。カメラの位置、向き、照明をここで固定する。収録後にカメラ位置を変えると、訓練データの条件がそろわなくなる。

### カメラが表示されない場合

Zoom、Teams、ブラウザなどカメラを使うアプリを閉じる。OS のプライバシー設定で、ターミナルまたは Miniforge Prompt にカメラの利用を許可する。`lerobot-find-cameras opencv` でカメラ番号をもう一度確認する。

## 次に行うこと

セットアップが完了したら、次は `lerobot-record` で ACT の模倣学習用データを収録する。収録では、この資料で決めたフォロワー・リーダーのポート、アーム ID、カメラ設定をそのまま使う。

最初のデータセットは、1 つの単純なタスクを 50 エピソード程度収録する。カメラと作業台を固定し、同じつかみ方・置き方でゆっくり操作する。収録後にデータを確認してから、Google Colab の GPU で ACT を訓練する。

## 参考にしたファイル・URL

- [`install.md`](./install.md): このワークショップ用の LeRobot インストール手順
- [`lerobot/docs/source/so101.mdx`](./lerobot/docs/source/so101.mdx): SO-101 のモーター設定とキャリブレーション
- [`lerobot/docs/source/il_robots.mdx`](./lerobot/docs/source/il_robots.mdx): テレオペレーション、データ収録、訓練、評価の流れ
- [`lerobot/docs/source/cameras.mdx`](./lerobot/docs/source/cameras.mdx): カメラの検出と設定
