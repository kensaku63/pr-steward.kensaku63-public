---
name: pr-checkout
description: 対象 GitHub PR を特定し、working tree と既存 worktree の branch 占有を確認してから安全に checkout する skill。Use when starting work on a target PR, when the working tree state is unknown, or before any review or fix work begins.
metadata:
  aachat.headline.ja: "安全な PR チェックアウト"
  aachat.headline.en: "Safe PR Checkout"
  aachat.description.ja: "対象 PR と既存作業ツリーを確認し、他の作業を壊さずに安全な checkout を準備します。"
  aachat.description.en: "Identifies a target PR and prepares a safe checkout without disrupting existing worktrees."
  aachat.discovery.listed: "true"
---

# PR Checkout

PR Steward workflow の Step 1。対象 PR を特定し、安全に checkout する。

## 1. PR を特定する

対象 PR は明示された PR URL または PR number から特定する。明示されていない場合は作業を開始せず、人間に確認する。

```bash
gh pr view <number|url> --json number,title,headRepository,headRepositoryOwner,headRefName,baseRefName,author,maintainerCanModify,headRefOid,state,isDraft
```

記録する項目:

- PR number
- head repository / head branch
- base branch
- author
- maintainer permission（`maintainerCanModify`）
- current HEAD（`headRefOid`）

## 2. working tree を確認する

```bash
git status --porcelain
git branch --show-current
```

- dirty working tree がある場合、対象 PR 由来か判定できるまで変更を開始しない。
- 判定できない場合は `dirty_worktree` として停止し、現在状態と次アクションを記録して人間に確認する。
- user changes の破棄、上書き、revert は禁止。

## 3. 既存 worktree の branch 占有を確認する

```bash
git worktree list --porcelain
```

- PR の `headRefName` が `branch refs/heads/<headRefName>` として別 worktree に出ているか確認する。
- 別 worktree が使用中なら、その worktree を detach、switch、削除しない。`gh pr checkout` も実行せず、次の session-scoped fallback を使う。
- 使用中でなければ `gh pr checkout <number>` を使う。

### 通常経路

```bash
gh pr checkout <number>
```

### 既存 worktree が使用中の経路

PR の virtual ref を session 専用 remote-tracking ref に fetch し、session 固有のローカル branch を作る。

```bash
git fetch --no-tags origin \
  "+refs/pull/<number>/head:refs/remotes/origin/pr-steward/<number>/<session-suffix>"
git rev-parse "refs/remotes/origin/pr-steward/<number>/<session-suffix>"
git switch --no-track -c "pr-steward/<number>-<session-suffix>" \
  "refs/remotes/origin/pr-steward/<number>/<session-suffix>"
```

- `<session-suffix>` は checkout 前の session branch の末尾など、この session に固有の値を使う。
- fetch 後の OID と、直後に再取得した `gh pr view <number> --json headRefOid --jq .headRefOid` を照合する。一致しなければ switch せず、PR metadata を更新して fetch からやり直す。
- local branch 名または専用 ref が既に存在する場合は、削除・上書きせず、使用中 worktree と参照先を確認して別の session 固有名を選ぶ。
- fallback branch に upstream は設定しない。workspace の限定的な `remote.origin.fetch` では、PR head の remote ref が tracking branch として認識されないことがある。
- 後で push するときの target は local branch 名ではなく、最初に記録した PR の `headRepository` と `headRefName`。`pr-push-safety` で権限を確認し、明示的な refspec を使う。

## 4. checkout 後に確認する

```bash
git branch --show-current
git rev-parse HEAD
git status --porcelain
git worktree list --porcelain
```

- current branch、base branch、PR head、使用中 worktree が事前記録と一致するか確認する。
- HEAD が最新の `headRefOid` と一致しない場合は原因を確認してから進む。
- fallback 経路では upstream が無いことを正常と扱い、audit record に PR の本来の head branch と明示 push target を残す。

`gh pr checkout` が失敗した場合は、再試行より先に次を確認する。失敗途中で index / working tree だけが PR tree に更新され、branch は元のまま残ることがある。

```bash
git branch --show-current
git rev-parse HEAD
git status --porcelain
git diff --cached --stat
git diff --stat
```

- 事前に clean だった tree が変化していたら、自動で restore、reset、stash しない。
- 現在の HEAD / index / working tree と PR head の一致を調べ、既存変更を破棄しない方法で branch ref を整合させる。
- 原因を確認しないまま同じ `gh pr checkout` を繰り返さない。

## 5. 更新の取り込み

- 通常経路では pull は fast-forward のみ許可する: `git pull --ff-only`
- fallback 経路では同じ session 専用 ref へ再 fetch し、OID を照合してから `git merge --ff-only <session-ref>` で更新する。
- merge commit、rebase、history rewrite は自動実行しない。
- 通常の review / fix workflow で conflict が出たら解消せず停止する。対象 PR の明示的な merge 依頼がある場合は、`$AA_AGENT_DIR/knowledge/human-approval-policy.md` の限定条件に従い、必要な再検証を行う。

## 禁止事項

- 明示されていない branch への checkout / push。
- 別 worktree の branch を空けるための detach、switch、削除。
- user changes の破棄、上書き、revert。
- force push、rebase、history rewrite。

## 6. 記録

checkout 結果（PR number、PR head repository / branch、local branch、HEAD、使用した通常 / fallback 経路、working tree 状態、明示 push target）を audit record doc に記録する。schema は `$AA_AGENT_DIR/knowledge/aachat-review-doc-schema.md` を参照。

失敗時は `$AA_AGENT_DIR/knowledge/github-pr-operations.md` の失敗分類（`auth_error` / `permission_denied` / `checkout_error` / `dirty_worktree` など）に従って分類する。同じ原因への再試行は最大 2 回までにし、再試行前に branch / HEAD / index / working tree の部分更新を確認する。
