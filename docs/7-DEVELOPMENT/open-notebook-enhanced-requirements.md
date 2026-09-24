# Open Notebook Enhanced 要件定義マスター

## 1. 目的と位置づけ

本書は、[Open Notebook](https://github.com/lfnovo/open-notebook)を基盤とする
「Open Notebook Enhanced」の製品全体の目的、境界、共通方針を定めるマスター文書。
個別機能の詳細と受け入れ条件は、[タスク別要件一覧](open-notebook-enhanced/README.md)から
各仕様書を参照する。

実装Issueは対応するタスク仕様書を実行可能な単位として扱い、仕様変更はまず該当する
タスク仕様書に反映する。複数タスクに影響する変更は本書も更新する。

## 2. 解決する課題

大量の資料を取り込むときのEmbedding処理と、利用者が検索を実行するときの処理を
独立させる。処理状態をGUIで見えるようにし、Hybrid Searchと任意のRerankerで検索品質を
高める。Reranker APIの設定・接続確認もGUIから行えるようにする。

## 3. 製品目標

1. 大量資料のEmbeddingを、通常処理とBatch処理に分けて管理する。
2. 利用者が資料・ジョブごとのモード、進捗、失敗をGUIで判断できる。
3. Embedding中も、利用可能なチャンクや全文検索で検索を継続できる。
4. 全文検索とベクトル検索を組み合わせ、必要に応じてRerankerで順位を調整する。
5. Reranker ProviderとAPI認証情報を設定画面から管理し、安全に接続テストできる。
6. Open Notebookの既存データと主要な利用フローを保つ。

## 4. 対象範囲

### 対象

- Embeddingモード（通常、Batch、自動）とEmbeddingジョブ管理
- 状態・進捗表示、一時停止、再開、キャンセル、失敗分の再試行
- Hybrid SearchとEmbedding未完了時のフォールバック
- RerankerのProvider抽象化、API設定GUI、Credential連携、接続テスト
- Rerankerを使った再順位付けと失敗時フォールバック
- 検索品質と処理性能を確認する評価・運用ドキュメント

### 対象外

- 複数ホストにまたがる分散ワーカー、マルチテナント権限
- Embeddingモデルの学習・ファインチューニング
- 初期版でのWebSocket通知
- すべてのReranker Providerへの対応
- Chat、Podcastなど既存機能の大規模な仕様変更

## 5. 全体処理の考え方

```text
資料追加
  └─ 通常 / Batch / 自動
       └─ Embeddingジョブと進捗管理
            └─ 完了済みデータから検索可能

検索
  └─ Vector / Full-text / Hybrid
       └─ 任意でReranker
            └─ 結果と実行状態をGUIに表示
```

ユーザー向けのBatchモードは、Embedding APIへ送る内部バッチサイズとは別の概念とする。
Batchモードはジョブの優先度・スケジューリング・状態管理を指し、APIバッチサイズは
各リクエストに含めるチャンク数を指す。

## 6. 共通設計方針

- 非同期のジョブ処理を使い、検索要求が大量Embeddingの完了を待たない構造にする。
- ジョブと資料の状態を永続化し、再起動後に状態を確認・復旧できるようにする。
- 既存のCredential機構を使い、API Keyを平文で返さず、ログにも出さない。
- Rerankerは任意機能とし、未設定・タイムアウト・API障害時には元の検索結果で応答する。
- API追加は後方互換を保ち、既存の検索・Ask・Notebookスコープを維持する。
- 初期版の進捗更新はポーリングを基本とする。
- データモデル、マイグレーション、Provider境界など構造的判断はADRに記録する。

## 7. 用語

| 用語 | 意味 |
|---|---|
| チャンク | 資料を検索用に分割したテキスト |
| Embedding | テキストをベクトルへ変換した値 |
| Embeddingジョブ | 1件以上の資料のEmbedding実行を管理する単位 |
| APIバッチ | Embedding APIへ一度に送るチャンク数 |
| Batchモード | 大量資料のEmbeddingジョブをキュー管理する利用者向けモード |
| Candidate | 初期検索で取得した再順位付け前の候補 |
| Reranker | Queryと候補の関連度を評価し順位を付け直すモデル |
| Hybrid Search | 全文検索とベクトル検索を統合する検索方式 |

## 8. タスク別仕様・Issue

各仕様書をタスクの詳細要件と受け入れ条件の一次参照先とする。

| Task | 仕様書 | GitHub Issue |
|---|---|---|
| T01 Enhanced共通契約とADR | [T01](open-notebook-enhanced/T01-foundation.md) | [#1](https://github.com/AiraCometes/open-notebook/issues/1) |
| T02 Embedding状態モデルと進捗API | [T02](open-notebook-enhanced/T02-embedding-status-api.md) | [#3](https://github.com/AiraCometes/open-notebook/issues/3) |
| T03 EmbeddingJobとBatchワーカー | [T03](open-notebook-enhanced/T03-batch-worker.md) | [#9](https://github.com/AiraCometes/open-notebook/issues/9) |
| T04 ジョブ操作と再試行 | [T04](open-notebook-enhanced/T04-job-controls.md) | [#7](https://github.com/AiraCometes/open-notebook/issues/7) |
| T05 Embeddingモード選択GUI | [T05](open-notebook-enhanced/T05-mode-ui.md) | [#8](https://github.com/AiraCometes/open-notebook/issues/8) |
| T06 Batch管理・検索可能状態UI | [T06](open-notebook-enhanced/T06-batch-dashboard.md) | [#10](https://github.com/AiraCometes/open-notebook/issues/10) |
| T07 Hybrid Search | [T07](open-notebook-enhanced/T07-hybrid-search.md) | [#5](https://github.com/AiraCometes/open-notebook/issues/5) |
| T08 Reranker境界とProvider | [T08](open-notebook-enhanced/T08-reranker-providers.md) | [#6](https://github.com/AiraCometes/open-notebook/issues/6) |
| T09 Reranker API設定GUI | [T09](open-notebook-enhanced/T09-reranker-settings-ui.md) | [#11](https://github.com/AiraCometes/open-notebook/issues/11) |
| T10 接続テストとモデル検出 | [T10](open-notebook-enhanced/T10-reranker-test.md) | [#2](https://github.com/AiraCometes/open-notebook/issues/2) |
| T11 検索統合とフォールバック | [T11](open-notebook-enhanced/T11-reranker-search.md) | [#12](https://github.com/AiraCometes/open-notebook/issues/12) |
| T12 品質・性能評価と運用文書 | [T12](open-notebook-enhanced/T12-evaluation-docs.md) | [#4](https://github.com/AiraCometes/open-notebook/issues/4) |

## 9. 依存関係の概略

```text
T01 → T02 → T03 → T04
             ├──→ T05 → T06
             └──→ T07
T01 → T08 → T09 → T10
             └──→ T11 ← T07
T02〜T11 → T12
```

個別Issueの着手条件や並行可能な範囲は、各タスク仕様書で管理する。

## 10. 参考資料

- [Open Notebook](https://github.com/lfnovo/open-notebook)
- [Embeddingコマンド](https://github.com/lfnovo/open-notebook/blob/main/commands/embedding_commands.py)
- [検索ユーザーガイド](https://github.com/lfnovo/open-notebook/blob/main/docs/3-USER-GUIDE/search.md)
- [大容量Embeddingの課題と内部バッチ対応](https://github.com/lfnovo/open-notebook/issues/536)
- [Hybrid Searchの設計議論](https://github.com/lfnovo/open-notebook/issues/1036)
- [Reranker対応の設計議論](https://github.com/lfnovo/open-notebook/issues/1087)
