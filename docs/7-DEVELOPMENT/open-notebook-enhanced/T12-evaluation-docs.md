# T12 品質・性能評価と運用文書

- Issue: [#4](https://github.com/AiraCometes/open-notebook/issues/4)
- 親仕様: [要件定義マスター](../open-notebook-enhanced-requirements.md)
- 依存: T02〜T11の対象機能が利用可能であること

## 目的

検索方式とEmbedding運用の効果を再現可能な方法で比較し、導入・運用に必要な情報をまとめる。

## 作業範囲

- Query、期待資料、期待チャンクを含む小規模な評価セットを用意する。
- Vector only、Full-text only、Hybrid、Hybrid + Rerankerを比較する。
- Recall@5、Recall@10、MRRまたはnDCGのうち適切な指標を算出する。
- Embedding処理量/時間、検索レイテンシ、Rerankerレイテンシ・失敗率を計測する。
- 初期値、APIコストやローカル負荷の注意、トラブルシュート、設定・移行手順を文書化する。

## 受け入れ条件

- 評価セットと実行方法がリポジトリに保存され、再実行できる。
- 各検索方式を同じQueryと候補データで比較できる。
- 品質指標と処理性能を別々に確認できる。
- 数値の限界、評価セットの偏り、Provider依存性を文書に明記する。
- セットアップ、Batch運用、Reranker設定、障害時対応が説明されている。

## 完了条件

評価手順と運用文書が整い、リリース判断に使える結果を再現できる。
