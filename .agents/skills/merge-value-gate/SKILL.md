---
name: merge-value-gate
description: PR を後続の自動レビュー・修正フェーズに進める価値があるかを pass / needs-human / reject で判定する skill。Use after checking out a target PR, before launching parallel reviewers, or when deciding whether a PR is worth automated stewardship.
metadata:
  aachat.headline.ja: "マージ価値ゲート"
  aachat.headline.en: "Merge Value Gate"
  aachat.description.ja: "PR を安全に後続レビューと修正へ進める価値があるかを判定します。"
  aachat.description.en: "Decides whether a pull request is worth safely advancing to review and remediation."
  aachat.discovery.listed: "true"
---

# Merge Value Gate

PR Steward workflow の Step 2。PR を後続の自動レビュー・修正フェーズに進める価値があるかを判定する。

## 1. 必ず答える問い

判定前に、PR の説明、差分、関連 Issue、repo 方針を読み、次に答える。

1. この PR は Issue、仕様、ユーザー価値、運用改善、バグ修正のいずれかに明確につながっているか。
2. repo の方針、プロダクト方向、公開範囲、セキュリティ方針に反していないか。
3. 変更量や複雑さに対して、得られる価値が釣り合っているか。
4. 既存機能を壊す可能性が、期待される価値より明らかに大きくないか。
5. 変更が未完成、実験途中、デバッグ用途、生成物の混入ではないか。
6. secret、credential、個人環境依存、危険な権限変更など、即時停止すべきリスクがないか。
7. テスト、ドキュメント、移行手順など、最低限の検証可能性があるか。
8. 同じ目的を満たす既存実装や既存 PR と重複していないか。
9. 後続の並列レビューに進めば、妥当な修正でマージ可能性を高められる状態か。

## 2. 判定

判定は `pass`、`needs-human`、`reject` のいずれかとする。

- `pass`: PR の目的と価値が確認でき、明確な停止リスクがなく、後続レビューと修正によってマージ可能性を高められる。
- `needs-human`: 価値判断に必要な文脈が不足している、プロダクト判断が必要、破壊的変更の許可が必要、または自動 reject には証拠が弱い。
- `reject`: PR が明確にマージ対象ではないと判断できる。例: repo 方針への明確な違反、目的不明の大量変更、無関係な生成物のみ、完全に重複した PR。

secret、credential、token、private key らしき差分、または危険な権限変更を検出した場合は `reject` にせず即時停止し、値を表示・保存・公開コメントせず `needs-human` として扱う。対応の正本は `$AA_AGENT_DIR/knowledge/human-approval-policy.md`。

## 3. ガードレール

- reject は例外扱いにする。迷ったら `needs-human` または `pass` に倒す。
- 実装が粗い、テスト不足、レビュー指摘が多そう、という理由だけでは reject しない。
- 後続レビューで直せる問題は merge value gate で止めない。

## 4. 結果の記録と通知

- 判定と根拠（答えた問いと証拠）を audit record doc に記録する。
- `reject` / `needs-human` の場合のみ、GitHub PR コメントに結論を投稿する。`pass` はコメント不要。ただし secret 等の検出が理由の場合は公開コメントを投稿せず、Project Ask だけを使う。
- reject コメントは事実ベースの理由、確認した証拠、再提出条件を最大 3 件で簡潔に書く。
- 作者の意図・能力への評価、証拠のない推測、secret の値そのものは書かない。
- `needs-human` の場合は、人間に選択してほしい判断事項と選択肢ごとの影響を asks に書く（`$AA_AGENT_DIR/knowledge/human-approval-policy.md` 参照）。

## 5. 次のステップ

- `pass`: `parallel-pr-review` に進む。
- `needs-human`: 人間の回答を待つ。回答後に再判定する。
- `reject`: コメント投稿と記録をして終了する。
