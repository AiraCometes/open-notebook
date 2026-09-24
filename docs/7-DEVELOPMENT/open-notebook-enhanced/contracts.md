# Open Notebook Enhanced 共通契約

T01（Issue [#1](https://github.com/AiraCometes/open-notebook/issues/1)）の成果物。
T02〜T11 の実装が共有する状態・データ・API・互換性の契約をここに固定する。
永続化・マイグレーション・後方互換性の構造的判断は
[ADR-009](../decisions/ADR-009-enhanced-state-persistence.md) に記録した。

実装時にこの契約と矛盾する必要が生じた場合は、先に本書と該当タスク仕様書を更新し、
複数タスクに影響する変更は[要件定義マスター](../open-notebook-enhanced-requirements.md)にも反映する。

## 1. 共通規約

### 1.1 レコードIDと識別子

| 対象 | ID形式 | 例 |
|---|---|---|
| Source | `source:<key>` | `source:abc123` |
| EmbeddingJob | `embedding_job:<key>` | `embedding_job:xyz` |
| EmbeddingJobItem | `embedding_job_item:<key>` | — |
| SourceEmbedding | `source_embedding:<key>` | — |
| RerankerConfig | `reranker_config:<key>` | — |
| Credential | `credential:<key>` | `credential:k1` |
| surreal-commands command | `command:<key>` | — |

APIはID文字列を受け取る。テーブル名が要求対象と一致しないIDは
`InvalidInputError`（400）で拒否する（`resolve_notebook_scope` と同じ方針）。

### 1.2 状態語彙の層分離

- `source.embedding_status`、`embedding_job.status` は**ドメイン状態**であり、
  UI・集計・検索可否判断が参照する唯一の状態。
- surreal-commands の `command.status`（`new`/`running`/`completed`/`failed`/`canceled`）は
  **実行基盤の内部状態**であり、ドメイン状態とは別物として扱う。
  EmbeddingJob の実行基盤が surreal-commands を利用しても、command.status を
  ドメイン状態として外部公開しない。既存の `source.command` フィールドと
  `SourceResponse.status`（加工処理コマンドの状態）は従来どおりの意味を持ち、
  `embedding_status` と混同しない。

### 1.3 エラー応答

- HTTPエラーは従来どおり FastAPI `{"detail": string}` 形式。
  型付き例外→HTTP の対応は既存マッピングに従う:
  `InvalidInputError`→400、`NotFoundError`→404、`AuthenticationError`→401、
  `RateLimitError`→429、`ConfigurationError`→422、
  `NetworkError`/`ExternalServiceError`→502、`OpenNotebookError`→500。
- 状態遷移の競合・不正遷移には **409 Conflict** を使う。
  新規の `ConflictError`（`OpenNotebookError` 派生）を追加し、
  グローバルハンドラで 409 に写像する（最初に 409 を返すタスクで追加する
  — T02 の retry または T04）。
- 構造化エラーが必要なフィールド（`last_error`、アイテム履歴、接続テスト結果）は
  次の共有オブジェクトを使う:

```json
{"kind": "rate_limit", "message": "Provider returned 429", "at": "2026-09-24T10:00:00Z"}
```

| `kind` | 意味 | 主なHTTP対応 |
|---|---|---|
| `validation` | 入力・データ不正（再試行しても同じ） | 400 |
| `configuration` | モデル未設定、Credential欠落など | 422 |
| `authentication` | API Key 不備・失効 | 401 |
| `rate_limit` | Provider のレート制限 | 429 |
| `timeout` | タイムアウト | 502 |
| `network` | 接続・DNS障害 | 502 |
| `provider` | 上記以外の Provider 応答異常 | 502 |
| `cancelled` | キャンセルによる中断 | — |
| `unknown` | 分類不能 | 500 |

- **秘密情報の非露出**: `message` およびログに API Key・認証ヘッダー値を含めない。
  Provider のエラー文をそのまま転記しない（必要に応じて分類済み文言に変換する）。

### 1.4 進捗通知

初期版はポーリングのみ（WebSocket なし）。状態APIは読み取り専用で、
読み取り時に集計クエリを走らせず永続化済みカウンタを返す（§4.4）。

## 2. Embeddingモードの意味

`embedding_mode` は「Embedding処理をどうスケジュールするか」の利用者向け指定であり、
Provider API への1リクエストあたりチャンク数（**APIバッチサイズ**）とは無関係の別概念。

| モード | 意味 | 実行経路 |
|---|---|---|
| `normal` | 資料追加の直後に Embedding を開始する。Batch ジョブより先に処理される（優先度クラス高）。従来の `embed=true` と同じ意味 | 既存の `embed_source` コマンド経路（Jobレコードを作らない） |
| `batch` | EmbeddingJob に積んでワーカーが順次処理する。大量投入向け | `embedding_job` + `embedding_job_item` 台帳経路 |
| `auto` | 投入時にシステムが `normal`/`batch` を判定する | 判定後、該当経路 |

### 2.1 要求モードと適用モード

- Sourceは `requested_mode`（`normal`/`batch`/`auto`）と `resolved_mode`
  （`normal`/`batch`）を別々に保持する。`auto` 以外は両者が一致する。
- `auto` の判定規則（初期値）: 推定チャンク数 `estimated = token_count(full_text)/CHUNK_SIZE`
  が設定値 `auto_batch_threshold`（既定 100 チャンク、ContentSettings で変更可）を
  超えたら `batch`、以下なら `normal`。
- 判定理由は `resolved_reason` に機械可読で記録する
  （例: `{"rule":"auto_batch_threshold","estimated_chunks":132,"threshold":100}`）。
  T05 のUIはこの値で「なぜそのモードになったか」を説明する。
- Embedding を行わない資料（`embed` 未指定で既定が never、等）は mode フィールドを
  持たない/None とし、`embedding_status` は `not_started` のまま。
- 既存の `SourceCreate.embed` フラグと `default_embedding_option`（ask/always/never）は
  「Embedding を**するか**」の契約として維持する。`embedding_mode` は
  「**どう**スケジュールするか」であり、embed 要求が成立した場合にのみ意味を持つ。
  `embed=true` + mode 省略 → 既定モード（初期値 `normal`）。

### 2.2 APIバッチサイズ（内部）

- `api_batch_size` = Embedding API の1リクエストに含めるチャンク数。
  既定値は既存の `OPEN_NOTEBOOK_EMBEDDING_BATCH_SIZE` 環境変数（50）を継承する。
- EmbeddingJob ごとに任意で上書き可能（範囲 1〜2048、Provider 上限に合わせる）。
- `concurrency` = ジョブ内で同時に発行する APIバッチ数（既定 1、範囲 1〜16）。
- どちらもジョブのモードに依存しない性能パラメータであり、
  「Batchモードだから大きい」等の連動はしない。

## 3. Source の Embedding 状態機械

`source.embedding_status` の値域（永続化されるドメイン状態）:

| 状態 | 意味 | 終端 |
|---|---|---|
| `not_started` | Embedding未要求・未実行。既存資料の移行既定値 | — |
| `queued` | ジョブ/コマンドに受理済み。ワーカー着手待ち、または一時停止中ジョブの未着手分 | — |
| `preparing` | ワーカーが着手。テキスト読込・チャンク分割・バリデーション中（チャンク未書込） | — |
| `processing` | チャンクのEmbeddingが進行中。カウンタが増加する | — |
| `retry_waiting` | 一時障害により再試行待ち（バックオフ中、または再試行の待機中） | — |
| `completed` | 全チャンク Embedding 済み | 終端 |
| `partial` | 一部チャンクが永続失敗、一部は利用可能 | 終端 |
| `failed` | 利用可能なチャンクなしで終了（永続エラー） | 終端 |
| `cancelled` | キャンセルで終了。済みチャンクは保持する | 終端 |

### 3.1 遷移図

```text
 not_started ──▶ queued ──▶ preparing ──▶ processing ──▶ completed
                                          │     ▲
                                          ▼     │
                                      retry_waiting ──┘
                                          │
                                          ▼
                                    partial / failed
```

| From → To | 契機 |
|---|---|
| `not_started → queued` | Embedding 要求が受理された（normal送信 / ジョブ所属） |
| `queued → preparing` | ワーカーが資料に着手 |
| `preparing → processing` | `total_chunks` 確定・最初のチャンク処理開始 |
| `preparing → failed` | 永続エラー（本文なし、モデル未設定等） |
| `preparing/processing → retry_waiting` | 一時障害でリトライ待ち |
| `retry_waiting → processing` | リトライ実行で再開 |
| `retry_waiting → failed/partial` | リトライ上限到達 |
| `processing → completed` | 全チャンク成功 |
| `processing → partial` | 一部チャンク永続失敗で終了 |
| `processing → failed` | 永続エラー、利用可能チャンク0 |
| 非終端 → `queued` | ジョブ一時停止・ワーカー再起動による退避 |
| 非終端 → `cancelled` | ジョブ/資料のキャンセル |
| `partial`/`failed`/`cancelled` → `queued` | 失敗分再試行（未Embeddingチャンクのみ） |
| 任意 → `queued` | 明示的な再Embedding（rebuild経路。カウンタをリセット） |

遷移規則:

- `preparing` 完了時に `total_chunks` が確定し、そのランの間不変。
- `processing → retry_waiting`: 一時障害でリトライがスケジュールされた場合。
  `retry_waiting → processing` で再開。リトライ上限到達で `failed` または `partial`。
- 終端（`completed`/`partial`/`failed`/`cancelled`）からの遷移は次の2つのみ:
  - **失敗分再試行**（`partial`/`failed`/`cancelled` → `queued`）:
    未Embeddingチャンクのみを対象に再投入する。
  - **明示的な再Embedding**（任意の状態 → `queued`）: 既存 rebuild 経路。
    カウンタをリセットして新規ランとして扱う。
- `completed` はジョブ操作の対象外（pause/resume/cancel/retry-failed は 409）。

### 3.2 カウンタ不変条件

常に `completed_chunks + failed_chunks <= total_chunks`。
`pending = total_chunks - completed_chunks - failed_chunks` は導出値（保存しない）。

| 状態 | 不変条件 |
|---|---|
| `not_started` | total=0, completed=0, failed=0 |
| `queued` | 前ランの値を保持するか、新規ランでは 0 |
| `preparing` | completed=0, failed=0（total は確定後 > 0） |
| `processing`/`retry_waiting` | completed + failed < total |
| `completed` | total > 0, completed == total, failed == 0 |
| `partial` | completed > 0, failed > 0, completed + failed == total |
| `failed` | completed == 0（preparing 中の失敗は total=0 もあり得る） |
| `cancelled` | completed + failed <= total（残りは未処理のまま残る） |

`completed_chunks` は `source_embedding` 実件数と一致する
（ジョブ/コマンド側が書き込みと同一トランザクション相当で更新する責務を持つ）。
`completed_chunks > 0` の資料はベクトル検索の対象に部分的に含まれる。

## 4. EmbeddingJob の状態機械

`embedding_job.status` の値域:

| 状態 | 意味 | 終端 |
|---|---|---|
| `queued` | 作成済み・ワーカー待ち（priority 順） | — |
| `running` | ワーカーがアイテムを処理中 | — |
| `paused` | 利用者が一時停止。新規チャンク処理は開始されない | — |
| `completed` | 全メンバー資料が `completed` | 終端 |
| `partial` | 完了と失敗が混在（≥1件が completed 以外の終端） | 終端 |
| `failed` | 全メンバー失敗、またはジョブ全体の永続エラー | 終端 |
| `cancelled` | キャンセル済み。**自動再開しない** | 終端 |

### 4.1 遷移図

```text
 queued ──▶ running ──▶ paused ──▶ queued（resume）
   │          │          │
   │          ├─▶ completed（全資料 completed）
   │          ├─▶ partial（完了・失敗混在）
   │          ├─▶ failed（全滅 or ジョブ全体の永続エラー）
   │          └─▶ cancelled
   ├─▶ paused（pickup前のpause要求）
   └─▶ cancelled        （paused ──▶ cancelled も可）

 running ──▶ queued（ワーカー再起動による再キュー: 孤立 running の回収）
 partial / failed ──▶ queued（retry-failed: 失敗アイテムのみ再投入、attempt+=1）
 cancelled / completed: 遷移なし（残作業は新規ジョブ）
```

- **pause/resume/cancel の粒度**: 制御はチャンクの APIバッチ境界で効く。
  実行中の APIリクエストは中断せず完了させ、次バッチを開始しない
  （T04 受入「一時停止後、新しいチャンク処理が開始されない」に対応）。
- **終端到達の競合**: 終端状態は不可逆。ワーカーが終端を観測した後の
  制御書込みは効かない。`cancelled` は自動再開しない（T04 受入）。
- `paused` → `queued`（resume）で同一 priority のまま待ち列に戻る。
  `paused` のまま pickup されることはない（ワーカーは `queued` のみ取得）。

### 4.2 制御チャネル（status と control の分離）

利用者の操作意図とワーカーの実行状態は別フィールドで持つ:

- `status`（§4.1）: ワーカーが遷移させるライフサイクル状態。
- `control`: `none` | `pause_requested` | `cancel_requested`。
  操作APIが書き、ワーカーがバッチ境界で読み、`status` に反映後 `none` に戻す。

これにより「要求したが未反映」（例: `status=running, control=pause_requested` =
pausing 中）を表現でき、操作要求とワーカー実行の競合を一貫して扱う（T04）。
`status`/`control` の更新は条件付きの単一 UPDATE（`WHERE status = $expected`）で
行い、ワーカーとAPIの二重書き込みを防止する。

### 4.3 操作の冪等性

| 操作 | 有効な現在状態 | すでに目的状態なら | それ以外 |
|---|---|---|---|
| pause | `queued`, `running` | `paused` → 200 no-op | 終端 → 409 |
| resume | `paused` | `queued`/`running` → 200 no-op | 終端 → 409 |
| cancel | `queued`, `running`, `paused` | `cancelled` → 200 no-op | 他終端 → 409 |
| retry-failed | `partial`, `failed` かつ失敗アイテム ≥1 | — | 実行中 → 409、失敗0件 → 409、`completed`/`cancelled` → 409 |

同一操作の多重送信で状態を壊さないこと（T04 受入）がこの表の目的。
同じ `control` 値を既に要求中（例: `control=pause_requested` で未反映）の
再送も 200 no-op とする。

### 4.4 責務と進捗集計

| レコード | 責務 |
|---|---|
| `source`（新規フィールド群） | 資料単位の利用者向け状態: mode、状態、カウンタ、モデル識別、最終エラー、現在のジョブ参照。一覧・詳細画面の一次参照。 |
| `embedding_job` | スケジューリング単位: メンバー資料集合（作成時スナップショット）、優先度、ライフサイクル、集計カウンタ、性能パラメータ、最終エラー。 |
| `embedding_job_item` | チャンク台帳: （job, source, content_hash）単位の pending/embedded/failed/skipped、attempts、エラー履歴、生成した source_embedding への参照。再開・重複防止・失敗分再試行の根拠。 |
| `source_embedding` | 成功済みチャンクのみ（従来どおり）。新規に `content_hash` を持ち、ジョブ外（normal経路）の重複判定にも使う。 |

集計ルール:

- アイテム書込み → 資料カウンタ更新 → 資料状態を §3.2 の不変条件で導出、を
  同一の書込み単位で行う（ワーカー責務）。
- ジョブの `total/completed/failed_chunks`・`*_sources` はメンバー資料の合計を
  **書込み時に維持**する（読み取り時の fan-out 集計をしない）。
- `GET` 系APIは保存済み値を返すのみ。`completed_sources` は資料状態が
  `completed` の件数、`failed_sources` は `partial`/`failed`/`cancelled` の件数とする。
- normal経路（ジョブ非所有）の資料も同じ Source カウンタ契約に従う。
  `embed_source` コマンドは `content_hash` 差分で既Embeddingチャンクをスキップし、
  失敗チャンクのみを再Embeddingできる（normal資料の「失敗分再試行」の実装根拠）。

### 4.5 再起動・重複の安全

- ワーカー起動時に `running` の孤立ジョブを `queued` に戻す
  （`paused`/`cancelled`/終端は触らない）。
- 作業単位 `(job, source, content_hash)` は一意。`embedded`/`skipped` アイテムは
  再実行しない。`failed` アイテムは retry-failed でのみ pending に戻る。
- 投入前に `(source, content_hash)` で既存 `source_embedding` を検査し、
  存在すればアイテムを `skipped`（参照を張り completed に計上）して
  二重 Embedding を防ぐ（T03 受入）。
- エラー履歴: アイテムの `errors` は追記専用（直近10件で打切り）、
  ジョブ/資料の `last_error` は最新1件。再試行で旧エラーを消さない（T04 受入）。

## 5. データ契約

### 5.1 `source` 追加フィールド（すべて `option<>` の追加変更のみ）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `requested_mode` | `option<string>` | — | `normal`/`batch`/`auto` |
| `resolved_mode` | `option<string>` | — | `normal`/`batch`（auto判定後） |
| `resolved_reason` | `option<object>` | — | auto判定の理由（§2.1） |
| `embedding_status` | `option<string>` | — | §3 の状態。未設定は `not_started` とみなす |
| `total_chunks` | `option<int>` | — | 現ランの総チャンク数 |
| `completed_chunks` | `option<int>` | — | Embedding済み数 |
| `failed_chunks` | `option<int>` | — | 永続失敗数 |
| `embedding_model` | `option<string>` | — | モデル識別のスナップショット（例 `openai:text-embedding-3-small`）。外部キーではない |
| `embedding_job` | `option<record<embedding_job>>` | — | 現在/直近のジョブ（normal経路では null） |
| `last_embedding_error` | `option<object>` | — | §1.3 の共有エラーオブジェクト |
| `embedding_updated_at` | `option<datetime>` | — | 状態・カウンタの最終更新 |

既存フィールド（`command`、`asset`、`full_text` など）は変更しない。

### 5.2 `embedding_job`（新規 SCHEMAFULL）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mode` | `string` | 必須 | `normal`/`batch`（解決済みモード。`auto` は格納しない） |
| `status` | `string` | 必須 | §4.1。作成時 `queued` |
| `control` | `string` | 必須 | §4.2。既定 `none` |
| `priority` | `int` | 必須 | 大きい方が先。normal経路の既定 100、batch 既定 0 |
| `source_ids` | `array<record<source>>` | 必須 | メンバー。作成後は不変 |
| `total_sources`/`completed_sources`/`failed_sources` | `int` | 必須 | §4.4 の集計 |
| `total_chunks`/`completed_chunks`/`failed_chunks` | `int` | 必須 | 同上 |
| `api_batch_size` | `option<int>` | — | §2.2。null は環境既定値 |
| `concurrency` | `option<int>` | — | §2.2。null は 1 |
| `model` | `option<string>` | — | 実行に使ったモデル識別スナップショット |
| `attempt` | `int` | 必須 | 再試行パス回数。初期 0、retry-failed で +1 |
| `last_error` | `option<object>` | — | §1.3 の共有エラーオブジェクト |
| `created`/`updated`/`started_at`/`finished_at` | `datetime`/`option<datetime>` | — | started_at=初回実行開始、finished_at=終端遷移時 |

### 5.3 `embedding_job_item`（新規 SCHEMAFULL）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `job` | `record<embedding_job>` | 必須 | 所有ジョブ |
| `source` | `record<source>` | 必須 | 対象資料 |
| `order` | `int` | 必須 | 資料内チャンク順序（0始まり） |
| `content_hash` | `string` | 必須 | チャンク本文の sha256（重複判定キー） |
| `status` | `string` | 必須 | `pending`/`embedded`/`failed`/`skipped` |
| `attempts` | `int` | 必須 | 実行試行数 |
| `errors` | `array<object>` | 必須 | §1.3 共有エラーの追記履歴（最大10件） |
| `embedding` | `option<record<source_embedding>>` | — | 生成された行への参照 |
| `created`/`updated` | `datetime` | — | |

一意性: `(job, source, content_hash)`。索引: `job`+`status`（作業取得用）、
`source`（資料別照会用）。

### 5.4 `source_embedding` 追加フィールド

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `content_hash` | `option<string>` | — | §4.5 の重複判定。既存行は null のまま |

### 5.5 `reranker_config`（新規 SCHEMAFULL）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `string` | 必須 | 表示名。1〜100文字 |
| `provider` | `string` | 必須 | `cohere`/`jina`/`custom_http`（T08の登録方式で拡張） |
| `model` | `option<string>` | provider依存 | cohere/jina は必須、custom_http は任意 |
| `base_url` | `option<string>` | provider依存 | custom_http は必須。`validate_url()` 経由（SSRF対策・ローカル許可は既存方針どおり） |
| `credential` | `option<record<credential>>` | — | 既存 Credential への参照。秘密値を複製しない |
| `enabled` | `bool` | 必須 | 既定 false（登録≠有効化） |
| `is_default` | `bool` | 必須 | 既定 false。既定は最大1件（設定時に他を原子的に解除） |
| `candidate_top_k` | `int` | 必須 | 初期候補数。1〜200、既定 50 |
| `final_top_n` | `int` | 必須 | 最終返却数。1〜50、既定 10 |
| `timeout_ms` | `int` | 必須 | 1,000〜60,000、既定 10,000 |
| `fallback_on_error` | `bool` | 必須 | 既定 true（失敗時は元順位で応答するマスター方針） |
| `last_test` | `option<object>` | — | `{at, status: success/failure, latency_ms, error: 共有エラー}` |
| `created`/`updated` | `datetime` | — | |

制約: `candidate_top_k >= final_top_n`。`is_default=true` は `enabled=true` が前提。
`credential` は存在する `credential:<id>` のみ（削除は参照がある限り 409 で拒否）。

## 6. API契約

すべて `/api` 配下の新規パス（既存パスの入出力変更は §6.5 の追加項目のみ）。
表の「必須」はリクエスト側の必須性を示す。

### 6.1 資料の Embedding 状態

`GET /api/sources/{source_id}/embedding-status` → `SourceEmbeddingStatusResponse`

| フィールド | 型 | 説明 |
|---|---|---|
| `source_id` | string | |
| `requested_mode`/`resolved_mode` | string \| null | §2.1 |
| `resolved_reason` | object \| null | auto判定理由 |
| `status` | string | §3。未設定資料は `not_started` |
| `total_chunks`/`completed_chunks`/`failed_chunks` | int | §3.2 |
| `pending_chunks` | int | 導出値（total−completed−failed） |
| `embedding_model` | string \| null | |
| `embedding_job_id` | string \| null | 所有ジョブ（あれば） |
| `job_status`/`job_control` | string \| null | 所有ジョブの状態（pause中の表示用） |
| `searchable_chunks` | int | = `completed_chunks` |
| `last_error` | object \| null | §1.3 共有エラー |
| `updated` | string \| null | |

エラー: 400 ID形式不正、404 資料なし。

`GET /api/embedding/summary` → 全体集計（T06 の Search/Ask 表示用）

```json
{ "searchable_chunks": 0, "sources_processing": 0, "sources_failed": 0,
  "sources_not_started": 0, "jobs_active": 0 }
```

- `searchable_chunks` = 全資料の `completed_chunks` 合計。
- `sources_processing` = `queued`/`preparing`/`processing`/`retry_waiting` 件数、
  `sources_failed` = `partial`/`failed`/`cancelled` 件数。
- `jobs_active` = `queued`/`running`/`paused` のジョブ件数。
- 実装は `source` / `embedding_job` への集計クエリ1本ずつでよい
  （§1.4 が禁じるのは行ごとの fan-out であり、GROUP ALL 集計は対象外）。

`POST /api/sources/{source_id}/embedding/retry` → `SourceEmbeddingStatusResponse`

- `partial`/`failed`/`cancelled` から失敗分のみ再投入（§3.1）。
  batch資料は所属ジョブの retry-failed 経路、normal資料は `embed_source` 再送
  （content_hash 差分で未分のみ）。
- エラー: 404 資料なし、409 実行中または `completed`/`not_started`（再試行対象なし）。

### 6.2 EmbeddingJob

`POST /api/embedding-jobs` → 201 `EmbeddingJobResponse`

| フィールド | 型 | 必須 | 制約 |
|---|---|---|---|
| `source_ids` | `array<string>` | 必須 | 1〜500件。重複除去。各要素は `source:` 形式 |
| `mode` | `string` | — | `normal`/`batch`、既定 `batch` |
| `api_batch_size` | int \| null | — | 1〜2048 |
| `concurrency` | int \| null | — | 1〜16 |
| `priority` | int \| null | — | 省略時はモード既定値 |

エラー: 400 空配列・ID形式不正・モード不正、404 資料なし、
422 モデル未設定（`ConfigurationError`）。

`GET /api/embedding-jobs` → `EmbeddingJobListResponse`

- クエリ: `status`（カンマ区切り複数可）、`mode`、`limit`（1〜200、既定50）、`offset`（既定0）。
- 応答: `{ "jobs": [...], "summary": { "queued": n, "running": n, "paused": n, "completed": n, "partial": n, "failed": n, "cancelled": n } }`。
  summary はフィルタ無視の全件集計（T06 の俯瞰用）。

`GET /api/embedding-jobs/{job_id}` → `EmbeddingJobResponse` + `sources` 配列
（メンバー資料の `{source_id, status, total/completed/failed_chunks}`、上限 200 件。
それ以上は items エンドポイントでページング）。

`GET /api/embedding-jobs/{job_id}/items` → アイテム台帳のページング取得
（`status` フィルタ、`limit`/`offset`。失敗プレビュー用）。

`EmbeddingJobResponse` = §5.2 の全フィールド（`id` 文字列化、日時は ISO 文字列）。

`POST /api/embedding-jobs/{job_id}/pause` | `resume` | `cancel` | `retry-failed`
→ 200 `EmbeddingJobResponse`（更新後の状態）

- 遷移・冪等性は §4.3 の表どおり。
- エラー: 404 ジョブなし、409 不正遷移
  （例 `{"detail": "Cannot pause a completed job"}`、
  `{"detail": "No failed chunks to retry"}`）。

### 6.3 RerankerConfig

`GET /api/rerankers` → `RerankerConfigResponse[]`（一覧、enabled/last_test 状態を含む）
`POST /api/rerankers` → 201 `RerankerConfigResponse`
`GET /api/rerankers/{id}` / `PUT /api/rerankers/{id}` / `DELETE /api/rerankers/{id}`
`POST /api/rerankers/{id}/test` → `RerankerTestResponse`
`POST /api/rerankers/{id}/discover-models` → `DiscoverModelsResponse` 相当

`RerankerConfigRequest`（POST/PUT、PUT は省略=変更なし）:

| フィールド | 型 | 必須 | 制約 |
|---|---|---|---|
| `name` | string | POST必須 | 1〜100文字 |
| `provider` | string | POST必須 | `cohere`/`jina`/`custom_http`（将来拡張） |
| `model` | string \| null | provider依存 | cohere/jina 必須 |
| `base_url` | string \| null | provider依存 | custom_http 必須。`validate_url()` 通過 |
| `credential_id` | string \| null | — | `credential:` 形式・存在必須。**キー値そのものは受け取らない** |
| `api_key` | string \| null | — | 指定時は新規 Credential を生成して参照（PUT で省略=既存保持、T09受入対応） |
| `enabled` | bool | — | 既定 false |
| `is_default` | bool | — | `enabled=true` が前提 |
| `candidate_top_k`/`final_top_n`/`timeout_ms`/`fallback_on_error` | — | — | §5.5 の範囲・既定値 |

`RerankerConfigResponse`: §5.5 の全フィールド + `credential_id`、`has_credential`。
**API Key 値を返さない**（`has_credential` と Credential 側の `has_api_key` 慣行に倣う）。

`RerankerTestResponse`: `{ "success": bool, "latency_ms": int, "candidates": int, "error": 共有エラー | null, "tested_at": string }`。
接続テストの失敗は **HTTP 200 + `success:false`** で返す
（`POST /credentials/{id}/test` の既存慣行どおり。テスト失敗はデータであり障害ではない）。
`discover-models` 非対応 provider は 200 で `{ "supported": false, "models": [] }`。

エラー: 400 入力不正（URL検証失敗を含む）、404 config/credential なし、
409 参照中 Credential の削除、422 制約違反（`candidate_top_k < final_top_n`、
provider 必須項目欠落、範囲外数値）。

### 6.4 検索方式

`SearchRequest` の追加（すべて任意・後方互換）:

| フィールド | 型 | 説明 |
|---|---|---|
| `type` | `"text" \| "vector" \| "hybrid"` | `hybrid` を Literal に追加（既存値は不変） |
| `rerank` | bool \| null | true 時、既定 Reranker で再順位付けを試行 |
| `reranker_id` | string \| null | `reranker_config:<id>` を明示指定（既定を上書き） |

`SearchResponse` の追加（すべて任意フィールドとして追加）:

| フィールド | 型 | 説明 |
|---|---|---|
| `effective_type` | string \| null | 実際に使った検索方式（縮退時は要求と異なる） |
| `degraded` | bool | 縮退応答か |
| `degradation_reason` | string \| null | 共有エラー `kind`（例 `configuration`） |
| `reranker` | object \| null | `{status: applied/skipped/fallback, config_id, reason, latency_ms}` |

- 既存 `search_type` フィールドは従来どおり要求値を返す（互換維持）。
- Hybrid = `fn::text_search` と `fn::vector_search` を RRF で統合（T07）。
  片系統不可時は利用可能側のみで応答し `degraded=true`。
- Notebookスコープ（ADR-008）は全経路に同じく適用する。
- Reranker の失敗（`authentication`/`rate_limit`/`timeout`/`network`/`provider`）は
  検索全体を失敗させず、`fallback_on_error` 時は元順位 +
  `reranker.status="fallback"`。`fallback_on_error=false` 時のみ 502 で応答する。

### 6.5 既存 API への追加項目

| 対象 | 追加 |
|---|---|
| `SourceListResponse`/`SourceResponse` | `embedding_status`、`total_chunks`、`failed_chunks`、`requested_mode`、`resolved_mode`（すべて任意。未設定資料は null） |
| `SourceCreate` | `embedding_mode`: `normal`/`batch`/`auto`（任意。`requested_mode` に保存。embed 要求がない場合は無視し、エラーにしない） |
| `SettingsResponse`/`SettingsUpdate` | `default_embedding_mode`（`normal`/`batch`/`auto`、既定 `normal`）、`auto_batch_threshold`（int、既定100） |
| `ContentSettings` | 同上フィールド（RecordModel の SCHEMALESS 記録、マイグレーション不要） |

## 7. 永続化・移行・後方互換性（要約）

詳細と却下案は [ADR-009](../decisions/ADR-009-enhanced-state-persistence.md)。

- 新規3テーブル（`embedding_job`/`embedding_job_item`/`reranker_config`）は SCHEMAFULL。
  既存テーブルへの変更は `option<>` フィールドの追加のみ（既存行・既存コードを壊さない）。
- 移行バックフィル: Embedding行がある資料 → `completed` + 実件数、
  実行中 command を持つ資料 → `processing`、それ以外 → `not_started`。
  `requested_mode`/`resolved_mode` は、従来経路でEmbedding済みの資料のみ
  `normal` で埋め、それ以外は未設定（null）とする。
- APIは追加のみ（新パス・新任意フィールド・Literal の新値）。既存クライアントは無変更で動く。
- マイグレーションの単位・`_down`・採番は ADR-006 の方針を継承する。

## 8. タスク別の利用契約

| タスク | 主に使う契約 |
|---|---|
| T02 | §3 状態機械・§3.2 不変条件・§5.1・§6.1 |
| T03 | §4 ジョブ機械・§4.4/4.5・§5.2/5.3/5.4・§6.2 |
| T04 | §4.2 制御チャネル・§4.3 冪等性・§6.1 retry・§6.2 操作API |
| T05/T06 | §2 モード・§6.1/6.2 読み取りAPI・§6.5 一覧追加項目 |
| T07 | §6.4 検索方式・縮退ルール |
| T08 | §5.5 RerankerConfig・§1.3 エラー分類 |
| T09/T10 | §6.3 CRUD/test/discover・Credential 参照規則 |
| T11 | §6.4 reranker ブロック・フォールバック |
| T12 | §6.2/6.4 の観測フィールド（latency、状態、件数） |
