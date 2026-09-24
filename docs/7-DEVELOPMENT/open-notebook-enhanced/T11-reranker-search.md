# T11 検索統合とRerankerフォールバック

- Issue: [#12](https://github.com/AiraCometes/open-notebook/issues/12)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 依存: [T07](T07-hybrid-search.md)、[T08](T08-reranker-providers.md)、[T09](T09-reranker-settings-ui.md)
- 後続タスク: T12

## 目的

Hybrid Searchの候補をRerankerで再評価し、失敗時も検索結果を返す。

## 作業範囲

- `Hybrid + Reranker`検索経路をAsk/Searchの対象に追加する。
- Candidate Top Kを取得してRerankerへ渡し、Final Top Nを返す。
- 資料ID、チャンク内容、スコアなどの対応を結果まで保持する。
- Timeout、認証エラー、レート制限、Provider不調ではHybrid順位へフォールバックする。
- 結果に検索モード、Reranker実行状態、フォールバック理由を含める。

## 受け入れ条件

- 有効な設定がある場合に候補の順位を再評価する。
- Candidate Top KとFinal Top Nが設定どおり反映される。
- Reranker未設定時はHybrid検索だけで応答する。
- Reranker失敗時も検索全体は成功し、元の順位を返す。
- Notebookスコープと検索フィルターが再順位付け後も維持される。
- GUIがReranker成功、未使用、フォールバックを判別できる。

## 完了条件

成功・未設定・タイムアウト・Providerエラーの統合テストが揃っている。
