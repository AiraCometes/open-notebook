# T09 Reranker API設定GUI

- Issue: [#11](https://github.com/AiraCometes/open-notebook/issues/11)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- API/Provider依存: [T01](T01-foundation.md)、[T08](T08-reranker-providers.md)
- 後続タスク: T10、T11

## 目的

管理者がReranker Provider、モデル、認証情報を設定画面から登録し、検索利用の既定値を管理できるようにする。

## 作業範囲

- Provider、表示名、Model、Base URL、API Keyを作成・編集・削除できるフォームを追加する。
- Candidate Top K、Final Top N、Timeout、失敗時Fallbackを設定できるようにする。
- Credential機構へ秘密情報を保存し、画面では設定済み状態と末尾などの限定表示を使う。
- ProviderとModel、接続状態、最終テスト日時を一覧表示する。
- enabled設定と既定Rerankerの選択を管理する。

## 受け入れ条件

- 設定を追加、編集、有効化、無効化、削除できる。
- API Keyは作成時以外に平文で再表示されず、APIレスポンスにも含まれない。
- Secret未指定の更新で保存済みCredentialを誤って消さない。
- URLと数値設定の入力検証が行われる。
- 設定画面で未設定、未テスト、成功、失敗、無効状態が分かる。

## 完了条件

設定CRUD APIとGUIがつながり、秘密情報のマスキングを確認できる。
