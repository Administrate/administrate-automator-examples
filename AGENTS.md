# 🛠️ Repository guidance

Administrate maintains this collection of importable n8n workflow examples. Keep changes small, focused and consistent with the example being edited. Read its README and workflow JSON before making changes, preserving user edits and comments.

## 📁 Layout

- Examples live in `automations/<lowercase-hyphenated-name>/` with a `README.md` and `workflow.json`.
- Examples with several workflows use a `workflows/` subdirectory with numbered JSON filenames in import order.
- Keep supporting assets with their example and update the root README's alphabetically sorted table when adding or renaming an example.
- GitHub CI and contribution templates live in `.github/`.

## 🎨 Style

- Use two-space indentation and double quotes in JSON. Preserve key order, node IDs, positions and unrelated export metadata.
- Node names are referenced by connections and n8n expressions. Update all references when renaming a node.
- Follow the surrounding embedded JavaScript style: `const` by default, `let` for reassignment, semicolons and camelCase local names. Config keys generally use `UPPER_SNAKE_CASE`.
- Keep reusable settings in a **Config** node where the example uses one. Use clear placeholders for instance URLs, record IDs and workflow IDs, documenting each required replacement.
- Use descriptive node names and brief comments or sticky notes for non-obvious logic. Avoid unnecessary abstractions and dependencies.
- New exports should be inactive with empty `pinData`. Do not include secrets, customer data or instance-specific credential IDs.
- Write new documentation in UK English with emoji headings, relative repository links, no em dashes and no Oxford commas. Preserve API identifiers and existing filenames.
- Route questions and bug reports to GitHub Issues. Pull requests for bug fixes and new examples are welcome; see [CONTRIBUTING.md](CONTRIBUTING.md).

## ✅ Checks

- For bug fixes, reproduce the failure before changing the workflow and repeat the check afterwards.
- Rely on CI for JSON and YAML syntax and n8n workflow validation. Keep these checks out of contributor checklists.
- Use only official GitHub Actions. Install trusted tools directly with pinned versions.
- Validate changed behaviour in Automator with test records, including relevant failure cases and repeat runs. Report any checks that could not be run.
- Keep the workflow table accurate after moving or adding examples.
- Inspect `git diff` and do not reformat unrelated workflows. CI validates files; it does not run workflows against Administrate.
