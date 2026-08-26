# pr-steward

PR Steward は、指定された GitHub Pull Request をマージ可能な状態へ安全に整える aachat agent です。

マージ価値の判定（merge value gate）、4 観点の並列レビュー、証拠精査、fresh context での統合修正設計、修正実装、最終 merge-blocker review、PR ブランチへの push までを 1 つのワークフローとして実行します。

## できること

- PR の目的・差分・リスクを読み、後続レビューに進める価値を `pass` / `needs-human` / `reject` で判定する。
- Outcome Gap / UX Friction / Code Quality / Release Hardening の 4 観点でレビューサブエージェントを並列起動し、課題を集める。
- GitHub PR 上の既存レビューを取得し、人間に加えて Cursor / Codex の指摘も出典付きで候補集合へ取り込む。
- サブエージェントの指摘を証拠・影響・スコープで精査し、問題の事実性と個別 reviewer の解法を分離する。
- 複雑な review では fresh planning Session を使い、全課題を共通 root cause、既存 authority、不変条件から一つの最小な設計へ統合する。
- 承認済み実装計画を新しい実装セッションに handoff し、最小限の修正だけを実装させる。
- 実装後に merge blocker が残っていないかだけを判定する最終レビューを、実装セッション内のサブエージェントで実行する。
- 安全チェックを満たした場合のみ PR head branch へ通常 push する。
- レビュー課題、実装計画、監査ログを aachat shared document に残し、GitHub PR コメントは簡潔な結論に限定する。

## 向いている依頼

- 「この PR をマージできる状態にして」
- 「この PR、マージする価値があるか判定して」
- 「この PR をレビューして、直すべき点を直して push して」
- 「外部コントリビューターの PR を安全に取り込みたい」

## 使い方

```bash
aachat agent clone owner/pr-steward --name pr-steward
```

clone 後、project に参加させてから session を起動します。

```bash
aachat project assign <project> --agent pr-steward
aachat session run pr-steward --project <project> "https://github.com/<owner>/<repo>/pull/<number> をマージ可能な状態にして"
```

対象 PR は PR URL または PR number で明示してください。

## ワークフロー

1. `pr-checkout`: PR の特定と安全な checkout。
2. `merge-value-gate`: マージ価値の判定。`reject` は例外扱いで、迷ったら人間判断に倒す。
3. `parallel-pr-review`: Cursor / Codex を含む既存 PR レビューの取得と、4 観点レビューの並列実行。
4. 全 candidate の intake ledger 化と、根本原因単位の cluster 化。
5. `review-issue-docs`: 小さな batch で証拠を精査し、有効な課題を `proposed` として 1 件ずつ固定。
6. review package の件数照合・重複排除・優先度付けと凍結。実装計画はまだ作らない。
7. `integrated-fix-planning`: 必要なら fresh planning Session で、全課題から一つの統合修正設計を確定。
8. `implementation-handoff`: planning Session から新しい実装セッションへの handoff。
9. `final-merge-blocker-review`: 実装後の merge-blocker 判定（実装セッション内のサブエージェントで実行）。
10. `pr-push-safety`: push 前チェックと通常 push。

## 安全方針

- force push、rebase、history rewrite は行わない。
- conflict 解消は、対象 PR を指定した明示的な merge 依頼に含まれる最小限の解消だけを許可する。それ以外は停止する。
- PR の merge / close / approve / request changes は人間の明示承認が必要。
- 仕様変更、UX 判断、API 契約変更、DB migration、認証・認可・課金・データ削除に関わる変更は人間判断へ回す。
- secret、credential らしき差分を検出したら、`reject` や公開コメントへ進まず即時停止して人間判断へ回す。値は出力しない。
- 同一 PR / 同一セッションの自動 fix-and-push は最大 3 回まで。

詳細は `$AA_AGENT_DIR/knowledge/human-approval-policy.md` を参照してください。

## 構成

- `identity.md`: エージェントの役割、行動原則、workflow と skill の対応の正本。
- `environment.yaml`: 必要な実行環境。依存は `config.packages` に、必要な env 名は `config.env[]` に書く。
- private runtime memory: 個別 project から切り離した経験、観察、未解決の仮説を置く非公開領域。公開 repository には含めない。
- `$AA_AGENT_DIR/knowledge/`: GitHub PR 操作手順、レビュー doc schema、優先度基準、人間承認ポリシー。
- `$AA_AGENT_DIR/.agents/skills/`: この agent 専用の実行時 skill。Discovery の子 skill カタログもここを優先して見る。
- `$AA_AGENT_DIR/scripts/check.sh`: agent repo の必須構造、skill metadata、正本参照、PR 固有状態の混入を検査する read-only check。

PR 固有の未完了状態、push 回数、再開手順、監査記録は、対象 Project の audit record shared document に残します。

## Repository check

```bash
bash "$AA_AGENT_DIR/scripts/check.sh"
```

この check はファイルを変更せず、構造・参照・方針の既知のドリフトを検出します。

## 実行時 Skill

- `pr-checkout`: 対象 PR の特定、working tree 確認、安全な checkout。
- `merge-value-gate`: `pass` / `needs-human` / `reject` の判定基準とガードレール。
- `parallel-pr-review`: Cursor / Codex を含む既存 PR レビューの intake、4 観点レビューサブエージェントの起動と共通ルール。
- `review-issue-docs`: レビュー課題の shared document 化と frontmatter 規約。
- `integrated-fix-planning`: evidence-validated な複数課題を fresh context で一つの最小な修正設計へ統合。
- `implementation-handoff`: 承認済み実装計画の新セッションへの引き継ぎ。
- `final-merge-blocker-review`: 実装後の merge-blocker 限定レビュー（実装セッション内のサブエージェントで実行）。
- `pr-push-safety`: push 前チェックリストと push ルール。

## 必要な GitHub CLI 認証と権限

GitHub 操作には、owner machine の GitHub CLI に保存されている active account を使います。session を起動する前に `gh auth status` が成功し、意図した member account が active であることを確認してください。その member には対象 repository の read / write 権限と、organization が要求する場合は有効な SSO 承認が必要です。

認証、権限、SSO のいずれかで失敗した場合は、その operation で停止し、member が通常の GitHub CLI 認証または権限を修復します。agent は provider token の選択、作成、差し替え、fallback を行いません。

## 注意

secret、token、JWT、PAT、秘密鍵は repo に含めないでください。
PR Steward の GitHub 操作用 credential を `environment.yaml` や repo 内文書へ追加しないでください。credential の管理と active account の選択は member の通常の GitHub CLI 認証に委ねます。

Skill Discovery 用の日本語・英語の説明と表示設定は、各 `.agents/skills/*/SKILL.md` の `metadata` に含まれます。

### description_ja（submit 用候補）

PR Steward は、GitHub Pull Request をマージ可能な状態へ安全に整えるエージェントです。マージ価値の判定、成果・UX・コード品質・リリース安全性の 4 観点並列レビュー、指摘の証拠精査、fresh context での統合修正設計、最小限の修正実装、最終 merge-blocker review、安全確認付きの push までを行います。判断材料と監査ログは aachat shared document に残し、破壊的 git 操作や仕様判断は人間承認に回します。

### description_en（submit 用候補）

PR Steward safely drives a GitHub pull request toward a mergeable state. It runs a merge-value gate, launches four parallel reviewers, separates evidence validation from proposed remedies, synthesizes complex findings into one coherent minimal design in a fresh planning session, implements that plan in another fresh session, runs a final merge-blocker-only review, and pushes with strict safety checks. Long-form reasoning and audit logs live in aachat shared documents; destructive git operations and spec-level decisions are escalated to humans.
