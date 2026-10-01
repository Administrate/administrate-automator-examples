# Alert Opportunity Owners When a Reserved Event Is Cancelled

## 🧩 Problem

Sales teams reserve places on Events against Opportunities. When one of those Events is cancelled, the Reservations and Interests stay attached to it, and nobody tells the people who own the Opportunities. Clients find out late, places on alternative Events fill up and the Opportunity owner has to piece together which deals were affected.

## 🤖 Automator Solution

When an Event is cancelled, this workflow finds every Opportunity Interest on it and emails each Opportunity owner. Each owner gets **one** email that lists all of their affected Opportunities, with links to the cancelled Event and to each Opportunity, so they can move the Reservations to another Event or remove them.

## ✨ Features

- Runs automatically when an Event is cancelled
- Groups Interests by Opportunity owner, so each owner receives a single email however many of their Opportunities are affected
- Lists each affected Opportunity once, with its Account name and a direct link
- Pages through Events with many Interests (100 per page, up to 5,000 Interests)
- Skips and logs any Interest with no Opportunity, no owner or an owner without a linked Contact
- Sends from a sending address of your choice, with optional CC addresses
- Passes all email content to the API as GraphQL variables, so names containing quotes or special characters don't break the request

## 🛠️ Setup Instructions

### 📋 Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read Events and Opportunities and to send ad hoc emails to Contacts
- A verified sending email address in Administrate
- Opportunity owners (Users) linked to a Contact record, because the email is sent to that Contact

### 📥 Installation

1. Download the [`workflow.json`](workflow.json) file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### ⚙️ Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Event Cancelled`
- `Find Event Interests`
- `Send Notification`

#### 2. Fill in the Config node

| Field | Default | Value |
|-------|---------|-------|
| `INSTANCE_SUBDOMAIN` | `YOUR-INSTANCE` | The part of your Administrate URL before `.administrateapp.com`. It's used to build the Event and Opportunity links in the email. |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` | The ID of the sending email address the emails come from. |
| `CC_EMAILS` | empty | Optional. Comma-separated addresses copied on every email, for example `admin@example.com, sales@example.com`. |
| `EMAIL_SUBJECT` | `Your client reservation is at risk: event cancelled` | The email subject. |
| `SIGN_OFF` | `Many thanks<br><br>Your Organisation` | The closing lines of the email. HTML such as `<br>` is allowed. |

To find the sending address ID, run this query in the Administrate GraphQL explorer:

```graphql
query {
  sendingEmailAddresses {
    edges { node { id name address verified } }
  }
}
```

The opening lines of the email are in the `body` field of **Build Owner Email**. Edit them there if you want different wording.

#### 3. Optional: record skipped Interests

**Log Skipped Interest** produces one item for each Interest that couldn't be emailed. Each item contains `status`, `reason`, `timestamp`, `interestId`, `opportunityId`, `opportunityName`, `accountName` and the Event details. To keep a record, connect a node after it, for example Slack, Google Sheets or an Administrate integration log.

### 🧪 Testing

1. Activate the workflow. The Administrate Trigger registers an *Event Cancelled* webhook for you.
2. Create a test Event and add Interests to it from two Opportunities owned by the same User, and one owned by a different User.
3. Cancel the Event.
4. Check that the first User receives one email listing both Opportunities and the second User receives one email listing theirs.
5. Repeat with an Opportunity whose owner has no linked Contact. That Interest should appear in **Log Skipped Interest** and no email should be sent for it.
6. Cancel an Event with no Interests. The run should stop at **Group Interests by Owner** without sending anything.

## 🔍 How It Works

1. **Event Cancelled**: an *Event Cancelled* webhook fires in Administrate.
2. **Config**: holds the instance subdomain, sending address, CC list and email wording.
3. **Find Event Interests**: fetches the Event and its Interests, 100 per page, with each Interest's Opportunity, Account and owner.
4. **Group Interests by Owner**: combines the pages, stops with an error if the API returned errors, groups the Interests by the owner's Contact and builds the list of affected Opportunities. Interests that can't be emailed are passed on with a `reason`.
5. **Owner Contact Found?**: sends owner groups to the email branch and skipped Interests to **Log Skipped Interest**.
6. **Build Owner Email**: assembles the recipient, subject, body, sending address and CC list.
7. **Send Notification**: calls the `sendContactAdhocEmail` mutation once for each owner.

## 🩺 Troubleshooting

**No emails are sent:**
- Check the workflow is active and the *Event Cancelled* webhook exists in Administrate.
- Check the Event had Interests before it was cancelled. Events with none end at **Group Interests by Owner**.
- Look at **Log Skipped Interest**. If every Interest is there, the Opportunity owners aren't linked to Contacts.

**`Send Notification` returns an error about the sending address:**
- Check `SENDING_ADDRESS_ID` is the ID of a verified sending address in this instance.

**The links in the email don't open the right record:**
- Check `INSTANCE_SUBDOMAIN` matches your Administrate URL. For `https://YOUR-INSTANCE.administrateapp.com` it's `YOUR-INSTANCE`.

**Only part of the Interests are listed:**
- **Find Event Interests** stops after 50 pages (5,000 Interests). Raise **Max Pages** in its pagination options if you need more.
