> [!CAUTION]
> **This file is a working draft and will be removed before the PR is submitted.**
> Branch: https://github.com/straubt1/terraform-provider-vault/tree/fix/aws-auth-sts-region-from-client

---

### Description

This PR fixes a compatibility break between the `auth_login_aws` provider block and Vault servers that have `use_sts_region_from_client = true` set on their AWS auth backend client config.

**Symptom**

When `use_sts_region_from_client = true` is enabled on the Vault AWS auth backend, authentication fails with:

```
Error: error authenticating to vault: error logging in to AWS auth method: Error making API request.
...
* error logging in: POST https://vault.example.com/v1/auth/aws/login
  Code: 400. Errors:
  * region missing from Authorization header
```

**Root cause**

Provider 5.7.0 changed `generateLoginData` to use `sts.PresignGetCallerIdentity`, which produces a signed **GET** request. In a presigned GET request the signature lives entirely in URL query parameters (`X-Amz-Signature=...`) — there is no `Authorization` header.

Vault's `use_sts_region_from_client` feature calls `awsRegionFromHeader(req.Header.Get("Authorization"))` on the incoming login request to extract the signing region. When the provider sends a presigned GET, `Authorization` is empty and Vault returns the error above.

Note: this setting is documented in Vault as an advanced option to allow Vault to use regional STS endpoints for validation. Operators who have turned it on to satisfy network/firewall policies (e.g. blocking `sts.amazonaws.com`) are fully broken on provider 5.7.0+.

**Fix**

Switch `generateLoginData` back to a manually SigV4-signed **POST** request (the pre-5.7.0 approach). A SigV4-signed POST includes:

```
Authorization: AWS4-HMAC-SHA256 Credential=AKID/YYYYMMDD/<region>/sts/aws4_request, ...
```

Vault's `awsRegionFromHeader` can parse the region from this header, so authentication succeeds regardless of the `use_sts_region_from_client` setting.

The `aws_sts_endpoint` field continues to work as before — it is used as the POST target URL when provided, otherwise the endpoint is derived from the signing region.

**Changes**

- `internal/provider/auth_aws.go`: Replace `PresignGetCallerIdentity` with a manually SigV4-signed POST in `generateLoginData`. Add `stsEndpoint string` parameter so the caller can pass through `aws_sts_endpoint`.
- `internal/provider/auth_aws_test.go`: Add `TestGenerateLoginData_SigV4POSTProducesAuthorizationHeader` and `TestGenerateLoginData_BodyAndServerIDHeader` unit tests.

**CHANGELOG entry** (to be added under `Unreleased` > `BUG FIXES` before submission):

```
* `provider/auth_login_aws`: Fix compatibility with Vault AWS auth backends configured with `use_sts_region_from_client = true`. The provider now sends a SigV4-signed POST to STS GetCallerIdentity, which includes an Authorization header that Vault can extract the signing region from.
```

Relates OR Closes #2766


### Reproduction steps

#### Pre-requisites

- Docker (for a local Vault dev server)
- Terraform ≥ 1.0
- AWS account with:
  - An IAM user with `sts:AssumeRole` permission on a target IAM role
  - Access key and secret for that user
  - An IAM role that trust the user (the "assumable" role)
- Vault server with `use_sts_region_from_client = true` on the AWS auth backend

#### Reproduce the error (provider 5.7.0)

**1. Start Vault dev mode**

```bash
docker run --rm -d \
  --name vault-dev \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  --cap-add IPC_LOCK \
  hashicorp/vault:latest
```

**2. Configure Vault**

```hcl
# vault-config/main.tf
resource "vault_auth_backend" "aws" {
  type = "aws"
  path = "aws"
}

resource "vault_aws_auth_backend_client" "config" {
  backend                    = vault_auth_backend.aws.path
  use_sts_region_from_client = true   # <-- this triggers the bug
}

resource "vault_aws_auth_backend_role" "vault_auth" {
  backend                  = vault_auth_backend.aws.path
  role                     = "vault-auth-role"
  auth_type                = "iam"
  bound_iam_principal_arns = ["arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"]
  resolve_aws_unique_ids   = false
  token_policies           = [vault_policy.read_secrets.name]
}
```

**3. Configure the provider**

```hcl
# test/provider.tf
provider "vault" {
  address          = "http://localhost:8200"
  skip_child_token = true

  auth_login_aws {
    mount                 = "aws"
    role                  = "vault-auth-role"
    aws_access_key_id     = var.aws_access_key_id
    aws_secret_access_key = var.aws_secret_access_key
    aws_role_arn          = var.aws_role_arn
    aws_region            = "us-east-1"
  }
}
```

**4. Run and observe the error**

```
$ terraform plan
│ Error: error authenticating to vault: error logging in to AWS auth method:
│   * region missing from Authorization header
```

#### Verify the fix

Build and reference the patched provider, then re-run `terraform plan` — authentication succeeds.

```bash
# build
cd /path/to/terraform-provider-vault
go build -o terraform-provider-vault .

# reference in ~/.terraformrc
provider_installation {
  dev_overrides {
    "hashicorp/vault" = "/path/to/terraform-provider-vault"
  }
  direct {}
}

# confirm fix
terraform plan   # no auth error
```


### Checklist

- [ ] Added [CHANGELOG](https://github.com/hashicorp/terraform-provider-vault/blob/master/CHANGELOG.md) entry (only for user-facing changes)
- [ ] Acceptance tests where run against all supported Vault Versions


### Output from acceptance testing

The existing unit tests pass without modification. Two new unit tests cover the fixed behavior directly:

```
$ go test ./internal/provider/ -run "TestAuthLoginAWS|TestGenerateLoginData" -v

=== RUN   TestAuthLoginAWS_Init
--- PASS: TestAuthLoginAWS_Init (0.00s)
=== RUN   TestAuthLoginAWS_LoginPath
--- PASS: TestAuthLoginAWS_LoginPath (0.00s)
=== RUN   TestAuthLoginAWS_getCredentialsConfig
--- PASS: TestAuthLoginAWS_getCredentialsConfig (0.00s)
=== RUN   TestAuthLoginAWS_RoleAssumption
--- PASS: TestAuthLoginAWS_RoleAssumption (0.00s)
=== RUN   TestAuthLoginAWS_SessionTokenWithRoleARN
--- PASS: TestAuthLoginAWS_SessionTokenWithRoleARN (0.00s)
=== RUN   TestAuthLoginAWS_CustomEndpoints
--- PASS: TestAuthLoginAWS_CustomEndpoints (0.00s)
=== RUN   TestAuthLoginAWS_Login
--- PASS: TestAuthLoginAWS_Login (0.00s)
=== RUN   TestGenerateLoginData_SigV4POSTProducesAuthorizationHeader
    --- PASS: .../explicit-region (0.00s)
    --- PASS: .../empty-region-falls-back-to-us-east-1 (0.00s)
    --- PASS: .../custom-sts-endpoint-with-explicit-region (0.00s)
--- PASS: TestGenerateLoginData_SigV4POSTProducesAuthorizationHeader (0.00s)
=== RUN   TestGenerateLoginData_BodyAndServerIDHeader
    --- PASS: .../body-is-sts-action (0.00s)
    --- PASS: .../server-id-header-present-when-set (0.00s)
    --- PASS: .../server-id-header-absent-when-not-set (0.00s)
--- PASS: TestGenerateLoginData_BodyAndServerIDHeader (0.00s)
PASS
ok  	github.com/hashicorp/terraform-provider-vault/internal/provider	0.63s
```

End-to-end testing was performed against a local Vault dev server (v1.18) with `use_sts_region_from_client = true` and a real AWS assumed-role credential flow. Authentication succeeds and a KV v2 secret is successfully read.


### Community Note

* Please vote on this pull request by adding a 👍 [reaction](https://blog.github.com/2016-03-10-add-reactions-to-pull-requests-issues-and-comments/) to the original pull request comment to help the community and maintainers prioritize this request
* Please do not leave "+1" comments, they generate extra noise for pull request followers and do not help prioritize the request


## PCI review checklist

<!-- heimdall_github_prtemplate:grc-pci_dss-2024-01-05 -->

- [x] I have documented a clear reason for, and description of, the change I am making.

- [x] If applicable, I've documented a plan to revert these changes if they require more than reverting the pull request.

  Revert by removing this PR. The presigned GET approach is preserved in git history.

- [x] If applicable, I've documented the impact of any changes to security controls.

  No changes to security controls. The fix switches the signing mechanism for the STS GetCallerIdentity call from presigned URL (GET) to a standard SigV4-signed POST. Both mechanisms are established AWS authentication patterns. Credentials are never logged or stored beyond the in-memory request lifecycle.
