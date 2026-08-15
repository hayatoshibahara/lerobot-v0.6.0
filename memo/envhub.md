# EnvHub — Hugging Face Hub からシミュレーション環境を読み込む仕組み

EnvHub は LeRobot v0.5.0 で導入された機能で、**シミュレーション環境を Python パッケージではなく Hugging Face Hub のリポジトリとして配布する**ための仕組みである。利用者は 1 行のコードで Hub 上の環境を読み込み、`lerobot-eval` や自作スクリプトからそのまま使える。

実装は `lerobot/src/lerobot/envs/` にあり、初出は 2025年11月4日の [PR #2121](https://github.com/huggingface/lerobot/pull/2121)（`feat(sim): EnvHub - allow loading envs from the hub`）である。

## 何の役に立つのか

従来、新しいシミュレーション環境を他人に使ってもらうには、PyPI パッケージにするか、LeRobot 本体へ PR を出して取り込んでもらう必要があった。どちらも手間が大きく、環境ごとに依存ライブラリが衝突しやすい。EnvHub はこの流れを次のように変える。

| 課題 | EnvHub での解決 |
| --- | --- |
| 環境の配布に PyPI 公開やパッケージングが必要 | Hub リポジトリに `env.py` を 1 枚置くだけで配布できる |
| LeRobot 本体に取り込まないと使えない | 本体を変更せず、第三者が自由に環境を公開できる |
| 環境のバージョン管理が曖昧 | Git のブランチ・タグ・コミットハッシュで固定できる |
| 環境ごとの依存が衝突する | 使う環境の分だけ手元に入れればよい（LeRobot 本体の依存は増えない） |
| 論文の実験を再現できない | コミットハッシュを指定して同一の環境を再現できる |

実際の応用として、次のような使われ方をしている。

- **NVIDIA IsaacLab Arena**：`nvidia/isaaclab-arena-envs` を EnvHub 経由で読み込み、`lerobot-eval` で SmolVLA や PI0.5 を評価する（[envhub_isaaclab_arena.mdx](./lerobot/docs/source/envhub_isaaclab_arena.mdx)）。
- **LeIsaac（LightWheel）**：**SO-101 の Leader / Follower** を IsaacLab 上で動かし、テレオペでデモを収集して学習まで回す（[envhub_leisaac.mdx](./lerobot/docs/source/envhub_leisaac.mdx)）。実機がなくても SO-101 のワークフローを試せるため、本ワークショップとの関連が最も深い。
- **Unitree G1 のシミュレーション**：`lerobot/src/lerobot/robots/unitree_g1/unitree_g1.py:302` で、シミュレーションモード時に `make_env("lerobot/unitree-g1-mujoco", trust_remote_code=True)` を呼んでいる。ロボットクラスの内部から EnvHub を使う例である。

つまり EnvHub は「環境を配る側」と「環境を試す側」の両方の摩擦を下げる機能であり、シミュレーション環境が **モデルやデータセットと同じように Hub 上で共有される資産になる**という位置づけである。

## 対応しているシミュレーション環境

LeRobot で使えるシミュレーション環境は、**EnvHub 経由のもの**と**本体に組み込まれているもの**の 2 系統に分かれる。

### EnvHub 経由（Hub 上のリポジトリ）

公式ドキュメントとコードから確認できるのは次の 5 つである。EnvHub は誰でも公開できる仕組みなので、これは「公式に案内されているもの」であって上限ではない。

| Hub リポジトリ | 中身 | シミュレータ | 参照 |
| --- | --- | --- | --- |
| `LightwheelAI/leisaac_env` | **SO-101** のピック・持ち上げ・片付け・布たたみ（4 タスク） | IsaacLab | [envhub_leisaac.mdx](./lerobot/docs/source/envhub_leisaac.mdx) |
| `nvidia/isaaclab-arena-envs` | GR-1 などのヒューマノイド操作。環境・対象物を設定で切り替える | IsaacLab / Isaac Sim | [envhub_isaaclab_arena.mdx](./lerobot/docs/source/envhub_isaaclab_arena.mdx) |
| `LightwheelAI/lw_benchhub_env` | LW-BenchHub（Lightwheel のタスク集） | IsaacLab | 同上（384 行目以降） |
| `lerobot/unitree-g1-mujoco` | Unitree G1 のシミュレーション | MuJoCo | [unitree_g1.py:302](./lerobot/src/lerobot/robots/unitree_g1/unitree_g1.py) |
| `lerobot/cartpole-env` | 動作確認用のリファレンス実装 | Gymnasium 標準 | [envhub.mdx](./lerobot/docs/source/envhub.mdx) |

LeIsaac の 4 タスクは `so101_pick_orange`（オレンジ 3 個を皿へ）、`so101_lift_cube`（赤いキューブを持ち上げる）、`so101_clean_toytable`（箱に片付ける）、`bi_so101_fold_cloth`（双腕で布をたたむ）である。最初の 3 つが単腕 SO-101 Follower、最後が双腕 SO-101 Follower を使う。

### 組み込み（`EnvConfig` 登録済み）

`--env.type=` で直接指定できるのは [envs/configs.py](./lerobot/src/lerobot/envs/configs.py) に `@EnvConfig.register_subclass` で登録されている次の 11 種類である。

| `--env.type` | 内容 | シミュレータ | 導入方法 |
| --- | --- | --- | --- |
| `aloha` | ALOHA 双腕（挿入・移送） | MuJoCo | `pip install -e ".[aloha]"` |
| `pusht` | 2D の物体押し込み | pymunk | `pip install -e ".[pusht]"` |
| `libero` | 130 タスク / 5 スイート、生涯学習 | MuJoCo + robosuite | `pip install -e ".[libero]"`（Linux のみ） |
| `libero_plus` | LIBERO に 7 種の摂動、約 1 万バリアント | 同上 | fork を clone |
| `metaworld` | 50 タスク（MT50）、Sawyer アーム | MuJoCo | `pip install -e ".[metaworld]"` |
| `robocasa` | RoboCasa365、キッチン 365 タスク | MuJoCo + robosuite | ソースから clone |
| `robotwin` | RoboTwin 2.0、双腕 50 タスク | SAPIEN | ソースから clone |
| `vlabench` | 言語条件付き 43 タスク | MuJoCo / dm_control | ソースから clone |
| `robomme` | 記憶を問う 16 タスク / 4 スイート | ManiSkill / SAPIEN | 手動 pip（Linux のみ） |
| `isaaclab_arena` | ← 上の EnvHub 経由（`HubEnvConfig` 継承） | IsaacLab | Isaac Sim を別途導入 |
| `gym_manipulator` | HIL-SERL 用（実機／`gym_hil` の Franka Panda） | MuJoCo | `pip install -e ".[hilserl]"` |

RoboCerebra には専用の `--env.type` がない。LIBERO のシミュレータをそのまま使うため、`--env.type=libero --env.task=libero_10` で評価する。

extras（`pip install` 一発）で入るのは **aloha / pusht / libero / metaworld** の 4 つだけで、残りは GitHub から clone する手順が各ドキュメントに書かれている。macOS では `libero` 系（`hf-libero` が Linux 限定）と `robomme` が使えない。

### ワークショップ視点での選び方

- **SO-101 をシミュレーションで触りたい** → LeIsaac 一択。ただし IsaacLab が動く GPU 環境（または NVIDIA Brev のクラウド）が要る。
- **とりあえず学習・評価の流れを試したい** → `pusht` か `aloha`。extras で入り、GPU 要件も軽い。
- **VLA の性能を測りたい** → `libero`。事前学習済みポリシーとデータセットが揃っている。ただし Linux 限定。

## 使い方

### 最小の例

```python
from lerobot.envs import make_env

# trust_remote_code=True が必須（後述のセキュリティを参照）
envs_dict = make_env("lerobot/cartpole-env", n_envs=4, trust_remote_code=True)

suite_name = next(iter(envs_dict))
env = envs_dict[suite_name][0]

obs, info = env.reset()
obs, reward, terminated, truncated, info = env.step(env.action_space.sample())
env.close()
```

戻り値は常に `{suite 名: {task_id: VectorEnv}}` の入れ子辞書である。単一タスクでも 1 要素の辞書に正規化されるため、LIBERO のような複数タスクのベンチマークと同じコードで扱える。

### URL の書式

`lerobot/src/lerobot/envs/utils.py:383` の `_parse_hub_url()` が `[repo_id][@revision][:path]` を解釈する。

| 書式 | 意味 | 例 |
| --- | --- | --- |
| `user/repo` | main の `env.py` を読む | `make_env("lerobot/pusht-env")` |
| `user/repo@revision` | ブランチ・タグ・コミットを指定 | `make_env("lerobot/pusht-env@v1.0")` |
| `user/repo:path` | `env.py` 以外のファイルを指定 | `make_env("lerobot/envs:pusht.py")` |
| `user/repo@rev:path` | 両方を指定 | `make_env("lerobot/envs@v1:pusht.py")` |

再現性とセキュリティのため、実験ではコミットハッシュで固定するのが推奨されている。

### CLI から使う

`lerobot-eval` は `EnvConfig` を CLI から受け取るため、文字列 URL をそのまま渡すことはできない。**`HubEnvConfig` を継承した登録済みの env 型**を `--env.type` で選び、`--env.hub_path` でリポジトリを指定する。v0.6.2 時点で登録されているのは `isaaclab_arena` のみである。

```bash
lerobot-eval \
    --policy.path=nvidia/smolvla-arena-gr1-microwave \
    --env.type=isaaclab_arena \
    --env.hub_path=nvidia/isaaclab-arena-envs \
    --env.environment=gr1_microwave \
    --env.embodiment=gr1_pink \
    --trust_remote_code=True \
    --eval.batch_size=1
```

`--trust_remote_code` は `EvalPipelineConfig` のトップレベルのフィールド（`lerobot/src/lerobot/configs/eval.py:43`）であり、`--env.` の下ではない点に注意する。

## 環境を公開する側の契約

Hub リポジトリに求められるのは実質 1 ファイルだけである。

```
my-environment-repo/
├── env.py             # 必須：make_env を公開する
├── requirements.txt   # 任意：依存ライブラリ（自動インストールはされない）
├── README.md          # 推奨：観測・行動空間、報酬、終了条件を説明する
└── assets/            # 任意
```

`env.py` は次のいずれかのシグネチャで `make_env` を公開する。

```python
def make_env(n_envs: int = 1, use_async_envs: bool = False):
    ...

# EnvConfig を受け取って設定を反映したい場合
def make_env(n_envs: int = 1, use_async_envs: bool = False, cfg: EnvConfig = None):
    ...
```

戻り値は次の 3 種類が許される（`_normalize_hub_result()` が辞書に正規化する）。

1. `gym.vector.VectorEnv` — 最も一般的
2. `gym.Env` — 自動で `SyncVectorEnv` に包まれる
3. `{suite 名: {task_id: VectorEnv}}` — 複数タスクのベンチマーク向け

なお `lerobot-eval` で評価まで行う場合は、`step()` が返す `info["is_success"]` などのベンチマーク側の約束も満たす必要がある。詳細は [adding_benchmarks.mdx](./lerobot/docs/source/adding_benchmarks.mdx) にまとまっている。

## 内部の動作

`make_env()`（`lerobot/src/lerobot/envs/factory.py:58`）の処理は次の順に進む。

1. **hub_path の決定** — 引数が文字列なら URL そのもの、`HubEnvConfig` なら `cfg.hub_path` を使う。通常の `EnvConfig` なら Hub 経路には入らず、ローカルの `cfg.create_envs()` が呼ばれる。
2. **ダウンロード**（`_download_hub_file()`）— `trust_remote_code` が `False` なら `RuntimeError` を送出して停止する。真なら `hf_hub_download()` で該当ファイルを取得し、失敗時は `snapshot_download()` にフォールバックする。
3. **読み込み**（`_import_hub_module()`）— `importlib.util.spec_from_file_location()` でダウンロードしたファイルをモジュールとして実行する。依存不足は `ModuleNotFoundError` に「どのパッケージが足りないか」を添えて再送出する。
4. **呼び出し**（`_call_make_env()`）— モジュールが `make_env` を持つか検査し、`EnvConfig` があれば `cfg=` 付きで呼ぶ。
5. **正規化**（`_normalize_hub_result()`）— 戻り値を `{suite: {task_id: VectorEnv}}` に揃える。

対応するコードは次のとおりである。

| ファイル | 役割 |
| --- | --- |
| [lerobot/src/lerobot/envs/factory.py](./lerobot/src/lerobot/envs/factory.py) | `make_env()` の入口。Hub 経路とローカル経路を分岐する |
| [lerobot/src/lerobot/envs/utils.py](./lerobot/src/lerobot/envs/utils.py) | URL パース、ダウンロード、モジュール実行、戻り値の正規化（383〜498 行目） |
| [lerobot/src/lerobot/envs/configs.py](./lerobot/src/lerobot/envs/configs.py) | `HubEnvConfig`（132 行目）と `IsaaclabArenaEnv`（649 行目） |
| [lerobot/src/lerobot/configs/eval.py](./lerobot/src/lerobot/configs/eval.py) | `lerobot-eval` の `trust_remote_code` フラグ |
| [lerobot/tests/envs/test_envs.py](./lerobot/tests/envs/test_envs.py) | URL パース、正規化、`trust_remote_code` 拒否のテスト |

### どこで実行されるのか — すべてローカル

誤解しやすい点なので明記する。EnvHub は**コードの配布経路であって、実行環境ではない**。シミュレーションは 100% 実行する側のマシンで動く。

[utils.py:369-376](./lerobot/src/lerobot/envs/utils.py) を見れば明らかである。

```python
spec = importlib.util.spec_from_file_location(module_name, path)
module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(module)   # ← make_env() を呼んだのと同じ Python プロセス内で実行
```

1. `hf_hub_download()` が `env.py` を**ローカルの HF キャッシュ**（`~/.cache/huggingface/hub`）へ落とす
2. `exec_module()` が、それを**同一プロセス内で**実行する
3. その中の `gym.make()` が IsaacLab や MuJoCo を**ローカルで起動する**

サンドボックスもコンテナ分離もない。ユーザー権限のまま動く。`trust_remote_code=True` が必須なのはこのためである。

したがって、公式ドキュメントの「インストール不要」は **`env.py` という接着コードの配布に packaging が要らない**という意味であって、シミュレータ本体まで不要という意味ではない。責任分界は次のとおりである。

| やること | 誰がやる |
| --- | --- |
| Isaac Sim / MuJoCo などシミュレータ本体の導入 | **利用者（手動）** |
| 環境の依存ライブラリ（`requirements.txt`） | **利用者（手動。自動インストールされない）** |
| `env.py` の取得とバージョン管理 | EnvHub |
| `make_env()` の呼び出しと戻り値の正規化 | LeRobot |

IsaacLab Arena のドキュメントもこの点を明示している ——「あなたの IsaacLab Arena 環境のコードは評価時にローカルで利用可能でなければならない。利用者が別途 clone するか、EnvHub リポジトリに環境コードとアセットを同梱すること」。EnvHub が省いてくれるのは**配布**の手間であって、**環境構築**の手間ではない。

GPU がない場合に LeIsaac が案内している NVIDIA Brev のクラウド実行も、EnvHub がリモート実行に対応したわけではなく、クラウド上のマシンが「実行する側」になるだけである。

## セキュリティ

EnvHub は**第三者が書いた Python コードを手元で実行する**機能である。そのため `trust_remote_code=True` を明示しない限り必ず失敗する設計になっている（`lerobot/src/lerobot/envs/utils.py:410`）。

```
RuntimeError: Refusing to execute remote code from the Hub for '...'
```

公式ドキュメントが挙げる注意点は次のとおりである。

1. 読み込む前に Hub 上で `env.py` の中身を読む。
2. コミットハッシュで固定する（`user/repo@a1b2c3d4`）。
3. `requirements.txt` に不審なパッケージがないか確認する。
4. 公式組織や素性の分かる作者のリポジトリを優先する。
5. 信用しきれない場合はコンテナや VM の中で実行する。

ワークショップで参加者に紹介する場合は、この「任意コード実行である」という点を必ず添える。

## 学習と評価での使い分け

`lerobot-train` と Hub 環境の関係は誤解しやすいので、先に整理しておく。

**模倣学習はデータセットから学習するので、環境そのものは不要である。** 実際 `TrainPipelineConfig` の `env` は `None` がデフォルトである（[train.py:112](./lerobot/src/lerobot/configs/train.py)）。環境が使われるのは、**学習の途中に定期的に挟む評価ロールアウト**のときだけである。

```python
# lerobot_train.py:847
if cfg.env and is_env_eval_step:        # env_eval_freq step ごと（デフォルト 20,000）
    with ... _make_eval_envs(cfg) as eval_env, ...:
```

この `_make_eval_envs()` 内の `make_env()` 呼び出しが `trust_remote_code` を渡していない（[lerobot_train.py:117-121](./lerobot/src/lerobot/scripts/lerobot_train.py)）。そのため `--env.type=isaaclab_arena` のような Hub 環境を指定すると、**この定期評価のタイミングで** `RuntimeError` になる。学習自体は問題なく回る。

回避策は、学習中の環境評価を切って、保存済みチェックポイントを `lerobot-eval` で評価することである。

```bash
lerobot-train ... --env_eval_freq=0     # 学習中の環境評価を無効化
lerobot-eval --policy.path=<checkpoint> --env.type=isaaclab_arena ... --trust_remote_code=True
```

これは LeRobot 自身が FSDP 分散学習のときに案内している手順とまったく同じである（[train.py:379-384](./lerobot/src/lerobot/configs/train.py)）。

> `...set --env_eval_freq=0 and evaluate with lerobot-eval on saved checkpoints.`

実際、IsaacLab Arena のドキュメントの「Training Policies」節は `lerobot-train --env.type=isaaclab_arena` という例を一切示さず、データセットへのリンクとポリシー学習ドキュメントへの誘導だけである。**「学習はデータセットから、評価は `lerobot-eval` で」が想定ワークフロー**ということである。

## 現時点での制限

コードを読んで確認できた制限は次のとおりである。

- **`lerobot-train` の学習中評価では Hub 環境を使えない**（上記「学習と評価での使い分け」を参照）。学習自体は動く。`--env_eval_freq=0` にして `lerobot-eval` で別途評価する。
- **`requirements.txt` は自動でインストールされない。** 依存は利用者が手動で入れる必要がある。シミュレータ本体も同様に手動導入である。
- **戻り値が辞書の場合は検証されない。** `_normalize_hub_result()` は `dict` をそのまま通すため、構造が誤っていても後段まで気づけない。
- **文字列で URL を渡す API は将来変更される可能性がある。** `factory.py:90` に `TODO: deprecate string API` のコメントがある。
- **`HubEnvConfig` を直接 CLI から選ぶことはできない。** 登録済みのサブクラス（現状は `isaaclab_arena`）を経由する必要がある。

## 参照

- [EnvHub 公式ドキュメント](https://huggingface.co/docs/lerobot/envhub) / ローカル: [lerobot/docs/source/envhub.mdx](./lerobot/docs/source/envhub.mdx)
- [lerobot/docs/source/envhub_leisaac.mdx](./lerobot/docs/source/envhub_leisaac.mdx) — SO-101 を IsaacLab 上で動かす
- [lerobot/docs/source/envhub_isaaclab_arena.mdx](./lerobot/docs/source/envhub_isaaclab_arena.mdx) — NVIDIA IsaacLab Arena との連携
- [lerobot/docs/source/adding_benchmarks.mdx](./lerobot/docs/source/adding_benchmarks.mdx) — ベンチマークとして統合する場合の要件
- [PR #2121 — feat(sim): EnvHub](https://github.com/huggingface/lerobot/pull/2121)
- [LeRobot v0.5.0 リリースブログ](https://huggingface.co/blog/lerobot-release-v050)
