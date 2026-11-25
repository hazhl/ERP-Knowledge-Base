# ERP Knowledge Base — System Design

**Document status:** Proposed MVP architecture  
**Deployment strategy:** Local development first; centralized company deployment after MVP approval  
**Target platform:** Windows desktop

## 1. Architecture overview

The ERP Knowledge Base uses a Tauri desktop client, an ASP.NET Core Web API, and Microsoft SQL Server. **Desktop clients communicate only with the API, never directly with SQL Server.** The API is responsible for authentication, authorization, validation, business rules, search, file access, and audit logging.

```text
Tauri 2 desktop application
(React + TypeScript + Tailwind CSS)
                  |
                  | HTTP on loopback for local development;
                  | HTTPS for company deployment
                  v
          ASP.NET Core Web API (C#)
                  |
                  | Entity Framework Core
                  v
              SQL Server

API --------> Protected attachment storage
```

The same application architecture and database model will be used locally and on the company server. The move to central hosting should require deployment configuration, infrastructure setup, a planned data migration, and validation—not a rewrite of the application.

### 1.1 Local-first development and MVP demonstration

- The Tauri application, ASP.NET Core API, SQL Server, and attachment storage run on the developer's Windows computer.
- The client calls an API bound to the loopback interface (`127.0.0.1` / `localhost`); SQL Server is accessible only to the local API and authorized development tools.
- Local development can use HTTP **only on loopback**. Use HTTPS for remote access or any shared pilot.
- This phase is intended for development and a controlled MVP demonstration, **not simultaneous access from multiple employees' PCs**.
- The local database is the single source of truth. Back up its data and attachment files before meaningful schema changes and before migration.
- Do not use real customer credentials, production secrets, or unapproved personal/customer data in test cases or attachments.

### 1.2 Company-server deployment

After management and IT approval:

- The ASP.NET Core API runs on a company-managed server, accessed through an approved HTTPS endpoint.
- SQL Server and attachment storage are centrally hosted on approved company infrastructure and accessible to the API, not desktop clients.
- Employees and managers run the Tauri desktop app on their own Windows computers and connect to the central API over the company LAN, VPN, or another approved access path.
- The application has **no offline editing or automatic cross-device synchronization** in the MVP. A network connection to the API is required for shared use.
- API endpoint settings differ by environment; no production hostname, database password, or secret is hardcoded into the desktop executable.

## 2. Technology choices

| Layer | Selected technology | Responsibility |
|---|---|---|
| Desktop shell | Tauri 2 | Windows desktop application and limited native integration |
| Frontend | React + TypeScript | Screens, forms, case editing, search, and manager workflows |
| Styling | Tailwind CSS | Consistent, responsive desktop UI |
| Localization | Frontend translation library (e.g., react-i18next) | English/Arabic labels and right-to-left layout |
| Backend | ASP.NET Core Web API + C# | API endpoints, authentication, permissions, validation, and workflows |
| Database access | Entity Framework Core | SQL Server access, transactions, and schema migrations |
| Database | Microsoft SQL Server | Central relational data and revision history |
| Authentication | ASP.NET Core Identity with an API-compatible session/token design | Account credentials, roles, and protected API access |
| Search | Parameterized SQL search initially; evaluate SQL Server Full-Text Search | Search published case fields, keywords, and error messages |
| Attachments | API-managed files + SQL Server metadata | Secure upload, authorization, download, and version association |
| API protocol | REST over HTTP(S), JSON | Communication between desktop app and API |
| Source control | Git and approved GitHub/company repository | Version control, reviews, and tracked changes |
| Initial distribution | Tauri Windows executable/package, subject to IT checks | Manual distribution without an obligatory installer |

Pin a supported .NET/EF Core release, compatible SQL Server version, Tauri release, Node.js toolchain, and frontend dependencies during implementation. Use development or properly licensed SQL Server editions as appropriate; **SQL Server Developer Edition must not be used for production**.

## 3. Authentication, account lifecycle, and authorization

### 3.1 Account registration and approval

1. An employee submits a work email and password using the registration form.
2. The API validates the email against a configurable approved company domain/policy, normalizes it, checks for duplicates, and securely stores the account using ASP.NET Core Identity.
3. The account remains **Pending** and cannot access protected knowledge-base functions.
4. An authorized manager approves or rejects the request. Approved accounts become **Active** and receive the Employee role. Rejected accounts remain unable to sign in to protected functions.
5. A manager can deactivate an Active account. Deactivation blocks further protected access without deleting historical cases, decisions, and audit records.
6. Registration, approval, rejection, and deactivation are audited. Authentication/registration endpoints may remain accessible as needed without granting knowledge-base access.

**First manager bootstrap:** The first manager account must be provisioned securely by the project administrator through a controlled setup or seed process. It must not be created by public registration or by a hardcoded default password. Later manager-role changes require authorized administration.

The MVP uses Identity-managed credentials. Enterprise SSO/Entra ID integration is **out of scope** unless company policy requires it before deployment. Work email validation does not by itself prove ownership of an address; before production, implement company-approved email verification, invitations, or administrator identity verification.

### 3.2 Sessions and token handling

- Use ASP.NET Core Identity with a clearly selected API authentication mechanism. Prefer short-lived bearer access tokens for the Tauri client; decide the specific token implementation during backend setup.
- Access credentials must be stored in protected Windows OS-backed storage where supported, **not in plaintext files or WebView local storage**. Keep access tokens in memory where practical.
- Protect refresh credentials if used. Their rotation, expiration, and server-side revocation must be implemented and tested.
- **Logout must remove local credentials and revoke server-side sessions/refresh credentials if implemented.** A token is not automatically invalidated merely because the UI signs out; short-lived access tokens may remain valid until expiry unless server-side invalidation is implemented.
- Account status and permissions must be checked server-side for protected requests; deactivated users must be denied even if they have an unexpired access token.
- Use HTTPS for all company-hosted traffic, and enforce sensible password, lockout, failed-sign-in logging, and rate-limiting policies.

### 3.3 Role and resource permissions

| Action | Employee | Manager |
|---|---|---|
| Register / request access | Yes | Yes, via controlled provisioning if needed |
| View/search published cases | Yes | Yes |
| Create a new case | Yes | Yes |
| Edit a case they originally created | Yes; publish new revision directly | Yes |
| Propose an edit to another author's case | Yes; requires manager review | Optional, according to manager workflow |
| Approve/reject another employee's edit request | No | Yes |
| Approve/reject registrations | No | Yes |
| Deactivate accounts | No | Yes |
| View audit logs and identifiable case authorship | No | Yes |
| Access attachments | Published-case attachments only, or own permitted submissions | As authorized for review and published cases |

**Author anonymity:** Employees must not see the identity, email, user ID, or other identifying metadata of authors of other employees' cases. The API must return role-appropriate response models; hiding a name in React is insufficient. Managers may view authorship and audit records. An employee may see that they own a case (and may access their own history), without exposing other authors through search results, revision history, download metadata, or API responses.

Every protected API action must independently enforce role, account status, and case ownership. A desktop client cannot be trusted to enforce permissions.

## 4. Case lifecycle and documentation requirements

### 4.1 Required case fields

| Field | Required | Notes |
|---|---|---|
| Case title | Yes | Searchable |
| ERP module/category | Yes | Select an active module |
| Problem description | Yes | Searchable |
| Error message | No | Searchable if provided |
| Root cause | No | Plain text/rich text, as supported |
| Solution | Yes | Practical resolution |
| Troubleshooting steps | Yes | Ordered/reproducible steps |
| SQL query/script | No | Render as a distinct formatted code block |
| Explanation of code | No | Separate from the code |
| Keywords | Yes | One or more searchable terms |
| ERP version | No | Optional environment context |
| Verification steps | Yes | How the employee confirmed resolution |
| Warnings/risks | No | Cautions, side effects, prerequisites |
| Attachments | No | API-controlled files |
| Related cases | No | References to other published cases |

The API—not only the frontend—must validate all required fields, lengths, allowable attachments, and referenced IDs. A new case is published after valid submission; editing one's own case produces a new published revision. Drafts are **not included** in this MVP unless added in a separate requirement change.

Render stored SQL/script content as **text**, never execute it from the knowledge base. Escape untrusted content and sanitize any supported rich text. Do not permit attachment names, code, or case descriptions to inject executable HTML/JavaScript.

### 4.2 Revisions and edit requests

1. Each case has a stable identity and a pointer to its **current published revision**.
2. Each revision contains a complete snapshot of the documentation. Revision **content is immutable** once submitted; publication/review state is tracked separately or via controlled status metadata.
3. The initial valid submission creates published revision 1.
4. An original author can edit their case by creating the next published revision directly; earlier published versions remain available.
5. Editing another author's case creates a proposed revision plus an edit request, with the current published revision recorded as its **base revision**. The proposal is invisible in normal employee search until approved.
6. Pending requests appear in the manager’s in-app review queue. **Review request** opens a split screen with the base published revision in one pane and the proposed revision in the other. The manager can approve or reject. Rejection requires nonblank feedback. The API retains the decision, reviewer, timestamp, and feedback.
7. Approval publishes the proposed snapshot as the next version and advances the current-revision pointer. Rejection preserves the current version and stores the feedback. For either outcome, the API writes the decision, applicable publication changes, audit event, and private in-app notification in one transaction.
8. If the current published revision has changed since a proposal was submitted, **approval must be blocked as a conflict**. The manager/requester can compare changes and submit an updated proposal against the current revision; no silent overwrite.
9. Publishing—including an author's direct edit—must use an optimistic concurrency check and a database transaction to avoid lost updates or duplicate published revision numbers.
10. Pending/rejected proposals and identifying reviewer/author information remain accessible only to authorized users. Employees can inspect published history without learning the identities of other authors.

### 4.3 In-app review and decision notifications

The manager queue lists pending edit requests for authorized managers. The review endpoint returns the base and proposed snapshots. If the base is stale, the UI flags the conflict and the API blocks approval. No decision notification is created for an unsuccessful approval attempt.

The employee’s notification inbox shows approved or rejected outcomes for their own requests. Rejection notifications display the manager’s feedback directly and link to the request. The API checks recipient ownership for reading or marking a notification as read. The MVP uses the queue and inbox for delivery. Email and direct messages are outside its scope.

## 5. Logical database design

The schema below is a logical design. SQL types, indexes, foreign-key behavior, and final table names will be determined through Entity Framework Core migrations.

| Entity | Important data and relationships |
|---|---|
| `AspNetUsers` (extended Identity user) | User ID, normalized email, credential data managed by Identity, account status, creation/decision dates; no plaintext passwords |
| `AspNetRoles` / `AspNetUserRoles` | Employee and Manager roles and their assignments |
| `RegistrationRequests` | Requester, Pending/Approved/Rejected status, submitted date, reviewing manager, decision date, optional note |
| `Modules` | ERP module ID, English/Arabic display names as applicable, active status |
| `Cases` | Stable case ID, original author ID, created timestamp, current published revision ID, concurrency token (`rowversion`), optional administrative state |
| `CaseRevisions` | Revision ID, case ID, submitted-by ID, submission timestamp, snapshot of all case form fields, base revision reference when applicable, published number if published |
| `CaseEditRequests` | Case ID, proposed revision ID, requester ID, base published revision ID, Pending/Approved/Rejected state, reviewer, review time, decision note required on rejection |
| `Notifications` | Recipient user, edit request, creation time, optional read time. The request supplies the outcome and rejection feedback. One notification per decided request. |
| `CaseRevisionKeywords` | Revision ID and normalized keyword, enabling searchable/filterable keywords per revision |
| `CaseRevisionRelatedCases` | Revision ID and related case ID, preserving links as of that revision |
| `Attachments` | Revision ID, FileContent, sanitized/display filename, MIME type, byte size, uploaded-by ID, upload date, optional checksum |
| `AuditLogs` | Event ID, actor ID (nullable for relevant system actions), action, target type/ID, outcome, UTC timestamp, safe metadata |

### 5.1 Key relationships and constraints

- One user can create many cases; **original case authorship is immutable** for ownership checks.
- One case has many revisions; at most one revision is designated as its current published revision, and it must belong to that case.
- One edit request references one proposed revision, one case, a requester, and its base published revision. Prevent duplicate processing of a reviewed request.
- Published revision numbers are unique **within each case**; pending proposals need not have a published revision number.
- A revision is associated with zero or more attachments, keywords, and related cases. Access to those records follows the revision's visibility rules.
- Use foreign keys and appropriate delete restrictions so disabling an account does not destroy cases or audit history.
- Use UTC for server-side timestamps; format displayed dates/times for the user's locale.
- A SQL Server `rowversion` on `Cases`, checked within a transaction, can support optimistic concurrency. The API must also compare each proposed edit's base revision with the current revision before publishing.

Published content should be retrieved through the `Cases.CurrentPublishedRevisionId` pointer, not by assuming the latest inserted revision is published.

### 5.2 Audit logs and sensitive information

Audit registration decisions, account activation/deactivation, case creation, direct revisions, proposals, review decisions, and relevant administrative changes. Logs should contain event identity, outcome, actor, target, and timestamp, but **must not record passwords, tokens, raw connection strings, sensitive attachment contents, or unnecessary customer details**. Restrict log access to managers/authorized administrators and define retention with company IT.

## 6. Search, localization, and attachments

### 6.1 Search and filtering

- Search **only currently published case content** in the ordinary case search. Exclude pending and rejected proposals.
- Search title, problem description, error message, solution, troubleshooting steps, and keywords; optional fields such as root cause may also be indexed.
- Filter by ERP module/category and available metadata such as ERP version.
- Start with parameterized SQL/EF Core text matching plus indexes appropriate to available filters. For larger datasets, evaluate SQL Server Full-Text Search after testing word breaking, Arabic/English queries, partial identifiers, and exact error codes.
- SQL Server Full-Text Search is **not guaranteed to match error substrings or SQL identifiers** the way users expect. Provide an exact/substring fallback for error-message searches where needed.
- Test searches in Arabic and English, including case-insensitive and expected normalization behavior. Never concatenate user input directly into SQL statements.

### 6.2 Arabic and English UI

- Provide a language selector with English and Arabic translations for screens, validation messages, and manager workflows.
- Apply `dir="rtl"` to Arabic UI containers; test navigation, tables, forms, and icons in both directions.
- Keep SQL/code blocks left-to-right and monospaced regardless of selected UI language.
- Case text itself may be written in Arabic, English, or both; store Unicode text in SQL Server (for example, `nvarchar` fields).
- Test mixed Arabic/English error messages and identifiers without distorting them.

### 6.3 File attachments

- Upload and download attachments through authenticated API endpoints. The API checks that the user has permission to access the associated case revision.
- Store the original file bytes in SQL Server as FileContent VARBINARY(MAX). Store the display filename, validated content type, and file size alongside the bytes. Save the file and its metadata in the same database transaction.
- Validate the extension, file size, content type, and file signature where applicable. Set a conservative file type allowlist and upload size limit before piloting. Treat uploaded filenames as untrusted display metadata.
- Associate each attachment with a specific case revision. A new revision can carry forward permitted attachments and include new uploads. Retain the attachments associated with earlier revisions so their history remains intact.
- Allow access to unpublished proposal attachments only to the submitter and authorized reviewers.
- Return attachment metadata in case responses and fetch the file bytes only when an authorized user opens or downloads an attachment. Follow company malware-scanning and retention policies where applicable.
- Include attachment bytes and metadata in SQL Server backups. Verify that a restored database can serve attachments for both current and historical revisions.

## 7. Desktop application distribution

**An installer is not required for the initial MVP.** A standalone Tauri executable or appropriately packaged ZIP may be used for an internal pilot **if tested on target Windows computers and permitted by company IT**.

- Build the Windows application with Tauri; a no-bundle build can produce the executable without an MSI/NSIS installer.
- Verify the Microsoft Edge WebView2 Runtime is available on recipient PCs, along with any application-specific runtime requirements.
- Test on a clean, non-developer Windows workstation before sharing. Do not assume copying an `.exe` automatically installs prerequisites.
- The desktop package contains only the UI/client. **It does not include the centrally hosted API or SQL Server** and must connect to the configured API endpoint.
- For a local demonstration on the developer's laptop, the API and database run there. Do not point colleagues at `localhost`, which would mean *their own* computers.
- Windows security controls may block unsigned/unapproved executables. Have company IT approve the pilot method; consider code signing or an installer if company policy requires it.
- Record each distributed app version and API compatibility. Automatic updates are outside the MVP.

## 8. Migration from local MVP to company server

The local database is authoritative until a deliberate cutover. There is **no automatic sync** between local and company databases.

1. **Prepare:** Approve the hosting plan with IT; provision the API host, supported/licensed SQL Server target, attachment storage, TLS certificate, database service identity, DNS, and firewall rules.
2. **Check compatibility:** Confirm backup/restore compatibility between SQL Server versions and that the selected API build matches the source database's EF Core migration history.
3. **Rehearse:** Test restoration and attachment copying into a non-production environment; validate sign-in, roles, revision history, and searches.
4. **Freeze source writes:** Stop edits/registrations on the local MVP, then take a verified full backup of SQL Server and a consistent copy of attachments. Keep both unchanged as the rollback source.
5. **Restore:** Restore the local database backup to the destination SQL Server. **Do not first create an unrelated database schema and then restore over it.** Restored databases already contain schema and EF migration history.
6. **Apply schema updates only if needed:** If the production API requires additional migrations beyond the restored schema, review and apply them **after restoration**, using an approved migration script/process and a database backup.
7. **Transfer attachments:** Copy the files while preserving their storage keys, or update their references through a controlled migration. Verify counts and sample downloads.
8. **Configure services:** Install/publish the API, set protected connection strings/secret configuration and storage paths, configure HTTPS and authorization, and initialize the required service permissions.
9. **Validate:** Test approved/pending/deactivated accounts, login/logout, manager bootstrap/roles, case search, author anonymity, revision approval/conflict handling, audit logs, file downloads, and Arabic/English screens.
10. **Cut over:** Configure distributed Tauri clients to use the central HTTPS API endpoint. Confirm connections from representative employee/manager PCs and formally designate the company database as the single source of truth.
11. **Rollback plan:** Keep the source backup for an agreed retention period. If production accepts new writes, rollback requires a separate reconciliation plan; simply pointing users back to the old local database would lose those writes.

## 9. Operations, security, and non-goals

### 9.1 Operational assumptions

- Local development uses loopback access; production uses TLS/HTTPS and an approved company network path.
- Database users and permissions follow least privilege. The application connects with a restricted service identity, not a shared administrator login.
- Secrets, certificates, token keys, and production connection strings are managed outside Git and outside the client bundle.
- Production backups include SQL Server data, audit/revisions, and all referenced attachment files. Restoration must be tested.
- Set API request/attachment limits, error handling, structured logging without secrets, and a basic health check before wider deployment.
- Company IT determines production hosting, licensing, backup retention, code signing, endpoint protection, and access policy.

### 9.2 Explicitly out of scope for the MVP

- AI-powered search or automatic solution generation.
- Direct ERP integrations or automatic case capture.
- Enterprise SSO/Entra ID integration unless mandated by company policy.
- Advanced notifications, real-time collaboration, or automatic desktop updates.
- Offline editing, client-to-client synchronization, and mobile application support.
- Export to PDF and user-managed drafts, unless later added to the approved MVP requirements.

## 10. Deployment diagram

```text
LOCAL DEVELOPMENT / MVP DEMO

Developer's Windows PC
  Tauri desktop client
           | HTTP loopback (localhost)
           v
  ASP.NET Core Web API
       |          |
       v          v
  SQL Server   Local attachment folder


COMPANY DEPLOYMENT (AFTER APPROVAL)

Employee / Manager Windows PCs
  Tauri desktop clients
           |
           | HTTPS over approved company LAN / VPN
           v
Company-managed API host
  ASP.NET Core Web API
       |          |
       |          +----> Protected attachment storage
       v
  SQL Server (private company network)
```

## 11. MVP design verification checklist

- [ ] A registration request cannot use protected functions before manager approval.
- [ ] The first manager is provisioned securely; an employee cannot self-assign Manager.
- [ ] Deactivation blocks access even if an earlier access token exists.
- [ ] Employee API responses never expose other case authors' identities.
- [ ] A case cannot be submitted without its required documentation fields.
- [ ] The original author can publish a new revision while preserving older revisions.
- [ ] Another employee's proposed revision does not replace published content before approval.
- [ ] Managers can open a pending request and compare its base and proposal side by side.
- [ ] Rejection requires feedback. Either decision creates exactly one private in-app notification for the requester, and the rejection notification shows the feedback.
- [ ] Stale proposals cannot silently overwrite newer published revisions.
- [ ] Only published cases appear in normal search results; keyword/error-message and module filtering work.
- [ ] Arabic UI renders RTL while SQL/code remains LTR.
- [ ] Attachments require authorization and remain associated with the correct revision.
- [ ] Manager decisions and important case/account actions produce safe audit records.
- [ ] A local demo works without exposing the development API on the network.
- [ ] A shared-client test uses the central API address, not `localhost`.
- [ ] A backup-and-restore rehearsal preserves users, revisions, logs, and files.

## 12. References for implementation

- [Tauri 2 configuration and bundling](https://v2.tauri.app/reference/config/)
- [Tauri Windows distribution and WebView2](https://v2.tauri.app/distribute/windows-installer/)
- [ASP.NET Core Identity API authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-api-authorization)
- [Entity Framework Core migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)
- [SQL Server database restore](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-a-database-to-a-new-location-sql-server)
