# ⚙️ Administrate Automator Examples

Official [n8n](https://n8n.io/) workflow examples for [Administrate Automator](https://getadministrate.com/features/automator/). Automate training management tasks including instructor scheduling, learner registration, attendance tracking, certificate renewals and integrations with your business systems.

Each example includes importable workflow JSON and a README explaining the problem it solves and how to use it. Browse the [automation examples](automations/) to find a starting point for your organisation.

## ✨ About Administrate Automator

Automator is a visual, drag-and-drop workflow builder within the [Administrate Training Management System (TMS)](https://getadministrate.com/), powered by n8n. Build workflows that respond to Administrate events, run on a schedule or start manually, with custom code when you need it. See the [Automator guide](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator) for how it works.

Connect your learning management system (LMS), CRM, ERP, HRIS and AI tools to automate processes across your training technology. Common uses include instructor invitations, booking approvals, learner waitlists and reporting. Learn more about [Automator's workflow automation features](https://getadministrate.com/features/automator/).

## 🚀 Getting started

You need an Administrate instance with the Automator paid add-on enabled and the **Automator Run** permission. Contact [Administrate Support](https://support.getadministrate.com/hc/en-us/requests/new) to enable it. See the [Automator setup guide](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator) for details.

1. Choose an example below and read its README for prerequisites and configuration.
2. Download its `workflow.json` file using GitHub's **Download raw file** button.
3. Open **Available Automations** in Administrate and select **Launch Automator** to open your n8n workspace.
4. Create a workflow, open the top-right menu (⋮) and select **Import from File** to load the JSON.
5. Configure the required credentials, instance URLs and parameters described in the example's README.
6. Follow the example's testing instructions with test records, review the execution logs and activate the workflow when ready.

Examples with several workflows have a `workflows/` subdirectory. Download the required JSON files and follow that example's README for import order and links between workflows.

## 🧩 Workflow examples

| Workflow | Description |
|----------|-------------|
| [Administrate MCP Server](automations/administrate-mcp-server/) | Expose Administrate search and update tools to AI agents such as Claude over the Model Context Protocol |
| [Alert Opportunity Owners When a Reserved Event Is Cancelled](automations/cancelled-event-reservation-alert/) | Email each opportunity owner once when an event with their reservations is cancelled |
| [Auto Assign Instructors to Event](automations/auto-assign-instructors-to-event/) | Automatically assign available approved instructors to new events based on availability and workplace location |
| [Auto Assign Tasks](automations/auto-assign-tasks/) | Redistribute tasks by task type to the account owner, booking owner, instructor and administrator |
| [Auto-Finalise Invoices Before Event Start](automations/auto-finalise-invoices/) | Finalise draft invoices and move bookings on as events approach, with a dry-run mode and a Slack summary |
| [Bulk Complete LMS Content](automations/bulk-complete-lms-content/) | Mark SCORM LMS content as complete in bulk for flagged learners on an event |
| [Bulk Register Delegates and Record Attendance](automations/bulk-register-delegates-record-attendance/) | A web form to register delegates onto an event and record their attendance and results in one go |
| [Bulk Resource Removal](automations/bulk-resource-removal/) | Remove all resources from an event in bulk |
| [Certificate Renewal Reminders](automations/certificate-renewal-reminders/) | Remind learners before their certificates or achievements expire, with dry-run mode, a send cap and a deduplication log |
| [Create Achievement Type from Course Template](automations/create-achievement-type-from-course-template/) | Create an achievement type from a course template and award it to qualified instructors |
| [Create Booking from Event](automations/create-booking-from-event/) | Create a booking directly from an event with the event automatically added as an interest |
| [Daily Event Resourcing Gaps Digest](automations/event-resourcing-digest/) | Daily email listing upcoming events that are missing instructors, rooms or other resources |
| [Delayed Email After Registration](automations/delayed-post-registration-email/) | Send a per-event follow-up email a set time after a learner registers |
| [Import Learners from a Spreadsheet](automations/import-learners-from-spreadsheet/) | Upload an Excel or CSV file to bulk-register learners onto an event, creating contacts and accounts as needed |
| [Instructor and Resource Utilisation Dashboards](automations/utilization-dashboards/) | Password-protected dashboards showing instructor and resource utilisation from Administrate data |
| [Instructor and Staff Invitations](automations/instructor-invitations/) | Invite instructors and staff to an event with secure personal links, chase non-responders and publish once staffing is complete |
| [Instructor Jump Ball](automations/instructor-jump-ball/) | Send RSVP requests to instructors and manage responses for event assignments |
| [Log Automation Failures to Administrate](automations/error-logging-to-administrate/) | Shared error workflow, logging sub-workflow and daily digest for recording automation failures in Administrate |
| [Logging vILT Attendance](automations/logging-vilt-attendance/) | Automatically record Zoom attendance in Administrate by matching email addresses |
| [Managing Low Attendance](automations/managing-low-attendance/) | Email participants with referral links when events are below target fill rate |
| [Merge Accounts](automations/merge-accounts/) | Merge duplicate accounts from HRIS/CRM syncs or manual entry |
| [Normalise Contact Mobile Numbers](automations/normalise-contact-mobile-numbers/) | Clean up contact mobile number formatting whenever a contact is created or updated (UK rules by default) |
| [OAuth - Authorisation Code](automations/oauth-authorization-code/) | Implement the OAuth authorisation code flow to execute API requests with the authenticated user's permissions |
| [QR Code Check-in and Attendance](automations/qr-code-check-in-attendance/) | QR code self check-in for classroom events, with walk-in registration, a daily reconciliation report and a live dashboard |
| [Review and Cancel Under-Subscribed Events](automations/cancel-under-subscribed-events/) | Flag low-fill events for human approval, re-check them, then cancel and notify instructors and learners |
| [Scheduler - Bulk Event Activation](automations/scheduler-bulk-event-activation/) | Activate hundreds or thousands of draft events created by Scheduler in bulk |
| [Sync Event Total Session Hours](automations/event-total-session-hours/) | Keep an event custom field equal to the total hours of its published sessions |
| [Tag an Entity](automations/tag-an-entity/) | Reusable sub-workflow that adds tags to a course template or learning path, creating missing tags |
| [Teams Virtual Classroom Auto Setup](automations/teams-virtual-classroom-auto-setup/) | Set up a Microsoft Teams virtual classroom automatically when a virtual event is created |
| [Updating Billing & Shipping Addresses](automations/updating-billing-shipping-addresses/) | Automatically copy the default address to billing and shipping fields when missing |
| [Upload Event Content](automations/upload-event-content/) | Reusable sub-workflow that creates LMS content on an event, uploading files from Smartsheet attachments |
| [Waitlist: Timed Offers with Accept/Decline](automations/waitlist-offer-accept/) | Offer freed places to waitlisted learners in turn, hold the place for a set time and record their accept/decline response |

## 💬 Help

For questions, bug reports or ideas about these examples, use [GitHub Issues](https://github.com/Administrate/administrate-automator-examples/issues).

For help with your Administrate instance or Automator access, contact [Administrate Support](https://support.getadministrate.com/hc/en-us/requests/new). The [Automator documentation](https://support.getadministrate.com/hc/en-us/articles/41797081434513-Administrate-Automator) also walks through building an instructor invitation workflow.

## 🤝 Contributing

This repository is primarily maintained by Administrate to share Automator examples. We also welcome pull requests for bug fixes and useful new examples. Read the [contribution guide](CONTRIBUTING.md) for the layout, documentation and testing expectations.

## 📄 Licence

These examples are licensed under the [BSD 2-Clause Licence](LICENSE). You can use, modify and redistribute them, including commercially and in closed-source projects. When redistributing, retain the copyright notice, licence conditions and disclaimer; for binary distributions, include them in the accompanying documentation or materials. The examples are provided as is, without warranty, and the licence limits the authors' liability.
