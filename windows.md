# LeRobot v0.6.0 — Windows インストールガイド

本ドキュメントは、Windows 上で LeRobot v0.6.0（本リポジトリの `lerobot/` サブモジュール）をセットアップする手順をまとめたものである。

## 0. 想定環境

| 項目 | 値 |
| --- | --- |
| OS | Windows 10 / 11（64bit） |
| GPU | NVIDIA GeForce RTX 3060（VRAM 6144 MiB / Ampere, sm_86） |
| CUDA | 13.2（`nvidia-smi` 表示のドライバ対応バージョン） |
| Python | 3.12（LeRobot は `requires-python = ">=3.12"`） |
| 用途 | 実機ロボット操作（SO-101 等）／ポリシー学習／シミュレーション評価／**世界モデルの実験** |

> **CUDA について**
> `nvidia-smi` が表示する「CUDA Version」はドライバが対応する上限であり、CUDA Toolkit を別途インストールする必要はない。PyTorch のホイールには CUDA ランタイムが同梱されているためである。

### 0.1 推論時 VRAM 早見表

**学習は Google Colab で行う前提とし、ローカルの RTX 3060 6 GB は「学習済みチェックポイントを推論で動かす」用途に絞る。** 以下はバッチ 1・単一〜2 カメラ・224–256 px 程度を想定した**推論時**の目安である。

| ポリシー | 規模 | 既定 dtype | 推論 VRAM 目安 | RTX 3060 6 GB |
| --- | --- | --- | ---: | --- |
| `act` | 約 80M | fp32 | 約 0.5–1 GB | ◎ 余裕 |
| `diffusion` | 約 100–260M | fp32 | 約 1–2 GB | ◎ |
| `vqbet` / `tdmpc` | 小 | fp32 | 約 1–2 GB | ◎ |
| `multi_task_dit` | 約 450M | fp32 | 約 2–3 GB | ○ |
| `smolvla` | 約 450M | fp32 | 約 2.5–3.5 GB | ○ |
| `evo1` | InternVL3-1B | fp32 | 約 4–5 GB | △ ぎりぎり |
| `xvla` | 約 0.9B | fp32 | 約 4–5 GB | △ ぎりぎり |
| `pi0` / `pi0_fast` / `pi05` | 約 3.3B | fp32 | 約 14–16 GB | ✕ |
| `groot` (N1.7-3B) / `eo1` (Qwen2.5-VL-3B) / `molmoact2` | 約 3B | fp32 | 約 13–16 GB | ✕ |
| `wall_x` | 大型 VLA | fp32 | 約 14–16 GB | ✕ |
| **`vla_jepa`** | Qwen3-VL-2B + DiT-B | **bf16** | **約 5–7 GB** | **△ 最も近いが厳しい** |
| **`fastwam`** | Wan2.2-TI2V-5B | **bf16** | **約 12–16 GB**（公式再現例は fp32 で 24 GB 超） | **✕** |
| **`lingbot_va`** | Wan2.2 DiT ~5B + VAE | — | **約 18–24 GB**（公式明記） | **✕** |

> **数値の出所について**
> `lingbot_va` の「18–24 GB」のみ公式ドキュメント（`docs/source/lingbot_va.mdx`）に明記された値である。それ以外は **パラメータ数 × dtype のバイト数から算出した重み量に、活性化・KV キャッシュ・CUDA コンテキスト分（数百 MB〜1 GB 程度）を上乗せした推定値**であり、公式の実測ではない。画像解像度・カメラ台数・`chunk_size` を上げれば増える。

**dtype に関する重要な注意**：LeRobot の `PreTrainedConfig` には全ポリシー共通の dtype 指定が存在しない。`--policy.torch_dtype` を持つのは **`fastwam` と `vla_jepa` の 2 つだけ**であり、他のポリシーは**推論時も fp32 で動く**。上表で 3B 級が 14–16 GB になるのはこのためである。VRAM を削りたい場合は `--policy.use_amp=true` を使う（`use_amp` は学習だけでなく評価にも適用される旨がコード内に明記されている）。

### 0.2 世界モデル系ポリシーの推論要件

v0.6.0 で追加された 3 つの世界モデル系ポリシーは、いずれも **数 B パラメータの動画生成バックボーンを土台にしている**。

| ポリシー | 構成 | 推論時に載るもの | 推論 VRAM | 追加ダウンロード |
| --- | --- | --- | ---: | --- |
| `vla_jepa` | Qwen3-VL-2B + V-JEPA2 + DiT-B | **Qwen + 行動ヘッドのみ**（世界モデルは学習時専用） | 約 5–7 GB | 約 5–10 GB |
| `fastwam` | Wan2.2-TI2V-5B | Wan VAE + DiT + テキストエンコーダ | 約 12–16 GB | 約 20 GB |
| `lingbot_va` | Wan2.2 DiT ~5B + VAE + UMT5-XXL | DiT + VAE（UMT5 は**既定で CPU**） | **約 18–24 GB** | **約 20 GB** |

**3060 で最も可能性があるのは `vla_jepa` である。** 理由は 2 つある。第一に、**推論時には世界モデル（V-JEPA2 予測器）が不要**で、Qwen3-VL-2B と DiT-B 行動ヘッドだけが載る。第二に、既定 dtype が bf16 である。それでも重みだけで 4.4 GB 前後を占め、6 GB では活性化とデスクトップ描画分を差し引くとほぼ余裕がない。**実測して OOM なら、まずディスプレイ出力を内蔵 GPU 側に移して 3060 の VRAM を空けることを試す。**

`fastwam` と `lingbot_va` は 6 GB では推論も成立しない。バックボーンが Wan2.2 の 5B 動画生成モデルであり、`lingbot_va` は VAE デコードも GPU 上で行うためである。

**運用方針の整理：**

1. **学習は Colab、推論はローカル**という分業を基本とする（§0.4）。
2. `vla_jepa` は 3060 でのローカル推論に挑戦する価値がある。まず `lerobot/VLA-JEPA-LIBERO` のような公開チェックポイントで試す。
3. `fastwam` / `lingbot_va` は**推論も Colab 側で行う**。`--policy.save_predicted_video=true`（`lingbot_va`）で予測動画を書き出せば、世界モデルが何を「想像」しているかは Colab 上でも十分に観察できる。
4. ローカル 3060 は「SO-101 でデータを録り、ACT を推論して実機を動かす」役割に置く。ここは 6 GB で十分に成立する。

### 0.3 Colab で学習する場合のランタイム選択

Colab のランタイムごとの VRAM と、載るポリシーの対応は以下である（学習時、AdamW 既定）。

| Colab ランタイム | VRAM | 学習できる範囲 |
| --- | ---: | --- |
| T4（無料枠） | 15 GB | `act`, `diffusion`, `vqbet`, `smolvla`(BS 小), `multi_task_dit` |
| L4（Pro） | 22.5 GB | 上記 + `xvla` / `evo1`、`vla_jepa` の `freeze_qwen` ファインチューン |
| A100 40 GB（Pro+） | 40 GB | `pi0` / `pi05` / `groot` / `eo1`、`vla_jepa` のフル学習 |
| — | 80 GB 級が必要 | **`lingbot_va`（LoRA 必須）/ `fastwam`（公式再現は 1×H20 140 GB）** |

**注意点：**

- **`lingbot_va` と `fastwam` は Colab の A100 40 GB でも厳しい。** `lingbot_va` は公式に「単一 24–32 GB GPU に載らない、LoRA (`--policy.use_peft=true`) と optimizer offload を使え」と書かれている。A100 40 GB + LoRA + `--batch_size=1` が最低ラインで、それでも通る保証はない。この 2 つは Hugging Face Jobs（`--job.target=a100-large` 等）か、より大きなクラウド GPU を検討したほうが早い。
- **Colab は Linux なので §0.4 の Windows 制約（flex-attention / LIBERO）を受けない。** `lingbot_va` の学習に必要な `--policy.attn_mode=flex` も Colab なら動く。この点で「学習は Colab」という方針は Windows 環境と非常に相性が良い。
- **セッション切断対策**：Colab は数時間で切れる。`--save_freq` を短めにし、`--output_dir` を Google Drive にマウントするか、`--policy.repo_id` を指定して Hub へ随時プッシュする。`--resume=true` で再開できる。
- **`--policy.scheduler_decay_steps`** を `--steps` に合わせること。短く打ち切ると学習率が減衰しないまま終わる。

> **`vla_jepa` の落とし穴**：VRAM 削減のために `--policy.freeze_qwen=true` を指定すると、`enable_world_model` が**エラーも警告もなく `False` に落とされる**（`configuration_vla_jepa.py` の `__post_init__`）。Qwen を凍結すると世界モデル側に勾配が流れないためである。**節約した瞬間に世界モデルの学習は行われていない。** 世界モデルそのものを実験したいなら凍結は選べず、A100 40 GB 級が要る。

### 0.4 世界モデルを Windows で扱う場合の注意（結論：WSL2 推奨）

世界モデル系ポリシーに限っては、**ネイティブ Windows ではなく WSL2 を使うべきである**。理由は 3 点ある。

1. **`lingbot_va` の学習には flex-attention が必須で、ネイティブ Windows では動かない。**
   公式ドキュメントは学習時に `--policy.attn_mode=flex` を要求する（既定の `torch` SDPA は推論専用）。実装（`src/lerobot/policies/lingbot_va/utils.py`）は `torch.compile(flex_attention)` を呼ぶが、`torch.compile` の CUDA バックエンドは **Triton** に依存し、Triton は公式には Windows をサポートしていない。したがってネイティブ Windows では `lingbot_va` の学習経路が成立しない。

2. **3 つの世界モデルの標準評価環境である LIBERO が Linux 限定である。**
   `pyproject.toml` の `libero` extra は `hf-libero>=0.1.4,<0.2.0; sys_platform == 'linux'` というマーカーを持つ。ネイティブ Windows では依存が黙って skip され、`--env.type=libero` が使えない。`lingbot_va` のもう一つの評価環境である RoboTwin 2.0 も SAPIEN + CuRobo が必要で、公式は Docker（Linux）を前提としている。

3. **CPU オフロード・LoRA・大容量チェックポイント周りの実績が Linux に偏っている。**
   `lingbot_va` は UMT5-XXL テキストエンコーダを既定で CPU に置く（`config.text_encoder_device`）構成であり、ホスト RAM を 32 GB 以上消費する。`accelerate` の CPU オフロードや `peft` の挙動も Linux での検証が厚い。

一方で、`fastwam` は **FlashAttention を使わず PyTorch の SDPA のみで動く**と明記されており（`flash-attn` パッケージを入れても効果はない）、この点に関しては Windows でも障害にならない。ただし上記 2.（LIBERO）が効くため、評価まで通すなら結局 WSL2 が必要である。

**加えて Windows 固有の実務上の注意：**

- **ディスクとキャッシュ**：3 モデルとも 5–20 GB のバックボーンを Hub から追加取得する。C ドライブが埋まるので §2.1 の `HF_HOME` 設定を必ず行う。WSL2 の場合も `/mnt/c/...` ではなく WSL 内のファイルシステムに置く。
- **長いパス**：Wan2.2 や Qwen3-VL のキャッシュはディレクトリ階層が深い。§2.1 の `LongPathsEnabled` を有効にしていないと展開時に失敗しやすい。
- **`--eval.batch_size=1`**：`lingbot_va` は KV キャッシュを使うストリーミング推論のため単一環境評価のみ対応である。`fastwam` の公式再現コマンドも `--eval.batch_size=1` を使っている。Windows の非同期環境まわりのトラブルを回避する意味でも、この設定を守ること。

---

## 1. ネイティブ Windows と WSL2 の使い分け

LeRobot は Windows でも WSL2（Windows Subsystem for Linux）でも動作するが、得意分野が異なる。**両方を用意し、用途で使い分けるのが最も現実的である。**

| 用途 | ネイティブ Windows | WSL2 (Ubuntu) |
| --- | --- | --- |
| SO-101 等の USB シリアル接続（`lerobot-find-port`, `calibrate`, `teleoperate`, `record`） | ◎ COM ポートがそのまま見える | △ `usbipd-win` による USB パススルーが必要 |
| キーボード遠隔操作（`pynput` の全画面フック） | ◎ そのまま動作 | ✕ ヘッドレスのため不可 |
| USB カメラ（OpenCV / MSMF） | ◎ | ✕ パススルー必須で不安定 |
| ポリシー学習（`lerobot-train`） | ◎ CUDA 対応 | ◎ CUDA on WSL 対応 |
| `pusht` / `aloha` などの MuJoCo 系シミュレーション | ○ | ◎ |
| LIBERO / RoboMME / RoboCasa 等のベンチマーク | ✕ **Linux 限定** | ◎ |
| 世界モデル（`vla_jepa` / `lingbot_va` / `fastwam`） | ✕ **§0.4 参照。flex-attention と LIBERO が使えない** | ◎（ただし VRAM が別途必要） |
| Feetech 公式ファームウェア更新ツール | ◎ **Windows 専用ソフト** | ✕ |

**LIBERO が Linux 限定である根拠**：`pyproject.toml` の `libero` extra が `hf-libero>=0.1.4,<0.2.0; sys_platform == 'linux'` というマーカーを持つ。ネイティブ Windows では依存が黙って skip されるため実行できない。RoboMME（ManiSkill / Vulkan 依存）と RoboCasa も同様に Linux 限定である。

**結論**：実機ロボットとカメラを扱う作業はネイティブ Windows、Linux 限定ベンチマークは WSL2。学習はどちらでもよい（同一 GPU を使う）。

---

## 2. ネイティブ Windows へのインストール

### 2.1 事前準備

以下を PowerShell（管理者権限）で導入する。

```powershell
winget install --id Git.Git -e
winget install --id GitHub.GitLFS -e
winget install --id CondaForge.Miniforge3 -e
```

**conda を PowerShell から使えるようにする。** Miniforge は意図的に PATH を書き換えないため、インストール直後に `conda` と打っても `用語 'conda' は、コマンドレット、関数、スクリプト ファイル...として認識されません` となる。以下の手順で初期化する。

まず `conda.exe` の場所を特定する（インストールスコープにより 2 か所のどちらかになる）。

```powershell
@("$env:USERPROFILE\miniforge3", "$env:LOCALAPPDATA\miniforge3", "C:\ProgramData\miniforge3") |
  Where-Object { Test-Path "$_\Scripts\conda.exe" }
```

次に実行ポリシーを緩める。`conda init` は PowerShell プロファイルに初期化スクリプトを書き込むが、既定の `Restricted` のままだとそのプロファイルが読み込まれず、**初期化しても `conda` が使えないまま**になる。

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

フルパスで初期化する（上で見つかったパスに置き換える）。

```powershell
& "$env:USERPROFILE\miniforge3\Scripts\conda.exe" init powershell
```

**PowerShell を開き直す。** プロンプト先頭に `(base)` が付き、`conda --version` が通れば成功である。

**それでも `conda` が認識されない場合は、PATH に直接追加する。** ユーザー環境変数なので管理者権限は不要であり、レジストリの昇格エラーも起きない。

```powershell
$base = "$env:USERPROFILE\miniforge3"   # 上で見つかったパスに置き換える
$add  = "$base;$base\Scripts;$base\Library\bin"
$p    = [Environment]::GetEnvironmentVariable("Path", "User")
if ($p -notlike "*$base\Scripts*") {
    [Environment]::SetEnvironmentVariable("Path", "$p;$add", "User")
}
```

3 つのディレクトリをすべて追加する点が重要である。`Scripts` に `conda.exe` が、`Library\bin` に conda で入れた DLL 群（ffmpeg のライブラリなど）が置かれるため、片方だけでは後段で実行時エラーになる。

設定後は **PowerShell を開き直してから**確認する。

```powershell
conda --version
where.exe conda
```

> **補足 1**：スタートメニューの **「Miniforge Prompt」** を使えば初期化も PATH 追加も不要で conda が使える。手早く進めたいならこれでもよい。
> **補足 2**：`C:\ProgramData\miniforge3`（マシンスコープ）に入った場合、後の `conda create` で書き込み権限エラーが出ることがある。`winget uninstall --id CondaForge.Miniforge3` の後 `winget install --id CondaForge.Miniforge3 -e --scope user` で入れ直すほうが後々楽である。
> **補足 3**：PowerShell 7（`pwsh`）と Windows PowerShell 5.1 はプロファイルが別である。両方使うならそれぞれで `conda init powershell` を実行する。
> **補足 4**：毎回長いパスを打ちたくない場合は、PowerShell プロファイル（`$PROFILE`）に `function lr { conda activate lerobot; Set-Location D:\lerobot }` のような関数を定義しておくと `lr` の一語で済む。プロファイルの読み込みには実行ポリシー `RemoteSigned` が必要である。

**長いパスの有効化**（推奨）。PyTorch や Hugging Face のキャッシュはパスが深くなりやすく、260 文字制限に当たることがある。

> **必ず管理者権限の PowerShell で実行すること。** `HKLM`（HKEY_LOCAL_MACHINE）への書き込みには昇格が必要であり、通常のウィンドウでは `要求されたレジストリ アクセスは許可されません` というエラーになる。スタートメニューで PowerShell を右クリック →「管理者として実行」、または `Start-Process powershell -Verb RunAs` で開き直す（タイトルバーに「管理者」と表示されていることを確認する）。

```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

`reg` コマンドでも同じ結果になる。既に値が存在する場合はこちらのほうが確実である。

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" /v LongPathsEnabled /t REG_DWORD /d 1 /f
```

確認（一般権限でも可）。`LongPathsEnabled : 1` が返れば成功である。**反映には再起動が必要**である。

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name LongPathsEnabled
```

**管理者権限が使えない場合**（会社支給 PC でグループポリシーによりブロックされている等）は、レジストリを触らずに「パスの起点を短くする」ことで回避する。次項の `HF_HOME` を `D:\hf` のような浅いパスに設定し、リポジトリも `C:\Users\<長いユーザー名>\Documents\...` ではなく `D:\lerobot` のような場所に置く。起点が短ければ深い階層でも 260 文字に届きにくい。

**Hugging Face キャッシュの移動**（任意だが強く推奨）。データセットとモデルで数十 GB になるため、C ドライブ以外に逃がしておく。世界モデル系を扱うなら 1 モデルあたり 5–20 GB を要するので事実上必須である。

```powershell
[Environment]::SetEnvironmentVariable("HF_HOME", "D:\hf", "User")
```

設定後は PowerShell を開き直す。反映確認は `$env:HF_HOME` で行う。

### 2.2 リポジトリの取得

本リポジトリはサブモジュール構成である。

```powershell
git clone --recurse-submodules <このリポジトリのURL> lerobot-v0.6.0
cd lerobot-v0.6.0
git lfs install
```

クローン済みの場合は次で十分である。

```powershell
cd lerobot-v0.6.0
git submodule update --init --recursive
```

### 2.3 Python 環境の作成

```powershell
conda create -y -n lerobot python=3.12
conda activate lerobot
```

> 以降、新しいシェルを開くたびに `conda activate lerobot` が必要である。

### 2.4 ffmpeg の導入（動画デコード用）

LeRobot はデータセット動画のデコードに **TorchCodec** を使う。TorchCodec は Windows 向けホイールを 0.7.0 以降で提供しており（`torch>=2.8` が必要）、本バージョンの LeRobot は `torch>=2.7,<2.12` を要求して実際には 2.11 系が入るため、**Windows でも TorchCodec が使える**。その前提として ffmpeg が必要である。

conda 環境に入れるのが最も確実である。

```powershell
conda install -y ffmpeg -c conda-forge
```

**`PackagesNotFoundInChannelsError` が出た場合は、バージョンを明示指定すればインストールできる。** バージョン指定なしだと conda が win-64 に存在しない組み合わせを解こうとして失敗することがあるためである。まず Windows 向けに実在するバージョンを一覧する。

```powershell
conda search -c conda-forge "ffmpeg" --subdir win-64
```

`--subdir win-64` を付けるのが要点である。これを省くと現在のプラットフォーム以外のビルドも混ざり、実際には入らないバージョンを選んでしまう。一覧の中から新しめのものを選び、明示的に指定する。

```powershell
conda install -y -c conda-forge ffmpeg=<一覧に出たバージョン>
```

導入後は encoder の有無を確認する。`libsvtav1` が見つからない場合は、一覧から別のバージョンを選び直す。

```powershell
ffmpeg -version
ffmpeg -encoders | Select-String "svtav1"
```

> **どうしても conda で入らない場合の代替が 2 つある。**
>
> **A. winget でシステム全体に入れる。** torch 2.11（TorchCodec 0.10 以降）はシステム全体の ffmpeg に動的リンクできる。
>
> ```powershell
> winget install --id Gyan.FFmpeg -e
> ```
>
> インストール後は PowerShell を開き直し、`ffmpeg -version` が通ることを確認する。
>
> **B. ffmpeg を使わず PyAV バックエンドで動かす。** LeRobot のデコーダは `torchcodec` と `pyav` の 2 系統があり、`pyav`（`av` パッケージ）は **Windows ホイールに ffmpeg 本体が同梱されている**ため、外部の ffmpeg が一切不要である。`av` は `dataset` extra に含まれるので追加インストールもいらない。
>
> ```powershell
> lerobot-train --dataset.video_backend=pyav ...
> ```
>
> デコードは torchcodec より遅いが、SO-101 のデータ記録と ACT の学習では実用上ほとんど問題にならない。なお `get_safe_default_video_backend()` が pyav へ自動フォールバックするのは **torchcodec が未インストールの場合のみ**である。torchcodec が入っていて ffmpeg だけが無い状態では自動では逃げないため、上記のように明示指定する必要がある。

### 2.5 PyTorch（CUDA 版）の導入

**ここが Windows で最も間違えやすい箇所である。** 先に PyTorch を明示的に入れてから LeRobot を入れる。

LeRobot は `torch>=2.7,<2.12.0` を要求するため、入るのは **torch 2.11 系**である。torch 2.11 の PyPI 既定ホイールは CUDA 13.0 ビルド（cu130）であり、CUDA 13 系はマイナーバージョン間で互換性があるため、**CUDA 13.2 対応ドライバの環境でそのまま動作する**。cu132 ホイールは torch 2.12 以降にしか存在せず、LeRobot の上限に抵触するので使わない。

確実を期すため、インデックスを明示して入れる。

```powershell
pip install --index-url https://download.pytorch.org/whl/cu130 torch torchvision
```

ドライバが古く cu130 が動かない場合のみ、フォールバックとして cu128 を使う（ドライバ下限 570.86）。

```powershell
pip install --index-url https://download.pytorch.org/whl/cu128 torch torchvision
```

導入確認。

```powershell
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

`True` と `NVIDIA GeForce RTX 3060` が出れば成功である。`False` の場合は §7 を参照。

### 2.6 LeRobot 本体の導入

サブモジュールのディレクトリで editable インストールする。用途に応じて extras を選ぶ。

```powershell
cd lerobot

# 実機ロボット操作 + 学習 + SO-101（Feetech モーター）
pip install -e ".[core_scripts,training,feetech]"

# シミュレーション評価も行う場合は追加
pip install -e ".[pusht,aloha]"

# 全部入り（重い。依存衝突も起きやすいので通常は非推奨）
# pip install -e ".[all]"
```

extras の意味は以下のとおりである。

| extra | 追加されるもの | 用途 |
| --- | --- | --- |
| `dataset` | `datasets`, `av`, `torchcodec`, `jsonlines` | データセットの読み書き |
| `training` | `dataset` + `accelerate`, `wandb` | `lerobot-train` |
| `hardware` | `pynput`, `pyserial`, `deepdiff` | 実機接続 |
| `viz` | `rerun-sdk`, `foxglove-sdk` | 記録・評価中の可視化 |
| `core_scripts` | `dataset` + `hardware` + `viz` | `lerobot-record` / `replay` / `calibrate` |
| `feetech` | Feetech SDK | SO-100 / SO-101 / Moss |
| `dynamixel` | Dynamixel SDK | Koch v1.1 |

> **注意**：`pip install -e ".[core_scripts]"` の `torch` 再解決で CPU 版に置き換わることがある。インストール後は必ず §2.5 の確認コマンドを再実行すること。置き換わっていたら §2.5 のコマンドを `--force-reinstall` 付きで打ち直す。

### 2.7 Hugging Face へのログイン

データセットや学習済みポリシーを Hub にアップロードする場合に必要である。

```powershell
hf auth login
```

### 2.8 動作確認

```powershell
lerobot-info
python -c "import lerobot; print(lerobot.__version__)"
```

---

## 3. 実機ロボット（SO-101 等）の接続

実機まわりはネイティブ Windows で行う。

### 3.1 モーターのファームウェア設定

Feetech の公式設定ソフト（FD / FT SCServo Debug）は **Windows 専用**である。モーター ID の初期設定やファームウェア更新はこのソフトで行う。詳細は `lerobot/docs/source/feetech.mdx` を参照。

### 3.2 ポートの特定

Windows ではシリアルポートが `COM3`、`COM4` のような名前になる（Linux の `/dev/ttyACM0` に相当）。

```powershell
lerobot-find-port
```

指示に従ってケーブルを抜き差しすると、該当する COM ポートが表示される。うまくいかない場合はデバイスマネージャーの「ポート (COM と LPT)」を直接確認する。

### 3.3 セットアップからテレオペまで

```powershell
lerobot-setup-motors --robot.type=so101_follower --robot.port=COM3
lerobot-setup-motors --teleop.type=so101_leader  --teleop.port=COM4

lerobot-calibrate --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower
lerobot-calibrate --teleop.type=so101_leader  --teleop.port=COM4 --teleop.id=my_leader

lerobot-teleoperate `
  --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower `
  --teleop.type=so101_leader  --teleop.port=COM4 --teleop.id=my_leader `
  --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" `
  --display_data=true
```

> PowerShell の行継続はバックスラッシュ `\` ではなくバッククォート `` ` `` である。1 行で書いてしまうのが最も安全である。

### 3.4 カメラの確認

```powershell
lerobot-find-cameras opencv
```

Windows の標準「カメラ」アプリは仮想カメラに対応していないため、動作確認にはこのコマンドを使う。なお LeRobot は Windows 上で `OPENCV_VIDEOIO_MSMF_ENABLE_HW_TRANSFORMS` を自動設定し、MSMF バックエンドの互換問題を回避している。

### 3.5 データ記録

```powershell
lerobot-record --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower --teleop.type=so101_leader --teleop.port=COM4 --teleop.id=my_leader --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" --dataset.repo_id=<HF_USER>/my_task --dataset.single_task="タスクを1文で説明" --dataset.num_episodes=50 --dataset.episode_time_s=30 --dataset.reset_time_s=10 --display_data=true
```

記録中のキー操作は **→**（次へ）、**←**（やり直し）、**ESC**（終了してアップロード）である。Windows デスクトップ上では `pynput` のグローバルフックが特別な権限なしに動作するため、キー操作はそのまま効く。

---

## 4. ポリシーの学習と推論

**基本方針は「学習は Colab / HF Jobs、推論はローカル 3060」である**（§0.1–0.3）。ここではローカルでも学習したい場合の設定と、ローカル推論の手順を扱う。

### 4.1 ローカル（6 GB）で学習できる範囲

VRAM 6 GB は LeRobot のポリシー群の中では小さい部類である。公式ガイド（`lerobot/AGENT_GUIDE.md` §6.2）でも **8 GB 未満は `act` 推奨**とされている。

| ポリシー | 6 GB でのローカル学習 | 6 GB でのローカル推論 | 備考 |
| --- | --- | --- | --- |
| `act` | ◎ | ◎ | 第一候補。単一タスクの把持・配置なら費用対効果が最良 |
| `diffusion` | △ | ◎ | バッチを絞れば学習も可能。学習ステップは ACT より多く必要 |
| `smolvla` | ✕ | ○ | 学習は Colab T4 以上で。推論はローカルで問題ない |
| `xvla` / `evo1` | ✕ | △ | 推論はぎりぎり。他の GPU 消費を止めれば載る可能性がある |
| `pi0` / `pi05` / `wall_x` / `groot` / `eo1` | ✕ | ✕ | 推論だけで 13–16 GB。Colab 側で完結させる |
| `vla_jepa` | ✕ | △ | **世界モデル系で唯一ローカル推論に望みがある**（§0.2） |
| `fastwam` / `lingbot_va` | ✕ | ✕ | 推論も 12–24 GB。学習・推論とも Colab / HF Jobs |

### 4.2 ローカル推論（学習済みチェックポイントを実機で動かす）

Colab で学習して Hub にプッシュしたポリシーは、`--policy.path` で直接参照できる。

```powershell
lerobot-record --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" --dataset.repo_id=<HF_USER>/eval_my_task --dataset.single_task="学習時と同じタスク説明" --dataset.num_episodes=10 --policy.path=<HF_USER>/act_my_task
```

連続稼働させるだけなら `lerobot-rollout` を使う。

```powershell
lerobot-rollout --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower --policy.path=<HF_USER>/act_my_task
```

推論で VRAM が足りない場合の対処は 3 つである。

- `--policy.use_amp=true` を付ける（`use_amp` は評価時にも適用される）。
- `fastwam` / `vla_jepa` に限り `--policy.torch_dtype=bfloat16` が使える。他のポリシーには dtype 指定が存在しない。
- ディスプレイ出力を CPU 内蔵 GPU 側に切り替え、3060 の VRAM を推論だけに使う。数百 MB の差が効く。

### 4.3 ローカル学習コマンド（ACT）

```powershell
lerobot-train --dataset.repo_id=<HF_USER>/my_task --policy.type=act --policy.device=cuda --policy.use_amp=true --batch_size=8 --num_workers=2 --output_dir=outputs/train/act_my_task --job_name=act_my_task --wandb.enable=true --policy.repo_id=<HF_USER>/act_my_task
```

### 4.4 6 GB で効くチューニング

- `--policy.use_amp=true` — 混合精度。既定は `false` なので必ず明示する。VRAM 削減効果が大きい。
- `--batch_size=8` — 既定値は 8。OOM が出たら 4 → 2 と下げる。なお本バージョンの `lerobot-train` に勾配累積オプションは存在しないため、バッチを下げた場合は `--steps` を増やして補う。
- `--num_workers=2` — 既定は 4。Windows の DataLoader は `spawn` 方式でプロセス生成コストが高く、ワーカー数だけメモリを消費する。エラーやハングが出る場合は `--num_workers=0` にする（`prefetch_factor` と `persistent_workers` は自動的に無効化されるので副作用はない）。
- `--prefetch_factor=2` — 既定は 4。ホスト RAM が 16 GB 程度なら下げておくと安定する。
- 学習中は他の GPU 消費アプリ（ブラウザのハードウェアアクセラレーション、ゲームランチャー等）を閉じる。6 GB では 500 MB の差が効く。
- 学習ステップ数の目安：50 エピソード × 30 秒 @ 30fps ≒ 45,000 フレーム。`batch_size=8` なら 1 エポック ≒ 5,625 ステップ、5〜10 エポックで 28k〜56k ステップである。

### 4.5 Colab / HF Jobs へのオフロード

**Colab で学習する場合**は、Colab のノートブック内で LeRobot をインストールして `lerobot-train` を実行する（Colab は Linux なので §0.4 の Windows 制約を受けない）。公式のノートブック例は `lerobot/docs/source/notebooks.mdx` にある。セッション切断に備え、以下は必ず付ける。

```bash
lerobot-train --dataset.repo_id=<HF_USER>/my_task --policy.type=smolvla --policy.device=cuda --batch_size=4 --steps=30000 --policy.scheduler_decay_steps=30000 --save_freq=2000 --policy.repo_id=<HF_USER>/smolvla_my_task --output_dir=/content/drive/MyDrive/lerobot/outputs
```

再開は同じ `--output_dir` に対して `--resume=true` を付ける。

**Hugging Face Jobs を使う場合**は、ローカル Windows からそのまま投げられる。ノートブックを開いたまま待つ必要がなく、切断もしない。

```powershell
lerobot-train --dataset.repo_id=<HF_USER>/my_task --policy.type=smolvla --job.target=a10g-small
```

利用可能なフレーバーは `hf jobs hardware` で確認する。`--job.timeout=8h` でタイムアウトを調整できる（既定は 48 時間）。世界モデル系なら `a100-large` 以上を選ぶ。

---

## 5. シミュレーション評価

### 5.1 ネイティブ Windows で動くもの

```powershell
pip install -e ".[pusht]"   # gym-pusht
pip install -e ".[aloha]"   # gym-aloha
lerobot-eval --policy.path=<HF_USER>/act_my_task --env.type=pusht --eval.n_episodes=50 --eval.batch_size=4
```

`--eval.batch_size` は同時に走らせる環境数である。既定の `0` は CPU コア数から自動決定するが、Windows では並列環境がプロセス生成のオーバーヘッドで不安定になることがある。その場合は明示的に小さい値を指定するか、`--eval.use_async_envs=false` で同期実行に切り替える。

### 5.2 WSL2 が必要なもの

`libero`、`robomme`、`robocasa` は Linux 限定である。§6 の手順で WSL2 環境を作り、その中で導入する。導入手順は `lerobot/docs/source/` 配下の対応する `.mdx`（`robomme.mdx`、`robocasa.mdx`、`vlabench.mdx`）に個別のレシピがある。

### 5.3 世界モデル系ポリシーを実験する（WSL2 または Colab）

推論 VRAM の壁（§0.2）は前提として、環境構築自体は WSL2 内で以下のように行う。まずは**学習ではなく、公開チェックポイントの推論・評価から始める**のが安全である。

**どこで動かすかの判断：**

| ポリシー | ローカル 3060（WSL2） | Colab |
| --- | --- | --- |
| `vla_jepa` の評価 | △ 挑戦する価値あり（約 5–7 GB） | ◎ T4 でも余裕 |
| `fastwam` の評価 | ✕ | ○ L4 / A100 |
| `lingbot_va` の評価 | ✕ | ○ A100 40 GB |
| 3 種すべての学習 | ✕ | §0.3 のとおり `lingbot_va` / `fastwam` は A100 でも厳しい |

なお LIBERO は Linux 限定なので、ローカルで評価する場合は必ず WSL2 側で実行する。Colab は Linux なのでこの制約を受けない。

```bash
# WSL2 の Ubuntu 内で、conda 環境を有効化した状態で
cd ~/lerobot-v0.6.0/lerobot

pip install -e ".[vla_jepa,libero]"     # VLA-JEPA
pip install -e ".[lingbot_va,libero]"   # LingBot-VA
pip install -e ".[fastwam,libero]"      # FastWAM
```

**VLA-JEPA の評価**（公開 LIBERO チェックポイント）:

```bash
lerobot-eval --policy.path=lerobot/VLA-JEPA-LIBERO --env.type=libero --env.task=libero_10 --eval.n_episodes=10 --eval.batch_size=5
```

**LingBot-VA の評価**（`--eval.batch_size=1` 固定）:

```bash
lerobot-eval --policy.path=lerobot/lingbot_va_libero_long --policy.device=cuda --env.type=libero --env.task=libero_10 --env.observation_height=128 --env.observation_width=128 --eval.n_episodes=50 --eval.batch_size=1 --output_dir=outputs/eval/lingbot_va_libero
```

`--policy.save_predicted_video=true` を付けると、モデルが「想像」した予測動画を `pred_episode_*.mp4` として出力する。世界モデルの挙動を観察したいなら最も分かりやすい入口である。

**FastWAM の評価**:

```bash
lerobot-eval --policy.path=ZibinDong/fastwam_libero_uncond_2cam224 --policy.device=cuda --policy.torch_dtype=float32 --policy.n_action_steps=10 --env.type=libero --env.task=libero_10 --env.observation_height=224 --env.observation_width=224 --eval.batch_size=1 --eval.n_episodes=50 --seed=0 --env.episode_length=600
```

**学習・ファインチューニングを行う場合の必須フラグ：**

| ポリシー | 必須／推奨フラグ | 理由 |
| --- | --- | --- |
| `vla_jepa` | `--policy.freeze_qwen=true` | Qwen3-VL backbone を凍結し行動ヘッドのみ学習。VRAM を大幅に削減できる |
| `vla_jepa` | `--policy.reinit_modules='[...]'` | ロボットの action/state 次元がチェックポイントと異なる場合に必要 |
| `lingbot_va` | `--policy.attn_mode=flex` | 学習時のブロック因果マスクに必須。**ネイティブ Windows では動かない** |
| `lingbot_va` | `--policy.use_peft=true` | 5B フル学習は 24–32 GB GPU に載らないため LoRA が必須 |
| `lingbot_va` | `--batch_size=1` | 公式例の値 |
| `fastwam` | `--policy.image_size='[224,448]'` | 各カメラ画像の幅の合計と一致させる必要がある |

> **`vla_jepa` の落とし穴**：`--policy.freeze_qwen=true` を指定すると、`enable_world_model` は **エラーにならず暗黙に `False` に落とされる**（`configuration_vla_jepa.py` の `__post_init__`）。Qwen を凍結すると世界モデル側に勾配が流れず JEPA 損失が無意味になるためである。つまり **VRAM 削減のために Qwen を凍結した時点で、世界モデルの学習は行われていない。** 世界モデルそのものを実験したいなら凍結は使えず、必然的に大容量 VRAM が要る。ログで `enable_world_model` の最終値を必ず確認すること。

---

## 6. WSL2（Ubuntu）へのインストール

### 6.1 WSL2 の準備

管理者権限の PowerShell で実行する。

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
```

再起動後、Ubuntu を起動してユーザーを作成する。GPU は Windows 側の NVIDIA ドライバがそのまま使われるため、**WSL 内に NVIDIA ドライバを入れてはならない**。確認は以下で行う。

```bash
nvidia-smi
```

### 6.2 環境構築

```bash
sudo apt update
sudo apt install -y build-essential cmake pkg-config git git-lfs ffmpeg libevdev-dev \
  libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev libswscale-dev \
  libswresample-dev libavfilter-dev

# miniforge
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh

conda create -y -n lerobot python=3.12
conda activate lerobot
conda install -y evdev -c conda-forge   # WSL では必須
```

### 6.3 PyTorch と LeRobot

WSL2 は Linux 扱いになるため、`pyproject.toml` の `tool.uv.sources` により **uv 経由では cu128 インデックスが既定**となる。CUDA 13.2 ドライバでも cu128 は動作するが、cu130 を使いたい場合は明示的に上書きする。

```bash
# pip の場合
pip install --index-url https://download.pytorch.org/whl/cu130 torch torchvision
cd /mnt/c/path/to/lerobot-v0.6.0/lerobot   # または WSL 内にクローンし直す
pip install -e ".[core_scripts,training,libero]"

# uv の場合（既定は cu128、上書きするなら）
uv pip install --force-reinstall torch torchvision --index-url https://download.pytorch.org/whl/cu130
```

> **性能上の注意**：`/mnt/c/...` 越しのファイル I/O は非常に遅い。学習データを扱うなら、リポジトリとデータセットは WSL 内のファイルシステム（`~/lerobot-v0.6.0` など）に置くこと。

### 6.4 WSL2 から USB シリアルを使う場合

どうしても WSL 側から実機に繋ぎたい場合は `usbipd-win` を使う。ただし遅延と切断のリスクがあるため、**実機操作はネイティブ Windows で行うことを推奨する**。

```powershell
winget install dorssel.usbipd-win
usbipd list
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

---

## 7. トラブルシューティング

| 症状 | 原因と対処 |
| --- | --- |
| `torch.cuda.is_available()` が `False` | LeRobot のインストール時に CPU 版 torch へ置き換わった。§2.5 のコマンドを `--force-reinstall` 付きで再実行する |
| `sm_86 is not compatible` 等のアーキテクチャエラー | CPU 版または古い CUDA ビルドが入っている。cu130 ホイールを入れ直す |
| `CUDA out of memory` | `--batch_size` を下げる、`--policy.use_amp=true` を付ける、`act` に切り替える、他の GPU 使用アプリを閉じる |
| `torchcodec` の import に失敗 | ffmpeg が無い／バージョン不一致。`conda install ffmpeg=7.1.1 -c conda-forge` を試す。それでも駄目なら LeRobot は自動的に `pyav` にフォールバックする |
| DataLoader がハング／`spawn` 関連のエラー | `--num_workers=0` にする |
| `lerobot-find-port` がポートを見つけない | USB-シリアルドライバ（CH340 / CP210x 等）が未導入。デバイスマネージャーで不明なデバイスを確認する |
| ビルドエラー（`cmake` / `av` のコンパイル） | conda 環境で `conda install -c conda-forge cmake ffmpeg` を先に入れる。Visual Studio Build Tools が必要な場合は `winget install Microsoft.VisualStudio.2022.BuildTools` |
| パス長エラー（`OSError: [Errno 2]` 等） | §2.1 の `LongPathsEnabled` を設定する（要再起動）。設定できないなら `HF_HOME` とリポジトリを浅いパスに移す |
| `要求されたレジストリ アクセスは許可されません` | 管理者権限の PowerShell で実行していない。`Start-Process powershell -Verb RunAs` で開き直す（§2.1） |
| `'conda' は、コマンドレット、関数、スクリプト ファイル...として認識されません` | Miniforge は PATH を書き換えない。§2.1 の手順で `conda init powershell` を**フルパスで**実行し、シェルを開き直す。それでも駄目なら §2.1 の PATH 追加スクリプト（`miniforge3`、`Scripts`、`Library\bin` の 3 つ）を実行する |
| `PackagesNotFoundInChannelsError`（ffmpeg 等） | バージョン無指定だと win-64 に無い組み合わせを解こうとして失敗する。`conda search -c conda-forge "ffmpeg" --subdir win-64` で実在するバージョンを調べ、`ffmpeg=<バージョン>` と明示指定する（§2.4） |
| `conda init` 済みなのに `conda` が使えない | 実行ポリシーが `Restricted` で PowerShell プロファイルが読み込まれていない。`Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` を実行する |
| `conda create` で書き込み権限エラー | `C:\ProgramData\miniforge3`（マシンスコープ）に入っている。`--scope user` で入れ直す（§2.1 補足 2） |
| `pip install ".[core_scripts]"` が構文エラーになる | PowerShell では角括弧を必ずダブルクォートで囲む |
| LIBERO 系のパッケージが入らない | ネイティブ Windows では `sys_platform == 'linux'` マーカーにより意図的に除外されている。WSL2 を使う |
| `lingbot_va` 学習時に Triton / `torch.compile` 関連のエラー | `--policy.attn_mode=flex` が Triton を要求するが Windows 非対応。WSL2 か Colab で実行する（§0.4） |
| 世界モデルの**推論**で `CUDA out of memory` | §0.2 参照。`vla_jepa` なら `--policy.torch_dtype=bfloat16` とディスプレイ出力の内蔵 GPU への切り替えを試す。`fastwam` / `lingbot_va` は 6 GB では諦めて Colab を使う |
| Wan2.2 / Qwen3-VL のダウンロードで C ドライブが枯渇 | `HF_HOME` を別ドライブに設定する（§2.1）。1 モデルあたり 5–20 GB を要する |

---

## 8. 用途別クイックリファレンス

```powershell
# --- 最小構成の確認だけしたい ---
conda create -y -n lerobot python=3.12; conda activate lerobot
pip install --index-url https://download.pytorch.org/whl/cu130 torch torchvision
pip install lerobot

# --- SO-101 を動かす（ネイティブ Windows） ---
conda install -y ffmpeg -c conda-forge
pip install -e ".[core_scripts,feetech]"

# --- ローカルで推論する（学習済みポリシーを実機で動かす）---
lerobot-rollout --robot.type=so101_follower --robot.port=COM3 --robot.id=my_follower --policy.path=<HF_USER>/act_my_task

# --- ローカルで学習する（ACT のみ推奨 / RTX 3060 6GB） ---
pip install -e ".[training]"
lerobot-train --policy.type=act --policy.device=cuda --policy.use_amp=true --batch_size=8 --num_workers=2 ...

# --- シミュレーション評価（Windows 可） ---
pip install -e ".[pusht,aloha]"

# --- LIBERO / RoboMME（WSL2 必須） ---
wsl
pip install -e ".[libero]"

# --- 世界モデルをローカル推論（vla_jepa のみ現実的 / WSL2） ---
wsl
pip install -e ".[vla_jepa,libero]"
lerobot-eval --policy.path=lerobot/VLA-JEPA-LIBERO --env.type=libero --env.task=libero_10 --eval.n_episodes=10 --eval.batch_size=5
```

### 役割分担のまとめ

| やること | 場所 | 理由 |
| --- | --- | --- |
| データ記録・キャリブレーション・実機テレオペ | **ネイティブ Windows** | COM ポート・カメラ・`pynput` がそのまま動く |
| `act` の学習 | ネイティブ Windows（6 GB で十分） | 約 1–3 GB で収まる |
| `smolvla` 以上の学習 | **Colab** | T4 15 GB 以上が要る |
| `vla_jepa` の学習 | **Colab（L4 以上）** | 凍結なら L4、フル学習は A100 40 GB |
| `lingbot_va` / `fastwam` の学習 | **HF Jobs / 大型クラウド GPU** | A100 40 GB でも厳しい（§0.3） |
| 学習済みポリシーの実機推論 | **ネイティブ Windows** | `act` / `smolvla` は 6 GB で動く |
| `vla_jepa` の推論・評価 | ローカル WSL2（挑戦）または Colab | 約 5–7 GB。LIBERO は Linux 限定 |
| `lingbot_va` / `fastwam` の推論・評価 | **Colab** | 12–24 GB 必要 |

---

## 参考

- `lerobot/docs/source/installation.mdx` — 公式インストール手順
- `lerobot/AGENT_GUIDE.md` — SO-101 の一連の手順、ポリシー選択・学習時間の指針
- `lerobot/docs/source/feetech.mdx` — Feetech モーターの Windows 専用設定手順
- `lerobot/docs/source/cameras.mdx` — カメラ設定と Windows 固有の注意点
- `lerobot/docs/source/hardware_guide.mdx` — ポリシー別 VRAM とクラウド GPU の選定指針
- `lerobot/docs/source/vla_jepa.mdx` / `lingbot_va.mdx` / `fastwam.mdx` — 世界モデル系ポリシーの詳細
- `lerobot/pyproject.toml` — extras とプラットフォームマーカーの定義（一次情報）
