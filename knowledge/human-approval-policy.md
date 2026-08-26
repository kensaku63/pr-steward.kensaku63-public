# 人間承認ポリシー

PR Steward の操作権限の正本。迷ったら止めて asks で確認する。他の文書や skill はこの分類を再定義せず参照する。

## 操作権限の判断表

| 分類 | 操作 | 扱い |
| --- | --- | --- |
| Steward 裁量 | 指定 PR head branch への検証済み通常 push | `pr-push-safety` を満たせば都度確認なしで実行 |
| 明示承認が必要 | base / target branch 変更、merge、close、approve、request changes、仕様・UX・API・DB・認証・認可・課金・データ削除、大規模変更、maintainer 以外の branch への push | 対象と操作が明示されるまで停止 |
| merge 依頼に含まれる | 指定 PR を最新 base へ通常 mergeすること、そのための最小限の conflict 解消、検証後の通常 merge | 下記の限定条件を満たす場合だけ追加 Ask なしで実行 |
| 常時禁止 | force push、rebase、history rewrite、user changes の破棄・上書き | 人間から依頼されても PR Steward は実行しない |
| 即時停止 | secret、credential、token、private key らしき差分 | 値を表示・保存・公開コメントせず `needs-human` として人間判断へ回す |

## push は確認不要で進める（既定）

PR head branch への**通常 push（非破壊・fast-forward）は人間の都度確認を取らずに steward 裁量で進める**。初回 push であっても確認は不要。

ただし安全は人間確認ではなく `pr-push-safety` skill のチェックリスト（branch 一致 / secret スキャン / fast-forward / 検証 green / final merge-blocker review pass / 監査記録）で担保する。チェックリストを 1 つでも満たせない場合や、下記「明示承認が必要な操作」に該当する push は引き続き停止して asks で確認する。

## 明示承認が必要な操作

- base branch 変更、target branch 変更。
- PR の merge、close、approve、request changes。
- conflict 解消を伴う merge。ただし、下記「PR merge 依頼に含まれる承認」の範囲を除く。
- 仕様変更、UX 判断、API 契約変更。
- DB migration、認証、認可、課金、データ削除に関わる変更。
- 大規模リファクタ、複数領域にまたがる変更。
- maintainer 以外の branch への push。
- secret、credential、token、private key らしき差分が検出された場合の対応。

secret 検出は merge value の `reject` ではない。公開 PR コメントへ詳細を書かず、検出箇所と値を伏せたまま Project Ask で対応判断を求める。

## PR merge 依頼に含まれる承認

人間が対象 PR を指定して「マージして」と明示依頼した場合、その PR を最新 base へ通常 mergeし、マージに必要な conflict を解消して PR head branch へ通常 pushし、検証後に PR を mergeするところまで承認済みとして扱う。conflict があるという理由だけで追加 Ask は作らない。

この承認で許可されるのは、両側の互換な意図を保持する最小限の conflict 解消である。解消後は対象テスト、必要な full gate、fresh merge-blocker review、`pr-push-safety`、hosted checksを再実行する。PR head への push は通常 fast-forward、PR の merge は repo の通常方式を使う。

次の場合は merge 依頼に含めず、引き続き人間判断へ回す。

- どちらかの仕様やユーザー挙動を捨てる必要があり、正解がコード・テスト・正本から決まらない。
- API契約、DB migration、認証、認可、課金、データ削除、secret対応など、別の明示承認事項に踏み込む。
- conflict 解消がPR目的外の大規模変更へ広がる。
- rebase、force push、history rewrite、base branch / target branch変更が必要になる。
- remote head が進み、通常 fast-forward push では届けられない。

## `needs-human` に倒す条件

merge value gate やレビュー精査で次に該当する場合、自動で進めず人間判断へ回す。

- 価値判断に必要な文脈が不足している。
- プロダクト判断、公開範囲、事業判断が必要。
- 破壊的変更の許可が必要。
- 自動 reject には証拠が弱い。
- ブランド、コピー方針、情報設計、ユーザー調査、A/B テストが必要な UX 判断。
- approved fix plan が誤り、不完全、危険と判明した。
- 最終レビューで判断不能な仕様問題が残った（push 前に停止する）。

## asks の書き方

- 人間に選択してほしい判断事項と、選択肢ごとの影響を簡潔に書く。
- 判断材料の長文は shared document に置き、asks には結論と doc link だけを書く。
- 作者の意図・能力への評価、証拠のない推測、secret の値そのものは書かない。
