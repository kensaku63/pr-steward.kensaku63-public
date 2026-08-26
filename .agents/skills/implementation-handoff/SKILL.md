---
name: implementation-handoff
description: 承認済み実装計画を新しい実装セッションへ handoff し、実装制約と参照 doc を伝える skill。Use when the approved fix plan is ready, when starting a fresh implementation session, or when constraining what an implementation session may change.
metadata:
  aachat.headline.ja: "実装ハンドオフ"
  aachat.headline.en: "Implementation Handoff"
  aachat.description.ja: "承認済みの最小修正計画を、制約と検証条件付きで新しい実装セッションへ渡します。"
  aachat.description.en: "Hands an approved minimal fix plan to a fresh implementation session with constraints and verification."
  aachat.discovery.listed: "true"
---

# Implementation Handoff

PR Steward workflow の Step 8。`integrated-fix-planning` を完了した planning Session は、承認済み実装計画を新しい実装 Session へ渡す。review 親 Session はこの skill を直接起動しない。

## 1. handoff doc を作る

handoff は aachat shared document に保存し、message には短い指示と doc link のみを書く。

推奨 path:

```text
aachat/projects/<team>/<project>/docs/pr-steward-handoff/<pr-number>-fix-plan.md
```

必須項目:

- PR URL / PR number
- 作成された shared document links（review issue docs、audit record doc）
- planning Session ID と reviewed HEAD

加えて、次を含めることを推奨する:

- base branch / head branch / 作業時の HEAD
- この PR が守る最小のユーザー価値と、十分な解決状態
- root cause / broken invariant、authoritative state / owner / caller boundary
- 比較した削除案・既存責務へ戻す案・追加案と採否理由
- approved fix plan（issue の羅列ではなく design change、対応 issue、優先度、実装順序）
- rejected / deferred / superseded issue と非目標
- 修正ごとの変更箇所、期待挙動、検証方法
- simplicity budget（追加を認める永続状態、正本、状態遷移、公開 API、caller protocol。人間の明示承認がなければすべて `none`。一般的な plan 承認ではなく、人間が各 architecture delta を明記して承認した場合だけ承認済みとみなす）
- 実行すべき lint / typecheck / test / build コマンド
- 実装セッションへの制約（次節をそのまま記載する）

## 2. 実装セッションへの制約

handoff doc に次の制約を明記し、実装セッションに遵守させる。

- approved fix plan に含まれる修正だけを実装する。
- 指摘を解決するために必要な最小限の周辺変更は許可する。
- レビュー指摘を要件として再解釈しない。handoff に書かれたユーザー価値と十分な解決状態を、既存の正本・責務を保つ最小変更で満たす。
- simplicity budget にない永続状態、第二の正本、状態機械、公開 API、複数 caller の protocol を追加しない。
- repo の既存パターンに沿ったテスト追加・更新は許可する。
- PR 目的と無関係なリファクタは禁止する。
- 仕様、UX、API 契約を独断で変えない。
- secret、credential、`.env`、秘密鍵、token を追加・変更しない。
- 失敗テストを削除・弱体化して通さない。
- approved plan が誤り、不完全、危険、人間判断が必要と判明したら停止して報告する。
- 実装中の修正が新しい race や不整合を作り、nonce、CAS、fence、例外分岐など次の防御を必要としたら、その防御を足さず停止する。元の修正を削る・戻す簡素化案とともに再計画を求める。

## 3. 新しい実装セッションを起動する

handoff doc を作成したら、同じ PR Steward agent の fresh session を起動して実装を任せる。既存 running session へ追加指示せず、独立した実装作業として新規 session を使う。

これは session 内側の agent コマンドなので、outside 用の `aachat` CLI ではなく `chat` を使う。

```bash
chat session run --agent <agent> --project <project> --stdin <<'EOF'
Step 8 の実装セッションです。
[[aachat/projects/<team>/<project>/docs/pr-steward-handoff/<pr-number>-fix-plan.md]] を読んで、approved fix plan の範囲だけを実装してください。
実装後は `final-merge-blocker-review` をサブエージェントで実行し（新規 session を起動しない）、blocker なしなら `pr-push-safety` に従って push まで進めてください。
完了・停止・人間判断が必要な場合は project に報告してください。
EOF
```

message には長い計画を書かない。次の内容だけを含める。

- Step 8 の実装セッションであること。
- handoff doc への wiki link（例: `[[aachat/projects/<team>/<project>/docs/pr-steward-handoff/<pr-number>-fix-plan.md]]`）。
- approved fix plan の範囲だけを実装すること。
- 実装後は `final-merge-blocker-review` をサブエージェントで実行し、push（`pr-push-safety`）まで進めること。
- 完了・停止・人間判断が必要な場合は project に報告すること。

実装セッションを起動できたら、この planning Session は以後の実装を続けない。今回の handoff で得た再利用可能な学びがある場合は `$AA_AGENT_DIR/memory/` に保存し、保存後に現在の session を終了する。

```bash
chat session finish
```

## 4. 実装セッションの必須行動（Step 8）

実装セッションは、approved fix plan を順番に実装する。

- 作業前に current branch と working tree を確認する。
- 変更は最小限に保つ。
- 作業開始時と各 blocker 修正前に architecture delta を simplicity budget と照合する。budget を超える場合は実装を続けない。
- 既存パターン、既存 helper、repo のテスト方針を優先する。
- 修正ごとに関連する review issue doc を更新する（`review-issue-docs` skill 参照）。
- 必要な lint、typecheck、test、build を実行する。
- 実行できない検証は理由を記録する。
- 実装と検証が終わったら、`final-merge-blocker-review` skill に従い、最終レビューを **同じ session 内のサブエージェント** として実行する。最終レビューのために新規 session を起動しない。
- blocker なしと判定したら `pr-push-safety` に進む。

## 5. handoff 後の planning Session

- Step 9（`final-merge-blocker-review`、サブエージェント実行）と Step 10（`pr-push-safety`）は実装セッションが続けて行う。planning Session が別のレビューセッションを起動しない。
- 実装セッションが停止・報告してきた場合は、計画を修正するか人間判断へ回す。
