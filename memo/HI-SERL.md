# HIL-SERL と SO-ARM 101 の対応状況

調査日: 2026-08-15
対象: このワークスペースの `lerobot`（commit `6adf5151`、`v0.6.1-24-g6adf5151`。ワークスペース名は v0.6.0）

## 結論

LeRobot には HIL-SERL（Human-in-the-Loop Sample-Efficient Reinforcement Learning）の実装、依存 extra、実機用チュートリアル、SAC のテスト群が存在する。したがって、**機能そのものは実装済みで、研究・検証用途には使える**。

しかし、**SO-ARM 101 を HIL-SERL で実機運用することは、このチェックアウトのままでは対応済みとはいえない**。HIL-SERL のチュートリアルは `so101_leader` を対応として記載する一方、実装中の `SOLeader` は HIL-SERL が必須とする `get_teleop_events()` を持たない。そのため、記録／学習パイプラインの初期化時に `TypeError` となる。これは公式 Issue #2952 と一致する。

SO-ARM 101 で人の介入を使って性能を改善したいワークショップでは、当面は HIL-SERL ではなく、公式に SO-101 を対応機器として明記している **DAgger ベースの HIL データ収集**を使い、ACT などを追加 fine-tune する経路を推奨する。これは強化学習ではなく、対話的な模倣学習である点に注意する。

## HIL-SERL とは

HIL-SERL は、少数の人間デモ、画像から成功を判定する報酬分類器、実機上での SAC/RLPD によるオンライン学習、人間の随時介入を組み合わせる手法である。LeRobot の実装は次の流れを採る。

1. 人がタスクを実演し、成功／失敗のラベル付きデータを収録する。
2. 画像ベースの報酬分類器を学習する。
3. actor（ロボット制御）と learner（GPU 学習）を別プロセスで起動し、gRPC で遷移と重みを交換する。
4. ポリシーの失敗時に人が介入し、その経験を混ぜて SAC を更新する。

模倣学習だけでは遭遇しにくい失敗状態を、人が安全に戻しながら探索できることが狙いである。一方で、成功判定、エンドエフェクタ座標系、IK、可動域、安全境界、分散プロセス、リアルタイム制御を同時に正しく設定する必要がある。初回の SO-101 ワークショップ向けの手軽な学習手法ではない。

## リポジトリ内の実装状況

| 項目 | 状況 | 根拠 |
| --- | --- | --- |
| HIL-SERL の公開位置 | 対応あり | [`lerobot/README.md`](./lerobot/README.md) が強化学習の一つとして HIL-SERL を掲載する。 |
| 依存関係 | 対応あり | [`pyproject.toml`](./lerobot/pyproject.toml) の `hilserl` extra は `gym-hil`、gRPC、データセット、kinematics 関連を含む。 |
| 実装本体 | 対応あり | [`rl/`](./lerobot/src/lerobot/rl/) に SAC、replay buffer、offline/online data mixer、actor、learner、gRPC service がある。 |
| 実機チュートリアル | 対応あり | [`hilserl.mdx`](./lerobot/docs/source/hilserl.mdx) にデモ収録、報酬分類器、actor/learner の手順がある。 |
| シミュレーション | 対応あり | [`hilserl_sim.mdx`](./lerobot/docs/source/hilserl_sim.mdx) と `gym-hil` の経路がある。実機前の検証先になる。 |
| ユニット／結合テスト | 一部あり | [`tests/rl/`](./lerobot/tests/rl/) に SAC、buffer、data mixer、actor/learner 通信のテストがある。ただし SO-101 実機を通すテストはない。 |
| 同梱済みの SO-101 用 HIL-SERL 設定 | なし | ドキュメント内の例は `so100` 名の外部設定を参照し、リポジトリには `env_config_so101*`／`train_config_hilserl_so101*` がない。 |

`uv run --extra test pytest tests/rl -q` も確認した。結果は `13 skipped`（実行失敗はなし）だったが、HIL-SERL extra の依存を入れずに実行したため、SAC／gRPC の実テストは走っていない。この結果は「自動テストで実機まで検証済み」を意味しない。

## SO-ARM 101 の互換性をコードで確認した結果

### ドキュメント上の主張

[`hilserl.mdx`](./lerobot/docs/source/hilserl.mdx) は `so101_leader` と `control_mode: "leader"` を設定する手順、介入用のキーボード操作、SO-101 リーダーを用いた映像を掲載している。SO-101 リーダーは登録済みのテレオペレータであり、通常のテレオペレーションやデータ収録には利用できる。

### 実装上の不一致

HIL-SERL の [`gym_manipulator.py`](./lerobot/src/lerobot/rl/gym_manipulator.py) は、アクション処理に `AddTeleopEventsAsInfoStep` を必ず追加する。このステップは [`hil_processor.py`](./lerobot/src/lerobot/processor/hil_processor.py) で、テレオペレータに `get_teleop_events()` があることを要求する。

しかし [`so_leader.py`](./lerobot/src/lerobot/teleoperators/so_leader/so_leader.py) の `SOLeader` にはこのメソッドがない。実装されているのはゲームパッドとキーボードのテレオペレータだけである。そのため SO-101 leader を指定した HIL-SERL は、実行開始前に次の趣旨の例外で停止する。

```text
Teleoperator SOLeader must implement get_teleop_events() method.
Compatible teleoperators: GamepadTeleop, KeyboardEndEffectorTeleop
```

さらに、HIL-SERL はエンドエフェクタ空間での操作、URDF による IK、ワークスペース境界を用いる。SO-101 の可動域・座標系・正規化を個別に調整しなければ、介入時の急な移動や極端に狭い操作範囲につながる。

## GitHub の状況と実用性評価

公開 GitHub Issues／PR を 2026-08-15 に確認した。リポジトリの GitHub Discussions は公開ページが利用できず、HIL-SERL に関する公開 Discussion は確認できなかった。初期の公式調整先は Issue #504 に記された Discord であり、ここでは検証可能な Issue／PR を判断根拠とした。

| 根拠 | 状態 | 読み取れること |
| --- | --- | --- |
| [#504: Porting HIL-SERL](https://github.com/huggingface/lerobot/issues/504) | Closed | 2024 年の移植計画。RLPD/SAC、介入、報酬分類器の基本機能は LeRobot に移植された。Issue 中には当初「leader arm を使う」タスクもある。 |
| [#3076: RL Stack Refactoring](https://github.com/huggingface/lerobot/issues/3076) | Open | メンテナが HIL-SERL を「working」と表現する一方、インターフェースの安定化、コアテスト、ベンチマーク、ドキュメント改善を今後の Phase 1 に置いている。基盤は稼働するが成熟途中と読むのが妥当である。 |
| [#1387: so101 can't work with HIL-SERL](https://github.com/huggingface/lerobot/issues/1387) | Open | SO-101 でエンドエフェクタ境界が極端に狭く介入できないこと、および CPU 経路の型処理を報告している。未解決のままである。 |
| [#1819: SO-101 leader を使った再現相談](https://github.com/huggingface/lerobot/issues/1819) | Open | SO-101 leader／SO-100 follower の境界値がほぼゼロになることや急な動作が報告され、公開された解決応答は見当たらない。 |
| [#2952: HIL-SERL SO101 is not implemented](https://github.com/huggingface/lerobot/issues/2952) | Open | チュートリアルの SO-101 設定が上記の `get_teleop_events()` 不足で起動しないことを、具体的なスタックトレース付きで報告している。手元の実装も同じ条件に該当する。 |
| [#3086: leader arm support for HIL-SERL recording](https://github.com/huggingface/lerobot/pull/3086) | Open・未マージ | #2952 を直す外部 PR。キーボードイベント、leader mode の関節位置処理、ゼロ姿勢への急移動防止を提案し、作者は実 SO-101 での記録を報告している。ただしレビューなし、PR 本文のテスト checklist は未完、ユニットテストは PR に含めていない。加えて actor/learner 学習時の出力スケールに関する質問も残る。採用済みの解決策ではない。 |
| [#3385: smooth human intervention](https://github.com/huggingface/lerobot/issues/3385) | Closed | 介入に入ると follower が leader の姿勢へ跳ぶ、という使用感・安全性の課題を記録している。#3086 に依存する提案だった。 |
| [#2337: pi0 を HIL-SERL で継続 RL できるか](https://github.com/huggingface/lerobot/issues/2337) | Open | 模倣学習済み π0 をそのまま HIL-SERL で継続学習する利用法も、公開回答がない。HIL-SERL は既定では Gaussian Actor + SAC の系であり、任意の VLA の汎用 RL fine-tune 経路ではない。 |

以上からの評価は次のとおりである。

| 観点 | 評価 | 理由 |
| --- | --- | --- |
| HIL-SERL コア（SAC／actor-learner） | 検証用途で利用可能 | メンテナが稼働を明言し、実装・チュートリアル・ソフトウェアテストがある。 |
| 報酬分類器を含む実機ワークフロー | 上級者向け・実験的 | 成功ラベル、カメラ、ROI、報酬閾値、分散プロセスを自分で検証する必要がある。公式ベンチマークや完成済み設定は不足する。 |
| SO-ARM 101 + SO-101 leader | 現行 main では非対応 | ドキュメントと実装が不一致で、起動を妨げる未解決 Issue があり、修正 PR も未マージである。 |
| SO-ARM 101 での人の介入を使った改善 | DAgger なら試行候補 | 公式 HIL データ収集は `so_leader` と `so_follower` を明示的に対応としており、回帰テストもある。ただし RL ではない。 |

## SO-ARM 101 ワークショップでの推奨方針

1. まず [`structure.md`](./structure.md) の標準フローで、SO-101 の組立、キャリブレーション、テレオペレーション、通常のデータ収録を安定させる。
2. ACT などでベースポリシーを作り、失敗モードを評価する。
3. 人の修正データを取り込みたい場合は、[`hil_data_collection.mdx`](./lerobot/docs/source/hil_data_collection.mdx) の `lerobot-rollout --strategy.type=dagger` を使う。SO100/SO101 leader と SO follower はこの方式の対応機器として明記されている。
4. HIL-SERL を試すなら、SO-101 実機ではなく先に `gym-hil` のシミュレーションで actor/learner、報酬分類器、ログ、停止処理を確認する。
5. SO-101 実機で HIL-SERL を再開する条件は、少なくとも #3086 相当が公式 main にマージされ、該当 release で SO-101 の収録、介入、actor/learner 学習を小タスクで再現できることを確認してからとする。未マージ PR を直接取り込む場合は、ロボットが急動作しないよう低速・無負荷・非常停止可能な環境で、fork 固有の変更として検証する。

## 参照先

- ローカルの HIL-SERL ガイド: [`lerobot/docs/source/hilserl.mdx`](./lerobot/docs/source/hilserl.mdx)
- ローカルの HIL データ収集ガイド: [`lerobot/docs/source/hil_data_collection.mdx`](./lerobot/docs/source/hil_data_collection.mdx)
- HIL-SERL 原論文: [HIL-SERL project / paper](https://hil-serl.github.io/)
- LeRobot の HIL-SERL 移植計画: [Issue #504](https://github.com/huggingface/lerobot/issues/504)
- SO-101 の未対応報告と修正候補: [Issue #2952](https://github.com/huggingface/lerobot/issues/2952)、[PR #3086](https://github.com/huggingface/lerobot/pull/3086)
- RL 基盤の現行ロードマップ: [Issue #3076](https://github.com/huggingface/lerobot/issues/3076)
