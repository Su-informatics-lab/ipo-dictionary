# GitHub Actions schema publishing

The `publish-schema.yml` workflow publishes the release `schema.json` artifact to the IPO dictionary bucket:

```text
s3://ipo-dictionary/<version>/schema.json
```

For example, release tag `1.0.2` is published to `s3://ipo-dictionary/1.0.2/schema.json`.

The workflow runs when a GitHub release is published. It also supports manual dispatch from `main` so an existing release can be backfilled or retried without deleting and recreating the release.

## AWS identity setup

The workflow uses GitHub Actions OpenID Connect (OIDC) to assume an AWS role. Do not create long-lived AWS access keys for this workflow.

Run these commands from PowerShell with access to the IPO AWS account:

```powershell
$Profile = "gaipo-tf"
$AccountId = "066964538924"
$Repo = "Su-informatics-lab/ipo-dictionary"
$Bucket = "ipo-dictionary"
$Region = "us-east-1"
$RoleName = "ipo-dictionary-github-actions-publish-schema"
$PolicyName = "ipo-dictionary-publish-schema"
$OidcArn = "arn:aws:iam::$AccountId:oidc-provider/token.actions.githubusercontent.com"
$RoleArn = "arn:aws:iam::$AccountId:role/$RoleName"

aws sts get-caller-identity --profile $Profile
aws s3api head-bucket --bucket $Bucket --profile $Profile
aws s3api get-bucket-location --bucket $Bucket --profile $Profile
```

Create the GitHub OIDC provider if the IPO account does not already have one:

```powershell
$Providers = aws iam list-open-id-connect-providers --profile $Profile | ConvertFrom-Json
if ($Providers.OpenIDConnectProviderList.Arn -notcontains $OidcArn) {
  aws iam create-open-id-connect-provider `
    --url https://token.actions.githubusercontent.com `
    --client-id-list sts.amazonaws.com `
    --profile $Profile
}
```

Create or update the role trust policy:

```powershell
$TrustPolicyPath = Join-Path $env:TEMP "ipo-dictionary-github-trust-policy.json"
@'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::066964538924:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:Su-informatics-lab/ipo-dictionary:ref:refs/tags/*",
            "repo:Su-informatics-lab/ipo-dictionary:ref:refs/heads/main"
          ]
        }
      }
    }
  ]
}
'@ | Set-Content -Path $TrustPolicyPath -Encoding utf8

aws iam get-role --role-name $RoleName --profile $Profile 2>$null
if ($LASTEXITCODE -eq 0) {
  aws iam update-assume-role-policy `
    --role-name $RoleName `
    --policy-document "file://$TrustPolicyPath" `
    --profile $Profile
} else {
  aws iam create-role `
    --role-name $RoleName `
    --description "Publish IPO dictionary schema releases to S3 from GitHub Actions." `
    --assume-role-policy-document "file://$TrustPolicyPath" `
    --profile $Profile
}
```

Create or update the inline permissions policy:

```powershell
$PermissionPolicyPath = Join-Path $env:TEMP "ipo-dictionary-github-permissions-policy.json"
@'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::ipo-dictionary"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::ipo-dictionary",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "*/schema.json"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::ipo-dictionary/*/schema.json"
    }
  ]
}
'@ | Set-Content -Path $PermissionPolicyPath -Encoding utf8

aws iam put-role-policy `
  --role-name $RoleName `
  --policy-name $PolicyName `
  --policy-document "file://$PermissionPolicyPath" `
  --profile $Profile
```

## GitHub setup

Set repository variables for the workflow:

```powershell
$Repo = "Su-informatics-lab/ipo-dictionary"
$RoleArn = "arn:aws:iam::066964538924:role/ipo-dictionary-github-actions-publish-schema"

gh variable set AWS_ROLE_ARN --repo $Repo --body $RoleArn
gh variable set AWS_REGION --repo $Repo --body us-east-1
gh variable set AWS_S3_BUCKET --repo $Repo --body ipo-dictionary
```

For this change, create and merge the pull request from the prepared branch:

```powershell
$Repo = "Su-informatics-lab/ipo-dictionary"
$Branch = "codex/publish-schema-to-s3"
$PrBodyPath = Join-Path $env:TEMP "ipo-dictionary-publish-schema-pr.md"

@'
## Summary
- Add a release/manual GitHub Actions workflow that publishes `schema.json` to S3.
- Document the GitHub OIDC AWS role setup, repository variables, and 1.0.2 backfill validation steps.

## Validation
- `yamllint .github\workflows\publish-schema.yml`
- `git diff --check`
- AWS OIDC provider, IAM role, and inline S3 policy created and read back in account `066964538924` with profile `gaipo-tf`.
'@ | Set-Content -Path $PrBodyPath -Encoding utf8

gh pr create `
  --repo $Repo `
  --base main `
  --head $Branch `
  --title "Add schema publishing workflow" `
  --body-file $PrBodyPath

Remove-Item $PrBodyPath -Force

gh pr merge --repo $Repo $Branch --squash --delete-branch
```

The workflow requires these permissions in `.github/workflows/publish-schema.yml`:

```yaml
permissions:
  contents: read
  id-token: write
```

`id-token: write` lets GitHub request an OIDC token for AWS. `contents: read` lets the workflow check out the release tag.

## Backfill or retry a release

After this workflow is on `main`, manually publish an existing release tag:

```powershell
$Repo = "Su-informatics-lab/ipo-dictionary"

gh workflow run publish-schema.yml --repo $Repo --ref main -f version=1.0.2
$RunId = gh run list --repo $Repo --workflow publish-schema.yml --limit 1 --json databaseId --jq ".[0].databaseId"
gh run watch $RunId --repo $Repo
```

Validate the object in S3:

```powershell
aws s3api head-object --bucket ipo-dictionary --key 1.0.2/schema.json --profile gaipo-tf

$DownloadPath = Join-Path $env:TEMP "ipo-dictionary-schema-1.0.2.json"
aws s3 cp s3://ipo-dictionary/1.0.2/schema.json $DownloadPath --profile gaipo-tf
$Schema = Get-Content $DownloadPath -Raw | ConvertFrom-Json
$Schema."_settings.yaml"._dict_version
```
