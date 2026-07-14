```
　.˚　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚
　　　　　　　　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡
　　　　　　　⋆｡°✩  open source contribution log  ✩°｡⋆
　　　　　　　　　hana lee  · ai 301
　　　　　　　　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡　⋆₊˚⊹♡
　.˚　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚　.　✦　.　˚
```

# Contribution #1: Add part button disappears in Parametric View

| | |
|---|---|
| **Student** | Hana Lee |
| **Issue** | [#11385 — add part button disappear if grid style is changed to parametric view](https://github.com/inventree/InvenTree/issues/11385) |
| **Branch** | [`fix-issue-add-parts-button`](https://github.com/hanielee/InvenTree/tree/fix-issue-add-parts-button) |
| **Pull Request** | [inventree/InvenTree#12392](https://github.com/inventree/InvenTree/pull/12392) |
| **Status** | ![Status](https://img.shields.io/badge/phase%20iv-iterating-white?labelColor=ffb6c1&style=flat-square) |

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
src/frontend/src/tables/general/ParametricDataTable.tsx
src/frontend/src/components/items/PartCreationMenu.tsx   (added post-review)
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

The final implementation went further than a single "Create Part" action — it brings Parametric View to full parity with Table View's Add dropdown:

| Action | Behavior |
|---|---|
| **Create Part** | Opens the standard Add Part modal, pre-filled with the current category |
| **Import from File** | Opens an importer session modal, pre-seeded with the current category, then hands off to the global importer overlay |
| **Import from Supplier** | Opens the existing `ImportPartWizard`, shown only if a supplier plugin is active |

A `refreshRef` was also threaded through `ParametricDataTable` so the table refreshes automatically once an import session completes.

### Post-Review Refactor: `PartCreationMenu`

The first version of the fix duplicated the entire "Add Parts" dropdown (modals, wizard, `ActionDropdown`) inside `ParametricPartTable`, on top of the copy that already existed in `PartListTable`. A maintainer flagged this during review:

> "The combination of 'create new part' / 'import parts' is a repeated pattern here — I think it would be worth offloading this to a common component e.g. `PartCreationMenu` which can be used in both locations."

In response, the entire dropdown (Create Part, Import from File, Import from Supplier, and their associated modals/wizard) was extracted into a new shared component, `src/frontend/src/components/items/PartCreationMenu.tsx`, parameterized by `categoryId`/`initialData`, `basePartInstance`, `enableImport`, and `refreshRef`. Both `PartListTable` and `ParametricPartTable` now render `<PartCreationMenu />` instead of maintaining separate copies of the same logic.

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

Add the `customActions` prop, plus a `refreshRef` so callers can trigger a table refresh externally (needed once an import session finishes):

```tsx
customActions?: ReactNode[];
refreshRef?: MutableRefObject<() => void>;

useEffect(() => {
  if (refreshRef) {
    refreshRef.current = table.refreshTable;
  }
}, [refreshRef, table.refreshTable]);

// inside InvenTreeTable props:
tableActions: customActions,
```

**Step 2 — `ParametricPartTable.tsx`**

Add imports for the "Add Parts" dropdown, the Add Part modal, the file importer, and the supplier wizard:

```tsx
import { UserRoles } from '@lib/enums/Roles';
import { t } from '@lingui/core/macro';
import { IconFileUpload, IconPackageImport, IconPlus } from '@tabler/icons-react';
import { useMemo, useRef } from 'react';
import { ActionDropdown } from '../../components/items/ActionDropdown';
import ImportPartWizard from '../../components/wizards/ImportPartWizard';
import { dataImporterSessionFields } from '../../forms/ImporterForms';
import { usePartFields } from '../../forms/PartForms';
import { useCreateApiFormModal } from '../../hooks/UseForm';
import { usePluginsWithMixin } from '../../hooks/UsePlugins';
import { openGlobalImporter } from '../../states/ImporterState';
import { useUserState } from '../../states/UserState';
```

Set up the three modals/wizards:

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

const importParts = useCreateApiFormModal({
  url: ApiEndpoints.import_session_list,
  title: t`Import Parts`,
  fields: importSessionFields,
  onFormSuccess: (response) =>
    openGlobalImporter(response.pk, { onClose: tableRefreshRef.current })
});

const supplierPlugins = usePluginsWithMixin('supplier');
const importPartWizard = ImportPartWizard({ categoryId });
```

Assemble the dropdown, gating each action independently:

```tsx
const tableActions = useMemo(() => [
  <ActionDropdown
    key="add-parts-actions"
    tooltip={t`Add Parts`}
    icon={<IconPlus />}
    hidden={!user.hasAddRole(UserRoles.part)}
    actions={[
      { name: t`Create Part`, icon: <IconPlus />, onClick: () => newPart.open() },
      {
        name: t`Import from File`,
        icon: <IconFileUpload />,
        onClick: () => importParts.open(),
        hidden: !enableImport
      },
      {
        name: t`Import from Supplier`,
        icon: <IconPackageImport />,
        hidden: !enableImport || supplierPlugins.length === 0,
        onClick: () => importPartWizard.openWizard()
      }
    ]}
  />
], [user, enableImport, supplierPlugins, newPart.open, importParts.open, importPartWizard.openWizard]);
```

An `enableImport` prop (default `true`) lets callers opt out of both import actions and keep only "Create Part".

**Step 3** — No backend changes. Permissions are already enforced by the API via `user.hasAddRole`. `ImportPartWizard` is an existing component, reused as-is rather than rebuilt.

**Step 4** — Reuse the existing `hasAddRole` gate exactly, so behavior matches Table View with no new permission concepts introduced. The supplier import action only appears when a supplier plugin is actually installed and active.

---

#### I — Implement

Branch: https://github.com/hanielee/InvenTree/tree/fix-issue-add-parts-button
PR: https://github.com/inventree/InvenTree/pull/12392

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

### Playwright End-to-End Tests

All tests added to `src/frontend/tests/pages/pui_part.spec.ts`:

- [x] Admin sees the "Add Parts" button after switching to Parametric View
- [x] Clicking the button opens the Create Part modal with the current category pre-filled
- [x] A read-only user does not see the Add Parts button in Parametric View
- [x] Submitting the form from Parametric View creates the part in the database (round-trip integration test)

### Manual Testing

Tested against the InvenTree demo dataset:

- [x] Toggled between Table View and Parametric View multiple times — button persists correctly in both
- [x] Clicked "Add Parts" > "Create Part" in Parametric View — modal opens with category pre-filled
- [x] Clicked "Add Parts" > "Import from File" — importer session opens pre-seeded with the current category, table refreshes after the import closes
- [x] Confirmed "Import from Supplier" is hidden when no supplier plugin is active, and opens `ImportPartWizard` when one is
- [x] Logged in as a reader user — button is absent in both views
- [x] Table View behavior is unchanged
- [x] Other `ParametricDataTable` consumers (no `customActions` passed) render as before

### Known Gap

The Playwright suite still only covers "Create Part" (visibility, form open, permission gate, round-trip creation). "Import from File" and "Import from Supplier" were added after those tests were written and are not yet covered by automated tests — manual testing only so far.

---

## Implementation Notes

### Phase 1 Progress

Reproduced the bug, traced the root cause to missing `tableActions` wiring in `ParametricDataTable`, designed the fix following the existing `customColumns`/`customFilters` extension pattern, implemented all three file changes, and wrote four Playwright tests covering admin visibility, form opening, read-only hiding, and round-trip part creation.

### Phase 2/3 Progress

Expanded the "Add Parts" dropdown beyond "Create Part" to match Table View's full feature set: added "Import from File" (wired to `dataImporterSessionFields` and the global importer overlay) and "Import from Supplier" (reusing the existing `ImportPartWizard`, shown only when a supplier plugin is active). Added a `refreshRef` to `ParametricDataTable` so the table refreshes automatically after an import session closes. Added an `enableImport` prop so callers can opt a parametric table out of both import actions and keep only "Create Part".

### Phase 4 Progress

Submitted [PR #12392](https://github.com/inventree/InvenTree/pull/12392) against `inventree/InvenTree:master`. In review, a maintainer pointed out that the "Add Parts" dropdown was now duplicated between `PartListTable` and `ParametricPartTable`. Extracted the shared logic into a new `PartCreationMenu` component and updated both tables to use it, removing the duplicated modals/wizard/`ActionDropdown` code from each. See [Maintainer Feedback](#maintainer-feedback) below.

### Code Changes

| File | Change |
|---|---|
| `src/frontend/src/tables/general/ParametricDataTable.tsx` | Added `customActions?: ReactNode[]` and `refreshRef?: MutableRefObject<() => void>` props, wired to `tableActions` in `InvenTreeTable` |
| `src/frontend/src/components/items/PartCreationMenu.tsx` | **New.** Shared "Add Parts" dropdown + Create Part modal + file importer + `ImportPartWizard`, parameterized by `categoryId`/`initialData`, `basePartInstance`, `enableImport`, `refreshRef` |
| `src/frontend/src/tables/part/ParametricPartTable.tsx` | Replaced the inline "Add Parts" dropdown with `<PartCreationMenu />` |
| `src/frontend/src/tables/part/PartTable.tsx` | Replaced the inline "Add Parts" dropdown with `<PartCreationMenu />`, removing the now-duplicated modal/wizard code |
| `src/frontend/tests/pages/pui_part.spec.ts` | Added 4 Playwright E2E tests covering "Create Part" (visibility, form open, permission gate, round-trip creation) |

**Approach decision:** Followed the existing `customColumns`/`customFilters` prop-injection pattern rather than modifying the internals of `ParametricDataTable`, keeping the generic table decoupled from Part-specific logic. For the import actions, reused `ImportPartWizard` and `dataImporterSessionFields` as-is instead of building parallel import logic for Parametric View. After maintainer feedback, consolidated both tables' dropdowns into `PartCreationMenu` rather than leaving the duplication in place.

---

## Pull Request

**PR Link:** [inventree/InvenTree#12392](https://github.com/inventree/InvenTree/pull/12392)

**Status:** Iterating (addressed one round of review feedback, awaiting re-review)

**Title:** `[UI] Keep Add Parts button visible in Parametric View`

**PR Description:**

> Users lose the ability to add parts as soon as they switch a Part Category view from Table View to Parametric View. The "Add Parts" button simply disappears from the toolbar, with no error and no indication that anything is wrong — only the per-row "Add Parameter" action remains. Since Parametric View is meant to be a different way of *looking at* the same part list, not a different *feature set*, this is a functional regression for anyone who prefers or needs to work in that view.
>
> The root cause is architectural rather than a permissions bug: `PartListTable` (Table View) and `ParametricPartTable` (Parametric View) are two separate components, and only `PartListTable` builds a toolbar `tableActions` array. The generic `ParametricDataTable` that Parametric View wraps has no concept of table-level actions at all — it only supports per-row actions. So the button isn't hidden by a broken permission check, it was simply never wired up for that view.
>
> This PR closes #11385 by:
> - Adding an optional `customActions` prop to `ParametricDataTable`, following the same prop-injection convention already used for `customColumns`/`customFilters`.
> - Wiring the existing "Add Parts" dropdown (Create Part, Import from File, Import from Supplier) into `ParametricPartTable` via that prop, gated by the same `hasAddRole(UserRoles.part)` permission check Table View already uses.
> - Adding a `refreshRef` to `ParametricDataTable` so the table refreshes automatically after an import session completes.
> - **(post-review)** Extracting the dropdown into a shared `PartCreationMenu` component, used by both `PartListTable` and `ParametricPartTable`, removing the duplication a maintainer flagged.
>
> **Acceptance criteria**
> - [x] Tests added (Playwright, `pui_part.spec.ts`)
> - [x] Tests passing locally (`yarn run test`)
> - [x] Lint / type-check passing (`yarn run lint`, `tsc --noEmit`)
> - [x] No breaking changes — `customActions`/`refreshRef` are optional, existing `ParametricDataTable` callers unaffected
> - [ ] Automated test coverage for Import from File / Import from Supplier (tracked as a follow-up, currently manual-only)
>
> **Before:** Add button visible in Table View, absent in Parametric View.
> **After:** Add Parts dropdown (Create Part / Import from File / Import from Supplier) present and permission-gated identically in both views.
>
> _Screenshots of both views to be attached to the PR._
>
> Closes #11385.

> [!IMPORTANT]
> **TODO before final submission:** attach actual before/after screenshots (Table View vs. Parametric View, Add Parts dropdown open) to the PR and embed them here.

### Maintainer Feedback

| Date | Reviewer Comment | My Response |
|---|---|---|
| 2026-07-13 | "The combination of 'create new part' / 'import parts' is a repeated pattern here — I think it would be worth offloading this to a common component e.g. `PartCreationMenu` which can be used in both locations." | Extracted the dropdown, its modals, and the supplier wizard into a new `src/frontend/src/components/items/PartCreationMenu.tsx`, parameterized by `categoryId`/`initialData`, `basePartInstance`, `enableImport`, and `refreshRef`. Updated both `PartListTable` and `ParametricPartTable` to render `<PartCreationMenu />` instead of maintaining duplicate copies (built on top of the original fix in commit `1370a9d7f`). About to push the follow-up commit to the PR branch; awaiting re-review. |

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how InvenTree structures its frontend: `InvenTreeTable` is a shared base, and feature-specific tables like `PartListTable` wrap it with actions, columns, and filters. Understanding this layering was key to identifying where the fix belonged.
- Practiced the prop-injection extension pattern used throughout this codebase (`customColumns`, `customFilters`, now `customActions`) rather than hardcoding feature logic into shared components.
- Got hands-on with `useCreateApiFormModal` and how InvenTree's form hooks connect to API endpoints.
- Wrote Playwright tests for a real open source project, including a full round-trip integration test that creates and deletes a part.
- Learned to recognize duplication *while writing it* rather than only after review — copying the dropdown into `ParametricPartTable` was the fastest path to a working fix, but it wasn't the right end state, and a maintainer caught what I should have caught myself.
- Practiced extracting a shared component (`PartCreationMenu`) from two call sites with slightly different needs (`PartListTable` passes `basePartInstance` for duplication, `ParametricPartTable` doesn't) — designing the prop surface so both callers stay simple.

### Challenges Overcome

The trickiest part early on was confirming this was a missing wiring issue rather than a permissions bug. The button is permission-gated in Table View, so if it was gone in Parametric View there were two possible explanations: either the permission check was failing, or the button was never rendered at all. Reading through `PartTable.tsx` and `ParametricDataTable.tsx` confirmed the latter.

Setting up the dev environment also took some effort. Parametric View renders blank without sample data that includes parameter templates, so the page looked broken before loading the demo dataset.

The second real challenge came after submitting the PR: taking the "offload this to a common component" feedback and actually doing it without breaking either caller. `PartListTable` passes a `basePartInstance` (for "duplicate part") and derives `initialData` from `props?.params`, while `ParametricPartTable` only ever needs `{ category: categoryId }`. Getting `PartCreationMenu`'s prop signature right so both tables could drop their duplicate code cleanly, instead of just moving the duplication one level up, took more thought than the original fix.

### What I'd Do Differently Next Time

Set up the full dev environment and load sample data before diving into the code — I spent time wondering if the page had actually loaded when the issue was just missing demo data. I'd also look harder, before opening the PR, at whether my fix was introducing duplication with an existing pattern (`PartListTable`'s dropdown) rather than waiting for a reviewer to point it out — the signal was there in the original diff (an almost line-for-line copy of `PartTable.tsx`'s `ActionDropdown`) and I should have caught it myself.

---

## Resources Used

- [InvenTree CONTRIBUTING.md](https://github.com/inventree/InvenTree/blob/master/CONTRIBUTING.md)
- [InvenTree issue #11385](https://github.com/inventree/InvenTree/issues/11385)
- [InvenTree PR #12392](https://github.com/inventree/InvenTree/pull/12392) — my pull request and review thread
- [`PartTable.tsx`](https://github.com/inventree/InvenTree/blob/master/src/frontend/src/tables/part/PartTable.tsx) — reference for the existing Add button pattern
- [Playwright docs: locators](https://playwright.dev/docs/locators)
- [InvenTree frontend tests](https://github.com/inventree/InvenTree/tree/master/src/frontend/tests)
