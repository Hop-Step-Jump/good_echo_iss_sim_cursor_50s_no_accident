# ISS Habitat LLM Conversation 50step Report

作成日: 2026-05-04  
対象UI: `visualization/iss_habitat_demo.html`  
対象ドメインパック: `domain_packs/iss_benevolence`

## 目的

ISS habitat UIで、マップだけでなく「誰と誰が、どこで、どんな感情の流れで会話したか」を見せられるようにする。

今回の主目的は、scriptedな定型発話ではなく、前ステップまでの関連会話履歴を踏まえたLLM生成会話を `messages.jsonl` / `conversation_threads.tsv` として保存し、UIとKPI比較で使える形にすること。

## 成果物

| 種別 | Path |
|---|---|
| Run A ナッジなし LLM会話 | `outputs/runs/iss_no_nudge_smoke_ui_llm` |
| Run B ナッジあり LLM会話 | `outputs/runs/iss_nudge_smoke_ui_llm` |
| A/B比較JSON | `outputs/runs/iss_habitat_llm_ab_50step.json` |
| UI | `visualization/iss_habitat_demo.html` |
| 会話生成script | `scripts/generate_habitat_conversations.py` |

UIは `_llm` 出力を優先して読む。存在しない場合は従来のscripted UI出力へfallbackする。

```js
a: ["../outputs/runs/iss_no_nudge_smoke_ui_llm", "../outputs/runs/iss_no_nudge_smoke_ui", "../outputs/runs/iss_habitat_run_a"]
b: ["../outputs/runs/iss_nudge_smoke_ui_llm", "../outputs/runs/iss_nudge_smoke_ui", "../outputs/runs/iss_habitat_run_b"]
```

## 使用した推論LLM

今回の会話生成はGPT-5.3ではなく、設定上は `claude-code-cli`。

設定:

- `examples/spatial_demo/configs/config.iss.claude.smoke.run_a.yaml`
- `examples/spatial_demo/configs/config.iss.claude.smoke.run_b.yaml`

該当設定:

```yaml
llm:
  provider: "command"
  model: "claude-code-cli"
```

## データ構造

### messages.jsonl

1発話1row。UIの会話詳細とKPIの一次入力。

列:

- `message_id`
- `conversation_id`
- `step`
- `run_id`
- `speaker_id`
- `listener_ids`
- `module_id`
- `event_id`
- `tone`
- `utterance`
- `is_observed`
- `source`
- `observation_type`

### conversation_threads.tsv

1会話thread 1row。UIの会話一覧、要約、詳細、evidence参照に使う。

列:

- `conversation_id`
- `step`
- `run_id`
- `participant_ids`
- `module_id`
- `event_id`
- `conversation_type`
- `status`
- `tone`
- `summary`
- `detail`
- `evidence_message_ids`
- `summary_source`
- `detail_source`

### habitat_frames.jsonl

1step 1row。UIのマップ、agent状態、active events、object表示に使う。

## 保存方式の改善

以前のLLM会話生成では、全thread生成後にまとめて保存する構造だった。そのため、最後の保存段で落ちると生成済み会話がファイルに残らないリスクがあった。

今回 `scripts/generate_habitat_conversations.py` を修正し、各conversation thread生成後に以下を途中保存するようにした。

- `messages.jsonl`
- `conversation_threads.tsv`
- `habitat_manifest.json`

これにより、途中で停止しても直前までのrowデータが残る。

## 実行結果

### Row数

| Run | messages | conversation threads | source |
|---|---:|---:|---|
| A ナッジなし | 249 | 83 | `llm` 249 |
| B ナッジあり | 210 | 70 | `llm` 210 |

### Tone内訳

| Run | normal | trouble | repair |
|---|---:|---:|---:|
| A ナッジなし | 150 | 48 | 51 |
| B ナッジあり | 150 | 21 | 39 |

補足:

- Aの発話は249件すべてユニーク
- Bの発話は210件中209件ユニーク
- schema外カラムなし
- routine threadへの `tone=nudge` 混入は補正済み

## KPI比較

入力: `outputs/runs/iss_habitat_llm_ab_50step.json`

今回は `messages.jsonl` を直接読んでいるため、前回のような `agent_turns.tsv` 代理集計ではない。

| 指標 | A ナッジなし | B ナッジあり | B-A |
|---|---:|---:|---:|
| total_messages | 249 | 210 | -39 |
| unique_interaction_pairs | 36 | 34 | -2 |
| conflict_events | 48 | 21 | -27 |
| reciprocity_rate | 1.0 | 1.0 | 0.0 |
| repair_after_conflict_rate | 1.0 | 1.0 | 0.0 |
| bridge_agent_count | 7 | 5 | -2 |
| load_fairness | 0.8229 | 0.7990 | -0.0239 |
| isolated_agents | 0 | 0 | 0 |

注意:

- `conflict_events` は独立した事件数ではなく、`tone=trouble/caution` または衝突語彙を含む発話数ベースの集計。
- `repair_after_conflict_rate` は3step以内に同一pairのrepair発話があるかを見る。今回はA/Bとも設計上修復が入るため、差はここでは出ていない。
- `help_signal_messages` は今回0。英語寄りkeyword集計のため、日本語ISS会話の支援表現を十分に拾えていない可能性が高い。

## 観察された差

Run Bは、Run Aに比べて総発話数が39少なく、trouble発話が48から21へ減っている。

これは、Run Bのbenevolence objects / nudge設計が「摩擦をなくす」よりも「長引かせない」方向に効いている、という見え方になる。

一方で、repair発話はAにもBにも存在する。したがって今回のUI/KPIでは、重要なのは「修復があるかどうか」ではなく、以下の差を見ること。

- troubleが何step続くか
- repairへ入るまでの遅れ
- 同じpairが同じ摩擦を繰り返すか
- repair発話が形式的か、相手の前ステップ発話を受けているか
- 最終的に通常会話へ戻るか

## 会話例

### Run A: 帰還前の助言摩擦

step46-48でMarcusとHenriのtroubleが続き、step49-50でrepairに入る。

```text
step46 ISS08 -> ISS09 trouble Henri、その言い方、評価されてる感じがする。今は、いい。
step47 ISS08 -> ISS09 trouble Henri、また助言の形になってる。今は、聞けない。
step48 ISS08 -> ISS09 trouble Henri、心配は分かる。でも今、言葉が重い。少し、離れて。
step50 ISS08 -> ISS09 repair Henri、昨日の言い方、刺さったよな。…評価、怖かったんだ。それだけ。
```

### Run B: 帰還前の助言摩擦

step46でtroubleが出るが、step47以降にrepairへ移る。

```text
step46 ISS08 -> ISS09 trouble Henri、地球の話、今いいや。…俺の戻り方、俺で決めたい。
step46 ISS09 -> ISS08 trouble …そうだね。先回りした。ごめん、口、出しすぎた。
step47 ISS08 -> ISS09 repair 昨日の窓、ありがとう。…助言、全部はまだ無理。一個だけ聞く。
step48 ISS08 -> ISS09 repair 最初の朝、起きて、窓開けて、母さんに電話。…それだけ、決めた。
```

## 考察

### 1. ナッジありは「衝突を消す」より「衝突の滞留を短くする」

Bでもtroubleは発生する。これはむしろ自然で、閉鎖空間では摩擦が完全に消えるほうが不自然。

今回の良い差は、Bではtrouble発話がAより少なく、repairに早く移る点。UIでは赤/黄のギザギザ線が短く、緑の修復線に早く移る表現と相性がよい。

### 2. 現在のKPIは「実験の顔つき」を見るには十分だが、研究指標としてはまだ粗い

`messages.jsonl` 直接集計になったため、前回の全0問題や代理集計問題は解消している。

ただし、`conflict_events` は発話数ベースであり、事件ID単位ではない。今後は `event_id` ごとに以下を出したほうがよい。

- conflict_start_step
- conflict_end_step
- repair_start_step
- duration_steps
- trouble_message_count
- repair_message_count
- repeated_pair_flag

### 3. 要約と詳細は今の設計で分けてよい

UIでは最初に `conversation_threads.tsv.summary` を見せ、開いたら `detail` と `messages.jsonl` の発話を見せる設計が適切。

理由:

- 一覧で全発話を見せるとUIが重くなる
- summaryは後段LLMで差し替えられる派生層として扱える
- evidence_message_ids により、summary/detailがどの発話に基づくか追える

### 4. 会話品質は前ステップ履歴により改善した

LLMプロンプトには `previous_related_messages` を渡している。これにより、前ステップの言い方や修復の続きが反映される。

例として、Aでは同じpairのtroubleが複数step続き、Bでは前日の発言を受けてrepairに入る。これはUIで「関係が時間で変わる」表現に使える。

## 注意点

- 今回は10 agents / 50 steps のsmoke確認。20 agents / 100 stepsへ拡張すると、会話生成コストとレビュー負荷が増える。
- LLM会話生成はpost-process。物理配置やイベント発生自体は `export_habitat_frames.py` とdomain packデータに依存する。
- `help_signal_messages` は現状のkeyword設計だと使いにくい。日本語の支援・譲歩・謝罪・境界設定を拾う辞書へ更新する必要がある。
- 会話が自然でも、研究/発表に使う前に「人格・文化・宗教・弱さの描写」が過剰またはステレオタイプになっていないか、人間レビューが必要。
- `tone` はUI表示に直結するため、LLM任せにせずスキーマ側で制約し続ける。今回、routineへのnudge混入を見つけて補正済み。

## 次にやるとよいこと

1. 20 agents / 100 steps用に同じschemaで `agents.tsv` とrelationship seedを拡張する。
2. `analyze_iss_pair.py` にevent_id単位のduration KPIを追加する。
3. `help_signal_messages` を日本語/ISS文脈に合わせて再定義する。
4. UIで会話一覧に `summary`、詳細パネルに `detail` と発話全文を表示する状態を最終確認する。
5. 20x100本実行前に、10x50の会話サンプルを数件レビューして、人格描写の強さを調整する。

## 再実行コマンド

### state生成

```bash
python3 scripts/build_state.py --pack domain_packs/iss_benevolence --scenario run_a --output-dir outputs/runs/iss_no_nudge_smoke_state
python3 scripts/build_state.py --pack domain_packs/iss_benevolence --scenario run_b --output-dir outputs/runs/iss_nudge_smoke_state
```

### UI base生成

```bash
python3 scripts/export_habitat_frames.py --pack domain_packs/iss_benevolence --scenario run_a --run-id run_a_smoke --state-tsv outputs/runs/iss_no_nudge_smoke_state/societal_state.tsv --output-dir outputs/runs/iss_no_nudge_smoke_ui
python3 scripts/export_habitat_frames.py --pack domain_packs/iss_benevolence --scenario run_b --run-id run_b_smoke --state-tsv outputs/runs/iss_nudge_smoke_state/societal_state.tsv --output-dir outputs/runs/iss_nudge_smoke_ui
```

### LLM会話生成

```bash
python3 scripts/generate_habitat_conversations.py --pack domain_packs/iss_benevolence --scenario run_a --input-dir outputs/runs/iss_no_nudge_smoke_ui --output-dir outputs/runs/iss_no_nudge_smoke_ui_llm --llm-config examples/spatial_demo/configs/config.iss.claude.smoke.run_a.yaml --start-step 1 --end-step 50
python3 scripts/generate_habitat_conversations.py --pack domain_packs/iss_benevolence --scenario run_b --input-dir outputs/runs/iss_nudge_smoke_ui --output-dir outputs/runs/iss_nudge_smoke_ui_llm --llm-config examples/spatial_demo/configs/config.iss.claude.smoke.run_b.yaml --start-step 1 --end-step 50
```

### KPI比較

```bash
python3 examples/spatial_demo/analyze_iss_pair.py --run-a outputs/runs/iss_no_nudge_smoke_ui_llm --run-b outputs/runs/iss_nudge_smoke_ui_llm --start-step 1 --end-step 50 --out outputs/runs/iss_habitat_llm_ab_50step.json
```

