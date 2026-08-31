---
id: adr-0013
title: ADR-0013 — 直接親とリポジトリidentityを現在のアカウントへ付け替える
status: proposed
updated: 2026-09-01
---

# ADR-0013: 直接親とリポジトリidentityを現在のアカウントへ付け替える

| Field | Value |
|-------|-------|
| Status | proposed |
| Date | 2026-09-01 |
| Deciders | repository owner |
| Author | Claude Code (AI agent) |
| Supersedes / Superseded by | ADR-0008のdirect parent条項をsupersedeする |

## Context

本リポジトリは `ea-Mitsuoka` アカウントへ移行済みである。`git remote get-url origin` は
`https://github.com/ea-Mitsuoka/secure-ga4-bq-template.git` を返す。一方、継承metadataは
旧 `Yukihide-Mitsuoka` アカウントを直接親としても自身のidentityとしても指したままである。

`ea-Mitsuoka/terraform-gcp-template` と `Yukihide-Mitsuoka/terraform-gcp-template` は別
アカウントが所有する別リポジトリであり、GitHubのrename redirectは適用されない。結果として
次の状態が発生している。

- scheduled Template Syncが、所有者の更新していないリポジトリを読む。停止した親はエラーを
  出さず、変更が届かないだけなので気づきにくい。
- `make doctor` が `Root README ownership is invalid (ADR-0011)` を報告する。
  `scripts/readme_ownership.py` はmarkerを `remote.origin.url` と比較する。
- `.github/CODEOWNERS` が、本リポジトリを所有しない `@Yukihide-Mitsuoka` へreviewを振るため、
  required reviewerが解決しない。
- `infra/envs/dev/variables.tf` の `github_repository` defaultが旧リポジトリ名のままである。
  この変数は、deployer/inspector SAへのimpersonationを許すWIF principal setを決める。既定値の
  まま `terraform apply` すると、現在Actionsを実行するリポジトリではなく旧リポジトリへ権限を
  与える設定が作られる。

親側は owner-qualified な契約rootを
`.ai/contracts/templates/yukihide-mitsuoka/terraform-gcp-template/` から
`.ai/contracts/templates/ea-mitsuoka/terraform-gcp-template/` へrenameする。本リポジトリは
このpathを `inherited_paths` と agent profileで宣言しており、`scripts/template_inheritance.py`
は `template` inputのpathを `.ai/contracts/templates/<owner>/<repository>/`（小文字）から
導出して検証する。したがってpath名は命名の好みではなく契約の一部であり、追随は必須である。

ADR-0008は `Yukihide-Mitsuoka/terraform-gcp-template` を唯一の直接親と規定している。受理済み
ADRは編集しないため、supersedeする決定が必要である。

## Options considered

### Option 1: 何もしない

すべてのowner-qualified参照を旧アカウントのまま残す。即座に壊れるものはなく、記録された親も
存在はし続ける。

しかし `make doctor` は失敗したまま、reviewの宛先は非所有者のまま、そして本リポジトリは維持
されている親から無期限にdriftする。親のrenameが取り込まれた時点で、宣言したpathと実体が
食い違い、`template_inheritance.py validate` が失敗する。

### Option 2: 直接親の参照だけを付け替える

`.github/inheritance/` 3ファイルとTemplate Sync sourceだけを変更し、契約rootのrenameと自身の
identityは据え置く。

差分は小さいが、親がrenameを取り込んだ瞬間に `inherited_paths` が親の公開しないpathを指す
ことになり、継承が空振りする。README ownershipもCODEOWNERSもWIF defaultも直らない。

### Option 3: 親・契約root・自身のidentityをまとめて付け替える

上記に加えて契約rootをrenameし（親変更のmanual port, ADR-0015）、CODEOWNERS、README ownership
marker、repository facts overlay、PR size policyのoperand、runbookの実行コマンド、`docs/usage.md`
の手順、`github_repository` default、そしてこれらを固定するテストを更新する。

差分は大きくなるが、中途半端な状態を残さない。

## Decision

Option 3を採用する。

本リポジトリは `ea-Mitsuoka/terraform-gcp-template` を唯一の直接親としてMUSTで宣言し、
ADR-0008の該当条項をsupersedeする。ADR-0008のそれ以外の制約（reviewed PRごとに
first-parent commitを1つずつ進める、子所有pathを保護する、Terraform family profileは
terraform-gcp-template経由でのみ受け取る）はすべて維持する。

継承contract rootは `.ai/contracts/templates/ea-mitsuoka/terraform-gcp-template/` をMUSTとする。
これは親PRのmanual portであり、ADR-0015の「同一reviewed PRで受理する」境界に従う。

受理済みlock commitは前進させない。`2099849c5deb3bc2ca843884f85ccc060b2ec608` は
`ea-Mitsuoka/terraform-gcp-template` に存在するため、本決定は継承状態を進めず、blobの再検証も
行わない。

`infra/envs/dev/variables.tf` の `github_repository` defaultは
`ea-Mitsuoka/secure-ga4-bq-template` へ変更する。これはコード上の既定値の訂正であり、
`terraform apply` を実行するまでcloud側の状態は変わらない。適用前の現行bindingは旧リポジトリ名
のままである点に注意する。

本決定の実装はGR-020のhard limit（20 files）に収めるため2つのreviewed PRに分割する。継承契約
そのもの（`.github/inheritance/`、contract rootのrename、Template Sync source、README ownership
marker、repository facts overlay、PR size policyのoperandと対応テスト）を先行PRとし、継承契約の
外にある運用上のidentity（`.github/CODEOWNERS` のreview routing、`docs/runbook/` の実行コマンド、
`github_repository` default）を後続PRとする。分割は決定内容を変えず、どちらのPRも単独で
`make doctor` / `make lint` / `make test` がgreenであることを条件とする。

履歴を記録するowner-qualified参照はMUST NOTで書き換えない。`CHANGELOG.md`、既存の
`.ai/decision-log.md` 行、受理済みADR本文、`docs/verification/`、`docs/roadmap.md`、
`docs/development-handoff.md` のissue・PR・Actions runリンクは、それらが実在したアカウントを
保持する。

reusable workflowとmoduleのsourceは本決定の対象外とする。`bq-cost-gate.yml` と
`bq-inspect.yml` の `uses:` が指す `Yukihide-Mitsuoka/gcp-cicd-workflows`、それと一致していなけ
ればならない `cost_gate_variables.tf` の `cost_gate_workflow_ref` default、および
`Yukihide-Mitsuoka/terraform-gcp-modules` のmodule sourceは、tag/SHA固定のartifact参照であって
継承のedgeではない。付け替えには、固定したtagが新アカウントに存在することの事前検証が必要で
あり、片側だけ動かすとWIFのjob_workflow_ref照合または `terraform init` が壊れる。

## Consequences

**Positive:**

- scheduled Template Syncが、所有者の維持する親を読む。
- `make doctor` が通る（`readme ownership: OK: ea-Mitsuoka/secure-ga4-bq-template`）。
- 親のrenameが到着したとき、宣言pathと実体が一致しているため継承が空振りしない。
- CODEOWNERSが解決し、required reviewerが機能する。
- 既定値のまま `terraform apply` しても、旧リポジトリへ権限を与える設定を作らない。

**Negative:**

- 親PRがmergeされる前にこのPRを取り込むと、宣言したcontract rootと親の公開pathが一時的に
  食い違う。したがってmerge順は親を先にする。
- リポジトリ内に新旧2つの表記が混在し、読み手は現在の参照と履歴リンクを区別する必要がある。
- `README.md` は移行済みの継承参照と未移行のartifact参照を同じ表に並べることになる。
- `github_repository` defaultの変更は `terraform apply` まで無効であり、live WIF bindingは
  依然として旧リポジトリ名を指す。移行後のActions OIDCでSA impersonationを使う前に、
  apply（または該当bindingの手動更新）が必要である。
- rollbackはこのPRのreviewed revertであり、親をrevertする場合は子を先に戻す。

**Follow-ups:**

- `terraform-gcp-modules` と `gcp-cicd-workflows` のtagが新アカウントに存在することを検証し、
  `uses:`、`cost_gate_workflow_ref`、module source、README表を1つのPRでまとめて付け替える。
- WIF bindingを再applyする、あるいは適用不要であることを確認して記録する。
- `docs/development-handoff.md` のsnapshotは2026-08-10時点のままであり、別途更新する。
