# ERP Knowledge Base — Project Requirements

## 1. Project scope

The ERP Knowledge Base will let employees record and find reusable ERP problem-solving cases. Managers will control account access, review proposed changes to other employees' cases, and monitor activity.

The first release (MVP) should be a practical, usable knowledge base. It will focus on essential documentation, access control, review, and discovery rather than enterprise integrations or advanced automation.

## 2. MVP definition

### Included in the MVP

| Feature | MVP scope |
|---|---|
| Employee registration | Employees can register using their work email and submit an access request. |
| Manager account approval | A manager can approve or reject pending registration requests. |
| Login and logout | Approved users can securely sign in and out. |
| Employee and manager roles | Access and available actions depend on the user's role. |
| Create documentation | Employees can create and submit new cases to the knowledge base. |
| Edit own documentation | Employees can edit cases they created. |
| Request edits to another employee's documentation | Employees can submit proposed changes for review; the published case remains unchanged until approval. |
| Manager review | Managers can approve or reject edit requests. Approved changes update the case; rejected changes do not. |
| Search | Users can search cases by title, keywords, problem description, and error message. |
| Filters and categories | Users can filter cases by ERP module/category and other available case metadata. |
| SQL and code blocks | Cases can include formatted SQL queries or scripts and an explanation of the code. |
| Audit logs | Managers can review a record of important account, case, and approval activity. |
| Arabic and English | The application supports both languages. Arabic screens should support right-to-left layout. |
| Basic attachments | Users can attach relevant files to a case. |

### Out of scope for the MVP

- AI-powered search
- ERP system integrations
- Advanced notification features

Basic notifications may be added later; they are not required for the first usable release.

## 3. User roles and permissions

### Employee

- Register and request access.
- Sign in and out after account approval.
- Search and view cases.
- Create cases and edit cases they created.
- Submit an edit request for a case created by someone else.
- Add relevant attachments to cases.

### Manager

- Approve or reject employee registration requests.
- Review, approve, or reject edit requests for other employees' cases.
- View audit logs and track knowledge-base activity.
- Deactivate accounts.
- Use employee capabilities where applicable.

## 4. User stories

### Employee stories

- As an employee, I want to register using my work email so that I can request access to the application.
- As an employee, I want to search documented cases so that I can find solutions to similar problems.
- As an employee, I want to document a problem and its solution so that the team can reuse my knowledge.
- As an employee, I want to include SQL queries or scripts with explanations so that others understand how the solution works.
- As an employee, I want to edit my own documentation so that I can correct or improve it.
- As an employee, I want to request changes to documentation created by others so that I can suggest improvements without changing the published case.
- As an employee, I want to search using error messages and keywords so that I can find relevant cases quickly.

### Manager stories

- As a manager, I want to approve or reject registration requests so that only authorized employees can access the knowledge base.
- As a manager, I want to review edit requests before changes are applied so that shared documentation remains accurate.
- As a manager, I want to view audit logs so that I can understand important activity in the knowledge base.
- As a manager, I want to deactivate accounts so that access can be removed when needed.
- As a manager, I want to track activity in the knowledge base so that I can monitor its use and contribution.

## 5. Case documentation form

When an employee selects **Add New Case**, the form should contain:

| Field | Required? |
|---|---|
| Case title | Yes |
| ERP module | Yes |
| Problem description | Yes |
| Error message | No |
| Root cause | No |
| Solution | Yes |
| Troubleshooting steps | Yes |
| SQL query / script | No |
| Explanation of code | No |
| Keywords | Yes |
| ERP version | No |
| Verification steps | Yes |
| Warnings / risks | No |
| Attachments | No |
| Related cases | No |

SQL queries and scripts must be displayed as code, separate from their explanatory text. Attachments are optional and limited to basic file attachment functionality in the MVP.

## 6. Core requirements

1. Registration requests must remain pending until a manager approves them; rejected or pending users cannot use approved-user functionality.
2. The application must enforce employee and manager permissions for protected actions.
3. Employees must be able to create cases and edit their own cases.
4. An employee's proposed change to another person's case must go through manager review. The published case must not change before approval.
5. Search must support case text, keywords, and error messages. Users must be able to narrow results using categories/ERP modules.
6. Case fields marked required above must be provided before a case can be submitted.
7. Important actions, including account decisions, case creation and edits, edit-request decisions, and account deactivation, must be recorded in audit logs.
8. The interface must be usable in English and Arabic, including right-to-left presentation for Arabic.

## 7. MVP success criteria

- An employee can request an account, receive manager approval, sign in, and sign out.
- An approved employee can create a case, find it through search or filters, and edit it.
- An employee can propose a change to another employee's case, and a manager can approve or reject it without losing the published version before approval.
- A manager can review registration requests, deactivate accounts, and inspect audit activity.
- Users can document and read and write cases in English or Arabic, including optional code and attachments.
