# Contribution [#]: [Issue Title]

**Contribution Number:** [1]  

**Student:** Hana Lee

**Issue:** [add part button disappear if grid style is changed to parametric view](https://github.com/inventree/InvenTree/issues/11385)

**Status:** [Phase III] [Complete]

---

## Why I Chose This Issue

I chose issue #11385 in inventree/InvenTree, "add part button disappear if grid style is changed to parametric view" because it's a focused UI consistency bug that I can approach confidently with my React/TypeScript background, and it sits inside a real inventory management system used in production.

I'm interested in this because:

- Missing UI elements based on view state are typically a conditional rendering problem, an area I have hands-on experience debugging
- The fix likely involves understanding how InvenTree manages toolbar actions across different view modes, which will teach me how this codebase handles shared UI state at a component level
- Inventory and parts management systems are new domain territory for me, so understanding how the parametric view differs architecturally from the table view is a learning goal in itself

From reading the issue, the "Add Part" button is rendered correctly in Table View but not carried over to Parametric View, suggesting the two views don't share a common toolbar or action layer. My contribution will make the button available consistently across views, respecting existing permission logic.
---

## Understanding the Issue

### Problem Description

From reading the issue, the "Add Part" button is rendered correctly in Table View but not carried over to Parametric View, suggesting the two views don't share a common toolbar or action layer. My contribution will make the button available consistently across views, respecting existing permission logic.

### Expected Behavior

The "Add Part" button should show up in both Table View and Parametric View.

### Current Behavior

The "Add Part" button ONLY shows up in Table View.

### Affected Components

Frontend - 
src/frontend/src/tables/part/PartTable.tsx
src/frontend/src/tables/part/ParametricPartTable.tsx

---

## Reproduction Process

### Environment Setup

1. **Backend dev server**
   - Created the Python virtual environment and installed dependencies via the project
     task runner: `invoke dev.setup-dev` (or `invoke install`), then `invoke dev.server`.
   - Loaded demo/sample data so the Parts categories actually contain parametric
     templates (otherwise the Parametric View renders with no parameter columns and is
     hard to eyeball): `invoke dev.import-records` / the bundled sample dataset.
2. **Frontend dev server**
   - `cd src/frontend`
   - `yarn install`
   - `yarn run dev` (Vite dev server, proxies API calls to the backend).
3. Logged in as an **admin / superuser** account so that `hasAddRole(UserRoles.part)`
   evaluates to `true` and the Add control is expected to be present.


| Challenge | Resolution |
| --- | --- |
| Parametric View showed no parameter columns at first, making it unclear whether the page even loaded. | Switched to a category that has active parameter templates (e.g. the demo "Electronics → Capacitors / Resistors" categories) so the parametric table renders meaningfully. |
| Needed to confirm the Add button is *permission*-gated, not globally removed. | Verified in `PartTable.tsx` that the Add dropdown is hidden only via `hidden={!user.hasAddRole(UserRoles.part)}`, so an admin should always see it. |
| Locating the divergence between the two views. | Traced the `SegmentedControlPanel` in `CategoryDetail.tsx` — each view renders a *different component* (`PartListTable` vs `ParametricPartTable`), which is the source of the inconsistency. |

### Steps to Reproduce

1. Start the backend and frontend dev servers and log in as an admin.
2. Navigate to a Part **Category** detail page that contains parts, e.g.
   `part/category/<id>/parts`.
3. Open the **Parts** tab. The view defaults to **Table View** — note the **Add**
   (➕ "Add Parts") button in the table toolbar.
4. Use the segmented control to switch to **Parametric View**.
5. **Observed:** the Add button is gone from the toolbar. Only per-row actions
   ("Add Parameter") remain, so there is no way to create a new part from this view.
6. **Expected:** the Add button should remain visible/enabled based on the user's
   `part` add permission, exactly as in Table View.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/hanielee/InvenTree/tree/fix-issue-add-part-button
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** 
Two different components back the same "Parts" panel:

- **Table View** → `PartListTable` (`src/frontend/src/tables/part/PartTable.tsx`)
- **Parametric View** → `ParametricPartTable`
  (`src/frontend/src/tables/part/ParametricPartTable.tsx`), which wraps the generic
  `ParametricDataTable` (`src/frontend/src/tables/general/ParametricDataTable.tsx`).

`PartListTable` builds a `tableActions` array (lines ~398–457) that includes an
`ActionDropdown` with the **Create Part / Import** actions, gated by
`hidden={!user.hasAddRole(UserRoles.part)}`, and passes it into `InvenTreeTable`.

`ParametricDataTable` only ever sets `rowActions` (the per-row "Add Parameter" action,
lines ~430–447) and **never accepts or renders any `tableActions`**. `ParametricPartTable`
likewise passes no add action. Therefore the toolbar-level **Add** button simply does not
exist in Parametric View — it is not a permission problem, it is a *missing wiring*
problem. Both views are mounted by the `SegmentedControlPanel` in
`src/frontend/src/pages/part/CategoryDetail.tsx` (lines ~282–309).

**Root cause:** `ParametricDataTable` provides no mechanism to surface custom
table-level actions, and `ParametricPartTable` never supplies an "Add Part" action, so
the button is absent regardless of the user's permissions.

**Match:** The existing pattern to copy is already in this codebase:

- **Where the button is built correctly:** `PartTable.tsx`, the `tableActions` `useMemo`
  — an `ActionDropdown` keyed `add-parts-actions`, icon `<IconPlus />`,
  `hidden={!user.hasAddRole(UserRoles.part)}`, whose first action ("Create Part") opens a
  `useCreateApiFormModal` pointed at `ApiEndpoints.part_list`.
- **How a generic table accepts caller-supplied extensions:** `ParametricDataTable`
  already accepts `customColumns` and `customFilters` and merges them
  (`tableColumns = [...customColumns, ...parameterColumns]`). We follow the **same
  prop-injection convention** by adding a `customActions` prop and merging it into the
  `InvenTreeTable` `tableActions`.

**Plan:**
1. **`ParametricDataTable.tsx`** — add an optional prop
   `customActions?: ReactNode[]` to the component's props type, and pass
   `tableActions={customActions}` (defaulting to `[]`) into the `InvenTreeTable`
   `props` object, mirroring how `customColumns` / `customFilters` are already threaded
   through. This makes the generic parametric table capable of showing toolbar actions
   without coupling it to the Part model.
2. **`ParametricPartTable.tsx`** — build the same "Add Parts" `ActionDropdown` used by
   `PartListTable`:
   - bring in `useUserState`, `useCreateApiFormModal`, `usePartFields`,
     `ActionDropdown`, `UserRoles`, `ModelType`, `ApiEndpoints`, `IconPlus`;
   - create the part-creation modal (`useCreateApiFormModal` → `ApiEndpoints.part_list`
     with `initialData: { category: categoryId }` so the new part defaults into the
     current category);
   - assemble a `customActions` array containing the `ActionDropdown`
     (`hidden={!user.hasAddRole(UserRoles.part)}`), and pass it plus the modal element
     into `ParametricDataTable`.
3. **No backend changes** — this is purely a frontend wiring fix; permissions are already
   enforced by the API and reflected by `user.hasAddRole`.
4. **Keep scope minimal** — reuse the existing permission gate (`hasAddRole`) so behavior
   matches Table View exactly; do not introduce a new permission concept.

**Implement:** https://github.com/hanielee/InvenTree/tree/fix-issue-add-part-button

**Review:** 
- **`CONTRIBUTING.md` / docs/develop/contributing:** feature/bugfix work goes on a
  **separate branch off `master`**, one concern per branch/PR — this fix qualifies as a
  single bugfix branch. No pushing directly to `master`.
- **Pre-commit:** run the configured `pre-commit` hooks (formatting/linting) before
  committing — the repo enforces style automatically on commit. For the frontend,
  ensure `yarn run lint` / type-check (`tsc`) pass.
- **No DB migrations** involved (frontend-only), so the "migrations must be committed"
  rule does not apply here.
- **Commit / PR conventions:** clear, scoped commit messages; PR title describing the
  bug and fix (e.g. "[UI] Keep Add button visible in Parametric part view"). Reference
  the original issue in the PR description. Match the `[UI]` prefix style seen in recent
  frontend commits.
- **Self-review checklist:** confirm the new prop is optional (no breakage for other
  `ParametricDataTable` callers), confirm imports are used, confirm the permission gate
  matches Table View, and run the frontend test suite.


**Evaluate:** InvenTree's frontend uses **Playwright** end-to-end tests
(`src/frontend/tests/`). The relevant spec is
`src/frontend/tests/pages/pui_part.spec.ts`, which already exercises the parametric view
via the `showParametricView(page)` helper (`src/frontend/tests/helpers.ts`).

Planned verification:

1. **Automated (Playwright):** extend the existing
   `Parts - Parameters by Category` test (or add a focused test) to, after
   `showParametricView(page)`, assert that the **Add** part action is visible for an
   admin — e.g. `await page.getByLabel('action-button-add-parts').waitFor()` (matching
   the toolbar action's accessible label/role) — and ideally assert it is *absent* for a
   read-only user, mirroring the existing Table View expectation.
2. **Run the suite:** `cd src/frontend && yarn run test` (Playwright) plus the
   type-check/lint steps to ensure nothing else regresses.
3. **Manual verification:** repeat the Steps to Reproduce above and confirm the Add
   button now persists across the Table ↔ Parametric toggle, that clicking it opens the
   "Add Part" form pre-filled with the current category, and that the button is hidden
   for a user lacking the `part` add role.
4. **Regression check:** confirm Table View is unchanged and that other consumers of
   `ParametricDataTable` (which pass no `customActions`) render exactly as before.

---

## Testing Strategy

InvenTree's frontend uses Playwright end-to-end tests. All new tests are added to
`src/frontend/tests/pages/pui_part.spec.ts`, following the same patterns as the existing
`Parts - Parameters by Category` test. `readeruser` was added to the imports from
`'../defaults'` to support the permission test.

### Unit Tests

- [ ] Test case 1: Admin sees Add button in Parametric View
      `test('Parts - Add button visible in Parametric View (admin)')`

Logs in as the default allaccess user, navigates to `part/category/4/parts`, switches
to Parametric View via `showParametricView(page)`, then asserts the button with
`aria-label="action-menu-add-parts"` is visible. Directly tests the reported bug.
- [ ] Test case 2: Add button opens Create Part form
`test('Parts - Add button opens Create Part form in Parametric View')`

Same setup. Clicks the Add Parts dropdown, clicks the "Create Part" menu item
(`aria-label="action-menu-add-parts-create-part"`), asserts the "Add Part" modal
appears, then dismisses. Confirms the modal is correctly wired, not just that the button renders.
- [ ] Test case 3:  Reader user does NOT see Add button
      `test('Parts - Add button hidden in Parametric View (reader)')`

Logs in as `readeruser` (username: `reader` / password: `readonly`). Switches to
Parametric View and asserts the Add Parts button has count 0. Verifies the permission
gate (`hasAddRole`) still works — the button is hidden, not just disabled.

### Integration Tests

- [ ] Create part via Parametric View submits to API
This test exercises the complete flow: open the Add Part modal from Parametric View,
fill in the name and description, submit the form, wait for the network to settle, then
assert the new part name is visible on the resulting page — confirming the API created
the record. Cleans up before and after using `deletePart()` (which calls the API
directly) to leave the database state unchanged.

This is the integration scenario: it validates that the modal's `initialData` is
correctly forwarded to `ApiEndpoints.part_list`, the API accepts the payload, and the
frontend navigates to the new part on success (`follow: true` in the modal config).

### Manual Testing

1. Started backend (`invoke dev.server`) and frontend (`yarn run dev`) with demo data loaded.
2. Logged in as admin, navigated to `part/category/4/parts` (Electronics — has parametric templates).
3. Confirmed Add button is visible in **Table View**.
4. Switched to **Parametric View** — Add Parts button now appears in the toolbar.
5. Clicked Add Parts → Create Part — modal opens pre-filled with the current category.
6. Switched back to Table View — no regression, button still present.
7. Logged in as `reader` — Add Parts button absent in both views (permission gate working).
8. Navigated to `part/category/5/parts` — button appears in Parametric View there too.

---

## Implementation Notes

### Week [3] Progress

The fix was completed in two files. The core insight was that `ParametricDataTable` — the generic component that backs Parametric View — had no way to accept toolbar-level actions from its callers; it only supported per-row actions. The solution follows the existing `customColumns` / `customFilters` prop-injection pattern already present in the component.

1. **`ParametricDataTable.tsx`** — added an optional `customActions?: ReactNode[]` prop to the component interface and wired it into the `InvenTreeTable` call as `tableActions: customActions ?? []`. The default of `[]` means every existing caller of `ParametricDataTable` (none of which pass this prop) is completely unaffected.

2. **`ParametricPartTable.tsx`** — imported `useUserState`, `useCreateApiFormModal`, `usePartFields`, `ActionDropdown`, `UserRoles`, `IconPlus`, and `t`. Constructed a `newPart` modal via `useCreateApiFormModal` pointing at `ApiEndpoints.part_list`, pre-seeding `initialData: { category: categoryId }` so new parts default into the current category. Assembled a memoized `tableActions` array containing an `ActionDropdown` gated by `user.hasAddRole(UserRoles.part)` — matching the exact permission gate used in Table View — and passed it to `ParametricDataTable` via the new `customActions` prop.

**Key decisions:**
- Reused the `customColumns`/`customFilters` convention rather than inventing a new pattern — keeps the diff minimal and stays idiomatic with the codebase.
- Kept the Add dropdown to "Create Part" only (no file import or supplier import actions) — those require additional wizard state that belongs in `PartListTable`, not a generic wrapper. Scope is limited to the reported bug.
- Used `hidden={!user.hasAddRole(UserRoles.part)}` (not `!user.hasAddPermission(ModelType.part)`) to match Table View's existing behavior exactly.

**Challenges faced:**
- Confirming this was not a permissions bug: traced `hasAddRole` call in `PartTable.tsx` and verified the role check returns `true` for admins in both views — the button was simply never rendered, not hidden by a permission check.
- Determining the right `initialData` for the new part modal: passing `{ category: categoryId }` ensures the form defaults to the category the user is currently browsing, which is the expected UX.
### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** `src/frontend/src/tables/general/ParametricDataTable.tsx`: Added `customActions?: ReactNode[]` prop; passed to `InvenTreeTable` as `tableActions`,`src/frontend/src/tables/part/ParametricPartTable.tsx`: Added Add Part modal, `ActionDropdown` toolbar action, and `customActions` prop wiring |
- **Key commits:** https://github.com/hanielee/InvenTree/tree/fix-issue-add-part-button
- **Approach decisions:** - **Why not modify `CategoryDetail.tsx`?** The bug is that `ParametricDataTable` has no way to surface add actions — fixing it there is a symptom fix. The root fix is making the generic component extensible, so any future parametric table (not just parts) can expose toolbar actions too.
- **Why keep the dropdown wrapper?** `PartListTable` uses an `ActionDropdown` around the "Create Part" action to allow future actions (import from file, import from supplier) to be added without changing the toolbar layout. Matching that structure keeps the two views visually consistent and leaves room for those actions to be added later.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
