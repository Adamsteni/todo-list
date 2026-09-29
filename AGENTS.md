# AGENTS.md

Guidance for AI coding agents (and humans) working in this repo.

## What this is

A single-file, offline task manager + notes app. `index.html` contains the markup,
the styles and the whole application — no build step, no dependencies, no server.

## Run it

- Open `index.html` in a browser, or run `python -m http.server 8000` and visit
  <http://localhost:8000>.
- To test a clean-slate install, clear site data / `localStorage` and reload.
  There is no test suite and no linter configured, so verification is manual:
  click through tasks, notes, drag-reorder, theme toggle, and resize across
  the 900px breakpoint.

## Architecture

- **Storage** — everything lives in `localStorage`, wrapped by the `load(key, fallback)`
  and `put(key, value)` helpers at the top of the script. Keys: `tasks_v1`, `notes_v1`,
  `cats_v1`, `theme`, `seeded_v1`. Never call `localStorage` directly elsewhere.
  The `_v1` suffix is the schema version: bump it (to `_v2`, etc.) when a shape changes
  so stale data can't be read back into the new code.
- **Data shapes**
  - task: `{ id, title, completed, cat, date }` where `date` is `"YYYY-MM-DD"` (`ymd()`)
  - note: `{ id, title, body, updated }` (`updated` is an epoch ms number)
  - category: `{ id, name, color }`
- **State** — a single `state` object: `{ page, date, cat, noteId }`. `page` is
  `"tasks"` or `"notes"`; `noteId` set means the note editor is open.
- **Rendering** — a full re-render on every change. `render()` switches the visible
  `<section>` and calls `renderWeek()` / `renderTabs()` / `renderTasks()` or
  `renderNotes()` / `renderEditor()`. Each renderer clears its container with
  `innerHTML = ""` and rebuilds nodes with the `el(tag, cls, text)` helper.
  There is no virtual DOM and no framework — follow that pattern instead of
  introducing one.
- **Event handling** — listeners are attached to freshly created elements inside the
  render functions; delegation is used only for the list's drag events.

## Conventions

- Plain ES5-flavoured JavaScript: `var`, `function`, string concatenation, no
  arrow functions, no `const`/`let`, no template literals, no optional chaining.
  Match the surrounding style rather than modernising it.
- Two-space indent, semicolons, `} catch (e) {}` for ignored storage errors.
- DOM text always goes through `textContent` via `el()` — never `innerHTML` with
  user data, so titles containing `<` can't inject markup.
- Colours come from CSS custom properties, not inline styles. The only inline style
  is the per-task category colour (`--c`), which is a data value.
- Dark mode is a `data-theme` attribute on `<html>` plus a `:root[data-theme="dark"]`
  block. Add new colours as variables in both blocks, never hardcode a hex value
  in a component.
- Short comments explain *why* a non-obvious line exists; don't narrate the obvious.

## Layout

- **Mobile first.** `.app` is a single 430px column with a fixed bottom nav
  (`.nav`) and a floating `+` button (`.fab`).
- **Desktop** is a single `@media (min-width:900px)` block at the end of the
  `<style>`. There, `.shell` becomes a two-column grid: the nav turns into a left
  rail, `#list` and `#notes` become card grids, the add/edit sheet becomes a
  centred dialog, and the note editor is centred at 780px. Keep new styles
  mobile-first and add desktop overrides inside that block rather than writing
  new breakpoints.

## Sample data

`SEED_TASKS` and `SEED_NOTES` at the top of the script are inserted by `seed()` on a
brand-new install only. The `seeded_v1` flag means the samples are never restored
after the user deletes them — don't set that flag anywhere else. Task dates are
offsets in days from today (`d: 0` is today, `-1` yesterday).

## Gotchas

- `applyVisibleOrder()` writes reordered tasks back into the slots they occupied in
  the full array, so hidden tasks keep their positions. Keep that invariant.
- Categories are created and renamed in `openCatSheet()` (same sheet as tasks, plus a
  colour swatch row); `deleteCat()` reassigns that category's tasks to the first
  remaining category and refuses to delete the last one. The `×` only appears on the
  selected tab, and right-click / long-press on a tab opens the edit-delete menu.
- Notes save on every keystroke, and a note left with an empty title *and* body is
  deleted when you navigate away.
- Don't run `render()` from inside a render function without a `return` — it will
  recurse.
