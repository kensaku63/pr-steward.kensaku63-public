---
name: final-merge-blocker-review
description: 実装後の PR に「マージを止める重大問題が残っているか」だけを判定する最終レビュー skill。Use after approved fixes are implemented, before push, or when deciding whether remaining issues are blockers or acceptable follow-ups.
metadata:
  aachat.headline.ja: "最終マージ阻害レビュー"
  aachat.headline.en: "Final Merge-Blocker Review"
  aachat.description.ja: "実装後に、マージを止める重大な問題だけを独立して確認します。"
  aachat.description.en: "Independently checks only for critical issues that should block a merge after implementation."
  aachat.discovery.listed: "true"
---

# Final Merge-Blocker Review

PR Steward workflow の Step 9。実装後の最終レビューは、改善点探しではなく「マージを止める重大問題が残っているか」の判定に限定する。

このレビューは **新規 session を起動せず、実装を行った session の中からサブエージェント（Agent tool）として実行する**。実装の文脈に引きずられない fresh context のレビュアーを 1 つ起動し、その判定を親（実装 session）が精査する。

指摘があった場合、親はまず「元の PR の欠陥」か「approved fix が作った欠陥」かを分類する。失敗条件が approved fix で追加した state、分岐、順序、API に依存し、base には存在しなかったなら後者とする。後者なら局所 patch を自動追加せず、修正自体の削除・簡素化を優先して再計画する。

## 1. サブエージェントの起動

実装 session が、merge-blocker 判定専用のレビューサブエージェントを 1 つ起動する。プロンプトには次を含める。

- PR number / base branch / head branch と、判定対象の diff 範囲（`git diff <base>...HEAD`）。
- approved fix plan doc と関連 review issue docs への参照。
- 下記「確認項目」「やらないこと」をそのまま制約として渡す。
- 出力形式: blocker の有無、blocker ごとの証拠（file:line）と影響、follow-up 扱いにした課題とその理由。

サブエージェントには判定だけをさせる。修正の実装、push、session 起動はさせない。

## 2. 確認項目（サブエージェントに渡す）

- CI、lint、typecheck、test が通るか。
- 修正が PR スコープを逸脱していないか。
- 修正が新しい正本、永続 lifecycle、状態機械、複数 caller の protocol を増やしていないか。増やしている場合、その複雑さがレビュー指摘ではなく承認済みのユーザー価値から直接必要か。
- 新しい回帰、例外、race、認可漏れ、データ破壊が入っていないか。
- エラー処理と境界条件が既存仕様に合っているか。
- テストが実装内容を実際に検証しているか。
- 残課題は merge blocker ではなく follow-up で許容できるか。

## 3. やらないこと（サブエージェントに渡す）

- 新しい改善提案、style 指摘、任意リファクタの追加。
- approved fix plan に含まれない課題の実装（merge blocker でない限り）。
- 失敗テストの削除・弱体化による「通す」行為。

## 4. 判定

親（実装 session）はサブエージェントの指摘を鵜呑みにせず、証拠を自分で確認してから判定する。

- blocker なし: `pr-push-safety` に進む。
- blocker あり（元の PR の欠陥で、単純な局所修正が可能）: 親が修正し、再度サブエージェントでこの確認を行う。fix-and-push の回数上限（3 回）に注意する。
- blocker あり（approved fix が作った欠陥、または解消に新たな state / nonce / CAS / fence / caller protocol が必要）: patch-on-patch を止める。push せず、approved fix の削除・簡素化を第一案として計画へ戻し、必要なら人間判断へ回す。
- 判断不能な仕様問題が残る場合: push 前に停止して人間判断へ回す（`$AA_AGENT_DIR/knowledge/human-approval-policy.md`）。

## 5. 記録

audit record doc に次を追記する。

- 実行した検証コマンドと結果（test / lint / typecheck / build）。
- 見つかった blocker と対応。
- blocker の由来（元の PR / approved fix）と、簡素化を検討した結果。
- follow-up として残した課題と、blocker ではないと判断した理由。
- 最終判定（push 可 / 停止 / 人間判断待ち）。
