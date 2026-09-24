# T03 EmbeddingJobとBatchワーカー

- Issue: [#9](https://github.com/AiraCometes/open-notebook/issues/9)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 詳細設計: [T01](T01-foundation.md)、[T02](T02-embedding-status-api.md)
- 後続タスク: T04、T05、T06

## 目的

複数資料のEmbeddingを永続ジョブとして扱い、通常処理とBatch処理の優先度を管理する。

## 作業範囲

- EmbeddingJobに対象資料、モード、優先度、状態、進捗、APIバッチサイズ、並列数、時刻、エラーを保持する。
- 既存の非同期コマンド／ワーカーを利用してEmbeddingを実行する。
- 通常処理をBatch処理より優先できるキュー制御を用意する。
- チャンク単位の結果を永続化し、再起動後に二重処理を避ける。
- API内部の送信件数をジョブモードから独立して設定できる。

## 受け入れ条件

- ジョブ作成から処理完了・部分失敗まで状態が永続化される。
- Batch投入中でも通常処理を優先して実行できる。
- ワーカー再起動後にジョブを再取得し、安全に継続できる。
- 重複投入で同じチャンクを不用意に再Embeddingしない。
- APIレート制限等の一時障害を再試行可能な結果として記録する。

## 完了条件

キュー、ワーカー、永続化と重複防止の自動テストが揃っている。
