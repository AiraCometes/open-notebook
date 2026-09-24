# T01 Enhanced共通契約とADR

- Issue: [#1](https://github.com/AiraCometes/open-notebook/issues/1)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 先行タスク: なし
- 後続タスク: T02〜T11

## 目的

各機能の実装が食い違わないように、状態・データ・API・互換性の共通契約を定める。

## 作業範囲

- SourceのEmbedding状態とEmbeddingJobの状態遷移を定義する。
- 通常・Batch・自動の意味と、内部APIバッチサイズとの境界を定義する。
- RerankerConfig、Credential参照、検索方式のAPI契約を定義する。
- 永続化、マイグレーション、後方互換性の方針をADRに記録する。
- 各後続Issueが参照する共有フィールド・エラー形式を決める。

## 受け入れ条件

- 状態遷移図に通常完了、部分失敗、再試行、キャンセルを含む。
- SourceとJobの責務、および両者の進捗集計方法が明確である。
- API契約に必須・任意項目、入力制約、代表的エラー応答が記載されている。
- 既存データの移行と既存API呼び出しの互換性方針が記録されている。
- 重要な構造判断がADRとして追加され、関連タスクから参照できる。

## 完了条件

設計文書とADRがコミットされ、T02以降が追加の前提確認なしに着手できる。
