# Open Notebook Enhanced 要件定義書

## 1. 文書の目的

本書は、[Open Notebook](https://github.com/lfnovo/open-notebook) を基盤に、
大量資料のEmbedding処理と検索時の低遅延処理を分離し、Rerankerによる検索結果の
再順位付けを追加する「Open Notebook Enhanced」の製品要件を定義する。

本書を実装Issue、設計判断、受け入れテストの基準とする。実装上の都合で仕様を変更
する場合は、本書または関連するIssueを先に更新する。

## 2. 背景と解決したい課題

Open Notebookは資料をチャンク化し、Embeddingを生成してベクトル検索に利用する。
一方で、大量の資料を一度に取り込む場合は、APIのレート制限、処理時間、失敗時の
再実行、検索可能になるまでの状態が利用者から分かりにくくなりやすい。

Enhancedでは、資料の搬入処理と検索処理を別のレーンとして扱う。

- **Batch処理**：大量資料をキューに積み、進捗を管理しながらバックグラウンドで処理する。
- **通常処理**：少量の資料を優先度高く処理し、早く検索可能にする。
- **検索処理**：Embedding処理の完了を待たず、検索可能なデータだけで応答する。
- **Reranker**：検索候補を別モデルで再評価し、最終的な関連度順に並べ替える。

## 3. 対象範囲

### 3.1 今回の対象

1. Embedding処理モード（通常、Batch、自動）の追加
2. Embeddingジョブの状態・進捗・失敗・再試行管理
3. Batch処理と通常処理をGUI上で明示
4. Hybrid Search（全文検索とベクトル検索の統合）
5. Rerankerの抽象化と検索パイプラインへの追加
6. Reranker API設定GUI
7. Rerankerの接続テスト、タイムアウト、検索結果へのフォールバック
8. 評価用の検索品質・処理性能メトリクス

### 3.2 今回の対象外

- 分散ワーカーを複数ホストへ展開する仕組み
- マルチテナント向けの権限管理
- Embeddingモデル自体の学習やファインチューニング
- WebSocketによるリアルタイム通知（初期版はポーリング）
- Reranker APIの全プロバイダーへの対応
- 既存Open NotebookのChatやPodcast機能の大幅な仕様変更

## 4. 用語

| 用語 | 意味 |
|---|---|
| チャンク | 資料を検索可能な単位に分割したテキスト |
| Embedding | テキストをベクトルへ変換した値 |
| Embeddingジョブ | 1件以上の資料のEmbedding処理を管理する単位 |
| APIバッチ | Embedding APIへ一度に送信するチャンク数 |
| Batchモード | 大量資料を低優先度でバックグラウンド処理するユーザー向けモード |
| Candidate | Vectorまたは全文検索で取得したReranker前の候補 |
| Reranker | Queryと候補文書の関連度を再評価して順位を付けるモデル |
| Hybrid Search | 全文検索とベクトル検索を統合した検索方式 |

## 5. プロダクト要件

### REQ-001：Embeddingモードを選択できる

資料追加時に、次のモードを選択できること。

- `normal`：通常処理。優先度を高くして速やかに処理する。
- `batch`：Batch処理。キューに積み、バックグラウンドで処理する。
- `auto`：ファイル数、ファイルサイズ、チャンク数などの閾値で自動判定する。

内部のAPIバッチサイズと、ユーザー向けのBatchモードは別の設定として扱う。

### REQ-002：Embedding状態を追跡できる

資料とジョブについて、少なくとも次の状態を保持すること。

`not_started`、`queued`、`preparing`、`processing`、`completed`、`partial`、
`failed`、`retry_waiting`、`cancelled`

状態には、総チャンク数、完了数、失敗数、処理モード、利用モデル、モデルバージョン、
最終エラーを関連付ける。

### REQ-003：Batchジョブを操作できる

ユーザーはBatchジョブを一覧で確認し、次の操作を実行できること。

- 一時停止
- 再開
- キャンセル
- 失敗チャンクのみ再試行
- 優先度変更

### REQ-004：Embedding未完了でも利用できる

Embedding処理中の資料について、Embedding済みのチャンクは検索対象にする。
Embeddingがない場合は、全文検索へフォールバックできること。

検索結果が不完全な可能性がある場合は、GUI上で検索可能数と処理中の件数を示す。

### REQ-005：Hybrid Searchを利用できる

検索は、全文検索とベクトル検索の結果を統合できること。初期版の順位統合方式は
Reciprocal Rank Fusion（RRF）とする。

### REQ-006：Rerankerを任意で有効化できる

Rerankerは検索パイプラインの後段に配置し、候補集合を再順位付けする。

初期値の目安は次のとおり。

- 候補取得数：50件
- 最終結果数：8件
- タイムアウト：3秒
- Reranker失敗時：Hybrid Searchの順位で継続

### REQ-007：Reranker APIをGUIから設定できる

設定画面で、Rerankerの登録・編集・有効化・無効化・削除ができること。

最低限、次の項目を設定できること。

- 表示名
- Provider
- Model名
- Base URL
- API Key
- Candidate Top K
- Final Top N
- Timeout
- 失敗時のフォールバック有無

API Keyは既存のCredential機構に暗号化して保存し、設定取得APIや画面へ平文で返さない。

### REQ-008：Reranker APIをテストできる

設定画面の「接続テスト」で、認証、モデル名、レスポンス形式、スコア取得、タイムアウトを
検証できること。テスト結果には成功・失敗理由・レイテンシを表示する。

### REQ-009：Reranker Providerを拡張できる

RerankerはProvider固有の処理を検索本体から分離する。初期版では、少なくとも次を想定する。

- Cohere系API
- Jina AI系API
- Custom HTTP（Cohere互換形式を基本とする）
- ローカルモデル（後続対応可能な抽象化を用意する）

### REQ-010：検索処理の方式を明示する

検索画面では、次の方式をユーザーが確認または選択できること。

- Vector only
- Full-text only
- Hybrid
- Hybrid + Reranker

検索結果には、実際に利用した方式と、Rerankerの成否を表示する。

## 6. GUI要件

### 6.1 資料追加画面

Embedding方法として「通常処理」「Batch処理」「自動」を表示する。各選択肢に用途を
1行で説明し、初期値は設定可能とする。

### 6.2 資料一覧

資料ごとに次を表示する。

- `[通常]`、`[Batch]`、`[自動]`のモードバッジ
- Embedding状態
- `完了数 / 総チャンク数`
- 失敗数
- 再試行・キャンセルなどの操作

### 6.3 Batch管理画面

ジョブ全体について、処理中、待機中、完了、失敗の件数を表示する。ジョブ単位で
一時停止、再開、キャンセルができること。

### 6.4 検索画面

検索方式、Rerankerの利用状況、検索可能なチャンク数、Embedding処理中の件数を表示する。
Rerankerがタイムアウトまたは失敗した場合は、その事実を結果画面に表示する。

### 6.5 Reranker設定画面

設定済みモデルをカードまたは一覧で表示する。各設定について、API Keyはマスク表示し、
接続状態、最終テスト日時、Provider、Model名を確認できること。

## 7. データ/API要件

### 7.1 Sourceへの追加項目

```text
embedding_status
embedding_mode
embedding_progress
embedding_total_chunks
embedding_completed_chunks
embedding_failed_chunks
embedding_model
embedding_version
last_embedding_error
```

### 7.2 EmbeddingJob

ジョブは、対象資料、処理モード、優先度、状態、進捗、APIバッチサイズ、並列数、
開始時刻、完了時刻、エラーを保持する。

### 7.3 RerankerConfig

```text
id
name
provider
model
base_url
credential_id
enabled
timeout_seconds
candidate_top_k
final_top_n
fallback_enabled
```

### 7.4 APIの初期案

```text
GET    /api/embedding-jobs
GET    /api/embedding-jobs/{id}
POST   /api/embedding-jobs/{id}/pause
POST   /api/embedding-jobs/{id}/resume
POST   /api/embedding-jobs/{id}/cancel
POST   /api/embedding-jobs/{id}/retry-failed

GET    /api/settings/rerankers
POST   /api/settings/rerankers
PATCH  /api/settings/rerankers/{id}
DELETE /api/settings/rerankers/{id}
POST   /api/settings/rerankers/{id}/test
POST   /api/settings/rerankers/discover-models
```

既存APIとの互換性を壊さず、オプション項目として追加する。

## 8. 非機能要件

- Embeddingジョブは再起動後も状態を失わず、再開または失敗として復旧できる。
- 一時的なAPIエラーは指数バックオフで再試行する。
- API Key、Authorizationヘッダー、外部APIのレスポンスに含まれる秘密情報をログへ出さない。
- Rerankerのタイムアウトで検索全体を失敗させない。
- 同じモデル設定と同じチャンクに対する不要な再Embeddingを避ける。
- 既存の検索、Ask、Notebookスコープ指定の動作を維持する。
- 主要な状態遷移、検索統合、Rerankerフォールバックを自動テストする。

## 9. 受け入れ基準

### Batch処理

1. 100件以上の資料をBatchとして投入できる。
2. UIに待機中、処理中、完了、失敗の状態が表示される。
3. 途中停止後に再開できる。
4. 失敗チャンクのみを再試行できる。
5. 処理中でも完了済みチャンクは検索できる。

### 検索

1. Vector only、Full-text only、Hybridを切り替えられる。
2. Embedding未完了時に全文検索へフォールバックできる。
3. Notebookスコープ指定が検索方式変更後も維持される。

### Reranker

1. API KeyをGUIから登録できる。
2. API Keyが平文で画面やAPIレスポンスに表示されない。
3. 接続テストの結果とレイテンシを確認できる。
4. Reranker有効時に候補順位が再評価される。
5. タイムアウト・認証エラー時にHybrid結果へフォールバックする。

## 10. 段階的な実装計画

### Phase 0：基盤と契約

要件、状態モデル、API契約、ADR、テスト方針を固定する。

### Phase 1：Embedding状態の可視化

Sourceの状態、進捗API、資料一覧のバッジを実装する。

### Phase 2：EmbeddingJobとBatch処理

ジョブキュー、優先度、一時停止、再開、キャンセル、失敗再試行を実装する。

### Phase 3：EmbeddingモードGUI

資料追加時の通常・Batch・自動選択と、Batch管理画面を実装する。

### Phase 4：Hybrid Search

全文検索とベクトル検索の統合、未Embedding時のフォールバックを実装する。

### Phase 5：Reranker設定と接続テスト

RerankerConfig、Credential連携、Providerアダプター、設定GUI、接続テストを実装する。

### Phase 6：Reranker検索統合と評価

候補取得、再順位付け、タイムアウト、フォールバック、品質・性能評価を実装する。

## 11. 実装Issue一覧

Issueは本書の要件IDを参照し、依存関係の順に着手する。

1. [Enhanced基盤：要件・状態遷移・API契約・ADR](https://github.com/AiraCometes/open-notebook/issues/1)
2. [Embedding状態モデルと進捗API](https://github.com/AiraCometes/open-notebook/issues/3)
3. [EmbeddingJobキューとBatchワーカー](https://github.com/AiraCometes/open-notebook/issues/9)
4. [Batchジョブの一時停止・再開・キャンセル・再試行](https://github.com/AiraCometes/open-notebook/issues/7)
5. [Embeddingモード選択GUIと資料一覧バッジ](https://github.com/AiraCometes/open-notebook/issues/8)
6. [Batch管理画面と検索可能状態の表示](https://github.com/AiraCometes/open-notebook/issues/10)
7. [Hybrid Searchと未Embedding時フォールバック](https://github.com/AiraCometes/open-notebook/issues/5)
8. [Reranker抽象化とProviderアダプター](https://github.com/AiraCometes/open-notebook/issues/6)
9. [Reranker API設定GUIとCredential連携](https://github.com/AiraCometes/open-notebook/issues/11)
10. [Reranker接続テストとモデル検出](https://github.com/AiraCometes/open-notebook/issues/2)
11. [Reranker検索統合・タイムアウト・フォールバック](https://github.com/AiraCometes/open-notebook/issues/12)
12. [検索品質・性能評価とリリースドキュメント](https://github.com/AiraCometes/open-notebook/issues/4)

## 12. 参考資料

- [Open Notebook](https://github.com/lfnovo/open-notebook)
- [Embeddingコマンド](https://github.com/lfnovo/open-notebook/blob/main/commands/embedding_commands.py)
- [検索ユーザーガイド](https://github.com/lfnovo/open-notebook/blob/main/docs/3-USER-GUIDE/search.md)
- [大容量Embeddingの課題と内部バッチ対応](https://github.com/lfnovo/open-notebook/issues/536)
- [Hybrid Searchの設計議論](https://github.com/lfnovo/open-notebook/issues/1036)
- [Reranker対応の設計議論](https://github.com/lfnovo/open-notebook/issues/1087)
