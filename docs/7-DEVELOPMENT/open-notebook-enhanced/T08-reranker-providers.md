# T08 Reranker境界とProvider

- Issue: [#6](https://github.com/AiraCometes/open-notebook/issues/6)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 依存: [T01](T01-foundation.md)
- 後続タスク: T09、T10、T11

## 目的

検索ロジックをProvider固有の通信方式から切り離し、設定・接続テスト・実検索で共通に使う。

## 作業範囲

- Query、文書候補、top_nを受け取り、順位とスコアを返すReranker契約を定義する。
- Cohere形式、Jina形式、Custom HTTPのProviderアダプターを追加する。
- Base URL、モデル名、認証情報を共通設定からProviderへ渡す。
- Provider応答の形式・順位範囲・候補IDを検証する。
- ローカルモデルを将来追加できる登録方式を用意する。

## 受け入れ条件

- Providerを切り替えても検索側の呼び出し契約が変わらない。
- 不正応答や未知の候補IDを安全に扱える。
- API Keyと認証ヘッダーが例外・ログに漏れない。
- Providerごとの必須設定と対応形式が文書化される。
- API接続でテストできるProvider境界がある。

## 完了条件

Provider契約とアダプターの単体テストが揃っている。
