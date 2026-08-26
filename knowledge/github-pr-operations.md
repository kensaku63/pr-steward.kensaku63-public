# GitHub PR 操作手順

PR Steward が GitHub PR を扱うときの操作手順と制約の正本。

## PR の特定と checkout

対象 PR は明示された PR URL または PR number から特定する。
checkout の実行手順は `$AA_AGENT_DIR/.agents/skills/pr-checkout/SKILL.md` を正本とする。

必須手順:

- `gh pr view <number|url> --json number,headRepository,headRefName,baseRefName,author,maintainerCanModify,headRefOid` で PR number、head repository、head branch、base branch、author、maintainer permission、current HEAD を確認する。
- 作業前に `git status --porcelain` を確認する。
- dirty working tree がある場合、対象 PR 由来か判定できるまで変更を開始しない。
- `git worktree list --porcelain` で PR head branch が別 worktree に占有されていないか、`gh pr checkout` より先に確認する。
- 別 worktree が使用中なら、その worktree を変更せず、PR の `refs/pull/<number>/head` から session 専用 local branch を upstream なしで作る。
- checkout 後に `git branch --show-current`、tracking branch、base branch、PR head を再確認する。
- 通常の review / fix workflow で更新を取り込むときは fast-forward のみ許可する。明示的な PR merge 依頼の例外は `$AA_AGENT_DIR/knowledge/human-approval-policy.md` を正本とする。

禁止事項:

- 明示されていない branch への checkout / push。
- user changes の破棄、上書き、revert。
- force push、rebase、history rewrite。人間から依頼されても PR Steward は実行しない。

## push ルール

push 前チェックは `$AA_AGENT_DIR/.agents/skills/pr-push-safety/SKILL.md` を正本とする。

- 通常 push のみ許可する。
- force push は常時禁止する。
- 同一 PR / 同一セッションの自動 fix-and-push は最大 3 回までとする。回数は対象 Project の audit record shared document に記録する。
- push 失敗時は原因を分類し、勝手に force push しない。
- `git push origin HEAD:refs/heads/<pr-head>` のような明示 refspec push は local remote-tracking ref を更新しない。push 後は remote branch と `refs/pull/<pr>/head` を session 用 ref へ再 fetch し、local commit、remote branch、GitHub `headRefOid`、PR ref の OID 一致と containment を確認する。終了時の local handoff 観測に古い snapshot を残さない。

## GitHub コメントポリシー

GitHub PR コメントには、PR 参加者が読むべき結論だけを書く。投稿してよい内容は、PR 参加者の判断または次の行動を変えるものに限る。

投稿してよい内容:

- merge value gate の `reject` または `needs-human` の結論。
- 根拠、影響範囲、期待される修正方針が揃った具体的な bug、risk、regression 指摘。
- マージ可否、修正要否、仕様判断に関わる blocker。
- 人間に選択してほしい判断事項と、選択肢ごとの影響。
- 修正後に PR 参加者が確認すべき挙動変更、残リスク、未解決事項。
- テスト結果がマージ判断を左右する場合の要点と、失敗時の原因・次アクション。

投稿してはいけない内容:

- 内部思考。
- 生ログ全文。
- 未確認の推測。
- 自動生成された大量コメント。
- secret、環境情報、credential の値。
- aachat 内部の会話そのもの。

## 既存 PR レビューの取得

並列レビュー開始前に、PR 上の既存 feedback を全 surface から取得する。Cursor / Codex の投稿だけを名前 filter で探すのではなく、全投稿を取得してから source metadata で識別する。

最低限取得する surface:

- conversation issue comments
- submitted reviews と review body
- inline review comments
- review thread の resolved / unresolved と comment の outdated / current

`gh pr view --comments` だけでは inline thread を網羅した証拠にならない。REST または GraphQL を併用し、pagination を最後まで処理する。

代表的な取得:

```bash
gh api --paginate repos/{owner}/{repo}/issues/{pr}/comments
gh api --paginate repos/{owner}/{repo}/pulls/{pr}/reviews
gh api --paginate repos/{owner}/{repo}/pulls/{pr}/comments
```

thread の resolution と current/outdated 状態が必要な場合は、`gh api graphql` で対象 PR の `reviewThreads` と各 thread の comments を取得する。

保存する field:

- GitHub ID / database ID
- URL
- author login、author association、取得できる app/bot metadata
- created / updated timestamp
- review state
- path、line、original line、commit / original commit
- resolved / outdated
- body

注意:

- author login は変更・派生し得るため、Cursor / Codex の login 名を正本として決め打ちしない。
- tool family は metadata または本文から明確な場合だけ付け、不明なら `unknown` とする。
- resolved / outdated の指摘も収集対象には含めるが、現在も有効かコード上で再検証する。
- post-push audit では `resolved=false / outdated=false` だけを blocker と判定しない。`original_commit_id`、final-head の実コードと line mapping、対応 regression test、final ready gate を突き合わせ、指摘が decision-level で残るかを判定する。GitHub thread の UI state と修正済みかどうかは別の事実として記録する。
- API で取得できない surface、pagination 未完了、権限不足があれば `未確認` と記録し、完全取得したと報告しない。

## GitHub CLI active account と Checks API

PR Steward の GitHub 操作は、`gh auth status` が示す一つの active member account を使う。PR metadata は取得できても `gh pr checks` や check-runs API が 403 を返す場合は、同じ active account の repository 権限または organization SSO 承認の不足として記録する。

permission failure を未計測や PASS に読み替えず、その operation で停止する。回復は member が通常の GitHub CLI account、対象 repository の権限、必要な SSO 承認を修復する経路だけとする。agent は別credential、別identity、PAT、GitHub App tokenへ切り替えず、command-localな環境操作やfallbackも行わない。

## 失敗の分類と扱い

失敗は次のカテゴリに分類する。

- `auth_error`
- `permission_denied`
- `checkout_error`
- `dirty_worktree`
- `conflict`
- `test_failure`
- `ci_failure`
- `tool_error`
- `unknown`

共通ルール:

- transient failure の自動再試行は最大 2 回まで。
- 同じ失敗が続く場合は停止し、現在状態、原因仮説、次アクションを記録する。
- conflict は通常 workflow では解消せず停止する。対象 PR の明示的な merge 依頼がある場合だけ、`$AA_AGENT_DIR/knowledge/human-approval-policy.md` の限定条件に従う。
- CI 失敗は、自分の変更起因か、既存失敗か、flaky / infra / permission / external service かを分類する。
- 不明な CI 失敗を解消するために無制限な修正 push を繰り返さない。
- 中断時は branch、未コミット差分、完了済み作業、未完了作業、再開手順を aachat shared document に記録する。
