# Upload Event Content

## 🧩 Problem

Setting up LMS content on an Event by hand is slow. Each file has to be uploaded to the Administrate document management system (DMS), filed in a sensible folder and then added to the Event as an LMS resource with the right name, order, completion settings and Learning Tags. When the content list and the files already live in a Smartsheet sheet, copying them across one by one is repetitive and error-prone.

## 💡 Automator Solution

This is a reusable sub-workflow. Another workflow calls it with an **Execute Workflow** node, passing one item per piece of content. For file content (SCORM packages, videos, documents and presentations) it downloads the file from an attachment on a Smartsheet sheet, uploads it to a DMS folder and links it to the Event. Other content types (external links, separators and discussions) are created directly. It returns one summary item with a result for every row.

## ✨ Features

- One call can add many content items, to one Event or several
- Files come from Smartsheet attachments, matched by row ID with the file name as a fallback
- Uploads files to the DMS with the two-step `requestUpload` and S3 POST flow
- Files each call's documents in one DMS folder, created inside a parent folder you choose and reused on later calls
- Creates LMS resources in `ordering` order, with display name, required and auto-complete settings
- Finds or creates the Learning Tags named on each row and applies them to the content
- All GraphQL values are sent as variables
- Checks every mutation for errors and returns a per-row result rather than failing silently
- A row that fails (missing file, rejected upload or rejected content) is reported and the remaining rows still run

## 🛠️ Setup Instructions

### Prerequisites

- Access to Administrate Automator
- An Administrate OAuth2 credential with permission to create DMS folders and documents, Learning Tags and Event LMS content
- A Smartsheet API access token for the sheet that holds the files (only needed when you upload files)
- A DMS folder to hold the uploaded files, for example `/Course Materials`

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file and save it
4. Note the workflow ID from its URL (`/workflow/<id>`). You need it, or the workflow name, when calling it

The workflow is called by other workflows, so it does not need to be activated.

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Fetch DMS Folders`
- `Create Target Folder`
- `Find Tag`
- `Create Tag`
- `Request Upload`
- `Create Content (File)`
- `Create Content (No File)`

Create a **Bearer Auth** credential with your Smartsheet API access token as the token and select it on:
- `List Sheet Attachments`
- `Get Attachment URL`

`Download Binary` and `Upload To DMS` need no credential. They use the short-lived URLs returned by Smartsheet and Administrate.

#### 2. Find the DMS parent folder ID

Run this query in the Administrate GraphQL explorer, replacing the name with your folder's name:

```graphql
query {
  folders(filters: [{ field: name, operation: eq, value: "Course Materials" }]) {
    edges { node { id legacyId name path } }
  }
}
```

Use the `id` (the base64 global ID) and the `path`. Do not use the `legacyId`.

#### 3. Fill in the Config node

| Key | Value |
|-----|-------|
| `GRAPHQL_URL` | Replace `https://YOUR-INSTANCE.administrateapp.com/graphql` with your Administrate GraphQL URL. |
| `SMARTSHEET_API_BASE_URL` | Leave as `https://api.smartsheet.com/2.0`. |
| `DMS_PARENT_FOLDER_ID` | Replace `REPLACE_WITH_DMS_PARENT_FOLDER_ID` with the `id` from step 2. Each call's folder is created inside it. |
| `DMS_PARENT_FOLDER_PATH` | Replace `REPLACE_WITH_DMS_PARENT_FOLDER_PATH` with the `path` from step 2, for example `/Course Materials`. It must describe the same folder as `DMS_PARENT_FOLDER_ID`. |
| `DEFAULT_CONTENT_TYPE` | MIME type used when a Smartsheet attachment has none. Defaults to `application/zip`. |

#### 4. Prepare the files in Smartsheet

Attach each file to the sheet, ideally to the row that describes it. The workflow matches a content item to an attachment by:

1. `smartsheetRowId`: a **file** attachment on that row (the first one if there are several)
2. otherwise `contentFile`: a **file** attachment anywhere on the sheet with that name (case-insensitive)

Link attachments (such as Google Drive or Box links) are ignored. The sheet must have fewer than 100 attachments in total (see Troubleshooting).

#### 5. Call it from another workflow

Add an **Execute Workflow** node to your workflow and set:

- **Source**: Database
- **Workflow**: choose **Upload Event Content** from the list (or select **By ID** and paste its workflow ID)
- **Mode**: **Run once with all items**, so all rows for one folder go in a single call
- **Workflow Inputs**: the fields below appear once the workflow is selected

Pass **one item per content row**.

**Inputs**

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `eventId` | String | Yes | GraphQL ID of the Event the content is added to. Rows in one call can target different Events. |
| `resourceType` | String | Yes | One of `scorm`, `video`, `document`, `presentation` (file types) or `external`, `separator`, `discussion` (no file). Not case-sensitive. |
| `displayName` | String | No | Name shown to learners. Also used as the DMS document name. |
| `ordering` | Number | No | Position in the Event's content list. Rows are processed in this order. |
| `isRequired` | Boolean | No | Whether the item is required to complete the Event. |
| `autoComplete` | Boolean | No | Whether the item completes automatically. |
| `externalActivityURL` | String | For `external` | The link for `external` content. Ignored for other types. |
| `tagsCsv` | String | No | Comma-separated Learning Tag names to apply. Missing tags are created. |
| `contentFile` | String | For file types | File name of the Smartsheet attachment, used to match when `smartsheetRowId` does not. Also used as the uploaded file name. |
| `smartsheetRowId` | String | For file types | Smartsheet row ID the file is attached to. Pass it as a string, because row IDs can be too large for a JavaScript number. |
| `smartsheetSheetId` | String | Yes | Smartsheet sheet ID that holds the attachments. Only the first item's value is used, so pass the same value on every item. |
| `dmsFolderName` | String | For file types | Name of the DMS folder to file this call's documents in, created inside `DMS_PARENT_FOLDER_ID` if it does not exist. Only the first item's value is used. |

Files are not passed as binary data. The caller only passes the values above; the sub-workflow fetches each file itself from Smartsheet.

Example items passed to the Execute Workflow node:

```json
[
  {
    "eventId": "REPLACE_WITH_EVENT_ID",
    "resourceType": "document",
    "displayName": "Pre-course reading",
    "ordering": 1,
    "isRequired": true,
    "autoComplete": false,
    "tagsCsv": "Pre-work, Reading",
    "contentFile": "pre-course-reading.pdf",
    "smartsheetRowId": "REPLACE_WITH_SMARTSHEET_ROW_ID",
    "smartsheetSheetId": "REPLACE_WITH_SMARTSHEET_SHEET_ID",
    "dmsFolderName": "Leadership Fundamentals"
  },
  {
    "eventId": "REPLACE_WITH_EVENT_ID",
    "resourceType": "external",
    "displayName": "Course forum",
    "ordering": 2,
    "isRequired": false,
    "externalActivityURL": "https://example.com/forum",
    "smartsheetSheetId": "REPLACE_WITH_SMARTSHEET_SHEET_ID",
    "dmsFolderName": "Leadership Fundamentals"
  }
]
```

Replace the `REPLACE_WITH_...` values with your own Event, Smartsheet row and sheet IDs.

**Output**

The call returns one item:

| Field | Type | Description |
|-------|------|-------------|
| `success` | Boolean | `true` if every row was created. |
| `message` | String | A short human-readable summary. |
| `createdCount` | Number | Rows created successfully. |
| `failedCount` | Number | Rows that failed. |
| `dmsFolderId` | String | Global ID of the DMS folder used, or empty if no files were uploaded. |
| `dmsFolderPath` | String | Path of that folder. |
| `folderWasCreated` | Boolean | `true` if the folder was created during this call. |
| `results` | Array | One entry per row with `eventId`, `displayName`, `resourceType`, `ordering`, `success`, `lmsContentId`, `documentId` and `error`. |

Branch on `success` with an **If** node in the calling workflow and read `results` to see which rows failed. Problems that affect the whole call stop the sub-workflow with an error on the calling Execute Workflow node before any content is created: a missing `dmsFolderName`, a duplicate or mismatched DMS folder, a Learning Tag that cannot be created or 100 or more sheet attachments. Connection and authentication failures, including a failed file download or S3 upload, also stop it.

### Testing

1. In Smartsheet, attach a small PDF to a row of a test sheet and note the sheet ID and row ID (from **File > Properties** and the row's **Properties**)
2. In Administrate, create a test Event that can hold LMS content and note its ID
3. Create a new workflow with a **Manual Trigger**
4. Add a **Code** node that returns two items like the example above: one `document` row using your sheet ID, row ID and file name, and one `external` row. Use the test Event ID on both
5. Add an **Execute Workflow** node that calls **Upload Event Content** with **Mode** set to **Run once with all items**, mapping each input from `$json`
6. Run the workflow. The result should show `success: true` and `createdCount: 2`
7. In Administrate, check that the Event has both content items in order, the PDF is in the DMS folder you named inside your parent folder and the tags are applied
8. Run it again. The folder is reused (`folderWasCreated: false`) and two more content items are added, because the workflow does not check for existing content
9. Change the file row's `contentFile` to a name that does not exist and clear its `smartsheetRowId`. The result should show `success: false`, with that row's `error` explaining that no attachment matched, while the `external` row is still created

## ⚙️ How It Works

1. **When Called**: receives one item per content row.
2. **Config**: holds the GraphQL URL, Smartsheet API URL, DMS parent folder and default MIME type.
3. **List Sheet Attachments**: lists the sheet's attachments once.
4. **Fetch DMS Folders / Find Target Folder**: looks for a folder named `dmsFolderName` inside the parent folder. If no row has a file type, folder handling is skipped.
5. **Folder Missing? / Create Target Folder / Folder Ready**: creates the folder if needed and passes on its global ID.
6. **Match Rows to Attachments**: matches each row to its Smartsheet attachment and flags file rows that have no match.
7. **Split Unique Tags / Find Tag / Find Missing Tags / Create Tag / Collect Tag Map**: finds or creates every Learning Tag named in the rows and adds `learningTagIds` to each row. Only one branch of each IF runs, so **Collect Tag Map** runs once.
8. **Sort by Order / Loop Content Rows**: processes the rows one at a time in `ordering` order.
9. **Route by Type**:
   - **Missing File**: records an error for the row.
   - **File Upload**: **Get Attachment URL** gets a temporary download URL, **Build Upload Body / Request Upload** calls `documents.requestUpload`, **Check Upload Request** checks for errors, **Download Binary** fetches the file and **Upload To DMS** posts it to the returned S3 URL within 60 seconds. **Create Content (File)** then calls `event.createLmsResource` with the new `documentId`.
   - **No Upload**: **Create Content (No File)** calls `event.createLmsResource` directly.
10. **Record Row Result**: checks the response for errors and records the row's result.
11. **Build Result**: returns the summary item once every row has been processed.

## 🩺 Troubleshooting

**The call stops with `dmsFolderName is required`:**
- Pass `dmsFolderName` on the first item whenever any row has a file type.

**The call stops with `Matched a folder at ... but expected ...`:**
- `DMS_PARENT_FOLDER_ID` and `DMS_PARENT_FOLDER_PATH` in **Config** point at different folders. Re-run the query in Configuration step 2 and copy both values from the same result.

**The call stops with `Found 2 folders named ...`:**
- Two folders with that name exist inside the parent folder. Remove or rename one in Administrate.

**Uploads fail with "The folder specified does not exist":**
- `DMS_PARENT_FOLDER_ID` must be the base64 global `id`, not the numeric `legacyId`, even though the folder exists.

**The call stops with `Smartsheet returned 100 attachments`:**
- Smartsheet returns attachments 100 at a time by default. Add a query parameter `includeAll` = `true` to **List Sheet Attachments**, or split the files across sheets.

**A row's error says no attachment matched:**
- Check the file is attached to the sheet as a file rather than a link and that `smartsheetRowId` or `contentFile` matches it.

**A row's error says `createLmsResource failed`:**
- Check `eventId` is a valid Event ID, `resourceType` is one of the supported values and `externalActivityURL` is set for `external` rows.

**Every call fails before anything happens, even for rows without files:**
- `smartsheetSheetId` is always required, because the sheet's attachments are listed at the start of every call. Pass a valid sheet ID and select the Smartsheet credential.

**Content appears twice:**
- The workflow does not check for existing content, so calling it twice for the same rows creates duplicates. Remove the extra items in Administrate.
