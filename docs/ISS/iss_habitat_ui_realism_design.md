# ISS Habitat UI Realism Design

この文書は `visualization/iss_habitat_demo.html` を目標UIとして、見た目の説得力と実験データの整合を保つための設計メモである。

## 目的

UIの第一目標は、ISS閉鎖環境で「誰がどこにいて、誰と話し、どこで摩擦が起き、何によって修復されたか」を一目で追えること。

実験の第一目標は、Run A（善性オブジェクトなし）とRun B（善性オブジェクトあり）の違いを、単なる平均スコアではなく、関係・会話・場所・修復の流れとして観測できること。

## リアルに見せるための原則

- 全員が毎ステップ話す必要はない。沈黙、回避、一人時間、短い返事も観測値として扱う。
- 会話は必ず `speaker_id` と `listener_ids` を持つ。誰と誰が話したかを曖昧にしない。
- 摩擦/言い合い/修復は `event_id` と `conversation_id` の両方に紐づける。
- 場所は会話と感情に影響する。共用部、個室、キューポラ、運動エリアでは自然な発話量と内容が違う。
- ナッジは万能にしない。Run Bでも摩擦は起きるが、短くなる、断り方が見える、修復の入口が増える、という差にする。
- UI上の位置、寝床割当、混雑は毎描画で変えない。step単位で連続性を持たせる。
- 内面ログと会話ログを混ぜない。`thought` / `private_talk` は本人内面寄り、`messages.jsonl` は実際の相互作用として扱う。

## 正準データ

UIに渡す主なデータは以下。

| 出力 | 役割 |
|---|---|
| `habitat_frames.jsonl` | UI用のstep別フレーム。場所、混雑、アクティブイベント、会話ID、KPIを束ねる |
| `messages.jsonl` | 発話1件単位の会話ログ |
| `conversation_threads.tsv` | 会話スレッド単位の要約・詳細・根拠message ID |
| `agent_positions.tsv` | stepごとの agent位置。UIの点配置に使う |
| `module_occupancy.tsv` | stepごとの場所別人数/定員/混雑状態 |
| `sleep_assignments.tsv` | stepごとの寝床割当 |
| `auto_events.tsv` | baseline/object/conflict/repair イベント |
| `societal_state.tsv` | 閉鎖空間ストレス、資源圧、対人摩擦などの状態 |

## 会話データ

`messages.jsonl` は発話単位にする。

必須:

- `message_id`
- `conversation_id`
- `step`
- `speaker_id`
- `listener_ids`
- `module_id`
- `event_id`
- `tone`
- `utterance`
- `is_observed`
- `source`

`conversation_threads.tsv` は会話単位にする。

必須:

- `conversation_id`
- `step`
- `participant_ids`
- `module_id`
- `event_id`
- `conversation_type`
- `status`
- `summary`
- `detail`
- `evidence_message_ids`
- `summary_source`
- `detail_source`

会話種別:

- `routine`: 普通の短い調整会話
- `support`: 相談・手伝い・支え合い
- `conflict`: 摩擦・言い合い
- `repair`: 謝罪・言い換え・距離の取り直し
- `object_mediated`: 善性オブジェクトを介した会話

## UI Frame

`habitat_frames.jsonl` はUIが最初に読む正規化済みフレームにする。

1行1stepで、以下を含む。

- `run_id`
- `step`
- `phase`
- `agent_states`
- `module_states`
- `active_objects`
- `active_events`
- `active_incidents`
- `conversation_ids`
- `metrics`
- `sleep_assignments`
- `summary`
- `detail`
- `source`

スキーマは `domain_packs/iss_benevolence/data/habitat_frame_schema.tsv` を正とする。`agent_states` の中身は `domain_packs/iss_benevolence/data/agent_state_schema.tsv` を正とし、感情/行動は `label + summary + detail + source + evidence` に分ける。

## 要約と詳細

UIでは最初に `summary` を見せ、必要な時だけ `detail` を開く。

- raw発話: `messages.jsonl` の `utterance`
- 会話要約: `conversation_threads.tsv` の `summary`
- 会話詳細: `conversation_threads.tsv` の `detail`
- 感情要約/詳細: `agent_states[].emotion_summary` / `emotion_detail`
- 行動要約/詳細: `agent_states[].action_summary` / `action_detail`

要約/詳細は派生データなので、必ず `summary_source` / `detail_source` / `emotion_source` / `action_source` を持たせる。後からLLM要約に差し替える場合も、rawを上書きしない。

## KPI

UIで見せるKPIは、リアルタイム表示と分析用で分ける。

UI表示:

- 平均ストレス
- 孤立気味人数
- 摩擦件数
- 修復率
- 個室待ち
- 高ストレス者

分析用:

- `total_messages`
- `unique_interaction_pairs`
- `help_signal_messages`
- `conflict_threads`
- `repair_threads`
- `unresolved_conflict_threads`
- `reciprocity_rate`
- `load_fairness`

## Run A / Run B の差

Run A:

- 摩擦が長引きやすい
- 修復会話が遅れる
- 個室待ちや声かけの曖昧さが残る
- 高ストレス者が会話から離れやすい

Run B:

- 摩擦は起きる
- オブジェクトが修復の入口になる
- 「断れる」「短く話せる」「場を移せる」選択肢が増える
- 全員が救われるのではなく、関係の一部が少し早く戻る

## 実装状態 / 実装順

1. 完了: `scripts/export_habitat_frames.py` で `habitat_frames.jsonl` / `messages.jsonl` / `conversation_threads.tsv` / `agent_positions.tsv` / `module_occupancy.tsv` / `sleep_assignments.tsv` を生成する。
2. 完了: UIデモは `outputs/runs/iss_habitat_run_a` / `outputs/runs/iss_habitat_run_b` が読める場合、生成データ由来で表示し、読めない場合はHTML内デモへフォールバックする。
3. 次: `run_agents.py` でLLM実発話の `messages.jsonl` を直接出力する。`private_talk` は内面ログとして分離する。
4. 次: A/B比較を `messages.jsonl` と `conversation_threads.tsv` ベースに戻す。
5. 次: 20 agents / 100 steps でも同じschemaでUIが動くようにする。
