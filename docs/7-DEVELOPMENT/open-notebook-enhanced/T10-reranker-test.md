# T10 Reranker接続テストとモデル検出

- Issue: [#2](https://github.com/AiraCometes/open-notebook/issues/2)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 依存: [T08](T08-reranker-providers.md)、[T09](T09-reranker-settings-ui.md)
- 後続タスク: T11

## 目的

保存した設定が実際に利用可能かを、検索に使う前に画面から確認できるようにする。

## 作業範囲

- 保存済み設定を使う接続テストAPIとGUI操作を追加する。
- 短い固定Queryと複数のサンプル文書で再順位付けを実行する。
- 認証、接続、タイムアウト、Provider応答形式、順位結果を検証する。
- 成否、エラー分類、レイテンシ、返却候補数、最終確認日時を保存・表示する。
- Providerが対応する場合にモデル検出APIと選択UIを追加する。

## 受け入れ条件

- 接続テストが検索パイプラインを変更せず単独実行できる。
- 認証エラー、ネットワーク障害、タイムアウト、不正応答を区別できる。
- テスト結果にAPI Keyや認証ヘッダーを含めない。
- テスト失敗でも保存済み設定と既存状態を破損しない。
- モデル検出非対応Providerでも手入力設定が可能である。

## 完了条件

接続テストAPI、Providerテスト、GUI結果表示が揃っている。
