# T02 Embedding状態モデルと進捗API

- Issue: [#3](https://github.com/AiraCometes/open-notebook/issues/3)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 詳細設計: [T01](T01-foundation.md)
- 後続タスク: T03、T05、T06

## 目的

Embeddingの進み具合を資料単位で追跡し、画面やジョブ処理が同じ状態を参照できるようにする。

## 作業範囲

- Sourceにモード、状態、総数、完了数、失敗数、モデル識別情報、最終エラーを持たせる。
- `not_started`、`queued`、`preparing`、`processing`、`completed`、`partial`、
  `retry_waiting`、`failed`、`cancelled`の遷移を実装する。
- 資料ごとの状態取得APIを追加し、一覧APIにも必要な集計値を含める。
- 既存資料に対する初期値と移行時の扱いを定義する。

## 受け入れ条件

- 状態とカウンターの組み合わせが不整合にならない。
- `completed_chunks / total_chunks`と失敗数をAPIから取得できる。
- 既存レコードが読み取り可能で、既存Source APIを壊さない。
- 不正な状態遷移を防ぎ、ログには秘密情報を含めない。

## 完了条件

モデル、マイグレーション、APIスキーマ、データ移行テストが揃っている。
