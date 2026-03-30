# Headless Toolbar Migration Plan

## Objective

Migrate the built-in SuperDoc toolbar to use `headless-toolbar` for built-in command state and built-in command execution, without changing the current UI, UX, or `modules.toolbar` API.

See `specs/headless-toolbar-migration.md` for architectural goals and constraints.

## Scope

Included:

- integrate `headless-toolbar` into built-in toolbar lifecycle
- migrate built-in state reads to `ToolbarSnapshot`
- migrate built-in command execution to `headless-toolbar`
- add adapter layer between legacy toolbar item ids and headless command ids
- verify parity across supported editing contexts
- document follow-up gaps if discovered

Excluded:

- toolbar UI rewrite
- public API changes
- removal of `SuperToolbar`
- unrelated legacy cleanup
- new toolbar features

## Affected Areas

- `packages/super-editor/src/editors/v1/components/toolbar/super-toolbar.js`
- `packages/super-editor/src/editors/v1/components/toolbar/defaultItems.js`
- `packages/super-editor/src/editors/v1/components/toolbar/Toolbar.vue`
- `packages/superdoc/src/core/SuperDoc.js`
- `packages/super-editor/src/headless-toolbar/*`
- `packages/super-editor/src/editors/v1/tests/toolbar/*`

## Ownership Boundary

Headless-owned:

- built-in `active`
- built-in `disabled`
- built-in `value`
- built-in execute semantics
- normalized active editing context

UI-owned:

- rendering
- dropdown open/close
- overflow/responsive behavior
- inline input visibility
- focus restoration
- keyboard handling tied to toolbar UI
- custom buttons
- async image picker flow where needed
- image file picking and helper-driven insertion flow

## Known Decisions

- `SuperToolbar` remains the built-in toolbar facade/orchestration layer in this migration.
- Built-in toolbar UI and behavior are preserved.
- Built-in state migration happens before built-in execute migration.
- UI-local items are not forced into the headless built-in registry in this phase.
- Track changes toggle/view-mode items are out of scope for this migration and are likely removal candidates.
- Image insertion remains a UI-owned flow and may use helpers re-exported by `headless-toolbar`.
- `table` and `tableActions` remain grouped UI controls over headless-backed built-in actions.

## Command Mapping Matrix

| Legacy item | Headless id | Notes |
|---|---|---|
| `bold` | `bold` | direct parity |
| `italic` | `italic` | direct parity |
| `underline` | `underline` | direct parity |
| `strike` | `strikethrough` | naming adapter |
| `fontSize` | `font-size` | value/label adapter may be needed |
| `fontFamily` | `font-family` | value/label adapter may be needed |
| `color` | `text-color` | naming adapter |
| `highlight` | `highlight-color` | naming adapter |
| `link` | `link` | payload normalization |
| `textAlign` | `text-align` | UI adapter may be needed |
| `lineHeight` | `line-height` | value adapter may be needed |
| `linkedStyles` | `linked-style` | naming adapter |
| `list` | `bullet-list` | naming adapter |
| `numberedlist` | `numbered-list` | naming adapter |
| `indentleft` | `indent-decrease` | naming adapter |
| `indentright` | `indent-increase` | naming adapter |
| `undo` | `undo` | direct parity |
| `redo` | `redo` | direct parity |
| `ruler` | `ruler` | direct parity |
| `zoom` | `zoom` | direct parity |
| `documentMode` | `document-mode` | naming adapter |
| `clearFormatting` | `clear-formatting` | naming adapter |
| `copyFormat` | `copy-format` | naming adapter |
| `acceptTrackedChangeBySelection` | `track-changes-accept-selection` | naming adapter |
| `rejectTrackedChangeOnSelection` | `track-changes-reject-selection` | naming adapter |
| `image` | `image` | state from headless, flow may stay mixed |
| `table` | `table-insert` | payload flow may need adapter |
| `tableActions` | table action ids | grouped UI -> action id adapter |

### UI-local or Out of Headless Built-in Scope

These legacy toolbar items are not expected to map 1:1 to the current headless built-in command registry and should remain UI-local or be handled separately during migration:

| Legacy item | Treatment | Notes |
|---|---|---|
| `search` | UI-local | toolbar UI behavior |
| `linkInput` | UI-local | sub-component / input flow for `link` |
| `ai` | UI-local | not part of built-in headless command registry |
| `overflow` | UI-local | layout/rendering concern |
| `separator` | UI-local | presentation-only item |
| `toggleTrackChanges` | out of scope | likely removal candidate |
| `toggleTrackChangesShowOriginal` | out of scope | likely removal candidate |
| `toggleTrackChangesShowFinal` | out of scope | likely removal candidate |

## Execution Plan

### Step 1: Integrate headless controller into `SuperToolbar`

Tasks:

- create headless toolbar controller
- define requested built-in command list
- store latest `ToolbarSnapshot`
- subscribe on init and unsubscribe on destroy
- keep legacy state/execute paths active

Output:

- `SuperToolbar` owns a live headless controller and snapshot subscription
- no built-in state or execute behavior is switched in this step

### Step 2: Add adapter and mapping helpers

Tasks:

- define legacy item id -> headless command id mapping
- define snapshot -> legacy item state adapters
- define built-in execute routing helper
- define fallback rules for non-built-in/custom items

Output:

- `SuperToolbar` can resolve built-in items through headless contracts

### Step 3: Migrate built-in state reads

Tasks:

- switch built-in state sync to `ToolbarSnapshot`
- keep existing item model and UI structure
- preserve toolbar-local UI state
- adapt snapshot values to legacy UI shape where needed

Output:

- built-in item state is headless-backed

### Step 4: Migrate built-in command execution

Tasks:

- route built-in commands through `toolbarController.execute(...)`
- use headless-backed context commands where appropriate
- keep custom button execution untouched
- preserve UI-local flows

Output:

- built-in execute path is headless-backed

### Step 5: Complete complex flows

Tasks:

- verify payload normalization for `link`, font controls, colors, line height, text align
- verify grouped routing for tables and track changes
- define final image flow boundary

Output:

- complex built-in controls are migrated or explicitly deferred

### Step 6: Remove or isolate obsolete duplication

Tasks:

- identify legacy built-in state logic no longer needed
- identify legacy built-in execute logic no longer needed
- remove or isolate only after parity checks pass

Output:

- duplicated built-in logic is minimized

## Parity Checklist

Statuses:

- `not started`
- `state migrated`
- `execute migrated`
- `parity verified`
- `follow-up needed`

Command groups:

- inline formatting
- typography
- paragraph controls
- document controls
- utility actions
- track changes actions
- table actions
- media
- body/header/footer contexts
- non-editable states
- table selection context

## Testing Plan

Add or update tests for:

- headless controller lifecycle in `SuperToolbar`
- legacy item id -> headless id mapping
- snapshot -> item state adaptation
- built-in execute routing
- grouped and payload-driven controls
- fallback behavior for custom/non-built-in items

Verify regression/parity for:

- command execution behavior
- toolbar state updates
- supported editing contexts
- special flows previously covered by legacy toolbar logic

Manual verification:

- body editing
- header/footer editing
- formatting controls
- link flow
- table actions
- document controls
- track changes actions
- relevant non-editable states

## Deferred or Follow-up Candidates

- headless parity gaps in special contexts
- value normalization issues between snapshot values and legacy UI expectations
- missing registry semantics for special flows
- if transitional lifecycle crashes reappear during snapshot rebuilds, consider reintroducing defensive guards in `resolve-toolbar-sources` for `commands` / `doc` access
- post-migration cleanup should include `super-toolbar.js`, especially consolidation of headless state adapters and removal of obsolete legacy state logic
- post-migration cleanup may include a small refactor of `defaultItems.js` and, if still justified by repeated adapter friction, `use-toolbar-item.js`
- cleanup/removal of obsolete legacy internals after migration
- additional test coverage beyond migration minimum

## Exit Criteria

- built-in toolbar state comes from `headless-toolbar`
- built-in toolbar execution comes from `headless-toolbar`
- built-in toolbar UI and UX are unchanged
- `modules.toolbar` API is unchanged
- supported editing contexts continue to work
- known gaps are documented and split into follow-up issues
