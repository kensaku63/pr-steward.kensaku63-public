---
name: parallel-pr-review
description: GitHub PR 上の既存レビュー（Cursor / Codex を含む）を取り込み、Outcome Gap / UX Friction / Code Quality / Release Hardening の 4 観点レビューサブエージェントを並列起動して、マージ可能性を高める課題候補を集める skill。Use after the merge value gate passes, when reviewing a PR from multiple perspectives, or when collecting review issues for validation.
metadata:
  aachat.headline.ja: "並列 PR レビュー"
  aachat.headline.en: "Parallel PR Review"
  aachat.description.ja: "既存レビューと四つの観点を並列に取り込み、検証対象の課題候補を集めます。"
  aachat.description.en: "Collects review candidates in parallel from existing feedback and four complementary perspectives."
  aachat.discovery.listed: "true"
---

# Parallel PR Review

PR Steward workflow の Step 3。親エージェントは、まず GitHub PR 上の既存レビューを取得し、その後 4 つのレビューサブエージェントを並列起動する。既存レビューとサブエージェント出力は、採用前の `candidate` として同じ intake ledger に集約する。

## 0. GitHub PR 上の既存レビューを取得する

4 観点レビューの開始前に、次の GitHub surface をすべて取得する。

- PR conversation の issue comments
- submitted review の summary/body
- inline review comments と review threads
- resolved / unresolved、outdated / current の状態

取得時に reviewer を人間だけへ限定しない。Cursor / Codex が投稿した review、summary、inline comment も必ず候補集合へ含める。bot 名を固定値で決め打ちせず、GitHub の `author.login`、app/bot metadata、comment URL / ID を保存する。Cursor / Codex と判定できない投稿は `unknown` のまま残し、推測で帰属させない。

取得方法と必要 field は `$AA_AGENT_DIR/knowledge/github-pr-operations.md` の「既存 PR レビューの取得」を正本とする。

各投稿は audit record の intake ledger に、最低限次を記録する。

- stable candidate ID
- source kind（issue comment / review summary / inline comment / parallel reviewer）
- author login と、確認できる場合だけ tool family（Cursor / Codex / human / other）
- URL / GitHub ID
- path / line / review state / resolved / outdated（存在する場合）
- 指摘本文の短い要約
- disposition（最初は `untriaged`）

レビュー本文を取得できなかった surface がある場合、レビュー取り込み完了とはしない。権限・pagination・API 制限などの理由を audit record に `未確認` として記録する。

## 1. 起動する 4 観点

### Outcome Gap Reviewer（`reviewer_role: outcome_gap`）

- PR が実現しようとしているユーザー成果、仕様、受け入れ条件に対して、足りない挙動、ズレた挙動、説明不足、回帰しそうな体験を課題として出す。
- 実装上のバグや境界条件漏れのうち、ユーザー成果、仕様、受け入れ条件の未達として説明できるものもこの担当が扱う。
- 改善案は、PR の目的を変えずに期待挙動へ近づける最小の仕様・UI・ドキュメント・テスト補強に限定する。

### UX Friction Reviewer（`reviewer_role: ux_friction`）

- PR が導入・変更する画面、フォーム、通知、エラー、空状態、読み込み状態、導線、文言、アクセシビリティ、体感速度、操作感について、ユーザーが迷う・失敗する・不安になる・戻れない・待たされる・反応が鈍いと感じる課題を出す。
- 改善案は、仕様やスコープを広げずに、ユーザーが次に何をすべきか分かる状態、失敗から回復できる状態、重要な情報に気づける状態、待ち時間を理解できる状態、操作が軽く自然に感じられる状態へ近づける最小の UI / copy / interaction / accessibility / feedback / perceived performance 改善に限定する。
- 好みのデザイン、全面的な UI 再設計、ブランド判断、A/B テストが必要な仮説は課題化せず、必要なら人間判断へ回す。

採用基準:

- UX 課題は、ユーザーが目的を達成するまでの摩擦、誤操作、理解不能、回復不能、不安、待ち時間、アクセシビリティ上の障害として説明できる場合だけ課題化する。
- 課題化するには、対象ユーザー、利用シナリオ、問題が起きる画面・状態・操作、ユーザーへの影響、最小の改善案を明示する。
- エラー文言は「何が起きたか」「ユーザーが次に何をすべきか」「再試行・戻る・問い合わせなどの回復手段」が不足している場合だけ指摘する。
- 読み込み中、空状態、成功後、失敗後、権限不足、入力途中、モバイル幅、キーボード操作、スクリーンリーダーの状態を確認対象に含める。
- レイテンシーは backend 性能だけでなく UX として扱う。クリック・入力・保存・検索・遷移・生成処理などで、反応が遅い、進行状況が分からない、二重操作できてしまう、待機中に次の行動が分からない場合は課題化する。
- 操作の気持ちよさは、主観的な好みではなく、反応の即時性、入力中の安定性、フォーカス維持、楽観更新、適切な disabled / loading / success feedback、アニメーションや遷移の過不足として具体化できる場合だけ扱う。
- 性能改善を提案する場合は、推測だけでなく、ユーザーが知覚する遅さ、発生する操作、影響範囲、確認方法を示す。大きな最適化ではなく、まず feedback、progress 表示、debounce、request dedupe、prefetch、skeleton、optimistic UI など PR スコープ内の最小改善を優先する。
- 見た目の好み、色や余白の趣味、全面的なデザイン刷新は提案しない。改善案は PR のスコープ内で実装できる最小変更にする。
- ブランド、コピー方針、情報設計、ユーザー調査、A/B テストが必要な判断は自動実装せず、human decision として扱う。

### Code Quality Reviewer（`reviewer_role: code_quality`）

- 将来の変更を安く・速く・安全にする観点で、PR が増やした変更コスト、認知負荷、不要な結合、浅い抽象、責務の散在、ドメイン語彙のズレ、テスト不能性を課題として出す。
- 改善案は、PR の目的と振る舞いを保存したまま、変更の局所性、読みやすさ、深いモジュール、驚きのなさ、ドメイン整合、エラーパス品質、必要な安全網を高める最小のリファクタリングに限定する。
- 変更予定のないコードの美化、命名・スタイルだけの変更、分割すること自体が目的の提案、カバレッジ数値の改善だけを目的にしたテスト追加は出さない。

採用基準:

- 品質課題は「綺麗かどうか」ではなく、「この PR または近い将来の変更コストを上げるか」で判断する。
- 課題化するには、変更が局所で済まない、一度に頭に入らない、浅い抽象が増える、名前や慣習から動作を推測できない、ドメイン語彙と構造がズレる、エラーパスが粗い、振る舞いを固定する安全網がない、のいずれかに該当する必要がある。
- 変更頻度が低い安定コード、sunset 予定のコード、PR で触れていない周辺コードの美化は提案しない。
- 抽象化や分割は、それによって隠せる複雑さが増え、caller が知るべき量が減る場合だけ提案する。小さくすること自体を成果にしない。
- 重複排除は、同じ変更が実際に同時発生しそうな場合だけ提案する。wrong abstraction を作るくらいなら重複を許容する。
- 品質改善案は、PR の観察可能な振る舞いを保存することを前提にする。振る舞い変更を伴う場合は Outcome Gap または Release Hardening の課題として扱う。

### Release Hardening Reviewer（`reviewer_role: release_hardening`）

- マージ後の事故につながるセキュリティ、権限、secret、性能、migration、ログ、監視、CI/CD、ロールバック困難性を課題として出す。
- 実装上のデータ整合性、race、認可漏れ、エラー処理漏れのうち、リリース後の事故や復旧困難性として説明できるものもこの担当が扱う。
- 改善案は、リリース前に必要な安全策、検証、観測性、ロールバック準備、運用手順の補強に限定する。

## 2. 共通ルール

- 各サブエージェントは担当領域外の指摘を出さない。
- 指摘は、観測した事実、直す価値、十分な解決状態が説明できるものに限定する。解法を示す場合は `solution hypothesis` と明記し、approved plan や要件として扱わない。
- style、命名、好み、軽微なリファクタは、マージ判断に影響しない限り出力しない。
- 長文成果物は message に書かず、まず audit record の intake ledger に candidate として固定する。
- candidate の段階では review issue doc を量産しない。親エージェントが有効と確認した unique cluster だけを、課題 1 件につき 1 doc へ昇格する。doc の path と schema は `review-issue-docs` skill と `$AA_AGENT_DIR/knowledge/aachat-review-doc-schema.md` に従う。
- 各課題は「何が問題か」だけで終えず、「なぜ今直すべきか」「どの状態になれば十分か」まで書く。局所解の最適性は主張しない。
- 書けない項目を無理に埋めない。確認できていない内容は `未確認` と明記する。
- `ありそう`、`可能性がある` だけの課題は、直す価値と十分な解決状態まで具体化できない限り課題化しない。
- テスト不足は単独では課題にしない。追加すべきテストがあると主張する場合は、守るべき振る舞い、壊れやすい変更点、既存テストでは検知できない理由、最小のテスト形態を明示する（`$AA_AGENT_DIR/knowledge/review-priority-rubric.md` のテスト指摘採用基準を参照）。
- 既存 PR レビューと同じ指摘を見つけた場合も削除せず、同じ cluster に source を追加する。どの reviewer が先に指摘したかを失わない。

## 3. サブエージェントへ渡す入力

各サブエージェントの prompt には最低限次を含める。

- PR URL / number、base branch、head branch
- PR の目的の要約と関連 Issue / 仕様へのリンク
- 担当領域の定義と採用基準（このファイルの該当節）
- audit record の intake ledger contract、candidate ID / source metadata / disposition の規約
- 親エージェントによる精査後に使う review issue doc の path 規約と frontmatter schema
- 「担当領域外の指摘を出さない」「採点や総評をしない」という制約

## 4. 完了条件

- GitHub PR の既存レビュー取得が完了し、Cursor / Codex を含む全投稿の candidate ID と source が intake ledger にある。
- 4 観点すべてのサブエージェントが完了し、その全候補が intake ledger に追記されている。
- 明らかな同一指摘を cluster 化しても、元 candidate ID と source が失われていない。
- candidate 総数と `untriaged` 件数を audit record doc に記録してから、親エージェントの batch 精査（`$AA_AGENT_DIR/knowledge/review-priority-rubric.md`）に進む。
