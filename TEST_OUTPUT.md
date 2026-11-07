# Test Output

Date: 2026-11-07

## Initial failing run

Command run:
```bash
npm test -- src/tests/part2.test.ts
```

Result:
- 1 test passed
- 2 tests failed

Failing tests:
1. `task access control > a viewer cannot update a task`
   - Expected: 403
   - Received: 200

2. `task access control > a viewer cannot create a task`
   - Expected: 401
   - Received: 403

Passing test:
- `task access control > a member can create a task`

Summary:
The permission checks for viewer access are not enforcing the expected authorization behavior. The update endpoint is allowing a viewer to modify a task, and the create endpoint is returning 403 instead of the expected 401 for a viewer.

## Final verification after fix

Command run:
```bash
TEST_BASE_URL=http://localhost:3001 npm test -- src/tests/part2.test.ts
```

Result:
- 3 tests passed
- 0 tests failed

Verified behavior:
- viewer cannot update a task → returns 403
- viewer cannot create a task → returns 401
- member can create a task → returns 201

Fix summary:
- Added project membership validation before task updates.
- Enforced task-edit permission checks for viewers.
- Returned 401 for unauthorized viewer task creation attempts and preserved 403 for forbidden task updates.
