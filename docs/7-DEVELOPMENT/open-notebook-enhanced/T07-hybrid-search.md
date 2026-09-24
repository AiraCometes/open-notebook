# T07 Hybrid Search

- Issue: [#5](https://github.com/AiraCometes/open-notebook/issues/5)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 依存: [T01](T01-foundation.md)、[T02](T02-embedding-status-api.md)
- 後続タスク: T11、T12

## 目的

固有語に強い全文検索と意味の近さを拾うベクトル検索を統合し、Embedding未完了時も検索可能にする。

## 作業範囲

- Vector only、Full-text only、Hybridの検索モードを提供する。
- Hybridは両検索の順位をReciprocal Rank Fusion（RRF）で統合する。
- 片方の検索が使えない場合は、利用可能な検索結果を返し、縮退状態を返す。
- Embedding未生成・未完了の場合は全文検索結果を利用する。
- Notebookスコープと既存の検索制約をすべての検索経路に適用する。

## 受け入れ条件

- RRF統合が同一レコードを重複させず、順位を安定して返す。
- Vector/Full-text各単独モードが維持される。
- Embedding設定なし、部分Embedding、片方の検索障害でも検索を継続する。
- Notebookスコープがフォールバック時にも維持される。
- 結果には実際に利用した検索モードと縮退状態を含められる。

## 完了条件

順位統合、重複、縮退、スコープ維持を含む自動テストが揃っている。
