```
　.˚　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚
　　　　　　　　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡
　　　　　　　⋆｡°✩  open source contribution log  ✩°｡⋆
　　　　　　　　　hana lee  ·  inventree  ·  ai 301
　　　　　　　　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡
　.˚　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚
```

# Contribution #1: Add part button disappears in Parametric View

| | |
|---|---|
| **Student** | Hana Lee |
| **Issue** | [#11385 — add part button disappear if grid style is changed to parametric view](https://github.com/inventree/InvenTree/issues/11385) |
| **Branch** | [`fix-issue-add-part-button`](https://github.com/hanielee/InvenTree/tree/fix-issue-add-part-button) |
| **Status** | Phase 3 Complete |

---

## Why I Chose This Issue

I chose issue #11385 in inventree/InvenTree because it is a focused UI consistency bug I can approach confidently with my React/TypeScript background, and it sits inside a real inventory management system used in production.

- Missing UI elements based on view state are typically a conditional rendering problem, which is an area I have hands-on experience debugging.
- The fix involves understanding how InvenTree manages toolbar actions across different view modes, which taught me how this codebase handles shared UI state at a component level.
- Inventory and parts management systems are new domain territory for me, so understanding how the parametric view differs architecturally from the table view was a real learning goal.

---

## Understanding the Issue

### Problem Description

The "Add Part" button renders correctly in Table View but is not carried over to Parametric View. The two views do not share a common toolbar or action layer, so the button is absent in Parametric View regardless of the user's permissions. My contribution wires the button into Parametric View while respecting the existing permission logic.

### Expected vs. Current Behavior

| | Behavior |
|---|---|
| **Expected** | "Add Part" button is visible in both Table View and Parametric View |
| **Current** | "Add Part" button only appears in Table View |

### Affected Components

```
src/frontend/src/tables/part/PartTable.tsx
src/frontend/src/tables/part/ParametricPartTable.tsx
```

---

## Reproduction Process

### Environment Setup

**1. Backend dev server**

```bash
invoke dev.setup-dev   # or: invoke install
invoke dev.server
invoke dev.import-records   # load demo data so parametric categories are populated
```

> Without demo data, Parametric View renders with no parameter columns and appears broken.

**2. Frontend dev server**

```bash
cd src/frontend
yarn install
yarn run dev
```

**3.** Log in as an **admin/superuser** so that `hasAddRole(UserRoles.part)` evaluates to `true` and the Add control is expected to be visible.

**Challenges encountered:**

| Challenge | Resolution |
|---|---|
| Parametric View showed no parameter columns, making it unclear whether the page loaded at all. | Switched to a category with active parameter templates (e.g. the demo "Electronics: Capacitors/Resistors" categories) so the table renders meaningfully. |
| Needed to confirm the Add button is permission-gated, not globally removed. | Verified in `PartTable.tsx` that the Add dropdown is hidden only via `hidden={!user.hasAddRole(UserRoles.part)}`, so an admin should always see it. |
| Locating the divergence between the two views. | Traced the `SegmentedControlPanel` in `CategoryDetail.tsx` — each view renders a different component (`PartListTable` vs `ParametricPartTable`), which is the source of the inconsistency. |

### Steps to Reproduce

1. Start the backend and frontend dev servers and log in as an admin.
2. Navigate to a Part **Category** detail page, e.g. `part/category/<id>/parts`.
3. Open the **Parts** tab. The view defaults to **Table View** — note the **Add Parts** button in the toolbar.
4. Use the segmented control to switch to **Parametric View**.
5. **Observed:** the Add button disappears. Only per-row actions ("Add Parameter") remain with no way to create a new part.
6. **Expected:** the Add button stays visible based on the user's `part` add permission, exactly as in Table View.

---

## Solution Approach

### Root Cause Analysis

`ParametricDataTable` has no mechanism for table-level toolbar actions, and `ParametricPartTable` never supplies an "Add Part" action. This is not a permissions bug — the button is simply never rendered.

Both views are mounted by `SegmentedControlPanel` in `CategoryDetail.tsx` (lines 282-309), but they use completely separate components:

| View | Component | Has "Add" toolbar action? |
|---|---|---|
| Table View | `PartListTable` | Yes |
| Parametric View | `ParametricPartTable` | No |

`PartListTable` builds a `tableActions` array with the "Add Parts" dropdown. `ParametricDataTable` never accepts or renders `tableActions` at all.

### Proposed Solution

Add an optional `customActions` prop to `ParametricDataTable`, following the same pattern as the existing `customColumns` and `customFilters` props. Then in `ParametricPartTable`, build the same "Add Parts" `ActionDropdown` used by `PartListTable` and pass it in. No backend changes needed since permissions are already handled by `user.hasAddRole`.

---

### Implementation Plan (UMPIRE)

#### U — Understand

Two different components back the same "Parts" panel:

| View | Component |
|---|---|
| Table View | `src/frontend/src/tables/part/PartTable.tsx` |
| Parametric View | `src/frontend/src/tables/part/ParametricPartTable.tsx` → wraps `src/frontend/src/tables/general/ParametricDataTable.tsx` |

`PartListTable` builds a `tableActions` array (lines ~398-457) with an `ActionDropdown` gated by:

```tsx
hidden={!user.hasAddRole(UserRoles.part)}
```

`ParametricDataTable` only ever sets `rowActions` (per-row "Add Parameter", lines ~430-447) and never touches `tableActions`. The Add button is absent from Parametric View not because of a permission failure, but because nothing ever wires it up.

---

#### M — Match

The extension pattern already exists in the codebase. `ParametricDataTable` accepts `customColumns` and `customFilters` and merges them in:

```tsx
tableColumns = [...customColumns, ...parameterColumns]
```

The fix follows the exact same convention: add a `customActions` prop and pass it through to `InvenTreeTable`.

---

#### P — Plan

**Step 1 — `ParametricDataTable.tsx`**

Add the prop to the type signature and wire it through:

```tsx
customActions?: ReactNode[]

// inside InvenTreeTable props:
tableActions={customActions ?? []}
```

**Step 2 — `ParametricPartTable.tsx`**

Add imports:

```tsx
import { UserRoles } from '@lib/enums/Roles';
import { ActionDropdown } from '../../components/items/ActionDropdown';
import { usePartFields } from '../../forms/PartForms';
import { useCreateApiFormModal } from '../../hooks/UseForm';
import { useUserState } from '../../states/UserState';
import { IconPlus } from '@tabler/icons-react';
```

Create the modal:

```tsx
const newPart = useCreateApiFormModal({
  url: ApiEndpoints.part_list,
  title: t`Add Part`,
  fields: newPartFields,
  initialData: { category: categoryId },
  follow: true,
  modelType: ModelType.part,
  keepOpenOption: true
});
```

Build and pass the action:

```tsx
const tableActions = useMemo(() => [
  <ActionDropdown
    key="add-parts-actions"
    tooltip={t`Add Parts`}
    icon={<IconPlus />}
    hidden={!user.hasAddRole(UserRoles.part)}
    actions={[{
      name: t`Create Part`,
      icon: <IconPlus />,
      onClick: () => newPart.open()
    }]}
  />
], [user, newPart]);
```

**Step 3** — No backend changes. Permissions are already enforced by the API via `user.hasAddRole`.

**Step 4** — Reuse the existing `hasAddRole` gate exactly, so behavior matches Table View with no new permission concepts introduced.

---

#### I — Implement

Branch: https://github.com/hanielee/InvenTree/tree/fix-issue-add-part-button

---

#### R — Review

- Branch off `master`, one bugfix per branch/PR (no direct pushes to `master`)
- Run pre-commit hooks and frontend checks before committing:

  ```bash
  yarn run lint
  tsc --noEmit
  ```

- PR title follows the `[UI]` prefix convention used in recent frontend commits:
  `[UI] Keep Add Part button visible in Parametric View`
- Reference issue #11385 in the PR description
- Confirm the new `customActions` prop is optional so other `ParametricDataTable` callers are unaffected

---

#### E — Evaluate

InvenTree uses Playwright E2E tests in `src/frontend/tests/`. The parametric view is already exercised via the `showParametricView(page)` helper in `src/frontend/tests/helpers.ts`.

Verification plan:

1. **Playwright tests** — assert Add button is visible for admin, absent for reader, and run a round-trip form submission test
2. **Full suite:**
   ```bash
   cd src/frontend && yarn run test
   ```
3. **Manual** — toggle between Table and Parametric View, confirm button persists, click through to form, verify category pre-fill, test with a reader account
4. **Regression** — confirm Table View is unchanged and other `ParametricDataTable` consumers (no `customActions` passed) render as before

---

## Testing Strategy

All new tests are in `src/frontend/tests/pages/pui_part.spec.ts`. `readeruser` was added to the imports from `'../defaults'` to support the permission test.

### Playwright Tests

- [x] **Admin sees Add button in Parametric View** — `test('Parts - Add button visible in Parametric View (admin)')`

  Logs in as the default allaccess user, navigates to `part/category/4/parts`, switches to Parametric View via `showParametricView(page)`, then asserts the button with `aria-label="action-menu-add-parts"` is visible. Directly tests the reported bug.

- [x] **Add button opens Create Part form** — `test('Parts - Add button opens Create Part form in Parametric View')`

  Same setup. Clicks the Add Parts dropdown, clicks the "Create Part" menu item (`aria-label="action-menu-add-parts-create-part"`), asserts the "Add Part" modal appears, then dismisses. Confirms the modal is correctly wired, not just that the button renders.

- [x] **Reader user does not see Add button** — `test('Parts - Add button hidden in Parametric View (reader)')`

  Logs in as `readeruser` (username: `reader` / password: `readonly`). Switches to Parametric View and asserts the Add Parts button has count 0. Verifies the permission gate (`hasAddRole`) still works.

### Integration Test

- [x] **Create part via Parametric View submits to API** — `test('Parts - Create part via Parametric View submits to API')`

  Exercises the complete flow: open the Add Part modal from Parametric View, fill in the name and description, submit the form, wait for network idle, then assert the new part name is visible on the resulting page. Confirms the API created the record. Cleans up before and after using `deletePart()` to leave the database unchanged. Validates that `initialData` is forwarded to `ApiEndpoints.part_list`, the API accepts the payload, and the frontend navigates to the new part on success (`follow: true`).

### Manual Testing

1. Started backend (`invoke dev.server`) and frontend (`yarn run dev`) with demo data loaded.
2. Logged in as admin, navigated to `part/category/4/parts` (Electronics — has parametric templates).
3. Confirmed Add button is visible in **Table View**.
4. Switched to **Parametric View** — Add Parts button now appears in the toolbar.
5. Clicked Add Parts > Create Part — modal opens pre-filled with the current category.
6. Switched back to Table View — no regression, button still present.
7. Logged in as `reader` — Add Parts button absent in both views (permission gate working).
8. Navigated to `part/category/5/parts` — button appears in Parametric View there too.

---

## Implementation Notes

### Week 3 Progress

The fix was completed in two files. The core insight was that `ParametricDataTable` — the generic component that backs Parametric View — had no way to accept toolbar-level actions from its callers; it only supported per-row actions. The solution follows the existing `customColumns`/`customFilters` prop-injection pattern already present in the component.

1. **`ParametricDataTable.tsx`** — added an optional `customActions?: ReactNode[]` prop to the component interface and wired it into the `InvenTreeTable` call as `tableActions: customActions ?? []`. The default of `[]` means every existing caller of `ParametricDataTable` (none of which pass this prop) is completely unaffected.

2. **`ParametricPartTable.tsx`** — imported `useUserState`, `useCreateApiFormModal`, `usePartFields`, `ActionDropdown`, `UserRoles`, `IconPlus`, and `t`. Constructed a `newPart` modal via `useCreateApiFormModal` pointing at `ApiEndpoints.part_list`, pre-seeding `initialData: { category: categoryId }` so new parts default into the current category. Assembled a memoized `tableActions` array containing an `ActionDropdown` gated by `user.hasAddRole(UserRoles.part)` — matching the exact permission gate used in Table View — and passed it to `ParametricDataTable` via the new `customActions` prop.

**Key decisions:**

- Reused the `customColumns`/`customFilters` convention rather than inventing a new pattern — keeps the diff minimal and stays idiomatic with the codebase.
- Kept the Add dropdown to "Create Part" only (no file import or supplier import actions) — those require additional wizard state that belongs in `PartListTable`, not a generic wrapper. Scope is limited to the reported bug.
- Used `hidden={!user.hasAddRole(UserRoles.part)}` (not `!user.hasAddPermission(ModelType.part)`) to match Table View's existing behavior exactly.

**Challenges faced:**

- Confirming this was not a permissions bug: traced `hasAddRole` call in `PartTable.tsx` and verified the role check returns `true` for admins in both views — the button was simply never rendered, not hidden by a permission check.
- Determining the right `initialData` for the new part modal: passing `{ category: categoryId }` ensures the form defaults to the category the user is currently browsing, which is the expected UX.

### Code Changes

| File | Change |
|---|---|
| `src/frontend/src/tables/general/ParametricDataTable.tsx` | Added `customActions?: ReactNode[]` prop, wired to `tableActions` in `InvenTreeTable` |
| `src/frontend/src/tables/part/ParametricPartTable.tsx` | Added Add Part modal, `ActionDropdown` toolbar action, and `customActions` prop wiring |
| `src/frontend/tests/pages/pui_part.spec.ts` | Added 4 Playwright E2E tests |

**Approach decisions:**

- **Why not modify `CategoryDetail.tsx`?** The bug is that `ParametricDataTable` has no way to surface add actions — fixing it there is a symptom fix. The root fix is making the generic component extensible, so any future parametric table (not just parts) can expose toolbar actions too.
- **Why keep the dropdown wrapper?** `PartListTable` uses an `ActionDropdown` around the "Create Part" action to allow future actions (import from file, import from supplier) to be added without changing the toolbar layout. Matching that structure keeps the two views visually consistent.

---

## Pull Request

**Status:** Not yet submitted

**Draft title:** `[UI] Keep Add Part button visible in Parametric View`

**Draft description:**

> Fixes #11385. The "Add Part" button was absent in Parametric View because `ParametricDataTable` had no mechanism for table-level toolbar actions and `ParametricPartTable` never supplied one. This PR adds an optional `customActions` prop to `ParametricDataTable` (following the same pattern as `customColumns` and `customFilters`) and wires the existing permission-gated "Add Parts" `ActionDropdown` into `ParametricPartTable`. No backend changes. Four Playwright tests added.

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how InvenTree structures its frontend: `InvenTreeTable` is a shared base, and feature-specific tables like `PartListTable` wrap it with actions, columns, and filters. Understanding this layering was key to identifying where the fix belonged.
- Practiced the prop-injection extension pattern used throughout this codebase (`customColumns`, `customFilters`, now `customActions`) rather than hardcoding feature logic into shared components.
- Got hands-on with `useCreateApiFormModal` and how InvenTree's form hooks connect to API endpoints.
- Wrote Playwright tests for a real open source project, including a full round-trip integration test that creates and deletes a part.

### Challenges Overcome

The trickiest part was confirming this was a missing wiring issue rather than a permissions bug. The button is permission-gated in Table View, so if it was gone in Parametric View there were two possible explanations: either the permission check was failing, or the button was never rendered at all. Reading through `PartTable.tsx` and `ParametricDataTable.tsx` confirmed the latter.

Setting up the dev environment also took some effort. Parametric View renders blank without sample data that includes parameter templates, so the page looked broken before loading the demo dataset.

### What I'd Do Differently Next Time

Set up the full dev environment and load sample data before diving into the code. I spent time wondering if the page had actually loaded when the issue was just missing demo data.

---

## Resources Used

- [InvenTree CONTRIBUTING.md](https://github.com/inventree/InvenTree/blob/master/CONTRIBUTING.md)
- [InvenTree issue #11385](https://github.com/inventree/InvenTree/issues/11385)
- [`PartTable.tsx`](https://github.com/inventree/InvenTree/blob/master/src/frontend/src/tables/part/PartTable.tsx) — reference for the existing Add button pattern
- [Playwright docs: locators](https://playwright.dev/docs/locators)
- [InvenTree frontend tests](https://github.com/inventree/InvenTree/tree/master/src/frontend/tests)
