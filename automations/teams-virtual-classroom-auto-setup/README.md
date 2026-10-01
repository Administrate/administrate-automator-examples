# Teams Virtual Classroom Auto Setup

## 🧩 Problem

Virtual Events that run on Microsoft Teams need the same setup every time: book the meeting host on the Event, switch the virtual classroom to Microsoft Teams and assign the host so Administrate can generate the Teams link. Doing this by hand for every new Event is slow and easy to forget, and a missed step means learners get no joining link.

## 🤖 Automator Solution

When an Event is created at your virtual location, the main workflow books the host contact with the host staff role, sets the virtual classroom type to Microsoft Teams and assigns the host. Administrate then creates the Teams meeting and its joining link.

An optional add-on workflow adds the Event's internal instructors as co-organisers of that Teams meeting through Microsoft Graph, so they can manage the meeting without the host account.

## ✨ Features

- Runs automatically on every *Event Created* webhook
- Only acts on Events at the location named in Config
- Books the host, configures Microsoft Teams and assigns the host in the order Administrate requires
- Two guards: one on the webhook payload and one on a fresh read of the Event, so an existing classroom or link is never overwritten
- Leaves Events that use another virtual classroom provider alone
- Produces success, error and skip log items that you can route anywhere
- Optional add-on: adds internal instructors as Teams co-organisers, keeps existing attendees and logs external instructors it skips

## 🛠️ Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials
- Administrate's **Microsoft Teams integration** connected in your Administrate instance
- A contact that acts as the Teams meeting host, linked to a Microsoft 365 account through the Teams integration
- A staff role for that host (for example "Virtual Classroom Host")
- A location used for your Teams Events (for example "Virtual")

For the optional co-host add-on you also need:

- A Microsoft Entra app registration with the Microsoft Graph **application** permissions `OnlineMeetings.ReadWrite.All` and `User.Read.All`, with admin consent granted
- An [application access policy](https://learn.microsoft.com/graph/cloud-communication-online-meeting-application-access-policy) that allows the app to manage online meetings for the host account. Without it, Graph rejects the meeting calls even with the permission granted.
- An n8n **OAuth2 API** credential using the client credentials grant for that app (token URL `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`, scope `https://graph.microsoft.com/.default`)

### Installation

This example has two workflows in the [workflows](workflows/) folder:

| File | Purpose | Required |
|------|---------|----------|
| [10-teams-virtual-classroom-setup.json](workflows/10-teams-virtual-classroom-setup.json) | Configures the Teams virtual classroom on new Events | Yes |
| [20-instructor-co-host-add-on.json](workflows/20-instructor-co-host-add-on.json) | Adds internal instructors as Teams co-organisers | No, optional |

1. Download the workflow JSON files you need
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Import `10-teams-virtual-classroom-setup.json` and, if you want the add-on, `20-instructor-co-host-add-on.json`

The two workflows are independent. The add-on only acts on Events that already have a Microsoft Teams classroom, which the main workflow sets up.

### Configuration

#### 1. Main workflow: set credentials

Select your Administrate OAuth2 credential on:
- `Administrate Trigger`
- `1. Add Personnel`
- `Check Classroom State`
- `2. Configure Virtual Classroom`
- `3. Assign Host`

#### 2. Main workflow: fill in the Config node

| Field | Value |
|-------|-------|
| `LOCATION_NAME` | Replace `REPLACE_WITH_VIRTUAL_LOCATION_NAME` with the exact name of the location used for Teams Events. The match is case sensitive. |
| `HOST_CONTACT_ID` | Replace `REPLACE_WITH_HOST_CONTACT_ID` with the GraphQL ID of the host contact. |
| `HOST_STAFF_ROLE_ID` | Replace `REPLACE_WITH_HOST_STAFF_ROLE_ID` with the ID of the host staff role. |
| `ADMINISTRATE_API_URL` | Defaults to `https://api.getadministrate.com/graphql`. Change it only if your instance uses a different API endpoint, such as `https://YOUR-INSTANCE.administrateapp.com/graphql`. |

To find the contact and staff role IDs, run these queries in the Administrate GraphQL explorer:

```graphql
query {
  contacts(filters: [{ field: lastName, operation: eq, value: "Host" }]) {
    edges { node { id firstName lastName emailAddress } }
  }
  staffRoles {
    edges { node { id name } }
  }
}
```

#### 3. Optional add-on: set credentials

- Select your Administrate OAuth2 credential on `Administrate Trigger`
- Select your Microsoft Graph OAuth2 credential on `Resolve Instructor Graph ID`, `Get Online Meeting` and `Add Co-organisers`. They are named "Microsoft Graph (App-only)" as a placeholder.

#### 4. Optional add-on: fill in the Config node

| Field | Value |
|-------|-------|
| `INTERNAL_DOMAINS` | Replace `REPLACE_WITH_INTERNAL_EMAIL_DOMAINS` with a comma-separated list of the email domains in your Microsoft tenant, for example `example.com,example.org`. Instructors with other domains are skipped. |
| `ORGANISER_MAILBOX_ID` | Replace `REPLACE_WITH_ORGANISER_MAILBOX_OBJECT_ID` with the Microsoft Entra object ID (a GUID) of the host account that owns the Teams meetings. You can find it on the user's page in the Entra admin centre. |
| `INSTRUCTOR_STAFF_ROLE_NAME` | Defaults to `Instructor`. Change it if your instructor staff role has a different name. |

### Testing

1. Activate the main workflow. The Administrate Trigger registers an *Event Created* webhook for you.
2. Create a test Event at the location in `LOCATION_NAME`.
3. Check the Event now has the host contact on its staff list, a Microsoft Teams virtual classroom and a joining link.
4. Create a test Event at a different location and confirm the workflow stops at **Location Matches?**.
5. For the add-on, activate it, add an internal instructor to the test Event, then edit and save the Event. Check the instructor appears as a co-organiser in the Teams meeting options.

## ⚙️ How It Works

### Main workflow

1. **Trigger**: an *Event Created* webhook fires in Administrate.
2. **Config**: holds the location name, host contact, host staff role and API URL.
3. **Location Matches?**: continues only when the Event's location name equals `LOCATION_NAME`.
4. **Already Configured (Payload)?**: skips the Event if it already has a Teams host or another provider is set.
5. **1. Add Personnel**: books the host contact on the Event with the host staff role. This must happen before the host can be assigned.
6. **Check Classroom State / Already Configured (Fresh)?**: re-reads the Event and skips it if the classroom was configured in the meantime.
7. **2. Configure Virtual Classroom**: sets the remote meeting type to Microsoft Teams.
8. **3. Assign Host**: sets the host contact and staff role. Administrate generates the Teams meeting and link.
9. **Build Log**: collects any GraphQL errors and returns a success or error item with the meeting URL.

### Optional co-host add-on

1. **Trigger**: an *Event Updated* webhook fires in Administrate.
2. **Teams Classroom Configured?**: continues only when the Event's classroom type is Microsoft Teams.
3. **Identify Instructors**: finds staff with the instructor role and splits them into internal and external using `INTERNAL_DOMAINS`.
4. **Resolve Instructor Graph ID**: looks up each internal instructor's Microsoft Graph user ID by email address. Instructors that cannot be found are dropped.
5. **Get Online Meeting**: finds the Teams meeting owned by `ORGANISER_MAILBOX_ID` that matches the Event's joining link.
6. **Build Merged Attendee List**: keeps existing attendees and adds the instructors with the `coorganizer` role, because the update replaces the whole attendee list.
7. **Meeting Found?**: continues only when a meeting matched. If none matched, or the lookup failed, **Log - No Meeting Found** returns a `no_meeting_found` item with the reason and the meeting is left unchanged.
8. **Add Co-organisers**: PATCHes the meeting and **Final Log** reports who was added and who was skipped.

## 🩺 Troubleshooting

**The main workflow does nothing on a new Event:**
- Check the workflow is active and the *Event Created* webhook exists in Administrate.
- Confirm `LOCATION_NAME` matches the location name exactly, including case.

**The Event has no Teams link:**
- Check Administrate's Microsoft Teams integration is connected and the host contact is linked to a Microsoft 365 account.
- Check `HOST_CONTACT_ID` and `HOST_STAFF_ROLE_ID` are correct. Errors from each mutation appear in the `errors` array of **Build Log**.

**The add-on does not run after I add an instructor:**
- Administrate has no webhook for staff changes. After adding the instructor, edit and save the Event to fire *Event Updated*.
- Each save re-runs the update for all current instructors. This is harmless because it sets the same role again.

**The add-on returns 403 or ends at `Log - No Meeting Found`:**
- The `note` on the `no_meeting_found` item says whether the lookup failed or simply found no meeting with that joining link.
- Check the Graph app has the `OnlineMeetings.ReadWrite.All` application permission with admin consent and that the application access policy covers the host account.
- Check `ORGANISER_MAILBOX_ID` is the object ID of the account that owns the meetings.

**An instructor is not added as a co-organiser:**
- External instructors (domains not in `INTERNAL_DOMAINS`) are skipped because they have no identity in your tenant. They appear in `skippedExternal`.
- Internal instructors whose email address does not match a Microsoft 365 user are dropped at **Collect Resolved Instructors**.
