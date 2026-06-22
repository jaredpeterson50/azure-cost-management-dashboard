# Deploy With GitHub Actions

This is the cleaner repeatable path after the first local build works. It uses GitHub Actions plus Azure OpenID Connect (OIDC), so you do not store an Azure client secret in GitHub.

This workflow uses repository-level secrets and repository-level variables. It does not use a GitHub Actions environment.

References:

- Azure Login recommends OIDC authentication and requires `permissions: id-token: write` in the workflow.
- Microsoft documents using Azure Login with OIDC for GitHub Actions.

## One-Time Azure Setup

Run these commands locally from PowerShell. Replace placeholders before running.

```powershell
$subscriptionId = "<your-subscription-id>"
$appName = "github-azure-billing-dashboard"
$githubOrgOrUser = "<your-github-user>"
$githubRepo = "azure-billing-dashboard"

az login
az account set --subscription $subscriptionId

$app = az ad app create --display-name $appName | ConvertFrom-Json
$clientId = $app.appId

az ad sp create --id $clientId | Out-Null

az role assignment create `
  --assignee $clientId `
  --role Contributor `
  --scope "/subscriptions/$subscriptionId"

$credentialPath = "$env:TEMP\github-main-federated-credential.json"

@{
  name = "github-main"
  issuer = "https://token.actions.githubusercontent.com"
  subject = "repo:$githubOrgOrUser/$githubRepo:ref:refs/heads/main"
  audiences = @("api://AzureADTokenExchange")
} | ConvertTo-Json | Set-Content -LiteralPath $credentialPath -NoNewline

az ad app federated-credential create `
  --id $clientId `
  --parameters "@$credentialPath"

az ad app show --id $clientId --query "{clientId:appId}" --output json
az account show --query "{subscriptionId:id, tenantId:tenantId}" --output json
```

Use the final `clientId`, `tenantId`, and `subscriptionId` values in GitHub.

## GitHub Repository Settings

Create these repository secrets:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`
- `BUDGET_CONTACT_EMAIL`

Create these repository variables:

- `AZURE_RESOURCE_GROUP`, for example `rg-azure-billing-dashboard`
- `AZURE_STORAGE_ACCOUNT`, globally unique lowercase storage name
- `BUDGET_START_DATE`, for example `2026-06-01T00:00:00Z`
- `BUDGET_END_DATE`, for example `2036-06-30T00:00:00Z`
- `MONTHLY_BUDGET_AMOUNT`, for example `2.6`

Because the workflow does not set `environment:`, only repository-level variables and secrets are required. If you later add a GitHub environment for approvals, you must also add a matching Azure federated credential subject such as `repo:<owner>/<repo>:environment:<environment-name>`.

## Run the Deployment

1. Push to `main`.
2. Open the GitHub repo.
3. Go to **Actions**.
4. Select **Deploy to Azure**.
5. Click **Run workflow**.

The workflow will:

- authenticate to Azure with OIDC
- run Terraform
- export the resource group and storage account from Terraform outputs
- refresh Cost Management data, with fallback static data if Cost Management is unsupported
- build the React app
- upload `dist` to Azure Storage static website hosting

## Enterprise Notes

For a production team, improve this further with:

- Terraform remote state in Azure Storage
- separate `dev`, `test`, and `prod` environments
- required reviewers before applying Terraform
- Checkov or Trivy IaC scanning
- Gitleaks secret scanning
- least-privilege custom Azure roles instead of broad `Contributor`