# BUGS.md

Top 4 findings, prioritized by business impact, based on live code review of the running app and API layer.

## 1) Critical: SQL injection in the task search endpoint
- File / line ref: `src/app/api/projects/[id]/tasks/route.ts:15-35`
- Category: Security
- Severity: Critical
- Description: The `GET` task-search route reads the `q` query parameter and interpolates it directly into a SQL string passed to `prisma.$queryRawUnsafe`. Because the user-controlled value is not parameterized or escaped, a crafted value such as `x' OR 1=1 --` can bypass the project filter and return unrelated tasks. This creates a direct data exposure risk for project content and a high-confidence injection vulnerability.
- AI confirmation:
  - Prompt: "Can a malicious `q` value break out of the task search SQL and return all rows in the project?"
  - Response: "Yes. The code builds a raw SQL string with `WHERE project_id = '${projectId}' AND (title ILIKE '%${q}%' ...)` and passes it to `$queryRawUnsafe`, so injection is possible if `q` includes a quote or SQL operator."

## 2) Critical: unauthorized task mutations were not consistently blocked
- File / line ref: `src/app/api/tasks/[id]/route.ts:15-52`, `src/app/api/projects/[id]/tasks/route.ts:44-61`
- Category: Security
- Severity: Critical
- Description: The original task update flow did not verify the authenticated user's project membership or task-edit permission before performing a patch. As a result, an unauthorized user could edit tasks in a project they were not allowed to change. The create route also had inconsistent auth checks, creating a second path for unauthorized task creation and making the permission model unreliable.
- AI confirmation:
  - Prompt: "Before the fix, did the task update endpoint validate role and membership before changing a task?"
  - Response: "No. The PATCH route updated the task immediately after parsing the payload and only checked for the task's existence; it did not verify the caller's project membership or `canEditTasks` role before writing the record."

## 3) High: the UI still exposes task-creation and task-edit controls to viewers
- File / line ref: `src/app/projects/[id]/page.tsx:77-112`, `src/components/TaskDetail.tsx:18-103`
- Category: Security
- Severity: High
- Description: The project page renders the "add a task" section and the task detail modal, including save and delete actions, for every authenticated user. Because the front end does not check the user's role before showing mutating controls, viewers can still see the workflow and attempt restricted actions even though server-side authorization should block them.
- AI confirmation:
  - Prompt: "If I log in as a viewer and open a project page, do I still see task creation or edit controls in the UI?"
  - Response: "Yes. The page unconditionally renders the add-task form and the `TaskDetail` modal includes save/delete buttons, so the UI does not hide mutating actions based on project role."

> When a viewer opens a project page, the app shows the add-task form and edit/delete task controls, but it should hide those controls or disable them unless the user is a project member or admin.

## 4) Medium: member emails are exposed in project detail views
- File / line ref: `src/app/api/projects/[id]/route.ts:19-34`, `src/app/projects/[id]/page.tsx:128-136`
- Category: Security
- Severity: Medium
- Description: The project detail response includes the full `owner` and `memberships.user` objects, and the UI renders each member's email directly. That makes personal contact data visible to any user who can open the project, even when the product only needs display names and roles. This creates a privacy and data-handling issue, especially in collaborative or cross-team environments.
- AI confirmation:
  - Prompt: "Does the project detail page reveal each member's email address in the UI, and does the API include that user object?"
  - Response: "Yes. The API includes `user: true` in the project memberships, and the UI serializes `{m.user.email} · {m.role}` under the 'members' section, so email addresses are exposed in the project view."

Summary: The biggest business risks are unauthorized task changes and raw-SQL injection, followed by a UI/permission mismatch and unnecessary exposure of user contact details. These issues are all high-impact because they affect project integrity, confidentiality, and operator trust.
