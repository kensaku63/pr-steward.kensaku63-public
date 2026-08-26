# レビュー優先度基準

親エージェントがサブエージェントの指摘を精査し、優先度を付けるときの正本。

## 精査チェックリスト

親エージェントはサブエージェントの出力を直接採用してはならない。各指摘について次を確認する。

- 証拠性: PR 差分、既存仕様、テスト失敗、実行ログ、再現手順のいずれかに基づくか。
- merge-blocker 性: 本番障害、データ破壊、セキュリティ問題、重大な UX 劣化、CI 失敗につながるか。
- スコープ適合性: PR 目的に対する修正であり、unrelated refactor ではないか。
- 複雑さ: 解決方針が過度に大きくないか。最小修正で目的を満たせるか。
- repo 方針との整合: 既存設計、命名、テスト方針、運用方針に沿っているか。
- 実装可能性: 変更箇所、期待挙動、検証方法が十分具体的か。
- UX 妥当性: 対象ユーザー、利用シナリオ、問題が起きる状態、最小の改善案が示されているか。好みや大規模再設計になっていないか。
- テスト必要性: 守るべき振る舞い、壊れやすい変更点、既存テストで検知できない理由、最小のテスト形態が示されているか。
- 副作用リスク: shared code、public API、DB schema、認可、非同期処理、既存互換性に触れるか。
- 人間判断の要否: 仕様、UX、API、スコープ判断が必要か。

GitHub PR 上の Cursor / Codex / 人間 reviewer の指摘にも、同じ基準を適用する。author や review tool の評判を根拠に採否を決めない。

## Review evidence gate

レビュー指摘の事実性と、その指摘で提案された解法の妥当性を分離する。review 親 Session が確定するのは症状、証拠、影響、scope、priority、十分な解決状態までである。個別の解法は approved plan にしない。

evidence-validated な `proposed` issue にする前に、cluster ごとに次を明記する。

1. 守るユーザー価値と十分な解決状態。
2. 問題を生んだ責務・不変条件。既存の正本へ戻す、変更を消す、元の単純な設計を保つ方法がないか。
3. reviewer が示した解法仮説と、その architecture delta。仮説は比較材料であり、採用判断ではない。
4. fresh planning が必要になる signal。新しい永続状態、正本、状態遷移、公開 API、caller が守る手順、変更経路が含まれるか。

次のいずれかに該当したら、review 親 Session は個別 guard を解法として採用せず、`integrated-fix-planning` の fresh planning 必須 signal として記録する。

- 下書き控えのために送信 lifecycle を持つ、表示補助のために第二の正本を持つなど、補助機構が本体の責務を引き受ける。
- 複数 caller が `begin / accept / reject` のような内部 protocol を知る必要が生じる。
- ある修正が作った race や不整合を、別の nonce、CAS、fence、例外分岐で塞ぐ必要が生じる。
- テストがユーザー契約ではなく、特定の関数呼び出し順や実装文字列を固定し始める。

複雑な解法仮説を退けても、元の candidate は失わない。ledger には「指摘は有効、解法仮説は非 authoritative」「fresh planning で再設計」のように、症状と解法を分けて残す。

## 認知負荷を超えない精査手順

親エージェントは全指摘を一度の prompt / 判断で処理しない。

1. まず全 candidate を lossless に intake ledger へ固定し、source と stable ID を付ける。
2. 同じ根本原因・失敗条件・修正対象を持つ candidate を cluster 化する。cluster 化は整理であり、採否判断ではない。
3. 原則最大 5 cluster の batch に分ける。P0候補、security、認証、データ破壊は 1 cluster ずつ扱う。
4. 各 cluster について、必要な diff・仕様・テスト・実行証拠だけを読み直し、この checklist と evidence gate で disposition を決める。
5. batch ごとに ledger と issue doc を更新して判断を外部化し、未処理件数を再計算してから次へ進む。
6. 全 batch 後に横断重複と優先度の整合を確認し、review package を凍結する。この Session では全体修正 plan を作らない。
7. `$AA_AGENT_DIR/.agents/skills/integrated-fix-planning/SKILL.md` の条件で fresh planning の要否を判定する。

完了時には、次の件数が保存則を満たすことを確認する。

```text
collected candidates
= validated + rejected + deferred + duplicate + superseded
```

`validated` は事実として有効で `status: proposed` の issue doc に昇格した candidate であり、実装採用を意味しない。`untriaged` が 1 件でも残る場合、planning に進まない。context 不足や時間不足で精査品質を維持できない場合は、未処理範囲を明示して次 session へ handoff し、完了扱いにしない。

## 重複排除

- 同じ根本原因、同じ修正対象、同じ失敗条件、同じ影響、片方の解決でもう片方も解消するものは 1 課題に統合する。
- 重複 doc は `status: duplicate` と `duplicate_of` で元 doc を参照する。

## 優先度

- `P0`: merge blocker。データ破壊、重大セキュリティ、主要機能停止、復旧困難。
- `P1`: マージ前修正必須。明確なバグ、高影響の回帰、仕様未達。
- `P2`: PR スコープ内なら修正。限定的不具合、必要性が説明できる安全網不足、運用上の注意。
- `P3`: follow-up 可。軽微な改善、低リスクの保守性課題。
- `defer`: 今回は扱わない。人間判断または別 PR が適切。

## Planning への境界

- review priority は問題の重要度であり、実装順序ではない。
- `validated` issue を全件実装する前提にしない。planning Session は横断設計の結果として `accepted` / `rejected` / `deferred` / `superseded` を再決定する。
- 実装順序、変更箇所、architecture delta、simplicity budget は `integrated-fix-planning` が確定する。

## テスト指摘の採用基準

- テスト追加は、カバレッジの数値改善ではなく、変更で壊れたら困る振る舞いを固定するために提案する。
- 既存テストで同じリスクを検知できる場合、新規テストは提案しない。
- テストが必要な場合は、unit / integration / characterization / golden / contract / e2e のどれが最小で十分かを示す。
- リファクタリングや構造変更に安全網がない場合は、まず現在の振る舞いを固定する characterization test を提案する。
- テストが private detail を固定し、将来の安全な変更を妨げる場合は、追加ではなく外部契約を固定する形への書き換えを提案する。

### Online migration の evidence boundary

- release risk が staged expand、validation、cleanup の transition にある場合、HEAD 適用後に fixture を作る final-schema test だけでは十分としない。migration 直前の schema に既存の正当な row を置き、実際の timestamp migration 列を境界ごとに通す upgrade test を優先する。
- deploy planner、retirement commit の ancestry、migration header は release 順の前提を証明するが、DB 内の transition behavior の証拠ではない。両者を代替関係にせず、planner gate と DB integration test の責務を分ける。
- `NOT VALID` constraint では、追加直後の `convalidated = false` と新規不正 write の拒否を同時に確認する。validation 後の既存 row 保持、legacy protection の存続、cleanup migration でだけ旧 constraint が消えることまで DB catalog と DML で観測する。
- migration SQL の文字列や private helper の呼び出し順を正本にしない。既存 integration test が同じ schema family を所有するなら、その test を upgrade contract へ直し、第二 fixture、SQL parser、test-only state model を増やさない。

## Rolling compatibility と snapshot freshness

- client / server、CLI / API のように別々に配備される契約変更では、release順を証拠として確認し、new client → old server と old client → new server の両方向をcontract testで固定する。片方向だけの互換性はrolling releaseの安全性を証明しない。
- legacy clientの不完全snapshotではdeactivateを省略してよいが、source watermarkや世代判定を迂回してはならない。stale snapshotは既存rowの復活だけでなく、削除済みの未登録rowのinsertもno-opでなければならない。
- payload budgetのため本文を省略する場合、hashやdigestが変わったrowに旧cacheを残さない。metadataだけでbudgetを超える場合も主要処理を停止させず、無効なpartial reportを送らない挙動を境界testで固定する。
