# Sync-OIDC-App — Terraform

Azure AD / Entra ID app registration for OIDC/MFA authentication, managed per environment (`dev` / `qa` / `prod`) via a single reusable module.

## File map

| File | Purpose |
|---|---|
| `oidc_app.tf` | Calls `module "oidc_app"` (app registration: display name, sign-in audience, redirect URIs, Graph scopes, `group_membership_claims`). Also defines the Enterprise App (`azuread_service_principal`) and the two `azuread_app_role_assignment` resources for user/group access. |
| `custom_attributes.tf` | Owns the `OIPAClientID` directory extension attribute and the optional claims (`extn.OIPAClientID`) on ID/Access tokens. **Sole owner** of `optional_claims` — do not duplicate elsewhere. |
| `variables.tf` | Declares `oidc_redirect_uris` (map keyed by env), `environment_name` (validated), `assigned_user_object_ids`, `assigned_group_object_ids`. |
| `modules/app_registrations/main.tf` | Reusable module — creates `azuread_application`. Shared across all envs/apps; no env-specific values here. |
| `modules/app_registrations/outputs.tf` | Exposes `client_id` and `object_id` for root files to reference. |
| `import.tf` | Temporary — only used to import pre-existing (manually created) resources into state. Delete after each import is applied. |

## Adding / changing redirect URIs per environment

Set in `variables.tf`:

```hcl
variable "oidc_redirect_uris" {
  type = map(list(string))
  default = {
    dev  = ["https://oidcdebugger.com/debug", "https://localhost:8001/federation/NGL_AzureAD/signin"]
    qa   = ["https://qa.example.com/auth/callback"]
    prod = ["https://example.com/auth/callback"]
  }
}
```

`oidc_app.tf` resolves the right list automatically:

```hcl
web_redirect_uris = var.oidc_redirect_uris[var.environment_name]
```

**To add a URL:** edit the list for that environment in `variables.tf` → `terraform plan` (expect `1 to change`, redirect URIs only) → `apply`.

> `environment_name` must exactly match a map key (`dev`/`qa`/`prod`, case-sensitive) or the plan fails immediately rather than silently resolving the wrong list.

## Adding a user or group

Managed by **Object ID** (not UPN/display name — avoids breakage on rename, and ambiguity when names collide):

```hcl
variable "assigned_user_object_ids" {
  type    = list(string)
  default = ["a8a73a79-4320-44b6-bfc5-d321f2193d89"]
}

variable "assigned_group_object_ids" {
  type    = list(string)
  default = ["a9217f84-7fd7-4133-83eb-9f6c99c69aca"]
}
```

Each entry becomes its own `azuread_app_role_assignment` via `for_each` — adding/removing one doesn't touch the others.

**To add a user:**
1. Get the Object ID: `az ad user show --id user@domain.com --query id -o tsv`
2. Add the GUID to `assigned_user_object_ids`.
3. `terraform plan` (expect `1 to add`) → `apply`.

**To add a group:**
1. Get the Object ID: `az ad group show --group "Group Name" --query id -o tsv`
2. Add the GUID to `assigned_group_object_ids`.
3. `terraform plan` → `apply`.

**To remove access:** delete the GUID from the list — Terraform destroys only that specific assignment.

> If the user/group was ever added manually in the portal first, `apply` fails with `EntitlementGrant entry already exists`. Import it — see below.

## Creating a brand-new app registration

The module is reusable — add a **new** `module` block, don't edit the existing `oidc_app` one:

```hcl
module "new_app" {
  source           = "../modules/app_registrations"
  display_name     = "<New-App-Name>-${var.environment_name}"
  sign_in_audience = "AzureADMyOrg"
  web_redirect_uris = var.new_app_redirect_uris[var.environment_name]
  required_resource_accesses = [ /* Graph scopes */ ]
  group_membership_claims = ["ApplicationGroup"]  # only if needed
}
```

1. Declare a matching `map(list(string))` redirect URI variable, same pattern as `oidc_redirect_uris`.
2. If it needs a custom extension claim, mirror the `null_resource` + `azuread_application_optional_claims` pattern from `custom_attributes.tf`, pointed at `module.new_app`.
3. If it needs its own user/group assignments, add its own `azuread_service_principal` + `azuread_app_role_assignment` resources.
4. `terraform plan` should show only additive changes — `0` diff on existing `Sync-OIDC-App` resources.

> Don't add app-specific config into `modules/app_registrations/main.tf` — that module is shared infrastructure.

## Importing pre-existing resources

If something was created manually in the portal before Terraform managed it, `apply` errors with "already exists." Add an `import` block to `import.tf`, `plan`/`apply`, then delete the file.

**Service Principal:**
```hcl
import {
  to = azuread_service_principal.oidc_app_sp
  id = "<service-principal-object-id>"
}
```

**App Role Assignment** (composite ID — get the real assignment ID from Graph first):
```bash
az rest --method GET \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/<sp-object-id>/appRoleAssignedTo" \
  --query "value[].{principalId:principalId, assignmentId:id}" -o table
```
```hcl
import {
  to = azuread_app_role_assignment.user_assignment["<user-object-id>"]
  id = "<sp-object-id>/appRoleAssignment/<assignment-id>"
}
```

**Optional Claims:**
```hcl
import {
  to = azuread_application_optional_claims.oidc_app_claims
  id = "/applications/<application-object-id>"
}
```

## Optional claims & groups claim — ownership rules

- **`extn.OIPAClientID`** → owned exclusively by `custom_attributes.tf`'s `azuread_application_optional_claims.oidc_app_claims`. Never re-declare `optional_claims` in the module call — two resources managing the same property will overwrite each other every apply, and the claim silently vanishes from the portal.
- **Groups claim** (`group_membership_claims = ["ApplicationGroup"]`) → set directly in the `module "oidc_app"` block in `oidc_app.tf`. Top-level `azuread_application` attribute, unrelated to `optional_claims`, required a one-line addition inside the module itself.

## Troubleshooting

| Error | Fix |
|---|---|
| `A resource with the ID "..." already exists` | Import it — see above. |
| `EntitlementGrant entry already exists` | User/group was assigned manually in the portal. Import the `app_role_assignment`. |
| `expected "..." to be a valid UUID` | A GUID was mistyped/truncated. Re-copy from the portal or CLI, don't retype from a screenshot. |
| `Reference to undeclared resource (data.azuread_user / data.azuread_group)` | Leftover from the old UPN/display-name lookup approach. Delete those `data` blocks — assignments use object IDs directly now. |
| `Invalid default value for variable` / type mismatch | Check for a duplicate `variable` declaration with a conflicting type: `grep -rn 'variable "<name>"' .` |
| `extn.OIPAClientID` disappears from the portal after apply | Two resources both managing `optional_claims`. Confirm `oidc_app.tf`'s module call has no `optional_claims` block. |
