# Open Notebook Enhanced タスク別要件

全体の目的・共通方針は[要件定義マスター](../open-notebook-enhanced-requirements.md)を参照。
実装では各GitHub Issueに対応する仕様書を使い、タスク固有の範囲と受け入れ条件を確認する。

| ID | タスク仕様 | Issue |
|---|---|---|
| T01 | [Enhanced共通契約とADR](T01-foundation.md) | [#1](https://github.com/AiraCometes/open-notebook/issues/1) |
| T02 | [Embedding状態モデルと進捗API](T02-embedding-status-api.md) | [#3](https://github.com/AiraCometes/open-notebook/issues/3) |
| T03 | [EmbeddingJobとBatchワーカー](T03-batch-worker.md) | [#9](https://github.com/AiraCometes/open-notebook/issues/9) |
| T04 | [ジョブ操作と再試行](T04-job-controls.md) | [#7](https://github.com/AiraCometes/open-notebook/issues/7) |
| T05 | [Embeddingモード選択GUI](T05-mode-ui.md) | [#8](https://github.com/AiraCometes/open-notebook/issues/8) |
| T06 | [Batch管理・検索可能状態UI](T06-batch-dashboard.md) | [#10](https://github.com/AiraCometes/open-notebook/issues/10) |
| T07 | [Hybrid Search](T07-hybrid-search.md) | [#5](https://github.com/AiraCometes/open-notebook/issues/5) |
| T08 | [Reranker境界とProvider](T08-reranker-providers.md) | [#6](https://github.com/AiraCometes/open-notebook/issues/6) |
| T09 | [Reranker API設定GUI](T09-reranker-settings-ui.md) | [#11](https://github.com/AiraCometes/open-notebook/issues/11) |
| T10 | [接続テストとモデル検出](T10-reranker-test.md) | [#2](https://github.com/AiraCometes/open-notebook/issues/2) |
| T11 | [検索統合とフォールバック](T11-reranker-search.md) | [#12](https://github.com/AiraCometes/open-notebook/issues/12) |
| T12 | [品質・性能評価と運用文書](T12-evaluation-docs.md) | [#4](https://github.com/AiraCometes/open-notebook/issues/4) |

Issue本文には該当仕様書のGitHub上のパーマリンクを記載する。仕様変更時はIssueと対応仕様書を
同時に更新し、他タスクにも影響する場合はマスターの共通方針を更新する。
