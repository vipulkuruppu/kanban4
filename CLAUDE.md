# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file Kanban demo ("UOB IT PMO — Project Board"). All of it lives in `index.html`: CSS in `<style>`, markup, and vanilla JS in one `<script>`. There is no build step, package manager, linter or test suite.

## Running

Open `index.html` in a browser, or serve the directory (for example `python3 -m http.server`) and load it from there. Tasks are kept only in memory, so a page reload resets the board to the seeded sample data.

## Architecture (the `<script>` block)

- **One state object.** `state` holds `tasks`, `filters`, `nextId` and `ui` (`pendingDeleteId`, `moveMenuId`). Actions (`addTask`, `moveTask`, `deleteTask`) change `state` and then call `renderBoard()`. Note that `addTask` does not render; its callers do.
- **One render path.** `renderBoard()` is the only function that draws cards. It rebuilds each column's `innerHTML` from `applyFilters()` and updates the count badges and `renderSummary()`. Transient card UI, such as the open move menu or the delete confirmation, is driven by `state.ui` and redrawn, not toggled in the DOM. After a re-render, `focusAfterRender(selector)` puts focus back where it belongs.
- **Event delegation.** One click handler on `#board` routes on `button[data-action]` (`move-toggle`, `move-to`, `delete-ask`, `delete-no`, `delete-yes`) and reads `data-id`. New card controls should follow this pattern rather than bind listeners per card, because cards are recreated on every render.
- **Escaping.** Every user value put into card HTML must go through `escapeHtml()`.
- **Constant lists.** `STATUSES`, `PROJECTS`, `CATEGORIES` and `PRIORITIES` fill the `<select>` elements in `init()` and are the whitelists that `validateForm()` checks against. The four status columns are hard-coded in the HTML with `data-status`, `data-list` and `data-count` attributes, and the summary uses `sum-<domKey(status)>` ids. Adding a status means changing both the constant and the markup and CSS.
- **Dates.** Dates are local `YYYY-MM-DD` strings built by `toLocalISO()`; do not use `toISOString()`, which shifts to UTC. Overdue checks and due-date validation compare these strings directly.
- **Task IDs.** `formatId()` produces `UOB-ITPM-0001` and so on.

## Email notification (FormSubmit)

`FORMSUBMIT_ENDPOINT` near the top of the script is the only place the recipient address is set. On submit, the card is added to the board immediately, and then `notifyNewTask()` POSTs JSON to FormSubmit. If that fails, the user sees a warning toast and the board is unaffected. FormSubmit can return HTTP 200 with `success: "false"`, and that case is treated as a failure. A new address needs a one-time activation: the first submission sends a confirmation email instead of delivering.

## Conventions

- Accessibility is deliberate throughout. Keep it intact when changing the UI: ARIA labels on icon buttons, `aria-invalid` with per-field `err-<name>` messages, the modal focus trap and Escape handling, the `aria-live` toast region, and keyboard move and delete as an alternative to drag and drop.
- Colors and spacing come from CSS custom properties on `:root`; use those tokens instead of hard-coded values.
