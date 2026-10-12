# Automation Management: Solution Guide

This guide covers how the solution works, how to set it up, the APIs and permissions it needs, how it was tested, and what you still have to verify in your own environments.

Contents
1. How it works
2. The form
3. Bot ID mapping and settings
4. Governance: requests, approvals, results and audit
5. UiPath provider (Orchestrator 2025.10 standalone)
6. Power Automate provider (Dataverse)
7. Security
8. Prerequisites
9. Setup steps
10. API reference (what is called, and where it is documented)
11. API permission checklist
12. Test evidence (offline)
13. First run in your environment (what you must run)
14. Known limits and unverified items
15. Changing the solution

---

## 1. How it works

```
Requester ── Main.xaml ── Form (AutomationManagement.uiform)
                │  Fetch Details ─► Requests\Fetch_Provider_Details ─► UiPath provider  ─► Orchestrator Dev/UAT/Prod REST
                │                                                   └► Power Automate provider ─► Dataverse Dev/UAT/Prod Web API
                │  Validate      ─► Requests\Validate_Deployment_Request (rules + live checks, changes nothing)
                │  Submit        ─► Requests\Submit_Request ─► queue AM_Requests  (reference AM|tool|BOT|type|path|version)
                │                   Dev to UAT and settings allow it ─► Requests\Execute_Request (deploys now)
                │                   UAT to Prod ─► stops here: "Nothing was deployed", waits for approval
Approver ───── ApproveRequest.xaml ─► queue AM_Approvals (reference APR|<request reference>)
Operator/robot ExecuteDeployment.xaml ─► Requests\Execute_Request
                   re-reads the request, re-fetches live details, re-validates (approval + previous stage mandatory for Prod)
                   ─► Deploy_Dev_To_UAT / Deploy_UAT_To_Prod (UiPath) or PA_Deploy_Solution (Power Automate)
                   ─► queue AM_DeploymentResults (RES|...|OK, RES|...|FAILED|ts, RES|...|REJECTED|ts) + audit file
```

Main design decisions:

* **The Bot ID is a business identifier.** It is looked up in `Config\BotMapping.json`, which names the process, package, folders, assets, queues and access (UiPath) or the solution and flows (Power Automate). It is never treated as an Orchestrator key. Aliases are supported (for example `INV-BOT` resolves to `BOT-1001`).
* **Submitting never changes Prod.** UAT to Prod always needs an `Approved` record from a listed approver who is not the requester, plus a successful Dev to UAT result for the same Bot ID and version. Both are checked when the request is submitted and checked again by `ExecuteDeployment.xaml` right before deploying.
* **Every step is idempotent.** Each object is read first and changed only when it differs. A package already in the target feed is not uploaded again, a process already on the version is left alone, an existing queue is never recreated, access roles are merged, and a version already recorded as `RES|...|OK` is not deployed again. A failed run can be re-run and resumes.
* **Values are kept apart.** The form shows the **Dev value** (retrieved), the **target state** (retrieved from UAT or Prod), the **proposed value** (editable, only in edit mode) and, after deployment, the **saved** status per object. The request stores Dev values from the robot's own fetch, never from the form, so edits are limited to the editable columns.
* **The two providers do not share API logic.** UiPath uses `Providers\UiPath\*`. Power Automate uses `Providers\PowerAutomate\*`. Both use the same framework workflows for HTTP, tokens, retries, errors and audit.

## 2. The form

`Form\AutomationManagement.uiform`, shown by `Form\Automation_Form_Controller.xaml` with Show Form (Continue workflow execution = On) and a Trigger Scope that reacts to the buttons and selection changes.

| Section | Shown when | Contents |
|---|---|---|
| Request | always | Automation Tool, Process Type, Deployment Path, Bot ID, **Fetch Details**, lookup status (searching, found, not found, error, warnings) |
| Process (New) | New | Process name, package, Dev version and folder per environment, all read only |
| Package | Enhancement or Bug Fix | Package name, version in Dev, UAT and Prod |
| Version | after a fetch | Requested Version, filled from the versions really available in the source (numeric sort, newest first). For UAT to Prod, only the version deployed to UAT is offered. |
| Configuration | UiPath + New | Assets, queues and access grids. Each row shows the Dev value, target state, action and proposed value. Plus proposed process name and target folder. **Edit Configuration** unlocks the editable columns for configuration editors. |
| Flows | Power Automate + New | Solution, version and cloud flows read from Dataverse, plus a warning about connection references |
| Change details | Enhancement / Bug Fix | Enhancement description and change reference, or bug description and incident reference (required, pattern-checked) |
| Review | after Validate | Summary, validation errors, warnings and submit result |

Buttons: **Fetch Details**, **Edit Configuration** (shown only to configuration editors), **Validate**, **Submit Request** (disabled until details for the current selection are loaded), **Cancel** (reads "Close" after a submission).

If the user changes the tool, type, path or Bot ID after a fetch, the details are marked stale. Validate and Submit then refuse to run until Fetch Details is clicked again.

Credential assets are shown as "Configured in Orchestrator (hidden)". The form never receives a secret value, and the grid has no field where one could be typed.

## 3. Bot ID mapping and settings

### BotMapping.json

```json
{ "botId": "BOT-1001", "aliases": ["INV-BOT"], "automationTool": "UiPath",
  "uipath": { "processName": "InvoiceProcessing", "packageId": "InvoiceProcessing",
              "folders": { "Dev": "Finance/Invoice", "UAT": "Finance/Invoice", "Prod": "Finance/Invoice" },
              "assets": [ { "name": "INV_ServiceAccount", "required": true,
                            "secretSource": { "UAT":  { "assetName": "STAGE_INV_ServiceAccount_UAT",  "folderPath": "Shared/SecretStaging" },
                                              "Prod": { "assetName": "STAGE_INV_ServiceAccount_Prod", "folderPath": "Shared/SecretStaging" } } } ],
              "queues": [ { "name": "INV_Invoices", "required": true } ],
              "access": [ { "name": "CORP\\Finance-RPA-Operators", "type": "DirectoryGroup", "accessLevel": "Operator" } ] } }
{ "botId": "BOT-2001", "automationTool": "Power Automate",
  "powerAutomate": { "solutionUniqueName": "PurchaseApprovals", "flows": ["Purchase approval"] } }
```

Rules checked when the file is loaded: Bot IDs and aliases are unique (ignoring case), each bot has the section for its tool, and each UiPath bot has a folder for Dev, UAT and Prod.

### Settings.json

| Key | Meaning |
|---|---|
| `botIdPattern` | Regex the Bot ID must match (also enforced in the form). |
| `http.timeoutSeconds`, `retryCount`, `retryDelaySeconds` | Passed to every HTTP Request activity: Timeout (for the call including retries), Retry count (retries after the first attempt) and Initial delay. The activity retries only 408, 429, 500, 502, 503, 504 and connection errors, with exponential back-off, honouring `Retry-After`. 401 re-authenticates once. 403 is never retried. |
| `http.pageSize`, `maxPages` | OData paging (`$top`/`$skip` for Orchestrator, `@odata.nextLink` for Dataverse). |
| `uipath.environments.<Dev/UAT/Prod>` | `orchestratorUrl`, `tenantName` (X-UIPATH-TenantName header), `tokenUrl` (`<orchestrator>/identity/connect/token`), `credentialAsset` and `credentialFolder` (the Orchestrator Credential asset holding the external application's client ID and secret), `scope`, and optional `feedId` (folder feed GUID). |
| `powerAutomate` | Entra `authorityUrl` and `tenantId`, `credentialAsset` (app registration client ID and secret), Dataverse `environmentUrl` per environment, async import polling, and `solutionArtifactStore` (where managed exports are kept with their SHA-256). |
| `governance` | Orchestrator environment and folder that hold the three governance queues, and their names. |
| `deploymentPaths` | `Dev to UAT` (no approval, `autoExecuteAfterSubmit: true`) and `UAT to Prod` (`requiresApproval`, `requiresSuccessfulPreviousStage`, `previousStage: "Dev to UAT"`, `autoExecuteAfterSubmit: false`). A path into Prod without approval is rejected at load time. |
| `authorization` | `configurationEditors` (who may use Edit Configuration), `approvers` per target environment, and `segregationOfDuties`. Entries are `DOMAIN\user` or `DOMAIN\group`; group membership is checked on the Windows robot. `*` allows everyone. |
| `roleMappings` | Access level (Read Only, Operator, Developer, Folder Administrator) to Orchestrator role names. |
| `allowedAccessLevels` | Which access levels may be granted per environment (Prod defaults to Read Only and Operator). |
| `security` | Asset-name patterns treated as sensitive, and the masking text. |
| `audit.folder` | Folder for the `audit-yyyyMMdd.jsonl` files. |
| `requestRules` | Required and optional fields per Process Type, and whether configuration is included (only for New). |

Settings contain **no secrets**: only names of Credential assets.

## 4. Governance: requests, approvals, results and audit

Create these queues in the governance folder (default `Shared/AutomationManagement` on the Dev Orchestrator) with **Enforce unique reference = On**:

| Queue | Reference | Content |
|---|---|---|
| `AM_Requests` | `AM|UIP|BOT-1001|New|Dev to UAT|1.10.0` (`PA` for Power Automate) | RequestId, BotId, tool, type, path, version, requester, and RequestJson: the full request with Dev values, target state, proposed changes and change details. No secrets. |
| `AM_Approvals` | `APR|<request reference>` | Decision (Approved/Rejected), Approver, Comment, ApprovedVersion, ApprovedAtUtc |
| `AM_DeploymentResults` | `RES|UIP|BOT-1001|Dev to UAT|1.10.0|OK`, or `...|FAILED|<timestamp>` / `...|REJECTED|<timestamp>` | Status, message, version, per-object results and the run's audit steps |

Because references are unique, the same request cannot be submitted twice, a request can be decided only once, and only one success record exists per Bot ID, path and version. That success record is the evidence the Prod stage checks.

**Audit file**: `Data\Audit\audit-yyyyMMdd.jsonl`, one JSON line per step, with timestampUtc, requestReference, botId, user, machine, operation, status and message. For example:

```
{"requestReference":"AM|UIP|BOT-1001|New|Dev to UAT|1.10.0","botId":"BOT-1001","user":"CORP\\jdoe","operation":"UAT asset INV_MaxInvoices","status":"Succeeded","message":"Updated and verified"}
```

Anything that looks like a credential is masked before it is written. The robot log carries the same lines.

## 5. UiPath provider (Orchestrator 2025.10 standalone)

**Fetch** (`Fetch_Bot_Configuration.xaml`). For Dev, UAT and Prod it reads the mapped folder, the process (release) and the package versions in the feed. For **New**, it also reads Dev assets, queues and folder access, plus their state in the target. Credential fields are never selected. A UAT or Prod read failure (for example HTTP 403) becomes a per-environment warning. A Dev failure makes the result an error.

**Deploy** (`Execute_UiPath_Deployment.xaml`, called by `Deploy_Dev_To_UAT` / `Deploy_UAT_To_Prod`):

1. **Package** (`Deploy_Package.xaml`). If the target feed already has the version, the transfer is skipped. Otherwise the package is downloaded from the source feed, its SHA-256 is recorded, it is uploaded to the target and the target feed is checked again. For UAT to Prod, the package comes from **UAT**, not Dev.
2. **Process**. Same version: no change. Different version: UpdateToSpecificPackageVersion. Missing: created, for New only. Enhancement or Bug Fix into an environment without the process is refused ("submit it as New"). The process is read back. If the version is wrong, the previous version is restored and the run fails.
3. **New only**: assets (`Update_Asset_Configuration`), queues (`Validate_Queue_Configuration`) and access (`Validate_User_Access`). Each is checked first, changed only as approved, and read back.
   * **Credential assets** are created from an approved staging Credential asset (`secretSource` in the mapping) read with Get Credential. The value goes straight to Orchestrator and is never read from Dev or from the form.
   * **Access**: accounts must already exist; they are never created. Roles are resolved by name. Because AssignUsers replaces an account's role list on the folder, current direct roles are merged in so nothing is removed. Inherited roles are not touched.
4. If any object fails, the remaining objects are still attempted, then the run fails with a per-object result. A re-run continues where it stopped.

**Validation rules** (`Validate_Deployment_Request.xaml`):
* required fields per type;
* version available in the source;
* no downgrade;
* process exists, or the type is New;
* Create versus existing asset or queue;
* required objects cannot be skipped;
* typed values (integer, boolean);
* secret handling;
* access levels allowed per environment;
* editor authorization for edited values;
* duplicate submission;
* approval and previous stage.

## 6. Power Automate provider (Dataverse)

The "Power Automate components" are the provider workflows in `Providers\PowerAutomate\`. They work through the Dataverse Web API, which is how solutions (and the cloud flows inside them) are exported and imported. No flow has to be built in Power Automate.

* **Fetch** (`PA_Fetch_Bot_Configuration`). It reads the organization name, the mapped solution's version and managed state in Dev, UAT and Prod, and the cloud flows (solution components of type 29, workflows with category 5) of the Dev solution. Only what the API returns is shown; no metadata is made up.
* **Dev to UAT** (`PA_Deploy_Solution`). It checks that the requested version is the current Dev version, exports the solution as **managed** (`ExportSolution`), and stores the file plus its SHA-256 in `Data\SolutionArtifacts`. Then it imports with `ImportSolutionAsync` (`OverwriteUnmanagedCustomizations: false`, `PublishWorkflows: true`), polls `asyncoperations(<id>)` until it completes, and reads the target version back.
* **UAT to Prod**. It imports the **same stored file** UAT got. The SHA-256 must match and the same version must be installed in UAT. Nothing is exported again for Prod.
* **Connection references and environment variables** are not set by this solution (`ComponentParameters` is empty). Configure the connections in UAT and Prod before the first import of a solution that uses them. The form shows this warning for Power Automate New requests.

## 7. Security

* **Secrets**:
  * Orchestrator external-application secrets, the Entra app secret and the staged credential values live only in Orchestrator Credential assets.
  * They are read at run time with Get Credential.
  * They are never in XAML, settings, the form, queue items, logs or audit files.
  * Access tokens are cached only in memory for the run (`in_Session`) and never logged.
* `project.json` `excludedLoggedData` hides arguments and variables matching `*password*`, `*secret*`, `*token*` and `*credential*` from robot execution logs.
* Asset reads use `$select` without credential fields. A non-credential asset whose name matches a sensitive pattern is masked in the form and cannot be migrated by value.
* Errors are logged as method, path, status, Orchestrator error code and correlation ID, never request or response bodies. Anything credential-shaped is masked.
* Least privilege:
  * Dev uses read scopes, plus what the governance queues and package download need (section 11).
  * Approvers and editors come from settings.
  * Segregation of duties is on by default.
  * Accounts and roles are never created.
* **No production change was executed while building this solution.** All runs were against local mocks.

## 8. Prerequisites

| Item | Version or requirement |
|---|---|
| UiPath Studio | 25.10 (Windows project, VB.NET) |
| Robot | 25.10, Windows. Attended for Main and ApproveRequest; attended or unattended for ExecuteDeployment. |
| Orchestrator | 2025.10.2 standalone, separate Dev, UAT and Prod (or tenants), reachable over HTTPS from the robot |
| Packages | UiPath.System.Activities 26.2.7, UiPath.Form.Activities 25.10.0, UiPath.WebAPI.Activities 2.4.0 (HTTP Request), UiPath.Excel.Activities 3.4.1 (pinned in project.json) |
| Orchestrator external applications | One confidential application per Orchestrator with application scopes (section 11) |
| Governance folder | `Shared/AutomationManagement` on the Dev Orchestrator, with the three queues and the credential assets |
| Secret staging folder | For example `Shared/SecretStaging`, readable only by the deployment robot, holding staged Credential assets per target environment |
| Power Automate (optional) | Entra app registration with a client secret, and a Dataverse **application user** in Dev, UAT and Prod with a security role that can export and import solutions |

## 9. Setup steps

1. **Unzip** and open `RPA_Automation_Management\project.json` in Studio 25.10. Let it restore the three packages.
2. **Orchestrator external applications** (each Orchestrator: Admin > External Applications > Add, confidential, application scopes as in section 11). Copy each client ID and secret.
3. **Credential assets** in the governance folder of the Dev Orchestrator (Username = client ID, Password = secret): `AM_Orchestrator_Dev_App`, `AM_Orchestrator_UAT_App`, `AM_Orchestrator_Prod_App`, and `AM_PowerPlatform_App` if you use Power Automate. Give the robot account View access to them and to nothing else it does not need.
4. **Governance queues** `AM_Requests`, `AM_Approvals` and `AM_DeploymentResults` in the same folder, with **Enforce unique reference** on.
5. **Secret staging** (only for Credential assets that must be created in UAT or Prod). Create the staged Credential assets named in the mapping in a restricted folder, filled in by the credential owner.
6. **Edit `Config\Settings.json`**: replace every `<...>` (hosts, tenant ID, Dataverse URLs, `DOMAIN\group` names). Set `feedId` only if a target folder uses its own feed. Review approvers, role mappings and allowed access levels.
7. **Edit `Config\BotMapping.json`**: replace the two template bots with your real Bot IDs, processes, folders and objects.
8. **Power Automate**: register the Entra app and create the Dataverse application user in each environment with the required role. Create the connections that the solutions' connection references need in UAT and Prod.
9. Publish the project to the governance Orchestrator. Create processes for the entry points: `Main` (requesters), `ApproveRequest` (approvers) and `ExecuteDeployment` (deployment robot). Point requesters, approvers and the deployment robot at them.
10. Do the first-run checks in section 13 **in Dev and UAT only** before allowing any Prod path.

## 10. API reference

"Guide" means the endpoint is described in the UiPath Orchestrator API guide or Microsoft's Dataverse Web API documentation. "Swagger" means it appears in the Orchestrator Swagger (`/swagger`) but has no worked example in the guide, so its exact body must be confirmed on your 2025.10.2 Swagger (section 14).

| Call | Used by | Source |
|---|---|---|
| `POST <orchestrator>/identity/connect/token` (client_credentials) | Get_Access_Token | Guide |
| `GET odata/Folders?$filter=FullyQualifiedName eq '...'` | Get_Folder | Guide |
| `GET odata/Releases?$filter=Name eq '...'`, `GET odata/Releases(<id>)` | Get_Process_Details, verify | Guide |
| `POST odata/Releases` | Deploy_Package (New: create process) | Swagger |
| `POST odata/Releases(<id>)/UiPath.Server.Configuration.OData.UpdateToSpecificPackageVersion` | Deploy_Package | Swagger |
| `GET odata/Processes/UiPath.Server.Configuration.OData.GetProcessVersions(processId='<id>')` | Get_Package_Details | Guide |
| `GET odata/Processes/UiPath.Server.Configuration.OData.DownloadPackage(key='<id>:<ver>')` | Deploy_Package | Guide |
| `POST odata/Processes/UiPath.Server.Configuration.OData.UploadPackage` (multipart `file`) | Deploy_Package | Swagger |
| `GET odata/Assets` (`$select` without credential fields), `POST odata/Assets` | Get_Asset_Details, Update_Asset_Configuration | Guide |
| `PUT odata/Assets(<id>)` | Update_Asset_Configuration | Swagger |
| `GET odata/QueueDefinitions` | Get_Queue_Details | Guide |
| `POST odata/QueueDefinitions`, `PUT odata/QueueDefinitions(<id>)` | Validate_Queue_Configuration | Swagger |
| `GET odata/Folders/UiPath.Server.Configuration.OData.GetUsersForFolder(key=..,includeInherited=..)` | Get_User_Access | Guide |
| `POST odata/Folders/UiPath.Server.Configuration.OData.AssignUsers` | Validate_User_Access | Guide |
| `GET odata/Users?$filter=UserName eq '...'`, `GET odata/Roles` | Validate_User_Access | Guide |
| `GET odata/QueueItems?$filter=QueueDefinitionId eq .. and Reference eq '..'` | Find_Queue_Item | Guide |
| `POST odata/Queues/UiPathODataSvc.AddQueueItem` | Add_Queue_Item | Swagger |
| `POST https://login.microsoftonline.com/<tenant>/oauth2/v2.0/token` (scope `<env url>/.default`) | Get_Access_Token (PA) | Guide (Microsoft) |
| `GET RetrieveCurrentOrganization(...)`, `GET solutions`, `GET solutioncomponents`, `GET workflows` | PA_Get_Solution_Details, PA_Get_Flows | Guide (Microsoft) |
| `POST ExportSolution`, `POST ImportSolutionAsync`, `GET asyncoperations(<id>)` | PA_Deploy_Solution | Guide (Microsoft) |

Every call above is made by the UiPath **HTTP Request** activity (UiPath.WebAPI.Activities 2.4.0), not by Invoke Code. All REST calls go through `Framework\Send_Http_Request.xaml`, which holds one HTTP Request activity per method: GET, POST, PUT, PATCH, DELETE, multipart upload and download to file. Token requests use the HTTP Request activity in `Framework\Get_Access_Token.xaml`, with Form URL encoded body type. The activity settings used are:

| Property | Value |
|---|---|
| Authentication | OAuth token (the bearer token from Get_Access_Token); None for the token request itself |
| Headers | `Accept: application/json`, `X-UIPATH-TenantName`, and `X-UIPATH-OrganizationUnitId` (folder ID) on folder-scoped calls |
| Body | Text (`application/json`, UTF-8) for POST, PUT and PATCH; Multipart (part `file`) for UploadPackage; Form URL encoded for tokens |
| Retry policy | Exponential back-off, Retry count = `http.retryCount` + 1 attempts, Initial delay = `http.retryDelaySeconds`, Retry-After preferred |
| Continue on error | True: error statuses come back in the result and are classified by Handle_API_Exception |
| Save raw request/response | False, so no headers or tokens are kept |
| Downloads | Save response as file into the artifact folder (Replace) |

## 11. API permission checklist

**Orchestrator external application, application scopes**

| Environment | Scopes (template default) | Why |
|---|---|---|
| Dev | `OR.Folders.Read OR.Execution OR.Assets.Read OR.Queues OR.Users.Read` | Read folders, processes, packages (download needs OR.Execution), assets and access. `OR.Queues` with write access is for the governance queues, which live on Dev. |
| UAT | `OR.Folders OR.Execution OR.Assets OR.Queues OR.Users` | Upload packages, create and update processes, assets, queues and folder role assignments |
| Prod | same as UAT | same, but only through an approved request |

- [ ] External application per Orchestrator, confidential, with the scopes above (reduce them if your 2025.10.2 offers finer scopes that still cover the calls in section 10).
- [ ] Confirm the app can see the mapped folders (a folder it cannot see is reported as "does not exist or is not visible").
- [ ] Robot account: View on the four `AM_*_App` credential assets and the staging credential assets, plus queue item access in the governance folder. Nothing in the Finance/... folders is needed for the robot itself.
- [ ] Governance queues created with unique references.
- [ ] Requesters, editors and approvers listed in `authorization` (Windows accounts or groups as the robot sees them).
- [ ] Entra app registration with client secret; Dataverse application user in each environment with a role that can read solutions and workflows and export (Dev) or import (UAT, Prod) solutions, for example System Customizer. Confirm your organisation's minimum.
- [ ] Outbound HTTPS from the robot to the three Orchestrators, login.microsoftonline.com and the Dataverse URLs.

## 12. Test evidence (offline)

All tests ran in a Linux build container with no access to your systems. **None of these results come from a real Orchestrator, Dataverse or Studio.**

| What | How | Result | Evidence |
|---|---|---|---|
| XAML load and expression compile | Every XAML loaded with the real UiPath.System.Activities 26.2.7, UiPath.Form.Activities 25.10.0, UiPath.WebAPI.Activities 2.4.0 (HTTP Request), UiPath.Platform 25.10 and UiPath.Workflow assemblies; `ActivityValidationServices` compiled every VB expression. Two Studio-internal assemblies (Activities.Contracts, CodedWorkflows) were empty stubs. | 38 of 38 loaded, 0 errors | `verification/xaml-validation.log` |
| .uiform file | Deserialised with UiPath.Studio.Forms.Runtime 25.10.0's form serializer | OK, 129 components, FormId matches the activities | same log |
| Form behaviour | Chromium with the form.io bundle from UiPath.Studio.Forms.Runtime 25.10.0. Covers section visibility per tool and type, events, field setting, stale details, buttons, grid editing, validation messages and secret masking. | 57 of 57 | `verification/form-browser-tests.log`, screenshots |
| Workflow logic end to end | The same expressions and Invoke Code bodies compiled as VB.NET (Option Strict On). The HTTP Request activity is emulated from its decompiled 2.4.0 logic: retry handler, continue on error, body types and response handling. Run against a **mock** Orchestrator (Dev/UAT/Prod), Identity Server, Entra ID and Dataverse with fault injection | 80 of 80 | `verification/workflow-tests.log` |

Workflow test areas:

| Area | Checks | What is checked |
|---|---|---|
| Configuration | 3 | Missing files, Prod path without approval, duplicate Bot ID |
| Bot IDs | 4 | Unknown ID, alias, tool mismatch, Cancel |
| UiPath New Dev to UAT | 20 | Fetch values, masking, edit mode, validation, submit, package, process, assets (including credential from staging), queue, access merge, Prod untouched, governance records, audit |
| Idempotency | 3 | Re-execute changes nothing, duplicate submit refused, objects then show Keep / No change |
| UiPath Enhancement UAT to Prod | 12 | Package-only details, version from UAT, waits for approval, blocked without approval, self-approval refused, wrong approver refused, approved deploy from the UAT feed, no config changes, decided once |
| UiPath Bug Fix | 5 | Incident reference required, process must exist, update path, downgrade refused |
| Validation rules | 8 | Disallowed role, duplicate create, required skip, invalid not submitted, non-editor edits, credential without secret source, stale details, Prod version rule |
| Partial failure | 4 | Persistent 500 on one object, retries, FAILED evidence, resume |
| Rollback | 2 | Wrong version after update triggers restore and failure, no OK evidence |
| API errors | 5 | 503 retry, 401 re-auth, 403 as an environment warning, rejected secret, governance queue down |
| Power Automate | 10 | Fetch, managed export, async import poll, Entra scope, artifact and hash, Prod waits for approval, tampered file refused, same file to Prod, import failure, version rule |
| Secrets | 4 | No secret or token in form data, logs, audit or queue items; every call authenticated; no credential fields selected |

Three workflow defects were found and fixed by these tests:
- an error message starting with "[" was parsed as JSON;
- the request kept the Bot ID as typed rather than the mapped one;
- the submission audit line lacked the request reference.

## 13. First run in your environment (what you must run)

Run these in order, in **Dev and UAT only**, and keep the logs.

1. **Open in Studio**. The project should open and restore without errors, including UiPath.WebAPI.Activities 2.4.0 for the HTTP Request activity (it is already in your local package cache). Run *Analyze Project* and *Validate*. (Validated here only with the runtime assemblies, not inside Studio.)
2. **Form designer**. Open `Form\AutomationManagement.uiform` and check it renders. Confirm the Show Form, Get/Set Form Fields and Form Trigger activities in `Form\Automation_Form_Controller.xaml` show the form as linked (FormId `8f1c2a7e-5b3d-4e69-a0c4-7d2e9b6f1a35`).
3. **Fetch only**. Run `Main` with a real Bot ID and use Fetch Details for Enhancement Dev to UAT, then New Dev to UAT. Compare what the form shows with Orchestrator. Check the robot log contains no secrets.
4. **Grid editing in the real form host**. In edit mode, change an asset value, a queue setting and an access level, then click Validate. Check that the summary shows your changes. This was tested with form.io in a browser, not with the UiPath form host.
5. **Swagger check**. Open `https://<uat-orchestrator>/swagger` and confirm the request bodies of the Swagger-only calls in section 10 (UploadPackage, POST Releases, UpdateToSpecificPackageVersion, Assets PUT, QueueDefinitions POST/PUT, AddQueueItem).
6. **Dev to UAT with a test bot**. Submit a New request for a throwaway process. Check the package, process, assets (including one credential from staging), queue and access in UAT, then the `AM_Requests` and `AM_DeploymentResults` items and the audit file. Run `ExecuteDeployment` again with the same reference and expect AlreadyDeployed with no changes.
7. **Approval flow without Prod**. To test UAT to Prod safely, temporarily point the `Prod` environment in a copy of the settings at a second non-production Orchestrator. Submit, try ExecuteDeployment (expect Rejected), approve with `ApproveRequest` as a different user, then execute.
8. **Power Automate**. Fetch a test solution, deploy Dev to UAT, and check the import in the Power Platform admin centre and the stored file and hash.
9. Only then point `Prod` at the real production Orchestrator and Dataverse.

## 14. Known limits and unverified items

* **Swagger-only endpoints** (section 10). The bodies were built from the Orchestrator API model (`UploadPackage` multipart field `file`; `UpdateToSpecificPackageVersion` body `{"packageVersion": ...}`; `POST Releases` with Name, ProcessKey, ProcessVersion and Description only, so a package with several entry points gets Orchestrator's default entry point; `AddQueueItem {itemData: {...}}`). They were not called against a real 2025.10.2 instance.
* **`feedId`**. The query parameter for folder feeds on GetProcessVersions, UploadPackage and DownloadPackage is not confirmed. Leave `feedId` empty for tenant feeds.
* **AddQueueItem 409**. The duplicate-reference status code on a unique-reference queue is assumed to be 409. The workflow also checks for an existing reference before adding, so a different status still prevents duplicates. It would just show as an error.
* **Secret-type assets** (Orchestrator "Secret" value type) are not migrated by value. They must be set in Orchestrator. Credential assets are supported through staging.
* **Per-robot asset values** are not migrated; a warning is shown.
* **Group membership** for editors and approvers is checked through Windows group membership on the robot machine. On non-domain-joined robots, list users explicitly.
* **Form host differences**. The UiPath form host may apply Set Form Fields values in a different order than the browser test.
* **Power Automate** connection references and environment variables must exist in the target (section 6). Unmanaged-to-managed conflicts in the target are reported by Dataverse and surfaced as a failed import.

## 15. Changing the solution

The XAML files, the form and the VB test twin are generated from one Python model in `verification/sources`:

| File | Purpose |
|---|---|
| `build.py` | Renders the XAML |
| `build_form.py` | Builds the form |
| `vb/` | Invoke Code bodies |
| `test/run_e2e.sh` | Mock end-to-end suite |
| `test/test_form.js` | Browser form tests |
| `validate.sh` | Assembly load and compile check |

Small edits can be made directly in Studio. If you regenerate, re-apply those edits.
