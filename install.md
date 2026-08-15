# LeRobot のインストール

## 目次

- [概要](#概要)
- [前提](#前提)
- [完成形](#完成形)
- [はじめる前に](#はじめる前に)
- [ステップ 1: Miniforge をインストールする](#ステップ-1-miniforge-をインストールする)
- [ステップ 2: ワークショップ用フォルダーを作成する](#ステップ-2-ワークショップ用フォルダーを作成する)
- [ステップ 3: LeRobot 用の Python 環境を作成する](#ステップ-3-lerobot-用の-python-環境を作成する)
- [ステップ 4: 必要なパッケージを準備する](#ステップ-4-必要なパッケージを準備する)
- [ステップ 5: LeRobot をインストールする](#ステップ-5-lerobot-をインストールする)
- [ステップ 6: インストールを確認する](#ステップ-6-インストールを確認する)
- [ステップ 7: 毎回の開始手順](#ステップ-7-毎回の開始手順)
- [次に行うこと](#次に行うこと)
- [参考にしたファイル・URL](#参考にしたファイルurl)

## 概要

この資料では、SO-ARM 101 の操作とデータ収録に使う PC へ LeRobot をインストールする。ローカル PC に GPU は不要である。モデルの学習は Google Colab の GPU で行う。

この手順では、SO-ARM 101 に必要なモーター制御、カメラ表示、データ収録・再生の機能をインストールする。

## 前提

この資料は LeRobot **v0.6.2** で動作を確認した。ステップ 2 では公式リポジトリの最新コードを取得するため、時期によって表示やコマンドが少し異なる場合がある。

対象の OS は次のいずれかである。

- macOS（Apple Silicon / Intel）
- Windows 11/10 のネイティブ環境

この入門資料では WSL2 を扱わない。SO-ARM 101 と USB カメラを WSL2 で使うには、Windows から WSL2 への USB 機器共有を別途設定する必要があり、初回の環境構築とトラブルシューティングが複雑になるためである。アーム操作とデータ収録は macOS または Windows ネイティブ環境で行う。

LeRobot 本体の取得には Git を使う。Git の確認方法と、未インストールの場合の案内はステップ 2 に記載する。

## 完成形

インストール後、次のコマンドが動けば準備完了である。

```bash
lerobot-info
```

次は、macOS（Apple Silicon）で GPU を使わない環境の出力例である。長い箇所は省略している。各バージョンは、インストールした時期によって異なってよい。

```text
- LeRobot version: 0.6.2
- Platform: macOS-26.5.2-arm64-arm-64bit
- Python version: 3.12.13
- Huggingface Hub version: 1.27.0
- Transformers version: N/A
- Datasets version: 4.8.5
- Numpy version: 2.2.6
- FFmpeg version: 9.0.1
- PyTorch version: 2.11.0
- Torchcodec version: 0.11.1
- Is PyTorch built with CUDA support?: False
- Cuda version: N/A
- GPU model: N/A
- Using GPU in script?: <fill in>
- lerobot scripts: [..., 'lerobot-calibrate', 'lerobot-find-port', 'lerobot-record',
                    'lerobot-replay', 'lerobot-setup-motors', 'lerobot-teleoperate', ...]
```

この例のように `lerobot-find-port`、`lerobot-calibrate`、`lerobot-record`、`lerobot-teleoperate` が含まれていれば、必要なコマンドを利用できる。CUDA と GPU が `N/A` でも問題ない。`Using GPU in script?: <fill in>` も、利用者が必要に応じて記録する欄なのでエラーではない。

## はじめる前に

- インターネット接続
- 20 GB 以上の空き容量
- ソフトウェアをインストールできるユーザー権限
- SO-ARM 101 を使う場合は、リーダー・フォロワーの USB 接続と電源

Python 3.12 と LeRobot 本体は、この後の手順で用意する。事前にインストールする必要はない。

## ステップ 1: Miniforge をインストールする

Miniforge は、Python と `conda`（Python の環境を分けて管理するための道具）をまとめて入れるソフトウェアである。この手順は **Miniforge がまだ入っていない** 前提で進める。

### 1-1. Miniforge をインストールする

macOS では「ターミナル」を開き、次の 2 行を実行する。Mac の CPU に合うインストーラーが自動で選ばれる。

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

Windows では [Miniforge のダウンロードページ](https://conda-forge.org/download/) から `Windows-x86_64` のインストーラーをダウンロードして実行する。インストール後は、スタートメニューから **Miniforge Prompt** を開く。以降の Windows 用コマンドは Miniforge Prompt で実行する。

### 1-2. macOS の質問に答える

`bash Miniforge3-...sh` を実行すると、英語の質問が表示される。以下のとおり入力する。

1. `Please, press ENTER to continue` と表示されたら、何も入力せず `Enter` を押す。
2. `Do you accept the license terms? [yes|no]` と表示されたら、次を入力して `Enter` を押す。

   ```text
   yes
   ```

3. `Miniforge3 will now be installed into this location:` と表示されたら、保存先は変更せず `Enter` を押す。通常は `/Users/<ユーザー名>/miniforge3` に入る。
4. `Do you wish to update your shell profile to automatically initialize conda?` と表示されたら、`yes` を入力する。
5. 完了したら、ターミナルを完全に閉じて新しく開く。

保存先を確認する画面で「すでに存在する」と表示された場合は、初回インストールではない。`Ctrl-C` でインストーラーを終了し、「Miniforge がすでに入っている場合」へ進む。

### 1-3. インストール後の確認

新しく開いたターミナルまたは Miniforge Prompt で、次を実行する。

```bash
conda --version
```

`conda` に続いてバージョン番号が表示されれば成功である。

### Miniforge がすでに入っている場合

`ERROR: File or directory already exists: '/Users/<ユーザー名>/miniforge3'` と表示された場合、Miniforge はすでにその場所に入っている。インストーラーを終了してから、次を実行する。

```bash
~/miniforge3/bin/conda --version
```

バージョンが表示されたら、Miniforge 本体は利用できる。続けて「`conda: command not found` と表示される場合」へ進む。

`~/miniforge3/bin/conda` も見つからない場合は、以前のインストールが途中で止まっている可能性がある。フォルダーを削除してやり直す前に、講師またはサポート担当者へ相談する。

Windows では Miniforge Prompt を開き、`conda --version` を実行する。バージョンが表示されれば再インストールは不要である。

### `conda: command not found` と表示される場合

Miniforge は入っていても、ターミナルが `conda` の場所を知らないと `command not found` と表示する。これを「パスが通っていない」と呼ぶ。

macOS の標準ターミナル（zsh）では、次を実行する。

```bash
~/miniforge3/bin/conda init zsh
```

ターミナルを完全に閉じて新しく開き、確認する。

```bash
conda --version
```

それでも見つからない場合だけ、`condabin` をパスへ追加する。次を実行してから、ターミナルを開き直す。

```bash
echo 'export PATH="$HOME/miniforge3/condabin:$PATH"' >> ~/.zshrc
```

Windows ではまず Miniforge Prompt を開く。通常の PowerShell でも使いたい場合は、Miniforge Prompt で次を実行してから PowerShell を開き直す。

```bat
conda init powershell
```

それでも使えない場合は、Windows の「環境変数」設定で、ユーザー環境変数の `Path` に次を追加する。

```text
C:\Users\<ユーザー名>\miniforge3\condabin
```

### Miniforge をアップグレードする場合

ワークショップ前のアップグレードは不要である。更新する場合だけ、次を実行する。

```bash
conda activate base
conda update -y -n base -c conda-forge --all
```

アップグレード後はターミナルを開き直し、次で確認する。

```bash
conda --version
```

既存のインストールがあるというエラーを解消する目的で、インストーラーに `-u` を付けない。

## ステップ 2: ワークショップ用フォルダーを作成する

作業用フォルダーには `lerobot-workshop` という名前を使う。ここに LeRobot 本体をダウンロードする。

macOS:

```bash
mkdir -p "$HOME/lerobot-workshop"
```

Windows ネイティブ（Miniforge Prompt）:

```bat
mkdir "%USERPROFILE%\lerobot-workshop"
```

Windows で「サブディレクトリまたはファイルは既に存在します」と表示されても問題ない。そのまま次へ進む。

### LeRobot 本体をダウンロードする

Git が使えることを確認する。

```bash
git --version
```

バージョンが表示されれば、そのまま次へ進む。macOS でインストールを求める画面が出た場合は、画面の案内に従う。Windows で `git` が見つからない場合は、[Git for Windows](https://git-scm.com/download/win) をインストールし、Miniforge Prompt を開き直す。

次を実行すると、`lerobot-workshop` の中に `lerobot` ディレクトリが作成される。

macOS:

```bash
git clone https://github.com/huggingface/lerobot.git "$HOME/lerobot-workshop/lerobot"
```

Windows ネイティブ:

```bat
git clone https://github.com/huggingface/lerobot.git "%USERPROFILE%\lerobot-workshop\lerobot"
```

ダウンロードには数分かかることがある。コマンド入力を待つ表示が戻るまで待つ。

ダウンロード後の構成は次の形にする。

```text
lerobot-workshop/
└── lerobot/
```

`lerobot/` は LeRobot 本体のソースコードであり、この後にインストールする対象である。

`lerobot` が作成されたことを確認する。

macOS:

```bash
ls "$HOME/lerobot-workshop"
```

Windows ネイティブ:

```bat
dir "%USERPROFILE%\lerobot-workshop"
```

### `destination path 'lerobot' already exists` と表示される場合

`lerobot` ディレクトリはすでに存在する。同じ場所へダウンロードし直さず、`ls` または `dir` で内容を確認する。

## ステップ 3: LeRobot 用の Python 環境を作成する

Python 3.12 の専用環境を作る。このコマンドは最初の 1 回だけ実行する。

```bash
conda create -y -n lerobot python=3.12
```

作成した環境を有効にする。

```bash
conda activate lerobot
```

コマンド行の先頭に `(lerobot)` と表示されれば成功である。

## ステップ 4: 必要なパッケージを準備する

### FFmpeg をインストールする

データセットの動画を扱うため、FFmpeg をインストールする。macOS と Windows のどちらでも、LeRobot 用環境を有効にしてから実行する。

```bash
conda install -y -c conda-forge ffmpeg
```

### Windows のみ: CPU 版 PyTorch をインストールする

GPU を搭載していない PC では、CPU 版の PyTorch を先に入れる。これにより CUDA（NVIDIA GPU 用）の大きなパッケージをダウンロードしない。

```bat
python -m pip install --index-url https://download.pytorch.org/whl/cpu torch torchvision
```

カメラを使う場合は、Windows のプライバシー設定でデスクトップ アプリのカメラ利用を許可する。

## ステップ 5: LeRobot をインストールする

コマンド行の先頭に `(lerobot)` が表示されていることを確認し、次を実行する。ディレクトリの移動は不要である。

macOS:

```bash
python -m pip install -e "$HOME/lerobot-workshop/lerobot[core_scripts,feetech]"
```

Windows ネイティブ:

```bat
python -m pip install -e "%USERPROFILE%\lerobot-workshop\lerobot[core_scripts,feetech]"
```

このコマンドには次が含まれる。

- `core_scripts`: キャリブレーション、テレオペレーション、データ収録、再生、映像表示
- `feetech`: SO-ARM 101 の Feetech モーターとの通信

`[training]` や `all` はローカル PC には入れない。ACT の学習は Google Colab で行うためである。

インストールには数分かかる。コマンド入力を待つ表示が戻るまで、ターミナルを閉じない。

### `lerobot` ディレクトリが見つからない場合

ステップ 2 に戻り、`lerobot-workshop` の中に `lerobot` があることを確認する。

## ステップ 6: インストールを確認する

次を実行する。

```bash
lerobot-info
```

LeRobot のバージョンと `lerobot scripts` の一覧が表示されれば成功である。表示例は冒頭の「完成形」を参照する。

続けて、SO-ARM 101 の USB 接続先を調べるコマンドが見つかることも確認する。

```bash
lerobot-find-port --help
```

### `lerobot-info` が見つからない場合

LeRobot 用環境が有効になっていない可能性がある。次を実行し、もう一度 `lerobot-info` を試す。

```bash
conda activate lerobot
```

それでも見つからない場合は、ステップ 5 のインストールをやり直す。

## ステップ 7: 毎回の開始手順

新しいターミナルまたは Miniforge Prompt を開いたら、最初に次を実行する。

```bash
conda activate lerobot
```

コマンド行の先頭に `(lerobot)` と表示されれば準備完了である。LeRobot のコマンドを使うだけなら、`lerobot` ディレクトリへ移動する必要はない。

## 次に行うこと

インストール後は、以下の順に進める。

1. `lerobot-find-port` でリーダーとフォロワーの USB ポートを調べる。
2. モーター ID・通信速度を設定し、キャリブレーションを行う。
3. テレオペレーションとカメラを確認する。
4. デモを収録して Hugging Face Hub にアップロードする。
5. Google Colab で ACT を使った模倣学習を行う。

SO-ARM 101 の設定以降は、`lerobot/docs/source/so101.mdx` を参照する。テレオペレーション、データ収録、学習、評価の流れは `lerobot/docs/source/il_robots.mdx` を参照する。

### `lerobot-find-port` がアームを見つけない場合

アームの電源、USB ケーブル、USB ハブを確認する。macOS のポート名は通常 `/dev/tty.usbmodem...` の形式である。Windows ではデバイス マネージャーの「ポート（COM と LPT）」で、アームに割り当てられた `COM` 番号を確認する。

### カメラが使えない場合

Zoom、Teams、ブラウザなど、カメラを使うアプリを閉じる。OS のプライバシー設定で、ターミナルまたは Miniforge Prompt にカメラの利用を許可する。

## 参考にしたファイル・URL

この資料は、以下のファイル・公式ページを参照して作成した。

- [`lerobot/pyproject.toml`](./lerobot/pyproject.toml): LeRobot のバージョン、Python 要件、追加パッケージ
- [`lerobot/docs/source/installation.mdx`](./lerobot/docs/source/installation.mdx): Python 環境、動画処理、LeRobot のインストール方法
- [`lerobot/docs/source/so101.mdx`](./lerobot/docs/source/so101.mdx): SO-ARM 101 用の Feetech SDK と USB ポート確認
- [`lerobot/docs/source/il_robots.mdx`](./lerobot/docs/source/il_robots.mdx): データ収録、学習、実機評価の流れ
- [Miniforge 公式リポジトリ](https://github.com/conda-forge/miniforge): Miniforge のインストール、シェル初期化、更新方法
- [LeRobot 公式リポジトリ](https://github.com/huggingface/lerobot): LeRobot 本体のダウンロード元
- [PyTorch CPU 版パッケージ一覧](https://download.pytorch.org/whl/cpu): GPU 非搭載の Windows PC 向け PyTorch の取得元
