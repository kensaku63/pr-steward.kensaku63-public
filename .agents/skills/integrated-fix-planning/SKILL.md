---
name: integrated-fix-planning
description: 検証済みの複数 PR review issue を fresh context で横断し、共通 root cause、既存 authority、不変条件から一つの最小な approved fix plan を作る skill。Use after review intake and evidence triage are complete, before implementation handoff, especially when findings span multiple responsibilities, four or more unique clusters, or any state, source-of-truth, wire/API, caller-protocol, auth, migration, or UX-contract change.
metadata:
  aachat.headline.ja: "統合修正計画"
  aachat.headline.en: "Integrated Fix Planning"
  aachat.description.ja: "検証済みの複数レビュー課題を、根本原因と既存責務から一つの最小な修正設計へ統合します。"
  aachat.description.en: "Synthesizes validated review findings into one minimal fix design based on root causes and existing ownership."
  aachat.discovery.listed: "true"
---

# Integrated Fix Planning

PR Steward workflow の Step 7。レビューで確定したのは問題の事実性と十分な解決状態であり、個別 reviewer の解法ではない。全 issue を一度に見渡し、PR 全体として最小の一つの設計へ再構成する。

## 1. Fresh Session が必須か判定する

次のいずれかなら、review 親 Session は plan を確定せず、同じ PR Steward agent の fresh planning Session を起動する。

- `proposed` の unique review issue が 4 件以上ある。
- issue が 2 つ以上の production responsibility、module、deployable、または user journey にまたがる。
- 永続 state、source of truth、state machine、lifecycle、public API、wire contract、caller protocol、認証・認可、DB migration、または仕様・重要 UX の変更候補がある。
- 一つの修正案が別の race、不整合、guard、nonce、CAS、fence、fallback を要求している。
- review 親 Session が context 不足、注意力低下、または局所解への anchor を疑う。

上記に該当せず、issue が最大 3 件、単一責務、architecture delta `none` の場合だけ、review 親 Session がこの skill の Step 3 以降を同じ Session で実行してよい。fast path でも issue ごとの案を連結するだけで plan を作らない。

## 2. Review package を凍結して fresh Session を起動する

review 親 Session は次を audit record に固定する。

- PR URL / number、base / head branch、reviewed HEAD。
- PR が守る最小のユーザー価値と十分な解決状態。
- review surface の完了状況と、未確認 surface。
- candidate conservation counts。新規 audit は `validated` を使う。
- `status: proposed` の evidence-validated review issue links。
- fresh planning が必要になった trigger。

reviewer transcript、candidate ごとの長い推論、個別の proposed remedy を planning prompt に複製しない。planner には audit record と issue doc の WikiLink だけを渡す。

```bash
chat session run --agent <agent> --project <project> --stdin <<'EOF'
PR Steward Step 7 の統合修正設計 Session です。
<audit-record-wikilink> と proposed review issue docs を読み、`integrated-fix-planning` に従ってください。
reviewer の解法を要件化せず、PR 全体の root cause、authority、不変条件から一つの最小な plan を作ってください。
plan が確定した場合だけ `implementation-handoff` で fresh 実装 Session を起動してください。人間判断が必要なら実装を起動せず報告してください。
EOF
```

起動後、review 親 Session は code、issue disposition、fix plan を変更しない。planning Session の ownership を尊重して終了する。

## 3. Evidence boundary を再確認する

- GitHub `headRefOid` が reviewed HEAD と一致することを確認する。不一致なら計画せず review の更新を求める。
- PR の仕様、diff、既存 authority / invariant、`proposed` issue の症状・証拠・十分な解決状態だけから判断を再構築する。
- issue doc に solution hypothesis が残る旧 record では、それを非 authoritative と扱う。先に root cause map を作り、その後で比較材料としてだけ読む。
- issue が事実でない、scope 外、すでに解消済みと判明した場合は `rejected` / `deferred` / `superseded` へ変更してよい。review 親の proposed 集合を全採用する義務はない。

## 4. 一つの設計へ統合する

次の順で考え、handoff に判断を残す。

1. 全 issue に共通する root cause と broken invariant を特定する。共通原因がない issue は責務ごとに明示的に分ける。
2. 変更後に authoritative である state、owner、transaction、projection、caller boundary を一枚の contract として説明する。
3. 少なくとも次を比較する。
   - PR の問題を生んだ変更を戻す・消す案。
   - 既存 authority / responsibility に処理を戻す案。
   - 新しい仕組みを足す案。
4. 最小の案で各 issue の十分な解決状態を満たせるか、具体的な挙動で比較する。
5. issue ごとの局所修正を並べず、同じ責務・変更経路に属する修正を一つの design change としてまとめる。
6. plan 後の全体が元 PR より説明しやすいか確認する。概念、state、branch、caller obligation が増えるなら、その増分が承認済み要件から不可避かを示す。

## 5. Planning disposition を確定する

各 `proposed` issue を次へ更新する。

- `accepted`: 統合 plan の design change で解消する。
- `rejected`: 問題が成立しない、scope 外、または修正価値がない。
- `deferred`: 問題は有効だが今回の PR では扱わない、または人間判断が必要。
- `superseded`: 別の accepted design change で独立修正が不要になった。

audit record に planning conservation を残す。

```text
validated unique issues
= accepted + rejected + deferred + superseded
```

未判定が 1 件でもあれば approved fix plan を作らない。

## 6. Approved fix plan の完了条件

`implementation-handoff` の schema に加えて、次を必須にする。

- reviewed HEAD と planning Session ID。
- 最小のユーザー価値と十分な解決状態。
- root cause / broken invariant map。
- authoritative state / owner / caller boundary。
- 比較した「戻す・既存責務へ寄せる・追加する」案と採否理由。
- issue ではなく design change 単位の approved plan。
- 各 design change が解消する issue links。
- explicitly rejected / deferred / superseded issue。
- simplicity budget と非目標。
- 実装順序、変更箇所、期待挙動、検証方法。

新しい永続 state、第二の正本、state machine、public API、caller protocol、仕様・重要 UX、認証・認可、DB migration が必要なら、人間の architecture-delta-specific approval なしに plan を approved にしない。Ask を出して停止する。

plan が確定した場合だけ `implementation-handoff` を使って fresh 実装 Session を起動する。planning Session 自身は code を編集しない。
