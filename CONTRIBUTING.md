# 🤝 Contributing to Administrate Automator Examples

You can submit a pull request directly without opening an issue first.

## 🧩 Adding an example

1. Fork the repository and create a branch with a descriptive name, such as `instructor-reminders`.
2. Add a directory under `automations/` using a lowercase, hyphenated name:

   ```text
   automations/my-workflow-name/
   ├── README.md
   └── workflow.json
   ```

   For examples with several workflows, use a `workflows/` subdirectory with numbered JSON files such as `10-error-handler.json` and `20-main-workflow.json`. Document the import order and how to connect them. Keep supporting files such as HTML forms with the example.

3. Document the problem, solution, prerequisites, setup, configuration, usage, testing and troubleshooting in the example's README. Include required permissions, credentials and any external services. Use an existing example as a starting point.
4. Export the workflow JSON, remove credentials and customer data and replace instance-specific URLs and IDs with clear placeholders. Leave workflows inactive and remove pinned execution data.
5. Add a link and a short description to the workflow table in the [main README](README.md), keeping it alphabetically sorted by title.
6. Submit a pull request explaining what the example does, why it is useful and how you tested it in Automator.

## 🛠️ Fixing an example

Keep the change focused on the bug and update any affected setup or usage instructions. Reproduce the failure before making the change, then repeat the same check to confirm the fix. Describe the failure and the testing performed in your pull request.

## 🎨 Workflow style

- Preserve the existing two-space JSON indentation and avoid unrelated export changes.
- Use descriptive node names and group related nodes visually.
- Use a **Config** node for values users need to change where practical, following the existing examples.
- Add brief sticky notes for setup requirements or non-obvious logic.
- Use placeholders such as `REPLACE_WITH_TABLE_ID` and `https://YOUR-N8N-HOST` and explain them in the README.
- Write documentation in UK English, with emoji in headings and relative links to repository files.

## 🧪 Validation

CI checks JSON and YAML syntax and runs the official n8n workflow SDK validator on pull requests and pushes to `main`. Validator errors fail CI; warnings are reported for review.

Workflow behaviour needs testing in Administrate Automator with test records, including relevant failure cases and repeat runs. Describe the results and your Automator/n8n version in the pull request, or say if you could not test it. CI does not connect to an Administrate instance or validate custom node parameters against instance-specific definitions.
