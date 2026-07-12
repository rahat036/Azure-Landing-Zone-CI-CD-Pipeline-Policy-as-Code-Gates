# Azure-Landing-Zone-CI-CD-Pipeline-Policy-as-Code-Gates
A reusable **GitHub Actions** pipeline template that enforces security, governance,
and cost guardrails on Azure infrastructure (Bicep or Terraform) **before** it
ever reaches Azure — not after.

Most teams find out about a misconfigured storage account or an ungoverned
public IP *after* deployment, via a portal alert or a compliance audit. This
pipeline shifts that check left: every pull request is linted, security-scanned,
policy-validated, and cost-estimated automatically, with a human approval gate
before anything touches a real subscription.

> Built by [Rezwanur](https://github.com/rahat036) — Cloud Solution Architect,
> Microsoft MVP, and author of *Pro Azure Governance and Security* (Apress).
> This repo is a practical companion to the governance patterns in that book.



## What this pipeline does

| Stage | Tool | Purpose |
|---|---|---|
| 1. Lint & format | Bicep linter / `terraform fmt` + `validate` | Catch syntax and style issues immediately |
| 2. Security scan | [Checkov](https://www.checkov.io/) | Fail the build on high/critical misconfigurations |
| 3. Policy compliance | [PSRule for Azure](https://azure.github.io/PSRule.Rules.Azure/) | Enforce naming, tagging, network isolation rules |
| 4. Cost estimate | [Infracost](https://www.infracost.io/) | Post a cost delta as a PR comment |
| 5. Plan preview | `terraform plan` / `az deployment ... what-if` | Reviewers see exactly what will change |
| 6. Deploy | GitHub Actions + OIDC federation | No stored secrets, environment-gated (dev → test → prod) |
| 7. Drift detection | Scheduled workflow | Nightly re-plan; opens a GitHub Issue if drift is found |

Everything is designed to be **forked and reused** — swap in your own Bicep
modules or Terraform config under `infra/` and the workflows pick them up.

---

## Repository structure

```
azure-landing-zone-cicd/
├── .github/workflows/
│   ├── pr-validation.yml      # Lint, scan, policy check, cost estimate, plan preview
│   ├── deploy.yml             # OIDC-authenticated deploy with environment approvals
│   └── drift-detection.yml    # Scheduled drift check → auto-files an issue
├── infra/
│   ├── bicep/                 # Sample Landing Zone modules (network, storage, Key Vault)
│   └── terraform/             # Equivalent Terraform implementation
├── policy/ps-rule/            # PSRule for Azure baseline + custom governance rules
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SETUP.md
│   └── CONTRIBUTING.md
├── examples/sample-workload/  # A minimal workload to test the pipeline against
└── LICENSE
```

---

## Quick start

1. **Fork this repo.**
2. Create an Entra ID App Registration with **federated credentials** (OIDC) —
   no client secrets required. Steps in [`docs/SETUP.md`](docs/SETUP.md).
3. Add these repository secrets/variables:
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID`
   - `AZURE_SUBSCRIPTION_ID`
   - `INFRACOST_API_KEY` *(optional, free tier available)*
4. Replace the contents of `infra/bicep/` or `infra/terraform/` with your own
   modules — or keep the samples to test the pipeline end-to-end.
5. Open a pull request. Watch the checks run.
6. Merge to `main` → deploys to **dev** automatically. **test** and **prod**
   require manual approval via [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment).

Full walkthrough: [`docs/SETUP.md`](docs/SETUP.md).

---

## Why policy-as-code instead of Azure Policy alone?

Azure Policy is excellent, but it evaluates resources *after* deployment
(or blocks at deployment time, with feedback buried in an ARM error).
PSRule and Checkov run against the *template source* in seconds, inside the
PR, with human-readable output — so a contributor sees "storage account
`stlogs001` has no `Environment` tag" as a PR comment, not a deployment failure
three minutes into a pipeline run.

Both approaches are complementary. This template assumes Azure Policy is
still enforced at the subscription/management-group level as a safety net;
the pipeline is the fast feedback loop.

## Customizing the governance rules

The default ruleset in `policy/ps-rule/ps-rule.yaml` enforces:
- Mandatory tags (`Environment`, `Owner`, `CostCenter`)
- No public network access on Storage Accounts or Key Vaults by default
- Naming convention matching `<resourceType>-<workload>-<env>-<region>`
- TLS 1.2 minimum on any resource exposing an HTTPS endpoint
- No NSG rules allowing inbound `*/*` from `Internet`

Edit `policy/ps-rule/ps-rule.yaml` or add custom rules in
`policy/ps-rule/Rules.Custom.Rule.ps1` to match your organization's standards.

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, adapt it for your own landing zone.

## Contributing

See [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md). Issues and PRs welcome,
especially additional PSRule rule packs or Terraform module examples.

