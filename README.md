# Administrate Automator Examples

A collection of workflow examples for [Administrate Automator](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator), demonstrating how to automate common tasks and build custom integrations with Administrate.

## What is Automator?

Administrate Automator lets you automate your work in Administrate - everything from custom workflows to integrations with systems and custom tools unique to your organization. Build automations that eliminate manual tasks, connect your systems, and scale your operations without adding headcount.

Automator is powered by n8n, a powerful workflow automation platform that we've white-labeled and integrated directly into Administrate.

## Getting Started

1. **Access Automator** in your Administrate instance
2. **Browse the examples** below to find workflows that solve your challenges
3. **Download the workflow JSON** file from the example directory
4. **Import into Automator** using the import function in your n8n interface
5. **Configure credentials** and customize as needed for your organization

For detailed setup instructions, visit the [Automator documentation](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator#h_01KC4HGRHKTPC1C95A840Q7T21).

## Available Workflows

| Workflow | Description | Version |
|----------|-------------|---------|
| [Scheduler - Bulk Event Activation](./scheduler-bulk-event-activation) | Activate hundreds or thousands of draft events created by Scheduler in bulk | 1.0.0 |
| [Instructor Jump Ball](./instructor-jump-ball) | Send RSVP requests to instructors and manage responses for event assignments | 1.0.0 |
| [Auto Assign Tasks](./auto-assign-tasks) | Automatically redistributes tasks by task type to the Account Owner, Booking Owner, Instructor, and Administrator. | 1.0.0 |
| [Bulk Resource Removal](./bulk-resource-removal) | Remove all Resources from an Event in bulk with 2 clicks | 1.0.0 |
| [Updating Billing & Shipping Addresses](./updating-billing-shipping-addresses) | Automatically copy Default Address to Billing and Shipping fields when missing | 1.0.0 |
| [Managing Low Attendance](./managing-low-attendance) | Email participants with referral links when Events are below target fill rate | 1.0.0 |
| [Merge Accounts](./merge-accounts) | Merge duplicate Accounts from HRIS/CRM syncs or manual entry | 1.0.0 |
| [Logging vILT Attendance](./logging-vilt-attendance) | Automatically record Zoom attendance in Administrate by matching email addresses | 1.0.0 |

## How to Import a Workflow

1. Navigate to the workflow directory you want to use
2. Download the `workflow.json` file
3. In your Adminsitrate Automator workspace > new workflow, click the menu (⋮) in the top right
4. Select "Import from File"
5. Choose the downloaded JSON file
6. Configure your Administrate OAuth credentials
7. Update any instance-specific URLs or parameters as noted in the workflow's README

## Contributing

We welcome contributions! If you've created a useful workflow or have improvements to existing ones:

1. Fork this repository
2. Create a new branch for your workflow or improvement
3. Follow the existing directory structure and README format
4. Submit a pull request with a clear description of what your workflow does

## Workflow Structure

Each workflow follows this structure:
```
workflow-name/
├── README.md           # Problem description, solution, setup instructions
├── workflow.json       # n8n workflow file
└── screenshots/        # Optional: workflow screenshots or examples
```

## Support

- **Automator Guide**: [Getting Started with Automator](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator)
- **Community**: Join discussions in the [Issues section](https://github.com/Administrate/administrate-automator-examples/issues)

## Stay Updated

- **Star this repo** to bookmark it
- **Watch this repo** to get notified of new workflows and updates
