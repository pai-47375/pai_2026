# 小型LLMによる長期タスク分解はVLAの実行を助けるか？ ― SmolVLA × LIBERO-Long での検証

Physical AI 応用1講座 最終課題（omnicampus ID: 47375）

## 概要

長い指示を小型LLMがサブタスクに分解し、VLA（SmolVLA）に1つずつ渡す階層型の構成が、
実行の成功率を上げるかをLIBEROシミュレーション上で検証しました。

**結論：LLMの分解は人間の分解と完全に一致していたにもかかわらず、成功率は50%から5%に下がりました。**
人間が学習時の言い回しを正しく選んでも成功は0%で、ボトルネックはLLMの計画能力ではなく、
計画をVLAに渡す言語のインターフェース（言い回し×シーンの組み合わせ）にあります。

| 条件 | SmolVLAに渡す指示 | 成功率 | 部分達成率 |
| --- | --- | --- | --- |
| baseline | LIBERO-10の長い指示をそのまま | 50% (10/20) | 57% |
| llm_decomp | Qwen2.5-1.5Bが分解したサブタスクを順に | 5% (1/20) | 5% |
| skill_grounded | 各サブタスクをLLMが学習済みの指示文（スキルライブラリ）に置き換えて順に | 5% (1/20) | 15% |
| oracle_grounded | 各サブタスクを人間が選んだ正しい学習済み指示文にして順に（Task 0・1・7のみ） | 0% (0/15) | 10% |

- 対象：LIBERO-10のタスク0・1・4・7（いずれも2つの物体を順に操作）、各5エピソード
- 条件間でエピソードごとの初期配置を固定し、同じ配置どうしで比較
- 同じ初期配置のペア比較では、baselineで成功した10回のうち9回が分解後に失敗し、改善は0回
- skill_groundedでは小型LLMがスキル選択を6回中3回誤ったため、oracle_groundedで言い回しの効果を切り分けた
- Task 0・1・7で1つ目のゴールを達成した割合：baseline 60% / llm_decomp 0% / skill_grounded 13% / oracle_grounded 20%

## 構成

```
.
├── notebooks/
│   ├── pai_final_01_baseline.ipynb     # 環境構築・評価ループ・ベースライン評価
│   ├── pai_final_02_llm_decomp.ipynb   # LLM分解・skill_grounded・比較集計
│   └── pai_final_03_oracle_grounded.ipynb  # 正しい言い回しでの追加評価・全条件の比較
└── results/
    ├── results.jsonl                   # 全エピソードの結果（1行1エピソード）
    ├── decompositions.json             # LLMによる分解結果（生出力つき）
    ├── grounded_decompositions.json    # スキル選択の結果
    ├── llm_prompt.json                 # 分解に使ったプロンプト
    ├── summary_overall.csv / summary_by_task.csv / paired_comparison.csv / summary_tasks_0_1_7.csv
    └── comparison.png
```

## 使用したもの

- VLA：SmolVLA（`lerobot/smolvla_libero`）推論のみ
- LLM：Qwen2.5-1.5B-Instruct（推論のみ）
- フレームワーク：[LeRobot](https://github.com/huggingface/lerobot)（コミット `3f797de` に固定）
- ベンチマーク：LIBERO（hf-libero 0.1.4、robosuite 1.4.0、MuJoCo）
- 実行環境：Google Colab（T4 GPU、無料枠で動作）

## 実行方法

1. `notebooks/pai_final_01_baseline.ipynb` をColabで開き、ランタイムをT4 GPUにして上から実行します。
   結果はGoogle Driveの `MyDrive/pai_final/` に保存されます。
2. 続けて `notebooks/pai_final_02_llm_decomp.ipynb`、`notebooks/pai_final_03_oracle_grounded.ipynb` を実行します（同じDriveフォルダを使います）。

### インストールについての注意

`lerobot[libero]` をそのままインストールすると、依存パッケージ `egl_probe` / `hf-egl-probe` のビルドがColab上で失敗します。
これらはLIBEROの学習用コード（`libero.lifelong`）でしか使われないため、ノートブックでは
`hf-libero` を `--no-deps` で入れ、シミュレーションの実行に必要な依存だけを追加しています。

## 実装の工夫

- **サブタスクの自動切り替え**：LIBEROのゴール定義（BDDL）の述語を1つずつ判定し、満たされた数 k に応じて k+1 番目の指示を渡します。切り替え時にはSmolVLAの行動キューを破棄します。
- **公平な比較**：エピソード番号ごとに初期状態を固定し、条件間でペア比較できるようにしました。
- **小型LLMの安定化**：分解は貪欲法で決定的に生成してキャッシュし、スキルの選択は番号の多肢選択にしました。one-shot例とスキルライブラリには評価対象のLIBERO-10の指示を含めていません。
- **途中再開**：結果を1エピソードごとに追記し、Colabが切断されても続きから再開できます。

## 限界

- 1条件20エピソードと試行数が少ない
- SmolVLAの行動生成のランダム性について、エピソードごとのシードは固定していない
- サブタスクの切り替えにシミュレータ内部のゴール判定（オラクル）を使っている
