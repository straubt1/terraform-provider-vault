---
layout: "vault"
page_title: "Vault: vault_database_secret_backend_credentials resource"
sidebar_current: "docs-vault-resource-database-secret-backend-credentials"
description: |-
  Obtains credentials for a database from a database secret backend in Vault.
---

# vault\_database\_secret\_backend\_credentials

Obtains credentials for a database using Vault's database secret backends.

~> **Important** All data provided in the resource configuration will be
written in cleartext to state and plan files generated by Terraform, and
will appear in the console output when Terraform runs. Protect these
artifacts accordingly. See
[the main provider documentation](../index.html)
for more details.

## Example Usage

```hcl
resource "vault_mount" "db" {
  path = "postgres"
  type = "database"
}

resource "vault_database_secret_backend_connection" "postgres" {
  backend       = "${vault_mount.db.path}"
  name          = "postgres"
  allowed_roles = ["my-role"]

  postgresql {
    role_url = "postgres://username:password@host:port/database"
  }
}

resource "vault_database_secret_backend_role" "role" {
  backend             = "${vault_mount.db.path}"
  name                = "my-role"
  db_name             = "${vault_database_secret_backend_connection.postgres.name}"
  creation_statements = "CREATE ROLE {{name}} WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';"
}

resource "vault_database_secret_backend_credentials" "creds" {
  backend = "${vault_database_secret_backend_role.role.backend}"
  name    = "${vault_database_secret_backend_role.role.name}"
}
```

## Argument Reference

The following arguments are supported:

* `name` - (Required) The name of the role to generate credentials with.

* `backend` - (Required) The unique name of the Vault mount the role exists on.

## Attributes Reference

In addition to the above arguments, the following attributes are available for
reference:

* `username` - The username of the generated credentials.

* `password` - The password of the generated credentials.

* `lease_renewable` - Whether these credentials can be automatically renewed or
  not.

* `lease_duration` - How long these credentials last.

* `lease_started` - The RFC 3339 timestamp these credentials were created, as
  reported by the machine running Terraform.