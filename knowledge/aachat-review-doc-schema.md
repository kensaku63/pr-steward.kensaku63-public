# aachat レビュー doc schema

PR Steward が aachat shared document に残す成果物の path 規約と schema の正本。

## shared document の用途

- 作業ログ
- 調査メモ
- 判断材料
- レビュー課題
- 実装計画
- テストログ要約
- リスク一覧
- ユーザー承認が必要な asks
- 複数エージェント間の handoff

長文成果物は message ではなく shared document に固定する。shared document は必ず次の形式の path に作る。

```text
aachat/projects/<team>/<project>/docs/<kind>/<id>.md
```

## review issue doc

課題 1 件につき 1 doc を作る。

推奨 path:

```text
aachat/projects/<team>/<project>/docs/review-issues/<pr-number>-<reviewer>-<slug>.md
```

doc link:

```text
[[aachat/projects/<team>/<project>/docs/review-issues/<pr-number>-<reviewer>-<slug>.md]]
```

frontmatter:

```yaml
---
title: ""
summary: ""
status: proposed
pr_number: 0
reviewer_role: code_quality
severity: medium
priority: p2
confidence: medium
category: code-quality
merge_blocker: false
duplicate_of: null
related: []
sources:
  - candidate_id: ""
    kind: parallel_review
    author: ""
    url: ""
    github_id: null
---
```

- `reviewer_role` は `outcome_gap`、`ux_friction`、`code_quality`、`release_hardening` のいずれか。
- `status` は evidence-validated な issue を `proposed` で作成する。review 親 Session は `accepted` にせず、integrated planning が `accepted` / `rejected` / `deferred` / `superseded` に更新する。実装検証後は `fixed` にする。
- 重複 doc は `status: duplicate` と `duplicate_of` で元 doc を参照する。
- `sources` は、元 candidate ID、source kind、author、URL / GitHub ID を持つ。複数 reviewer の同一指摘を統合しても source を失わない。

必須本文セクション:

- 課題の核心
- 観測した事実
- 起きる条件・分からない条件
- 直す価値
- 十分な解決状態
- 既存の責務・正本・不変条件
- 解法仮説（任意・非 authoritative）
- planning で再検討すべき architecture risk
- 親エージェント向け判断メモ

書けない項目を無理に埋めない。確認できていない内容は `未確認` と明記する。

## audit record doc

各 PR セッションで 1 つ作り、次を記録する。

- PR URL / number
- base branch / head branch / checked out branch
- current HEAD before and after work
- merge value gate 判定
- 起動したサブエージェントと担当領域
- GitHub PR review intake の取得 surface、取得結果、未確認 surface
- Cursor / Codex を含む既存 review candidate の source metadata
- 全 candidate の intake ledger、cluster、disposition
- collected / validated / rejected / deferred / duplicate / superseded / untriaged の review 件数照合
- validated unique issue の planning disposition（accepted / rejected / deferred / superseded）と件数照合
- 作成された review issue docs
- review 親エージェントの evidence validation と、planning Session の採用 / 却下 / defer 判断
- approved fix plan
- 実装内容
- 実行した commands と結果要約
- test / lint / typecheck / build の結果
- GitHub に投稿したコメント
- push した commit
- 残リスク
- 人間判断が必要な項目

推奨 path:

```text
aachat/projects/<team>/<project>/docs/pr-steward-audit/<pr-number>-<session-id>.md
```

intake ledger の各 candidate は最低限次を持つ。

- candidate ID
- source kind、author、URL / GitHub ID
- path / line / review state / resolved / outdated（存在する場合）
- 指摘本文の短い要約
- canonical cluster ID
- disposition と理由

disposition:

- `untriaged`: 未精査
- `validated`: 有効な課題として `status: proposed` の issue doc へ昇格。実装採用ではない
- `rejected`: 証拠不足、スコープ外、非課題
- `deferred`: follow-up または人間判断へ移送
- `duplicate`: 同じ canonical cluster へ統合
- `superseded`: 後続 commit や別修正ですでに解消

新規 audit では review disposition に `validated` を使う。過去 audit の `accepted` は historical record として書き換えない。planning 後は別の planning ledger で、`validated = accepted + rejected + deferred + superseded` を照合する。

## handoff doc

実装セッションへの handoff は shared document に保存し、message には短い指示と doc link のみを書く。schema は `$AA_AGENT_DIR/.agents/skills/implementation-handoff/SKILL.md` を参照。
