# Tag an Entity

## 🧩 Problem

Learning Tags make Course Templates and Learning Paths easier to find and report on, but adding them from an automation takes several steps. Each tag name has to be looked up to get its ID, missing tags have to be created first and the right `addTags` mutation has to be called for the type of record. Repeating that logic in every workflow that tags something is slow to build and easy to get wrong.

## 💡 Automator Solution

This is a reusable sub-workflow. Any other workflow can call it with an **Execute Workflow** node, passing the record type, the record ID and a comma-separated list of tag names. It finds each Learning Tag by exact name, creates any that do not exist, adds them all to the record and returns a single result item that says whether it worked.

## ✨ Features

- One call tags a Course Template or a Learning Path
- Tag names are passed as plain text, so callers never need tag IDs
- Missing Learning Tags are created automatically
- Duplicate and blank names in the list are ignored
- All GraphQL values are sent as variables
- Checks every lookup and mutation for errors and returns them to the caller instead of failing silently
- Returns a clear `success` flag, a message, the tag IDs used and the names of any tags it created
- Does nothing (and reports success) when the tag list is empty

## 🛠️ Setup Instructions

### Prerequisites

- Access to Administrate Automator
- An Administrate OAuth2 credential with permission to read and create Learning Tags and to edit Course Templates and Learning Paths

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file and save it
4. Note the workflow ID from its URL (`/workflow/<id>`). You need it, or the workflow name, when calling it

The workflow is called by other workflows, so it does not need to be activated.

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Find Tag`
- `Create Tag`
- `Add Tags`

#### 2. Fill in the Config node

| Key | Value |
|-----|-------|
| `GRAPHQL_URL` | Replace `https://YOUR-INSTANCE.administrateapp.com/graphql` with your Administrate GraphQL URL. |

#### 3. Call it from another workflow

Add an **Execute Workflow** node to your workflow and set:

- **Source**: Database
- **Workflow**: choose **Tag an Entity** from the list (or select **By ID** and paste its workflow ID)
- **Mode**: **Run once for each item**, so each record gets its own call
- **Workflow Inputs**: the three fields below appear once the workflow is selected

**Inputs**

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `entityType` | String | Yes | `courseTemplate` or `learningPath` (case-sensitive). |
| `entityId` | String | Yes | The GraphQL ID of the Course Template or Learning Path. |
| `tagsCsv` | String | No | Comma-separated Learning Tag names, for example `Leadership, Online, Level 2`. Empty means nothing to do. |

Example mapping on the Execute Workflow node:

```text
entityType = courseTemplate
entityId   = {{ $json.courseTemplateId }}
tagsCsv    = {{ $json.tags }}
```

**Output**

The call returns one item:

| Field | Type | Description |
|-------|------|-------------|
| `success` | Boolean | `true` if every tag was found or created and `addTags` returned no errors. Also `true` when there were no tags to add. |
| `message` | String | A short human-readable summary. |
| `entityType` | String | As passed in. |
| `entityId` | String | As passed in. |
| `tagNames` | Array | The cleaned, de-duplicated tag names. |
| `learningTagIds` | Array | The Learning Tag IDs that were (or would have been) added. |
| `createdTagNames` | Array | Tags that did not exist and were created during this call. |
| `errors` | Array | Error messages. Empty on success. |

Example result:

```json
{
  "success": true,
  "message": "Added 2 tag(s) to courseTemplate <COURSE_TEMPLATE_ID>.",
  "entityType": "courseTemplate",
  "entityId": "<COURSE_TEMPLATE_ID>",
  "tagNames": ["Leadership", "Online"],
  "learningTagIds": ["<LEARNING_TAG_ID_1>", "<LEARNING_TAG_ID_2>"],
  "createdTagNames": ["Online"],
  "errors": []
}
```

Branch on `success` with an **If** node in the calling workflow to handle failures. Input problems (an unsupported `entityType`, a missing `entityId` or more than one item in a single call) and GraphQL errors are returned this way. Connection or authentication failures on the HTTP nodes still stop the sub-workflow and show as an error on the calling Execute Workflow node.

### Testing

1. Create a new workflow with a **Manual Trigger**
2. Add a **Set** node with three string fields: `entityType` = `courseTemplate`, `entityId` = the ID of a test Course Template and `tagsCsv` = `Automator Test, Another Test`
3. Add an **Execute Workflow** node that calls **Tag an Entity**, mapping the three inputs from the Set node
4. Run the workflow. The result should show `success: true` and list any tags it created in `createdTagNames`
5. Open the Course Template in Administrate and check that both tags are on it
6. Run it again. It should still succeed, with `createdTagNames` empty because the tags now exist
7. Change `entityType` to `event` and run it again. The result should show `success: false` with an error explaining the allowed values
8. Clear `tagsCsv` and run it. The result should show `success: true` with the message that nothing was changed

Repeat with `entityType` = `learningPath` and a test Learning Path if you plan to tag those too.

## ⚙️ How It Works

1. **When Called**: receives `entityType`, `entityId` and `tagsCsv` from the calling workflow.
2. **Config**: holds the GraphQL URL.
3. **Validate Input**: checks the inputs and splits `tagsCsv` into unique, trimmed names. If the input is invalid or there are no tags, **Return Early Result** returns the result straight away.
4. **Split Tags / Find Tag**: looks up each tag by exact name with the `learningTags` query.
5. **Find Missing Tags**: lists the tags that were not found. It always outputs at least one item, so only one branch of **Any Tags to Create?** runs.
6. **Create Tag**: creates each missing tag with the `learningTags.create` mutation.
7. **Collect Tag Ids**: gathers the IDs of found and created tags and records any lookup or creation errors.
8. **Can Add Tags?**: if any tag failed, skips the mutation and goes straight to the result.
9. **Build Add Tags Request / Add Tags**: calls `courseTemplate.addTags` or `learningPaths.addTags` with the tag IDs.
10. **Build Result**: checks the mutation response for errors and returns the result item.

## 🩺 Troubleshooting

**The result says `entityType must be one of courseTemplate, learningPath`:**
- The value is case-sensitive. Use exactly `courseTemplate` or `learningPath`.

**The result says `Expected one item per call`:**
- Set the calling **Execute Workflow** node's **Mode** to **Run once for each item**.

**An `addTags failed` error mentions the ID:**
- Check that `entityId` is the GraphQL ID of a record of the type given in `entityType`. A Learning Path ID passed with `courseTemplate` fails.

**A tag was created even though the call failed:**
- Tags are created before `addTags` runs and are not removed if it fails. Run the call again once the problem is fixed; the existing tags are reused.

**A tag is created again with different capitalisation:**
- Tags are matched by exact name. Keep tag names consistent in the calling workflow.

**The calling workflow shows an authentication or connection error:**
- Check the Administrate OAuth2 credential on **Find Tag**, **Create Tag** and **Add Tags**.
- Check `GRAPHQL_URL` in **Config**.
