# CLAUDE.md

このファイルは、本リポジトリで作業する際にClaude Code（claude.ai/code）へ指針を与えるものである。

## リポジトリの目的

本リポジトリ（`lerobot-0.6.0`）は、Hugging Faceの[LeRobot](https://github.com/huggingface/lerobot) v0.6.0リリースを検証するための薄いラッパーである。リポジトリ自体には固有のコードは存在せず、実体となるLeRobotのソースコードは`lerobot/`ディレクトリのgitサブモジュール（現在`v0.6.0-40-g0d383d09`をチェックアウト済み）に置かれている。ルートの`README.md`には、v0.6.0における変更点が日本語でまとめられている。すなわち、世界モデル系の新規ポリシー（VLA-JEPA、LingBot-VA、FastWAM）、VLAモデルの追加（GR00T N1.7、MolmoAct2、EO-1など）、統一された`lerobot.rewards` API、深度情報に対応したデータセット、`lerobot-eval`による評価ベンチマークの追加、FSDP/HF Jobsによる学習インフラの拡張、基本依存関係の削減、などである。

## `lerobot/`サブモジュール内での作業

`lerobot/CLAUDE.md`は`lerobot/AGENTS.md`へのシンボリックリンクであり、このファイルにはLeRobotプロジェクト自体の開発コマンド（`uv sync`、`uv run pytest`、`pre-commit run --all-files`）とアーキテクチャ（`policies/`、`processor/`、`datasets/`、`envs/`、および`robots/`・`motors/`・`cameras/`・`teleoperators/`配下のハードウェア抽象化レイヤー）がすでに記載されている。`lerobot/`配下のコードを変更する際は必ずこのファイルを参照すること。ここでの重複記載は行わない。また`lerobot/AGENT_GUIDE.md`には、ハードウェアのセットアップ、データ収録、ポリシーの選定・学習に関する、ユーザー向けのコピー＆ペースト可能なガイドが記載されている。SO-101やロボット関連のコマンドを提案する前には必ずこのファイルを確認すること。なお同ファイルには、コマンドを提案する前にユーザーの目的・保有ハードウェア・学習用マシンについて必ず質問するようエージェントに指示する記述がある点にも留意する。

## README.md（「エージェントへのお願い」）による指示

ルートの`README.md`には、本リポジトリにおける2つの恒常的な要求事項が定められている。

- 回答は日本語の「である」調を用いること（丁寧語・敬体は用いない）。
- コードを実装する際は可読性を優先し、ジュニアエンジニアが理解できるようコメント・docstringを追記すること。
- コミットはConventional Commitsに準拠し、1行の英語にまとめること
</content>
