# PuntoBello SPO Permissions Governance

> **Mirror:** This repository is a mirror of [diemobiliar/puntobello-spopermissionsgov](https://github.com/diemobiliar/puntobello-spopermissionsgov), published under the [MIT License](LICENSE.md). Please open issues and pull requests in the original repository.

PuntoBello SPO Permissions Governance governs **selected-scope API permissions** in SharePoint Online and Microsoft Graph. It covers `Sites.Selected`, `Lists.SelectedOperations.Selected`, `ListItems.SelectedOperations.Selected` and `Files.SelectedOperations.Selected`.

The solution has three parts:

- A **SharePoint site** where app owners request access and site owners approve or reject it.
- A scheduled **Azure Container Apps Job** that validates requests, grants and revokes permissions, sends notifications and runs a yearly recertification.
- An **Azure Developer CLI (azd)** setup, so a single `azd up` deploys everything.

## Contents

- [Why](#why)
- [How it works](#how-it-works)
- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Usage](#usage)
- [Operations](#operations)
- [Security considerations](#security-considerations)
- [Customisation](#customisation)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Known limitations](#known-limitations)

## Why

With the `*.Selected` permissions, an application gets access only to the sites, lists, list items or files it was explicitly granted, not to the whole tenant. Declaring the permission on the app registration is not enough, though: someone still has to grant access per resource, decide who may approve it, and remove access that is no longer needed.

This solution makes that process self-service and auditable:

- **Self-service requests**: app owners request access in a SharePoint list. No admin ticket is needed.
- **Validation**: every request is checked automatically. The checks cover the app, its owner, the declared permission, and the target site and resource.
- **Owner approval**: the owners of the target site, Team or private channel approve or reject the request.
- **Yearly recertification**: owners must reconfirm every granted permission once a year. Unanswered requests are rejected automatically.
- **Automatic revocation**: when a permission is rejected or expires, it is revoked.
- **Detection of unmanaged permissions**: the job reports apps and service principals (for example managed identities) that hold a governed permission without having gone through the process.

## How it works

### Components

```
 App owner ──request──►  SharePoint governance site  ◄──approve / reject── Site / Team owners
                         ├─ Requests list
                         └─ My Recertification list
                                   ▲
                                   │ PnP PowerShell + Microsoft Graph (managed identity)
                                   │
                    Azure Container Apps Job  (weekdays 18:00 UTC)
                    └─ job/Invoke-Recertification.ps1
                                   │
                                   ├─► grants / revokes *.Selected permissions on target sites
                                   └─► sends notification mails (Graph Mail.Send)
```

### Governed permissions

The job governs these application permissions (app roles):

| Permission scope (list value) | API | Grants access to |
|---|---|---|
| `Sites.Selected (Graph)` | Microsoft Graph | a whole site |
| `Sites.Selected (SPO)` | Office 365 SharePoint Online | a whole site |
| `Lists.SelectedOperations.Selected` | Microsoft Graph | a single list or document library |
| `ListItems.SelectedOperations.Selected` | Microsoft Graph | a single list item |
| `Files.SelectedOperations.Selected` | Microsoft Graph | a single file |

### Lifecycle

Each job run goes through three phases.

**1. Process new requests** (Requests list, status `New`)

The request is valid if all of these checks pass:

- the app registration exists in the tenant,
- the requester is an owner of the app registration,
- the app registration declares the requested permission scope as an application permission,
- the target site exists,
- for list, item and file scopes, the target list, item or file exists.

Then:

- **Valid**: the job creates an item in *My Recertification* (status `New`) and gives the target owners edit access and the app owners read access to it. The request is set to `MovedToRecertificationList`.
- **Invalid**: the request is set to `Invalid`, and the requester receives a *ValidationFailed* mail.
- **Check failed with an error** (for example Graph throttling): the request stays `New` and is checked again in the next run.

**2. Process recertification items** (*My Recertification* list)

```
New ──InitialMail──► ApprovalRequested
                        │  owner sets Status
                        ├─ Approved ──► permission granted, ApprovedMail,
                        │               next recertification = today + 1 year
                        │                    │ (after 1 year)
                        │                    └─RecertificationMail──► ApprovalRequested
                        ├─ Rejected ──► permission revoked, RejectedMail
                        └─ no answer:
                             ≥ 14 days ─► ReminderMail
                             ≥ 28 days ─► ReminderMail2
                             ≥ 35 days ─► Rejected automatically (Approval Action By = System)
```

- Approval requests and reminders go to the owners of the target site.
- *ApprovedMail* and *RejectedMail* go to the site owners, with the app owners in CC.
- The *Mail Status* column records the last mail sent. It prevents the same mail from being sent twice.

**3. Detect unmanaged permissions**

The job finds every app that holds a governed permission:

- app registrations that declare one of the governed roles,
- service principals that have one of these roles assigned, for example managed identities or multi-tenant apps.

Each app that has no item in *My Recertification* yet gets a new item with status `Unmanaged`. The governance mailbox receives an *UnmanagedMail* about it once a week. When a managed item exists for the same app ID, the job removes the `Unmanaged` item.

### Who gets which owners

The job finds the owners of the target from the site template:

| Target | Site template | Owners |
|---|---|---|
| Team or group-connected site | `GROUP#0` | Owners of the Microsoft 365 group |
| Teams private channel | `TEAMCHANNEL#*` | Owners of the private channel |
| Communication or classic team site | `SITEPAGEPUBLISHING#0`, `STS#*` | Members of the site's Owners group (Entra ID groups in it are expanded) |

The job does not support other site templates.

## Repository structure

```
.
├── azure.yaml                  # azd project: Terraform infra + preup/postup hooks
├── hooks/
│   ├── preup.ps1               # Provisions the SharePoint site and lists, configures forms and permissions
│   └── postup.ps1              # Grants the managed identity FullControl (Sites.Selected) on the governance site
├── infra/                      # Terraform
│   ├── provider.tf             # Providers, Graph / SharePoint service principal lookups
│   ├── variables.tf            # Input variables
│   ├── main.tfvars.json        # Variable values, mapped from azd environment values
│   ├── main.tf                 # Resource group, Log Analytics
│   ├── main-uami.tf            # User-assigned managed identity and its permissions
│   ├── main-cae.tf             # Container registry, image build, storage, Container Apps environment
│   ├── main-caj.tf             # Container Apps Job (schedule, environment variables)
│   └── output.tf               # Outputs used by the postup hook
├── job/                        # Runs inside the container (uploaded to the file share)
│   ├── Invoke-Recertification.ps1     # Entry point: the three phases described above
│   ├── RecertificationFunctions.psm1  # Validation, owner lookup, grant / revoke, list helpers
│   ├── MailFunctions.psm1             # Mail rendering and sending via Microsoft Graph
│   └── mails/                         # HTML mail templates
├── spo/
│   ├── solutions.json          # Governance site URL, title, templates
│   └── assets/Site.xml         # PnP provisioning template (lists, fields, views, groups, home page)
└── .devcontainer/              # Git submodule: PuntoBello Installer (dev container, Dockerfile, scripts)
```

The image built from `.devcontainer/Dockerfile` serves as both the dev container and the job's runtime image. The scripts in `job/` are not baked into the image. Terraform uploads them to an Azure file share that is mounted at `/mnt/scripts`. To roll out a script change, run `azd up` (or `azd provision`) again.

## Prerequisites

- **Microsoft 365 tenant** with SharePoint Online. You need a **SharePoint Administrator** to create the site and grant site permissions.
- **Azure subscription** and an identity that can:
  - create resource groups and resources, and assign Azure roles (*Owner*, or *Contributor* + *User Access Administrator*),
  - grant Microsoft Graph and SharePoint **application permissions** to a managed identity (*Privileged Role Administrator* or *Global Administrator*).
- **Entra ID app registration for the installer**: the hooks connect to SharePoint with it through PnP PowerShell. It needs the SharePoint application permission `Sites.FullControl.All`, with certificate authentication. See the [PuntoBello Installer](https://github.com/diemobiliar/puntobello-installer).
- **Sender mailbox**: an Exchange Online mailbox (a shared mailbox is fine) to send notifications from. It also receives the unmanaged-app alerts.
- **Docker**: Docker Desktop or another Docker daemon must run on the deployment machine. Terraform builds the container image locally and pushes it to Azure Container Registry.
- **VS Code** with the Dev Containers extension, recommended. Alternatively, install `azd`, Terraform, PowerShell 7 and PnP.PowerShell locally.

## Deployment

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/diemobiliar/puntobello-spopermissionsgov.git
cd puntobello-spopermissionsgov
```

If you already cloned without submodules:

```bash
git submodule update --init
```

### 2. Let the dev container use Docker

`azd up` builds the image from inside the dev container, so the container needs access to the host's Docker daemon. Check `.devcontainer/devcontainer.json` and `.devcontainer/Dockerfile`:

1. `devcontainer.json` must mount the Docker socket:

   ```json
   "mounts": [
     "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
   ]
   ```

2. The `node` user cannot access the socket. Comment out this line in the `Dockerfile`:

   ```dockerfile
   # USER node
   ```

> **Local development only.** Running the dev container as root removes a security boundary. Do not use this change in CI/CD or shared environments, and do not commit it to the installer.

### 3. Open the dev container and sign in

In VS Code, run **Dev Containers: Rebuild and Reopen in Container**. Then sign in inside the container:

```bash
az login
azd auth login
```

### 4. Configure the installer

Edit `.devcontainer/scripts/config.psm1`:

- `$global:M365_TENANTNAME`: your tenant name, for example `contoso` for `contoso.sharepoint.com`.
- `$global:loginSelector`: the login variant, plus the values for that variant (app ID, certificate path and password, …).

Keep certificates in `.secrets/`. This folder is ignored by git. Never commit real credentials, in this repository or in the installer.

### 5. Configure the governance site

`spo/solutions.json` defines the site that hosts the lists. The URL is relative to `https://<tenant>.sharepoint.com/sites/`.

```json
{
  "sites": [
    {
      "Url": "pb_spopergov",
      "Title": "SPO Permissions Governance",
      "LCID": 1033,
      "solutions": [],
      "templates": [
        { "templateName": "Site.xml", "relativePath": "./spo/assets", "sortOrder": 100 }
      ]
    }
  ],
  "termStore": []
}
```

### 6. Create the azd environment

```bash
azd env new <env-name>
azd env set AZURE_LOCATION "switzerlandnorth"
azd env set PB_SENDER_MAIL "spogov@contoso.com"
```

| Value | Set by | Purpose |
|---|---|---|
| `AZURE_ENV_NAME` | `azd env new` | Becomes part of all resource names |
| `AZURE_LOCATION` | you | Azure region |
| `AZURE_SUBSCRIPTION_ID` | azd (prompts you) | Target subscription |
| `PB_SENDER_MAIL` | you | Sender mailbox, and recipient of the unmanaged-app alerts |
| `PB_SITE_URL`, `PB_TENANT_NAME` | preup hook | Governance site URL and tenant name |
| `PB_UAMI_APP_ID`, `PB_UAMI_APP_NAME` | Terraform outputs | Used by the postup hook |

> **Environment name:** use **at most 7 characters, lowercase letters and digits only** (for example `dev` or `prod`). The name becomes part of the storage account name, which allows at most 24 lowercase alphanumeric characters.

### 7. Deploy

```bash
azd up
```

| Phase | What happens |
|---|---|
| **preup hook** | Creates the governance site and applies `Site.xml`. Hides job-managed fields from the forms, shows the target resource fields only for the scopes that need them, gives the `SPOPermissionsGovUsers` group read access to the home page only, removes the Visitors group from *My Recertification*, and stores `PB_SITE_URL` and `PB_TENANT_NAME`. |
| **Terraform** | Creates the resource group, Log Analytics workspace, container registry, storage account and file share, user-assigned managed identity, Container Apps environment and job. Builds and pushes the image, uploads `job/`, and grants the identity its permissions. Then waits 3 minutes for the permissions to propagate. |
| **postup hook** | Grants the managed identity `FullControl` on the governance site through `Sites.Selected`. Running it again does not create a second grant. |

Azure resources created:

| Resource | Purpose |
|---|---|
| Resource group | Contains all resources (tagged with `azd-env-name`) |
| Log Analytics workspace | Job logs (30 days retention) |
| Container registry (Basic) | Job image |
| Storage account + file share | `job/` scripts and mail templates, mounted at `/mnt/scripts` |
| User-assigned managed identity | Identity of the job |
| Container Apps environment + job | Runs `Invoke-Recertification.ps1` on weekdays at 18:00 UTC |

Permissions of the managed identity:

| Permission | API | Used for |
|---|---|---|
| `Sites.FullControl.All` | Microsoft Graph | Granting and revoking `*.Selected` permissions, reading sites, lists, items and files |
| `Sites.FullControl.All` | SharePoint Online | Reading site owners and setting item permissions through PnP |
| `Sites.Selected` | SharePoint Online | Access to the governance site |
| `Application.Read.All` | Microsoft Graph | Reading app registrations, their owners and permission assignments |
| `Group.Read.All` | Microsoft Graph | Reading Microsoft 365 group, Team and channel owners |
| `User.Read.All` | Microsoft Graph | Resolving users |
| `Mail.Send` | Microsoft Graph | Sending notification mails |
| `AcrPull` | Container registry | Pulling the job image |

### 8. Grant users access to the governance site

Add the people who may request permissions, usually app owners and developers, to the **SPOPermissionsGovUsers** SharePoint group. They can add items to the *Requests* list and read the home page. Site owners who approve requests get access to their items in *My Recertification* automatically.

### 9. Verify

Start the job manually instead of waiting for the schedule:

```bash
az containerapp job start --name <job-name> --resource-group <resource-group>
az containerapp job execution list --name <job-name> --resource-group <resource-group> -o table
```

The resource names follow the pattern `pb-spopergov-<random>-<env>-caj` and `pb-spopergov-<random>-<env>-rg`.

## Usage

### App owners: request a permission

1. In the app registration, add the application permission you need (see [Governed permissions](#governed-permissions)) and have an admin grant consent.
2. Open the governance site and create a new item in the **Requests** list:

   | Field | Value |
   |---|---|
   | App-ID | Application (client) ID of the app registration |
   | Target Url | URL of the target site, for example `https://contoso.sharepoint.com/sites/finance` |
   | Permission Scope | The scope the app declares (see the table above) |
   | Permission | `Read` or `Write` |
   | Reason | Why the app needs the access |
   | Target Resource Name | List or library scopes only: display name of the list or document library |
   | Target Resource Id | Item scope: numeric item ID. File scope: file path relative to the library root, for example `Reports/2026/q1.xlsx` |

3. You must be an **owner of the app registration**, otherwise the request is invalid.
4. The next job run validates the request. If it is invalid, you receive a mail. Fix the request and submit it again as a new item.
5. You get read access to the resulting item in *My Recertification*, and receive the approval or rejection mail in CC.

### Site and Team owners: approve or reject

1. You receive a mail when an app requests access to your site, and again each year for recertification.
2. Open the item in **My Recertification** from the link in the mail, and set **Status** to `Approved` or `Rejected`.
3. The next job run grants or revokes the permission and notifies everyone involved.

If you don't answer within 35 days, the request is rejected automatically. Any permission that was already granted is then revoked.

### Governance admins: handle unmanaged apps

`Unmanaged` items in *My Recertification* are apps that hold a governed permission without having gone through the process. For each of them:

- ask the app owner to submit a request. The `Unmanaged` item is removed once a managed item exists for the same app, or
- remove the permission from the app if it is not needed.

### Status values

**Requests – Request Status**

| Value | Meaning |
|---|---|
| `New` | Waiting for validation |
| `Invalid` | Validation failed, and the requester was notified |
| `MovedToRecertificationList` | Valid. The approval continues in *My Recertification* |

**My Recertification – Status**

| Value | Set by | Meaning |
|---|---|---|
| `New` | Job | Item created, approval mail not yet sent |
| `ApprovalRequested` | Job | Waiting for the owner's decision |
| `Approved` | Owner | Permission granted (or will be in the next run) |
| `Rejected` | Owner / Job | Permission revoked (or will be in the next run) |
| `Unmanaged` | Job | Governed permission found without a request |
| `Deleted` | — | Reserved. The job does not process it |

## Operations

### Logs

Job output goes to the Log Analytics workspace:

```kusto
ContainerAppConsoleLogs_CL
| where ContainerJobName_s startswith "pb-spopergov"
| project TimeGenerated, Log_s
| order by TimeGenerated desc
```

These environment variables of the job, set in `infra/main-caj.tf`, control how much the job logs:

| Variable | Default | Effect |
|---|---|---|
| `INFORMATION` | `1` | `Write-Information` output (one line per action) |
| `VERBOSE` | `1` | `Write-Verbose` output (detailed tracing) |
| `DEBUG` | `0` | `Write-Debug` output, including PnP internals (very noisy) |

### Updating

Change scripts, templates or infrastructure, then run `azd up` again. Re-running the hooks is safe.

### Schedule

The default schedule is `0 18 * * 1-5` (weekdays at 18:00 UTC). To change it, edit `cronExpression` in `infra/main-caj.tf`. With `triggerType = "Manual"` the job only runs when started manually.

## Security considerations

- **The job identity is highly privileged.** To grant and revoke `*.Selected` permissions on any site, it holds `Sites.FullControl.All` on Microsoft Graph and SharePoint. Treat the resource group like a tenant admin credential: restrict who can change the job, its image, the file share, or the managed identity.
- **The file share holds executable code.** Anyone with write access to the storage account can change what the job runs.
- **`Mail.Send` covers every mailbox by default.** Restrict it to the sender mailbox with [Exchange Online RBAC for Applications](https://learn.microsoft.com/exchange/permissions-exo/application-rbac) or an application access policy.
- **The container registry has the admin user enabled.** The Terraform Docker provider needs it to push the image. The job itself pulls with its managed identity.
- **Revoking a site-scope permission removes everything.** For `Sites.Selected`, the job revokes all site permissions the app holds on that site, including grants made outside this process.
- **Approvers are the owners of the target.** Anyone who can become owner of a site or Team can approve requests for it.

## Customisation

- **Mail templates**: `job/mails/*.html`. The `<title>` becomes the mail subject. These placeholders are replaced:

  | Placeholder | Replaced with |
  |---|---|
  | `[App]` | App display name |
  | `[Url]` | Target site URL |
  | `[SPOGovSiteUrl]` | Home page of the governance site |

- **Reminder and expiry periods**: the 14, 28 and 35 day thresholds and the 1 year recertification interval are in `job/Invoke-Recertification.ps1`.
- **Governed permissions**: add the app role ID to `$roleIdToDisplayName` in `job/Invoke-Recertification.ps1`. Add the display name as a choice of *Permission Scope* in `spo/assets/Site.xml` (both lists), and implement the grant and revoke in both `switch` blocks of the script.
- **Job permissions**: `infra/main-uami.tf`.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `azd up` fails building the image (`permission denied ... docker.sock`) | Docker socket not mounted, or the dev container still runs as `node`. See [step 2](#2-let-the-dev-container-use-docker). |
| Storage account name is invalid or too long | The environment name is longer than 7 characters or contains `-` / uppercase letters. Create a new environment with a shorter name. |
| The first job run fails with `401` / `403` | Permission grants can take longer than the 3 minute wait to propagate. Start the job again after a few minutes. |
| Job fails immediately with `SITE_URL and SENDER_MAIL must be set` | `PB_SENDER_MAIL` was not set before `azd up`. Set it and run `azd up` again. |
| Requests stay `New` with a warning "could not be completed" | A temporary Graph or SharePoint error. The request is retried in the next run. Check the logs for the actual error. |
| Request is `Invalid` although the app exists | The requester is not an owner of the app registration, the scope is not declared as an *application* permission, or the target list name, item ID or file path is wrong. The log line `ValidationFailed: ...` shows which check failed. |
| Mails are not sent | The sender mailbox doesn't exist, or an Exchange policy blocks the identity from sending as it. |

## Uninstall

```bash
azd down --purge
```

This removes all Azure resources. These items are **not** removed:

- the SharePoint governance site (delete it in the SharePoint admin center),
- the permissions the job granted on target sites (revoke them first by rejecting the items, if you want to clean up).

## Known limitations

- Only app registrations in the tenant can request permissions, because the requester must be an owner of the app registration. Managed identities and multi-tenant apps without an app registration are reported as `Unmanaged`, but cannot go through the request process.
- Item permissions on *My Recertification* are set once, when an item is created. Later changes to site or app owners are not applied to existing items, but notifications always go to the current owners.
- The job reads the lists in full on every run. Lists beyond the SharePoint list view threshold (5,000 items) are not supported.

## Built with

- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- [Terraform](https://www.terraform.io/)
- [Azure Container Apps jobs](https://learn.microsoft.com/azure/container-apps/jobs)
- [PnP PowerShell](https://pnp.github.io/powershell/)
- [Microsoft Graph](https://learn.microsoft.com/graph/overview)
- [PuntoBello Installer](https://github.com/diemobiliar/puntobello-installer)

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on submitting pull requests, and follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Authors

- **Fabian Hutzli** and **Mattias Bürgi**: initial design and implementation
- **Nicole Beck-Dekkara** and **Nello D'Andrea**: contributors to the broader PuntoBello ecosystem

See also [MAINTAINERS](MAINTAINERS).

## License and acknowledgment request

This project is licensed under the MIT License. See [LICENSE.md](LICENSE.md) for details.

If you use this solution, especially in commercial products or managed services, please consider mentioning **"Powered by Die Mobiliar – PuntoBello"** in your documentation or product materials as a voluntary acknowledgment of the authors' work.
