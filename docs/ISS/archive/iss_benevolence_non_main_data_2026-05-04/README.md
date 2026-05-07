# ISS Benevolence Non-Main Data Archive

このディレクトリは、`domain_packs/iss_benevolence/data/` から外した非主線データの退避先です。

退避理由:

- 現在の主線実行（`build_state.py` / `export_habitat_frames.py` / UI smoke）では直接参照しない
- 一部が旧tail実験用、または設計メモ用のTSVであり、正準実行データと混ざると誤読しやすい
- 必要になった場合は、明示的にこのarchiveから参照する

退避ファイル:

- `chronic_stress_event_templates.tsv`
- `conversation_schema.tsv`
- `preload_column_design.tsv`
- `run_matrix.tsv`
- `tail_trouble_design_protocol.tsv`
- `world_events.tsv`
- `events_tail_low.tsv`
- `events_tail_medium.tsv`
- `events_tail_high.tsv`
- `world_events_tail_low.tsv`
- `world_events_tail_medium.tsv`
- `world_events_tail_high.tsv`
