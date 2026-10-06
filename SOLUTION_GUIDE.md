# RPA Request & Code Movement Automation: Solution Guide

Target: UiPath Studio 2025.10.17, UiPath Forms (Form Activities 25.10.x), standalone Orchestrator 2025.10.x, Microsoft Dataverse Web API v9.2 (Power Automate solutions). Language: VB.NET, Windows (.NET 8) project.

Sections 1 to 17 follow the requested output order. Section 18 lists API limitations; section 19 says exactly what was and was not verified.

---

## 1. Architecture

```
 Requester (attended robot)                     Back end (same project, also runs unattended)
 ───────────────────────────                     ──────────────────────────────────────────────
 Main.xaml
  ├─ GetConfiguration      Config.xlsx  ──────►  Config dictionary (Settings, Environments, tables)
  ├─ GetEntityConfiguration EntityConfiguration.json (or Entities sheet)
  ├─ GetCurrentUser         Windows account + AD display name / mail / employee ID
  ├─ loop: WF_ShowRequestForm ─► Forms/RPARequestForm.xaml (UiPath Form, dynamic, form.io JSON)
  │        WF_ValidateRequest  (server-side, never trusts the form)
  ├─ WF_CreateRequestID     RPA-yyyy-NNNNN, unique even with parallel robots
  ├─ WF_SendNotification    "Submitted"
  └─ WF_ProcessRequest ─┬─ WF_LoadEntityConfiguration
                        ├─ UiPath target:  WF_GetOrchestratorToken → WF_CreateOrchestratorFolder → Assets → Queue
                        │                  → WF_CreateMachineTemplate → WF_AddOrchestratorUser (+ roles) → BOT robot account
                        │                  → WF_MoveUiPathCode (download package from source, upload to target, create/update process, verify, roll back)
                        ├─ Power Automate: WF_AssignSecurityRoles (Dataverse) → WF_MovePowerAutomateCode
                        │                  (export or stored artifact → import → env variables → connection refs → turn flows on → validate)
                        ├─ WF_UpdateRequestStatus  Submitted → In Progress → Provisioning → Deployment In Progress → UAT/Production → Completed
                        └─ WF_SendNotification "Completed"
       any exception ─► Framework/Exceptions/WF_HandleException  (BusinessRuleException → Rejected, everything else → Failed; status + mail)
```

* **One project, two modes.** Attended: Main shows the form. Unattended: start Main with `in_RequestJson` (the form output JSON); the same validation and back end run without UI, which is how a queue/trigger based performer would call it.
* **Least privilege.** Each environment has its own confidential client (DEV/UAT/PROD) with only the OR scopes it needs; secrets live in Orchestrator credential assets; the Power Platform app user only needs System Customizer + the roles it assigns.
* **Idempotent by design.** Every create is "look up, create if missing, re-read on 409". Running the same request twice changes nothing (tested, section 14).
* **HTTP layer.** One `SendApiRequest` workflow (HttpClient in Invoke Code) for Orchestrator, Identity and Dataverse: OAuth token refresh 5 minutes before expiry, retries only 429/500/502/503/504 and timeouts with exponential back-off and Retry-After, multipart upload, binary download, correlation IDs, body logging only when the body holds no secret. Tokens are kept as `SecureString` and only converted at the moment the header is written.

## 2. Project structure

```
RPA_Request_Management/
├─ project.json                    entry point Main.xaml; excludedLoggedData masks secrets
├─ Main.xaml
├─ Config/
│  ├─ Config.xlsx                  Settings, Environments, Entities, Roles, Assets, Queues, Machines, PowerAutomate,
│  │                               PA_EnvironmentVariables, PA_ConnectionReferences
│  ├─ EntityConfiguration.json     entity catalogue (drives the form and the back end)
│  └─ company-logo.svg             placeholder logo (Config "CompanyLogoPath")
├─ Forms/
│  ├─ RPARequestForm.xaml          Create Form activity
│  └─ RPARequestForm.form.json     form schema (form.io JSON, import in Form Designer)
├─ Workflows/                      business workflows named in the brief (WF_*)
├─ Orchestrator/                   one workflow per Orchestrator operation
├─ PowerAutomate/                  one workflow per Dataverse / solution operation
├─ Framework/
│  ├─ Utilities/                   GetToken, SendApiRequest, ParseApiResponse, HandleApiError, GetConfiguration,
│  │                               GetEntityConfiguration, GetCurrentUser, GenerateRequestID, WriteAuditLog
│  ├─ Logging/SetLogContext.xaml   Add Log Fields (RequestID, Entity, RPATool, ...)
│  └─ Exceptions/WF_HandleException.xaml
└─ docs/  SOLUTION_GUIDE.md, WORKFLOWS.md (every workflow with its arguments)
```

Requested name → file:

| Requested | File |
|---|---|
| Main | Main.xaml |
| WF_ShowRequestForm, WF_ValidateRequest, WF_CreateRequestID, WF_LoadEntityConfiguration | Workflows/ |
| WF_CreateOrchestratorFolder / Assets / Queue, WF_AddOrchestratorUser, WF_CreateMachineTemplate, WF_AssignRoles | Workflows/ (call Orchestrator/ building blocks) |
| WF_MoveUiPathCode, WF_MovePowerAutomateCode, WF_UpdateRequestStatus, WF_SendNotification | Workflows/ |
| WF_HandleException | Framework/Exceptions/ |
| WF_GetOrchestratorToken | Orchestrator/ |
| Power Automate Export / Import / UpdateEnvironmentVariables / UpdateConnectionReferences / ActivateFlows / ValidateDeployment | PowerAutomate/WF_ExportSolution, WF_ImportSolution, WF_UpdateEnvironmentVariables, WF_UpdateConnectionReferences, WF_ActivateFlows, WF_ValidateDeployment |
| GetToken, SendApiRequest, ParseApiResponse, HandleApiError, GetConfiguration, GetEntityConfiguration, GenerateRequestID, WriteAuditLog | Framework/Utilities/ |
| (added) WF_ProcessRequest | Workflows/: the back end orchestration, reusable unattended |

## 3. Packages

| Package | Version | Used for |
|---|---|---|
| UiPath.System.Activities | 26.2.7 | Invoke Workflow File, Invoke Code, Get Credential, Add Log Fields, Message Box, Log Message |
| UiPath.Excel.Activities | 3.4.1 | Read Range (workbook) for Config.xlsx |
| UiPath.Form.Activities | 25.10.2 | Create Form |

Versions are the ones bundled with Studio 2025.10.17 according to its release notes. No UiPath.WebAPI or Mail package is needed: HTTP and SMTP use .NET (`HttpClient`, `SmtpClient`) inside Invoke Code so retries, SecureString tokens and multipart upload are under our control. Newtonsoft.Json comes with System.Activities. Package feed for 2025.10: `https://pkgs.uipath.com/official/index.json`.

## 4. Configuration

Config.xlsx is read once into a `Dictionary(Of String, Object)`:

* `Settings` (Name | Value | Description) → `Config("<Name>")`. Relative `...Path` values are resolved against the Config folder. A setting whose name looks like a secret (password, secret, token, apikey) and is not an `...Asset`, `...AssetFolder`, `...Mode`, `...Url` or `...Path` name is refused at start-up, so nobody can paste a secret into the workbook.
* `Environments` (one row per DEV/UAT/PROD) → `Config("UAT.OrchestratorURL")` etc. Columns: OrchestratorURL, TenantName, IdentityTokenURL (`{OrchestratorURL}/identity/connect/token`), ClientID, ClientSecretAsset, CredentialAssetFolder, Scope, IdentityScope. **DEV, UAT and PROD are fully separate**: own URL, own client, own secret asset.
* Every other sheet → `Config("Table.<Sheet>")` as a DataTable.

Key settings: `RequireHttps` (True), `AllowDevToProd` (False), `RetryCount` 3, `RetryDelaySeconds` 5, `ApiTimeoutSeconds` 60, `MaxFormAttempts` 5, `RequestStorePath` (use a UNC share in production), `SolutionArtifactStorePath`, `AllowDirectoryUserAssignment`, `CreateMissingRobotAccounts`, `BotFolderRoleName`, `SecretAssetMode`, `VaultCredentialFolder`, `PowerPlatformTenantID/ClientID/ClientSecretAsset`, SMTP settings, `CompanyLogoPath`.

Other sheets:

| Sheet | Columns | Meaning |
|---|---|---|
| Entities | same fields as EntityConfiguration.json | used when `EntityConfigurationSource = Excel` |
| Roles | PolicyRole, RPATool, Environment (DEV/UAT/PROD/ALL), TargetRoleName, Scope (Folder/Tenant/Dataverse) | maps the 4 policy roles to Orchestrator folder/tenant roles or Dataverse security roles per environment |
| Assets | AssetGroup, Environment, AssetName, AssetType (Text/JSON/Integer/Bool/Credential/Secret), Value, ValueFromCredentialAsset, UpdateIfExists, Description | `{Entity}` and `{Environment}` are expanded; secret values are never in the sheet, they are copied from a vault credential asset |
| Queues | QueueName (`*` = default), Description, MaxNumberOfRetries, AcceptAutomaticallyRetry, EnforceUniqueReference | |
| Machines | MachineTemplateName (`*` = default), UnattendedSlots, NonProductionSlots, Description | |
| PowerAutomate | Entity (`*`), ExportAsManaged, OverwriteUnmanagedCustomizations, PublishWorkflows, ActivateFlows | |
| PA_EnvironmentVariables | Entity, Environment, SchemaName, Value | target values per environment |
| PA_ConnectionReferences | Entity, Environment, LogicalName, ConnectionId, ConnectorId | pre-created connections in the target |

EntityConfiguration.json: one object per entity with `PolicyRole1..4`, `BotID`, `PowerAutomateBotID`, `Dev/UAT/ProdFolder`, `Dev/UAT/ProdMachineTemplate`, `QueueName`, `AssetConfiguration` (asset group), `PowerAutomateDev/UAT/ProdEnvironment` (Dataverse URLs), `PowerAutomateSolution`, `AllowedMovements`, `NotificationEmail`. Adding an entity = adding an object; the form picks it up without changes.

Credential assets you must create (folder `Shared/RPA Platform`, robot account with View on it):

| Asset | Password holds |
|---|---|
| OR_ClientSecret_DEV / _UAT / _PROD | client secret of the confidential app in each Orchestrator |
| PP_ClientSecret | Entra app registration secret (application user in every Dataverse environment) |
| VAULT_{Entity}_AppLogin_UAT / _PROD, VAULT_{Entity}_ApiKey_{Environment} | values copied into new entity credential assets |
| SmtpCredentialAsset (optional) | SMTP relay login |

## 5. Form design

One UiPath Form (Create Form activity), not Apps/HTML/WinForms/WPF. All dynamic behaviour is form.io logic inside the form JSON, driven by a hidden `EntityConfig` field that the robot fills with the entity catalogue:

| Section | Fields | Rules |
|---|---|---|
| Header | company logo (`{{ data.CompanyLogo }}`, Base64 data URI from `CompanyLogoPath`, placeholder if missing), title | |
| Server validation | red alert | shown only when the robot returned validation errors |
| Requester | Requester Name, Email, Employee ID (prefilled from AD when available), Windows Account (read-only, auto-detected) | required, e-mail format, ID pattern |
| Request | Entity, RPA Tool (UiPath / Power Automate), Process Type (New / Enhancement / Bug Fix), Priority (Critical / High / Medium / Low), Movement Type, Source / Target Environment (calculated) | Movement options = the entity's `AllowedMovements`; DEV → PROD only if both the entity and `AllowDevToProd` allow it |
| Entity configuration | Policy Role 1-4 (read-only, from entity) each with "Grant this role", BOT ID, Queue, Asset configuration, Source / Target folder, Target machine template, PA source / target environment | at least one role must be granted; UiPath fields hidden for Power Automate and vice versa; BOT ID switches to the PA bot ID |
| UiPath deployment | Package Name, Package Version (empty = version running in the source folder) | shown for UiPath |
| Power Automate deployment | Solution unique name (defaults to entity solution), Export as managed | shown for Power Automate |
| Process details | New: Application Name, Process Name, Process Description. Enhancement: Existing Process Name, Enhancement Description. Bug Fix: Existing Process Name, Bug Description, Severity | sections switch with Process Type |
| Additional information | Request Description, Business Justification, Comments, Grant access to (optional DOMAIN\user or e-mail; empty = requester) | |
| Buttons | SUBMIT, CANCEL | Cancel sets `FormAction = Cancel` and skips validation |

The robot repeats every check on submit (WF_ValidateRequest) and recomputes the entity-driven values, so a tampered form cannot change policy roles, environments or movement rules. Errors are sent back into the form (up to `MaxFormAttempts`) with the previous values kept. A screenshot is in `verification/form-preview.png`.

**Importing the form:** open `Forms/RPARequestForm.xaml`, double-click Create Form → Open Form Designer → Import (or paste the JSON into the JSON editor) → select `Forms/RPARequestForm.form.json` → Save. Component keys must stay as they are; the workflow maps `EntityConfig`, `CompanyLogo`, `ValidationMessage` and reads the output by key.

## 6. Form

`Forms/RPARequestForm.form.json` (80 components). Built for form.io 4.x, the engine UiPath Forms uses.

## 7. Main workflow

`Main.xaml` arguments: `in_ConfigPath` (default `Config\Config.xlsx`), `in_RequestJson` (empty = attended form; JSON = unattended), `out_RequestID`, `out_Status`.
Flow: load config and entity catalogue → current user → form loop (show, validate, re-show with errors) → Cancel ends with status "Cancelled" and creates nothing → Request ID → "Submitted" mail → WF_ProcessRequest in Try/Catch → WF_HandleException on error → result message box (attended). An outer Try/Catch logs Fatal, tells the user, and rethrows so Orchestrator marks the job faulted.

## 8. Workflows

Every workflow, its purpose and all arguments (direction, type, description) are listed in `docs/WORKFLOWS.md`, generated from the XAML. Naming: `in_`, `out_`, `io_` arguments; variables `str`, `int`, `lng`, `bool`, `dict`, `lst`, `dt`, `sec` (SecureString).

## 9. VB.NET code

The VB.NET is inside the XAML as Invoke Code activities (29 bodies). The same code is in `docs/vb/` for review. Highlights:

| Body | Purpose |
|---|---|
| HttpSend | HttpClient call with token refresh, retries (429/5xx/timeouts, exponential back-off, Retry-After), multipart upload, download to file, header handling, OData headers, SecureString token |
| TokenRequest | OAuth2 client credentials (Orchestrator Identity and Entra), secret only from SecureString |
| HandleApiError | classifies status codes; 401/403 → UnauthorizedAccessException, others → HttpRequestException with masked summary |
| RequestBuild | server-side validation and the fixed request schema |
| RequestId | race-free Request ID (FileMode.CreateNew reservation per number) |
| RequestStore | request register JSON with status history, atomic write |
| AuditLine | JSON-lines audit log with masking |
| AssetPlan / AssetBody | expands asset groups per entity/environment, builds Orchestrator asset bodies |
| RoleMap / RoleMatch / TenantRoles / FolderAssignment | policy roles → Orchestrator roles; merge instead of overwrite |
| MatchUser / DomainUserBody | user lookup and AD assignment |
| SolutionParams / ImportBody / SaveExport | Dataverse component parameters, import body, export artifact + SHA-256 |
| ClassifyException | Rejected vs Failed, masks secrets in the summary |

## 10. Orchestrator API (2025.10, standalone)

Auth: `POST {OrchestratorURL}/identity/connect/token`, `grant_type=client_credentials`, per-environment client, scopes `OR.Folders OR.Assets OR.Queues OR.Machines OR.Users OR.Execution` (Identity calls: `PM.RobotAccount PM.RobotAccount.Write`). Folder-scoped calls send `X-UIPATH-OrganizationUnitId`.

"Guide" = shown in the UiPath Orchestrator API guide for 2025.10. "Swagger" = present in the Orchestrator Swagger (`/swagger`) and used by the community, but not shown as an example in the guide: check it in your instance's `/swagger/index.html` before go-live.

| Operation | Call | Source |
|---|---|---|
| Find folder | GET odata/Folders?$filter=DisplayName eq '..' and ParentId eq .. / FullyQualifiedName eq '..' | Guide |
| Create folder | POST odata/Folders {DisplayName, ParentId, ProvisionType, PermissionModel} | Swagger |
| Assets | GET odata/Assets?$filter=Name eq '..'; POST odata/Assets {Name, ValueScope:Global, ValueType Text/Bool/Integer/Credential, ...}; PUT odata/Assets({id}) | Guide |
| Queue | GET/POST odata/QueueDefinitions | Swagger |
| Machine template | GET/POST odata/Machines {Name, Type:Template, UnattendedSlots, NonProductionSlots}; GET Machines/...GetAssignedMachines(folderId=); POST Folders/...AssignMachines | Swagger |
| Roles | GET odata/Roles | Guide |
| Users | GET odata/Users?$filter=...; GET odata/Users({id}) | Guide |
| Folder roles | GET Folders/...GetUsersForFolder(key=,includeInherited=false); POST Folders/...AssignUsers (merged with current roles because AssignUsers replaces them) | Guide |
| Tenant roles | POST odata/Users({id})/UiPath.Server.Configuration.OData.AssignRoles {roleIds} (merged: the call overwrites) | Guide |
| AD user/group | POST Folders/...AssignDomainUser {assignment:{Domain, UserName, UserType, RolesPerFolder}} | Swagger |
| Robot account | POST identity/api/RobotAccount {partitionGlobalId, name, groupIDsToAdd} | Guide (Identity API) |
| Package versions | GET Processes/...GetProcessVersions(processId='..') | Guide |
| Download package | GET Processes/...DownloadPackage(key='id:version') | Guide |
| Upload package | POST Processes/...UploadPackage (multipart, field `file`) | Swagger |
| Process (release) | GET/POST odata/Releases, GET Releases({id}), POST Releases({id})/...UpdateToSpecificPackageVersion {packageVersion} | Guide (Releases) / Swagger (UpdateToSpecificPackageVersion) |

Package movement: the .nupkg is downloaded from the source Orchestrator and uploaded to the target tenant feed (skipped if that version exists), then the process is created or updated to the exact version and read back; if the read-back version is wrong the previous version is restored.

## 11. Power Automate / Dataverse API

Auth: Entra client credentials `POST {PowerPlatformAuthorityUrl}/{TenantID}/oauth2/v2.0/token`, scope `{environment URL}/.default`. The app registration must be an application user in every environment. Base `{environment URL}/api/data/v9.2/`, headers `OData-MaxVersion: 4.0`, `OData-Version: 4.0`.

| Step | Call |
|---|---|
| Find solution | GET solutions?$filter=uniquename eq '..'&$select=solutionid,version,ismanaged |
| Export | POST ExportSolutionAsync {SolutionName, Managed} → AsyncOperationId, ExportJobId; poll GET asyncoperations({id}) until statecode 3 (statuscode 30 ok, 31 failed, 32 cancelled); POST DownloadSolutionExportData {ExportJobId} → Base64 zip saved to SolutionArtifactStorePath with SHA-256 |
| Import | POST ImportSolutionAsync {OverwriteUnmanagedCustomizations, PublishWorkflows, CustomizationFile, ImportJobId, ComponentParameters (env variable values + connection references)}; poll asyncoperations. Skipped when the same or newer version is installed |
| Environment variables | GET environmentvariabledefinitions (with values); PATCH environmentvariablevalues({id}) or POST environmentvariablevalues with EnvironmentVariableDefinitionId@odata.bind |
| Connection references | GET connectionreferences?$filter=connectionreferencelogicalname eq '..'; PATCH connectionreferences({id}) {connectionid} |
| Turn flows on | GET solutioncomponents (componenttype 29) → GET workflows (category 5) → PATCH workflows({id}) {statecode:1, statuscode:2} |
| Validate | version, env variables, connection references, flows on |
| Security roles | GET systemusers?$filter=domainname eq '..'; GET businessunits (root); GET roles?$filter=name eq '..' and _businessunitid_value eq ..; POST systemusers({id})/systemuserroles_association/$ref |

UAT → PROD: a managed solution cannot be exported, so the managed zip exported from DEV (stored per version in `SolutionArtifactStorePath`) is imported into PROD. Run DEV → UAT first for that version.

## 12. Error handling

| Situation | Behaviour |
|---|---|
| 429, 500, 502, 503, 504, timeout, connection error | retried `RetryCount` times, back-off `RetryDelaySeconds × 2^n`, Retry-After honoured (max 120 s) |
| 400, 404, 409 | not retried; 409 on create = "already exists" → re-read (idempotent) |
| 401, 403 | not retried; UnauthorizedAccessException → request **Failed** |
| Validation errors | form re-shown with messages; unattended → **Validation Failed** |
| Business rule (movement not allowed, missing solution/package, API limitation, unknown user) | BusinessRuleException → request **Rejected**, requester mailed |
| System error (API down, bug) | request **Failed**, CoE mailed, stack trace in the robot log only |
| Deployment verification fails | UiPath: previous process version restored, then Failed. Power Automate: Failed with "fix forward or uninstall" (managed downgrade not possible) |
| SMTP failure | warning only, never fails the request |

## 13. Logging

* Robot log: Info for each step, Trace for request URLs, Warn/Error with status, category, error code, correlation ID. `Add Log Fields` adds RequestID, Entity, RPATool, ProcessType, SourceEnvironment, TargetEnvironment to every line, so Orchestrator / Kibana can filter per request.
* Audit log: `<RequestStore>\audit\audit-yyyyMMdd.jsonl`, one JSON line per operation (timestamp, level, requestId, entity, tool, workflow, operation, status, message, machine, user).
* Request register: `<RequestStore>\requests\<RequestID>.json` with full status history.
* Never logged: passwords, client secrets, tokens, credential values, secret assets. Request bodies with secrets are written as `<not logged>`; summaries are masked; `project.json` `excludedLoggedData` hides `*password*`, `*secret*`, `*token*`, `sec*`, `Private:*`.

## 14. Test cases

Executed here against a stateful mock of 3 Orchestrators, Identity Server, Entra ID, Dataverse and SMTP (see section 19). **85 of 85 end-to-end checks and 28 of 28 form checks pass.** Logs are in `verification/`.

| # | Scenario | Expected / observed |
|---|---|---|
| 1 | UiPath DEV → UAT, attended, first submit invalid | form re-shown with message; Completed; folder, 6 assets (secret as Credential), queue, machine template assigned, user + BOT roles, process at source version 1.0.4, history Submitted → … → Completed, 2 mails |
| 2 | Same request again | Completed, no duplicate folder/asset/queue/machine/process, no create calls |
| 3 | UAT → PROD, AD user, injected 503×2 and 429 | retried and Completed; AD user added via AssignDomainUser; folder + tenant roles merged; PROD machine slots from sheet |
| 4 | Update with wrong version after update | Failed, process rolled back to previous version, failure mail |
| 5 | Power Automate DEV → UAT | managed import, env variables, connection reference, flows on, Dataverse role, artifact stored, no Orchestrator calls |
| 6 | Power Automate UAT → PROD | stored DEV artifact imported, PROD values, no export from UAT |
| 7 | Missing solution / flow cannot be turned on | Rejected / Failed with fix-forward message |
| 8 | Movement not allowed, DEV → PROD, tampered policy role | Validation Failed, no Request ID |
| 9 | Unknown user, directory add disabled | Rejected, "API limitation: …" |
| 10 | Wrong Orchestrator / Entra secret | Failed, token requested once (no retry) |
| 11 | Cancel button, window closed | Cancelled, nothing created |
| 12 | 40 parallel Request IDs | all unique |
| 13 | HandleApiError 400/401/403/404/409/429/500/502/503 | correct category and transient flag; 403 throws UnauthorizedAccessException |
| 14 | Secret scan of robot log, console, audit, register and mails | no secret or token found |
| 15 | Submission produced by the real form in Chromium | passes server validation, Completed |
| Form | entity options, roles per entity, movement filtering, tool and process-type sections, calculated folders/BOT/PA URLs, required fields, DEV → PROD rule, role rule, submit, cancel, validation alert | all pass |

UAT test plan for your environment: run 1, 2, 3 (with a real AD account), 5, 6 and 8 against DEV/UAT Orchestrators and Dataverse sandbox environments before PROD.

## 15. Deployment

1. Orchestrator (each of DEV/UAT/PROD): Admin → External Applications → add a confidential application with the application scopes in the Environments sheet (add `PM.RobotAccount*` only if `CreateMissingRobotAccounts = True`). Create custom roles named in the Roles sheet (`RPA Support`, `RPA Read Only`, `RPA Tenant Monitor`) or rename them in the sheet.
2. Create the credential assets (section 4) in `Shared/RPA Platform` in the Orchestrator where the robot runs; give its robot account View permission on that folder only.
3. Entra: app registration with a client secret; in each Dataverse environment add it as application user with System Customizer; create connections in UAT/PROD and put their IDs in PA_ConnectionReferences.
4. Fill Config.xlsx and EntityConfiguration.json (URLs, client IDs, tenant ID, folders, SMTP). Put `RequestStorePath` and `SolutionArtifactStorePath` on a share the robot can write and requesters cannot.
5. Open the project in Studio 2025.10.17, restore packages, import the form JSON into Create Form (section 5), run Main once in Debug against DEV.
6. Publish to Orchestrator; create the process in the folder of the requesters' attended robots (Assistant). For unattended use, start Main with `in_RequestJson`.

## 16. Production readiness checklist

- [ ] Studio opens every XAML without validation errors; form imported and saved
- [ ] All `/swagger`-only endpoints (section 10) confirmed on your 2025.10 Orchestrator
- [ ] Separate client per environment, least scopes, secrets only in credential assets, rotation date recorded
- [ ] `RequireHttps = True`, `AllowDevToProd = False` (unless approved)
- [ ] Request store and artifact store on a protected share with backup
- [ ] SMTP relay tested; NotificationEmail mailboxes exist
- [ ] Roles sheet reviewed by security (policy role → Orchestrator / Dataverse role per environment)
- [ ] Custom roles exist in each Orchestrator; Dataverse roles exist in each environment
- [ ] Test cases 1-15 run in DEV/UAT
- [ ] Log level Info in PROD; Kibana/ELK or Insights dashboard on RequestID
- [ ] Runbook: how to re-run a Failed request (same form JSON, unattended), how to uninstall a bad managed solution
- [ ] Change approval: who may submit UAT → PROD (Orchestrator folder permissions on the requester process)

## 17. Operations notes

* Re-running a Failed request is safe: everything is idempotent.
* Adding an entity: add it to EntityConfiguration.json (and Roles/Assets rows if it needs new ones). No workflow change.
* Status values: Submitted, Validation Failed, In Progress, Provisioning, Deployment In Progress, UAT, Production, Completed, Failed, Rejected (plus "Cancelled" as Main output only; nothing is stored).

## 18. API limitations (no documented API; supported alternative used)

| Need | Limitation | What the solution does |
|---|---|---|
| Create local Orchestrator users | Orchestrator 2025.10 standalone documents no API for local users | Directory users/groups via AssignDomainUser; otherwise request Rejected with "API limitation" and the manual step |
| Secret asset type | create-body for the newer Secret asset type is not documented | stored as Credential asset (`SecretAssetMode = Credential`) |
| Export from UAT (managed) | managed solutions cannot be exported | reuse the DEV-exported managed artifact |
| Create connections | OAuth connections need interactive sign-in | pre-create connections; map IDs in PA_ConnectionReferences |
| Roll back a managed solution | downgrade is blocked | fail with "fix forward or uninstall" |
| Add a person to Dataverse | users come from Entra sync | must already exist (environment security group); roles are then assigned by API |

## 19. What was verified, and what was not

**Verified here**
* All 49 XAML files are generated from one model together with an equivalent VB.NET module. That module (same expressions, same Invoke Code bodies) compiles with Option Strict On and was run end to end against the mock services: 85/85 checks.
* A static check confirms every Invoke Workflow argument exists in the target with the right direction and type, and every Assign target has the right type.
* The form JSON was rendered with form.io 4.14 in Chromium: 28/28 checks, and its submission was processed by the back end.
* The Config.xlsx guard, masking and secret scan.

**Not verified (cannot be done from here)**
* Opening the project in UiPath Studio 2025.10.17. The XAML follows Studio's format (namespaces, references, ViewState-free), but attribute names of Read Range, Message Box, Add Log Fields, Get Robot Credential and especially Create Form (`FormFieldsCollection`, `FormFieldsInputData`, `FormFieldsOutputData`, `SelectedButton`) were written from documentation and earlier projects, not from Studio. If Studio flags one, replace that activity from the toolbox with the same expressions.
* Importing the JSON into the UiPath Form Designer (UiPath Forms is form.io-based; custom JS and data sources are standard form.io).
* Live Orchestrator 2025.10 and Dataverse. The mock follows documented shapes; the Swagger-only endpoints in section 10 must be checked in your instance.
* Real AD lookup (PowerShell ADSI) and a real SMTP relay with TLS.
