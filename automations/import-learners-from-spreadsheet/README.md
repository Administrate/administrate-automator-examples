# Import Learners onto an Event from a Spreadsheet

## Problem

Customers often send a list of delegates as a spreadsheet. Adding them to an Event means searching for each person, creating the contacts that don't exist under the right account, and registering them one at a time. For large groups this is slow and easy to get wrong.

## Automator Solution

Attach the spreadsheet to the Event and run **Import Learners from Spreadsheet** from the Event's Automator menu. For each row the workflow finds the contact by email, or creates it under the named (non-individual) account. It then registers the contact on the Event unless they are already booked. A summary of what happened to every row is appended to the Event's internal notes.

## Features

- One-click import from a spreadsheet attached to the Event
- Matches existing contacts by email address
- Creates missing contacts in one batch under the named organisation account
- Skips contacts who are already registered on the Event
- Refuses to create contacts under Individual accounts or accounts that don't exist
- Writes a per-row outcome summary to the Event's internal notes
- Configurable column headers and file selection
- Names, emails and account names are sent as GraphQL variables, so quotes and apostrophes (e.g. O'Brien) can't break the queries

## Spreadsheet Format

An `.xlsx` (or `.xls`) file whose first row contains these headers (the names can be changed in **Config**):

| Account Name | email | First Name | Last Name |
|---|---|---|---|
| Example Ltd | learner@example.com | Alex | Smith |

- **Account Name** must exactly match an existing non-individual account in Administrate. It is only used when a new contact has to be created.
- Rows with an empty email are ignored.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Permission to read documents, create contacts, register learners and update events

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Select one **Administrate OAuth2** credential on every Administrate node:
- `Import from Excel Webhook` (Administrate Trigger)
- `Administrate API - Get DmsDoc URL` (Administrate node)
- All `HTTP Request - …` nodes, plus `Retrieve Existing Event Internal Notes` and `Apply status messages to Event Internal Notes`

`HTTP Request - Get Excel Data` downloads the file from a signed URL and needs no credential.

#### 2. Review the `Config` node

| Key | Default | Description |
|---|---|---|
| `apiUrl` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint used by the HTTP Request nodes |
| `allowedFileExtensions` | `xlsx,xls` | File extensions treated as the import spreadsheet. The **Extract from File** node reads XLSX; to import CSV, add `csv` here and set that node's operation to CSV. |
| `documentNameFirstChoice` | `attendees` | If several spreadsheets are attached, prefer the one whose name contains this |
| `documentNameSecondChoice` | `import` | Second preference; otherwise the first spreadsheet found is used |
| `colEmail` | `email` | Column header for the email address |
| `colFirstName` | `First Name` | Column header for the first name |
| `colLastName` | `Last Name` | Column header for the last name |
| `colAccountName` | `Account Name` | Column header for the account name |
| `dmsFolderName` | `Student Import Files` | Reference only: a suggested DMS folder for import files. Not used by the logic. |

There are no `REPLACE_WITH_…` placeholders and no Data Tables in this workflow.

#### 3. Activate the workflow

When the workflow is activated, the Administrate Trigger registers a **Manual Event** webhook named **Import Learners from Spreadsheet**. It appears in the Automator menu on Events.

### Testing

1. Activate the workflow
2. Create a small spreadsheet with a mix of rows: an existing contact, a new contact under an organisation account, a contact under an Individual account, and an unknown account name
3. Attach it to a test Event (Outline tab → Documents)
4. Choose **Import Learners from Spreadsheet** from the Event's Automator menu
5. Check the Event's learners, then read the summary in the Event's internal notes

## How It Works

1. **Trigger**: a user runs **Import Learners from Spreadsheet** on an Event
2. **Find the spreadsheet**: reads the Event's documents and picks the spreadsheet using `documentNameFirstChoice`, then `documentNameSecondChoice`, then the first match
3. **Download and read**: gets a signed download URL (`downloadDocument`), downloads the file and extracts the rows. Rows with no email are dropped.
4. **Map columns**: maps the configured headers to email / first name / last name / account name
5. **Look up contact by email** (email passed as a GraphQL variable)
   - **Found**: checks whether they already have a learner record on the Event. If not, they are registered.
   - **Not found**: looks up the account by name (passed as a GraphQL variable)
     - Account missing or Individual → the row is skipped with a message
     - Otherwise → the contacts are created in one `createBatch` call and then registered on the Event
6. **Record outcomes**: each row produces a timestamped status message
7. **Update internal notes**: all messages are combined and appended to the Event's existing internal notes. The notes are sent as a GraphQL variable, so existing HTML and names with quotes are safe.

## Troubleshooting

**Menu option not showing:**
- Make sure the workflow is active and the Administrate Trigger credential is set

**Nothing imported:**
- Check that the file is attached to the Event itself and has an allowed extension
- Check that the header row matches `colEmail`, `colFirstName`, `colLastName` and `colAccountName` exactly

**"Account Name provided not found":**
- The account name must match an existing account's name exactly

**"Account specified is an Individual Account":**
- New contacts can only be created under organisation (non-individual) accounts

**Registration errors in the notes:**
- The message after "Reason:" is the Administrate error (e.g. the Event is full or not open for registration)
