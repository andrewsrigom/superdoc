# Rebuild Built-in SuperDoc Toolbar on Headless Toolbar API

## Overview

This work migrates the built-in SuperDoc toolbar to use the `headless-toolbar` API as its underlying state and command integration layer.

The migration is internal. The current built-in toolbar UI, its behavior, and the public `modules.toolbar` configuration API must remain unchanged.

The goal is to make `headless-toolbar` the source of truth for built-in toolbar command state and command execution, while keeping the built-in toolbar implementation and consumer-facing API stable.

## Goals

- Migrate the built-in SuperDoc toolbar to use `headless-toolbar` for built-in command state.
- Migrate the built-in SuperDoc toolbar to use `headless-toolbar` for built-in command execution.
- Preserve existing toolbar UI and behavior across supported editing contexts.
- Preserve the current public toolbar configuration API via `SuperDoc.js` and `modules.toolbar`.
- Validate that the headless toolbar API is sufficient to support the current built-in toolbar in practice.
- Document any parity gaps or behavioral mismatches discovered during migration and split them into follow-up issues where appropriate.

## Non-Goals

- No redesign or rewrite of the built-in toolbar UI.
- No change to the public `modules.toolbar` API.
- No intentional behavior changes to the built-in toolbar UX.
- No removal of `SuperToolbar` in this phase.
- No requirement to fully remove all legacy toolbar internals if they still serve UI orchestration purposes.

## Current Architecture

Today, the built-in toolbar combines multiple concerns:

- built-in command state derivation
- built-in command execution logic
- Vue item/view-model construction
- toolbar-specific UI state and rendering
- toolbar lifecycle integration with `SuperDoc`

At the same time, `headless-toolbar` already provides:

- normalized active toolbar context
- built-in command state snapshot
- built-in command execution via registry-backed `execute()`
- subscription/lifecycle model for toolbar updates

This creates duplication between the built-in toolbar internals and the public headless toolbar model.

## Target Architecture

After migration:

- `headless-toolbar` is the source of truth for built-in command state.
- `headless-toolbar` is the source of truth for built-in command execution.
- `SuperToolbar` remains as a facade/orchestration layer for the built-in toolbar.
- Existing toolbar Vue components remain in place and continue rendering the current built-in toolbar UI.
- Built-in toolbar UI state is adapted from `ToolbarSnapshot` rather than re-derived independently in legacy code.

### Responsibility Split

`headless-toolbar` owns:

- active editing context for toolbar integration
- built-in command `active` / `disabled` / `value`
- built-in command execution semantics

`SuperToolbar` owns:

- initialization and teardown
- subscription to headless toolbar updates
- adaptation from headless snapshot to existing toolbar item model/UI props
- legacy toolbar-only UI orchestration
- preservation of public toolbar config behavior

Toolbar Vue components own:

- rendering
- local UI interactions
- dropdown open/close state
- presentation-only concerns

## Compatibility Constraints

The following must remain unchanged in this migration:

- `SuperDoc` consumer API for toolbar configuration
- `modules.toolbar` config shape
- current built-in toolbar visual structure
- current toolbar interaction model
- support for existing toolbar config options such as:
  - `selector`
  - `groups`
  - `icons`
  - `texts`
  - `fonts`
  - `hideButtons`
  - `responsiveToContainer`
  - `excludeItems`
- current supported editing contexts
- current built-in toolbar integration through `SuperDoc.js`

## Migration Design

### Core Design Decision

The built-in toolbar will not be replaced by a separate headless-demo-style implementation.

Instead, the existing built-in toolbar stack will be internally rewired so that:

- `SuperToolbar` creates and owns a headless toolbar controller
- `SuperToolbar` subscribes to `ToolbarSnapshot` updates
- existing built-in toolbar items receive their built-in state from the snapshot
- built-in command execution is delegated to headless `execute()` or `context.target.commands`
- existing UI rendering remains intact

### Expected Implementation Shape

`SuperDoc.js`

- continues creating the toolbar through the existing module path
- continues passing toolbar configuration exactly as today

`SuperToolbar`

- initializes `createHeadlessToolbar({ superdoc, commands: [...] })`
- stores the latest `ToolbarSnapshot`
- updates built-in toolbar item state from snapshot data
- routes built-in command execution through the headless controller
- preserves existing lifecycle/event wiring expected by `SuperDoc`

Existing toolbar Vue/UI layer

- stays visually and structurally the same
- keeps local UI-only state
- does not remain an independent source of truth for built-in command state

## Source of Truth Rules

For built-in toolbar commands:

- state must come from `headless-toolbar`
- execution must go through `headless-toolbar`
- any mismatch between legacy state/execute logic and headless behavior must be treated as a parity issue, not silently reimplemented in the built-in UI

For non-built-in or toolbar-local UI behavior:

- local UI state may remain owned by the built-in toolbar implementation
- custom buttons may remain outside headless source-of-truth if they are not part of the built-in command registry
- async or UI-driven flows may still use headless helpers where applicable without forcing everything into synchronous `execute()`

## Command Mapping

A legacy-to-headless mapping layer is expected where naming differs.

Known examples:

- `bold` -> `bold`
- `italic` -> `italic`
- `underline` -> `underline`
- `strike` -> `strikethrough`
- `fontSize` -> `font-size`
- `fontFamily` -> `font-family`
- `highlight` -> `highlight-color`
- `linkedStyles` -> `linked-style`
- `documentMode` -> `document-mode`

A full mapping table should be produced during implementation and reviewed as part of the migration.

## Supported Behavior Scope

The migrated toolbar must continue to work across supported contexts, including:

- standard body editing
- header/footer editing
- editable vs non-editable states where relevant
- document-level controls such as zoom, ruler, and document mode
- table-related contexts
- link and formatting contexts
- tracked changes selection actions where supported

## Implementation Planning

Implementation will follow a phased rollout model.

The detailed execution sequence, command mapping, parity checklist, testing plan, and deferred items are tracked in the companion plan document:

- `specs/headless-toolbar-migration-plan.md`

This spec remains the source of truth for migration goals, architectural boundaries, compatibility constraints, and acceptance criteria.

## Acceptance Criteria

- the built-in SuperDoc toolbar uses `headless-toolbar` as the core integration layer for built-in command state
- the built-in SuperDoc toolbar uses `headless-toolbar` as the core integration layer for built-in command execution
- the current toolbar UI and UX remain unchanged
- the current `modules.toolbar` API remains unchanged
- existing toolbar behavior continues to work across supported editing contexts
- known gaps or mismatches discovered during migration are documented and split into follow-up issues where needed

## Risks

- legacy item names and headless command ids do not always match 1:1 and require careful adapter mapping
- some toolbar item labels or selected values may need transformation from snapshot values into legacy UI model shape
- complex flows such as image handling, table insertion, and certain dropdown interactions may be partly UI-owned and need explicit boundary decisions
- there may be edge-case parity differences in special editing contexts that only appear once the built-in toolbar fully switches to headless state/execute

## Open Questions

- full reviewed mapping of all legacy built-in toolbar item ids to headless command ids
- exact list of built-in items that remain partially UI-owned
- whether any existing legacy toolbar-only state derivation should remain temporarily as a fallback during transition
- what parity checks/tests should be required for each command group before the migration is considered complete

## Follow-up Expectations

If migration exposes missing nuances in the headless toolbar API, those should be handled as explicit follow-ups rather than hidden in new built-in-toolbar-only logic.

Examples of likely follow-up categories:

- parity gaps in special editing contexts
- value-shape normalization improvements
- additional registry coverage for special flows
- cleanup/removal of now-unused legacy toolbar internals
