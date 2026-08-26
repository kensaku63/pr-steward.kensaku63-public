---
name: review-issue-docs
description: レビュー課題を課題 1 件につき 1 つの aachat shared document として固定し、frontmatter と必須本文を満たした形で管理する skill。Use when a reviewer creates a review issue, when the parent agent updates issue status, or when deduplicating review issues.
metadata:
  aachat.headline.ja: "レビュー課題ドキュメント"
  aachat.headline.en: "Review Issue Documents"
  aachat.description.ja: "検証済みレビュー課題を、一件ずつ追跡可能な共有ドキュメントとして記録・更新します。"
  aachat.description.en: "Records and updates validated review findings as individually traceable shared documents."
  aachat.discovery.listed: "true"
---

# Review Issue Docs

レビュー課題を aachat shared document として固定するときの規約。schema の正本は `$AA_AGENT_DIR/knowledge/aachat-review-doc-schema.md`。

## 1. 作成ルール

- 課題 1 件につき 1 doc を作る。複数課題を 1 doc にまとめない。
- 長文成果物は message に書かず、doc に固定する。message には要約と doc link だけを書く。

path:

```text
aachat/projects/<team>/<project>/docs/review-issues/<pr-number>-<reviewer>-<slug>.md
```

doc link:

```text
[[aachat/projects/<team>/<project>/docs/review-issues/<pr-number>-<reviewer>-<slug>.md]]
```

## 2. frontmatter

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
- `severity` / `confidence` は `low` / `medium` / `high`。
- `priority` は `p0` / `p1` / `p2` / `p3` / `defer`（基準は `$AA_AGENT_DIR/knowledge/review-priority-rubric.md`）。
- `merge_blocker` はマージを止めるべき課題のときだけ `true`。
- `sources` には元 candidate ID、kind、author、URL / comment ID を残す。Cursor / Codex / 人間 / 並列 reviewer の複数指摘を統合した場合も、全 source を保持する。

## 3. 必須本文セクション

- 課題の核心
- 観測した事実
- 起きる条件・分からない条件
- 直す価値
- 十分な解決状態
- 既存の責務・正本・不変条件
- 解法仮説（任意・非 authoritative）
- planning で再検討すべき architecture risk
- 親エージェント向け判断メモ

ルール:

- 「何が問題か」だけで終えない。「なぜ今直すべきか」「どの状態になれば十分か」まで書く。個別 issue doc では最適解を確定しない。
- 解法仮説を書く場合は事実・十分な解決状態と分離し、fresh planning が採用し直すまで approved と呼ばない。
- 書けない項目を無理に埋めない。確認できていない内容は `未確認` と明記する。
- 観測した事実には、PR 差分、既存仕様、テスト失敗、実行ログ、再現手順のどれに基づくかを書く。
- GitHub 上のレビュー本文は、それ自体を事実とみなさない。指摘内容をコード・仕様・実行結果で再検証し、source URL と検証結果を分けて書く。

## 4. intake から issue doc へ昇格する条件

- GitHub PR の既存レビューと並列 reviewer の出力は、最初に audit record の intake ledger へ `candidate` として集める。
- 親エージェントが証拠と影響を確認した有効な unique cluster だけを review issue doc に昇格する。
- 証拠不足、スコープ外、既に解消済み、outdated、非課題と判断した candidate は、doc を量産せず ledger に disposition と理由を残す。
- duplicate は元 candidate を消さず、昇格先 doc の `sources` または ledger の canonical cluster を参照させる。

## 5. status の遷移

| status | 意味 |
| --- | --- |
| `proposed` | 親エージェントが有効な unique cluster と確認し、issue doc へ昇格した直後 |
| `accepted` | integrated planning が統合 plan に採用 |
| `rejected` | 証拠不足、スコープ外、過剰として却下 |
| `deferred` | follow-up または人間判断へ回す |
| `superseded` | 統合 plan の別 design change で独立修正が不要 |
| `duplicate` | 他課題と統合。`duplicate_of` に元 doc を書く |
| `fixed` | 実装セッションが修正し、検証済み |

- review 親エージェントは evidence-validated な unique issue を `proposed` のまま planning へ渡す。実装採用の `accepted` へ変更しない。
- integrated planning Session は全 issue の横断設計後に `accepted` / `rejected` / `deferred` / `superseded` を確定し、判断メモへ統合 plan との対応を追記する。
- 実装セッションは修正ごとに該当 doc を更新し、修正内容と検証結果を追記する。

## 6. 重複排除

- 同じ根本原因、同じ修正対象、同じ失敗条件、同じ影響、片方の解決でもう片方も解消するものは 1 課題に統合する。
- 重複 doc は `status: duplicate` と `duplicate_of` で元 doc を参照し、本文は削除しない。
