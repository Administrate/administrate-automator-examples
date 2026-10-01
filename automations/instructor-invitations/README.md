# Instructor and Staff Invitations

## 🧩 Problem

Before an Event can go live, someone has to find people to deliver it. Coordinators email instructors and other staff, wait for replies scattered across inboxes, chase the ones who go quiet and keep a mental tally of who has said yes. Replies such as "maybe" or "I can, but my record says I can't" get lost. Nothing in Administrate shows at a glance how close an Event is to being fully staffed, and the Event stays in draft until someone remembers to publish it.

Simple RSVP links also have a hidden risk: corporate mail scanners open every link in an email before the recipient sees it. If opening a link records an answer, invitations answer themselves on delivery.

## 💡 Automator Solution

A set of seven linked workflows turns staffing into a tracked process:

- You invite people from the Event in Administrate as you do today, with a link to a response page.
- Each person gets a **personal link with a random token** and answers **Yes**, **No** or **Maybe**, with comments, on a form. Opening a link never changes anything; only submitting the form does.
- A **Yes** assigns them to the role the Event still needs. A **No** takes them off the Event if they were already on it.
- The coordinator and the responder are emailed, every answer is logged on the Event and a staffing summary custom field is kept up to date.
- A daily sweep reminds people who have not given a firm answer and tells the coordinator when an invitation expires.
- When staffing is complete, the coordinator gets a personal link to review and publish the Event, or it can publish automatically.

## ✨ Features

- Works for both invitation patterns: candidates who are not on the Event (Email to Contacts) and people already staffed who need to confirm (Email to Staff)
- Personal, unguessable tokens on every link that can lead to a change, checked on the page and again on submission
- Scanner safe: pages are read only, and every change happens on a form POST
- Roles are read from the Event's required personnel, so it works for instructors, assistants or any other staff role
- On-behalf-of recording: the coordinator can record an answer that arrived by phone or email
- Clear handling when a contact's record lacks the role the Event needs: the coordinator is told how to fix it
- Staffing summary on the Event, for example `1 of 2 Instructor, 1 of 1 Assistant confirmed - 1 still needed, 1 undecided`
- Opt-in reminders and expiry notices per Event, driven by a custom field
- Optional automatic publishing once staffing is genuinely complete, plus an opt-out field
- Every outcome and failure written to Administrate's External Integration Log against the Event
- All wording, role names, labels and styling in **Config** nodes

## 🗂️ The Workflows

| File | Workflow | Trigger | Purpose |
|---|---|---|---|
| `workflows/10-error-handler.json` | Instructor Invitations - Error Handler | Error Trigger | Writes failures from the other workflows onto the affected Event |
| `workflows/20-recompute-event-summary.json` | Instructor Invitations - Recompute Event Summary | Execute Workflow (called by 40 and 50) | Calculates the staffing summary, writes it to the Event and, when staffing has just completed, publishes automatically or sends the staffing complete email |
| `workflows/30-response-form.json` | Instructor Invitations - Response Form | Webhook (GET) | The page an invitation link opens. Checks the token and shows the form. Writes nothing |
| `workflows/40-response-handler.json` | Instructor Invitations - Response Handler | Webhook (POST) | Records answers, emails personal links, assigns or removes staff |
| `workflows/50-reminders-and-expiry.json` | Instructor Invitations - Reminders and Expiry | Schedule (08:00 daily) | Sends reminders and expiry notices and refreshes summaries, which can complete staffing |
| `workflows/60-publish-event-page.json` | Instructor Invitations - Publish Event Page | Webhook (GET) | The coordinator's review page from the staffing complete email. Writes nothing |
| `workflows/70-publish-event-handler.json` | Instructor Invitations - Publish Event Handler | Webhook (POST) | Publishes the Event and optionally tells everyone staffed |

The chain is: your invitation email links to **30** → **30** posts to **40** → **40** calls **20** → when staffing completes, **20** emails a link to **60** → **60** posts to **70**. **50** runs daily and also calls **20**, so an Event that becomes fully staffed during the sweep gets the same email. **10** watches 30, 40, 50, 60 and 70.

### How invitations start

**No workflow in this example sends the first invitation.** The coordinator sends it from the Event in Administrate, using **Email to Contacts** for candidates who are not on the Event or **Email to Staff** for people already staffed, with a communication template that links to **30 Response Form** (see [Invitation email template](#7-invitation-email-template)). Administrate does not tell n8n when this email is sent, so an invitation only becomes a row in the invitations Data Table when the person asks for their personal link or when an answer is recorded.

## 🔐 Personal Link Tokens

Every link that can lead to a recorded change carries a random token:

| Link | Format | Token stored in | Created by |
|---|---|---|---|
| Personal response link | `30 URL?e=<event>&c=<contact>&t=<token>` | `token` column of the invitations row (event plus contact) | 40 when the person asks for their link, or 50 when it sends a reminder to a row without one |
| Publish link | `60 URL?e=<event>&c=<contact>&t=<token>` | `token` column of the publish links row (one per Event) | 20 just before it sends the staffing complete email, whether 40 or 50 called it |

- Tokens are 32 random bytes from a secure source (`crypto.randomBytes`, or the Web Crypto API), hex encoded. They are never derived from Event or Contact IDs, so knowing the IDs is not enough.
- The token is checked on the GET page and again on the POST, using a constant-time comparison. A missing or wrong token shows an *invalid or expired link* page and nothing is written.
- The invitation you send from Administrate carries only the Event and Contact. Opening it shows a **Get your personal response link** page with an **Email me my link** button. Pressing the button (a POST, so mail scanners do not trigger it) creates or reuses the token and emails the personal link to the address on the Contact record. The page that follows is the same whether or not anything was sent, so it does not reveal whether an Event or Contact exists.
- Asking again reuses the existing token, so it cannot break a link already sent. Requests within `LINK_RESEND_MINUTES` of the last one send nothing.
- A new staffing complete email replaces the Event's publish token, so links in older emails expire.

## 🛠️ Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials able to read Events and Contacts, add and remove Event staff, update Events, send ad hoc emails and write External Integration Logs
- An n8n API credential in the same n8n project (used by the error handler to read failed executions)
- n8n Data Tables enabled
- A verified Administrate sending address
- Staff roles set up in Administrate, including one for the coordinator (for example **Coordinator**) and the roles you invite people for (for example **Instructor**)
- Required personnel set on your Events, so the workflows know which roles are needed
- Code nodes able to create secure random values. The token Code nodes use `crypto.randomBytes` and fall back to the Web Crypto API. If your n8n blocks both, set `NODE_FUNCTION_ALLOW_BUILTIN=crypto`.

### Installation

Import all seven files from the `workflows/` folder in this order. In your Automator instance, click the menu (⋮) and select "Import from File" for each.

1. `10-error-handler.json`, first, so it exists when you set it as the Error Workflow on the others
2. `20-recompute-event-summary.json`, so you can copy its ID into 40 and 50
3. `30-response-form.json`
4. `40-response-handler.json`
5. `50-reminders-and-expiry.json`
6. `60-publish-event-page.json`
7. `70-publish-event-handler.json`

### Configuration

#### 1. Create the Data Tables

Create three n8n Data Tables. Every column is of type **string**.

| Table (suggested name) | Columns |
|---|---|
| `InstructorInvitations` | `event_id`, `event_title`, `contact_id`, `contact_name`, `contact_email`, `staff_role`, `status`, `responded_at`, `response_channel`, `last_reminder_at`, `updated_at`, `token`, `link_sent_at` |
| `InstructorInvitationCorrespondence` | `event_id`, `contact_id`, `direction`, `channel`, `occurred_at`, `answer`, `comments`, `submitted_by` |
| `InstructorInvitationPublishLinks` | `event_id`, `token`, `created_at` |

The invitations table holds one row per Event and Contact. Its `status` is one of `invited`, `coordinator`, `maybe`, `accepted`, `declined`, `assign_failed`, `reminded` or `expired`. The correspondence table is an append-only history of every answer, stored exactly as written. Treat the `token` columns as secrets: anyone who can read them can answer for that person.

#### 2. Create the Event custom fields

| Field (suggested label) | Type | Used by | Purpose |
|---|---|---|---|
| Staffing Summary | Text | 20 | Shows the summary. Written by the workflows |
| Send Reminders | Yes/No | 50 | Yes opts the Event in to reminders and expiry notices |
| Publish Automatically | Yes/No | 20 | Yes publishes the Event as soon as staffing is complete |
| Suppress Completion Email | Yes/No | 20 | Yes stops the staffing complete email to the coordinator, whether staffing completes on an answer or during the daily sweep |
| Do Not Publish Automatically | Yes/No | 70 | Yes stops the publish page from publishing |

Read each field's `key` with this query in the Administrate GraphQL explorer:

```graphql
query {
  customFieldTemplate(type: Event) {
    customFieldDefinitions { key label type }
  }
}
```

#### 3. Set credentials

- **Administrate OAuth2**: every HTTP Request node in 10, 20, 30, 40, 50, 60 and 70 except **Fetch Failed Execution**
- **n8n API**: **Fetch Failed Execution** in 10

#### 4. Link the workflows

The workflows call each other by URL and by ID. The imported files already contain matching webhook paths, so for URLs you normally only need to replace `YOUR-N8N-HOST` with your n8n host name. n8n can change a webhook path on import if it clashes with an existing one, so **after importing, open each webhook node, copy its Production URL and compare**:

| Where | Config field | Must equal |
|---|---|---|
| 30 Config | `SUBMIT_URL` | 40 `Response Submitted` production URL |
| 40 Config | `RESPONSE_BASE_URL` | 30 `Invitation Link` production URL |
| 20 Config | `PUBLISH_PAGE_URL` | 60 `Publish Link` production URL |
| 50 Config | `RESPONSE_BASE_URL` | 30 `Invitation Link` production URL |
| 60 Config | `PUBLISH_SUBMIT_URL` | 70 `Publish Submitted` production URL |
| 10 Config | `N8N_API_URL` | `https://YOUR-N8N-HOST/api/v1` |
| 40 and 50 Config | `SUMMARY_WORKFLOW_ID` | ID of 20, from its URL (`/workflow/<id>`). Replace `REPLACE_WITH_ID_OF_RECOMPUTE_EVENT_SUMMARY_WORKFLOW` |

Your invitation template (step 7) must also use the 30 production URL. Once invitations have gone out, do not change the 30 path.

#### 5. Error workflow

Open **Workflow Settings** on 30, 40, 50, 60 and 70 and set **Error Workflow** to `Instructor Invitations - Error Handler`. The imported files do not carry this setting, because workflow IDs differ between instances.

#### 6. Placeholders and settings

| Workflow | Config field | Default | Replace with |
|---|---|---|---|
| 20, 30, 40, 50 | `INVITATIONS_TABLE_ID` | `REPLACE_WITH_INVITATIONS_TABLE_ID` | ID of the invitations Data Table |
| 40 | `CORRESPONDENCE_TABLE_ID` | `REPLACE_WITH_CORRESPONDENCE_TABLE_ID` | ID of the correspondence Data Table |
| 20, 60, 70 | `PUBLISH_LINKS_TABLE_ID` | `REPLACE_WITH_PUBLISH_LINKS_TABLE_ID` | ID of the publish links Data Table |
| 20 | `SUMMARY_FIELD_KEY` | `REPLACE_WITH_STAFFING_SUMMARY_FIELD_KEY` | Key of the Staffing Summary field |
| 50 | `REMINDERS_FIELD_KEY` | `REPLACE_WITH_SEND_REMINDERS_FIELD_KEY` | Key of the Send Reminders field |
| 20 | `AUTO_PUBLISH_FIELD_KEY` | `REPLACE_WITH_AUTO_PUBLISH_FIELD_KEY` | Key of the Publish Automatically field |
| 20 | `SUPPRESS_EMAIL_FIELD_KEY` | `REPLACE_WITH_SUPPRESS_COMPLETION_EMAIL_FIELD_KEY` | Key of the Suppress Completion Email field |
| 70 | `SUPPRESS_PUBLISH_FIELD_KEY` | `REPLACE_WITH_DO_NOT_PUBLISH_FIELD_KEY` | Key of the Do Not Publish Automatically field |
| 20, 40, 50, 70 | `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` | ID of a verified sending address, from `sendingEmailAddresses` |
| 30, 40 | `FALLBACK_ROLE_ID` | `REPLACE_WITH_FALLBACK_STAFF_ROLE_ID` | Staff role ID used when an Event has no required personnel, from `staffRoles` |
| 70 | `CONFIRMED_TEMPLATE_ID` | `REPLACE_WITH_EVENT_CONFIRMED_TEMPLATE_ID` | ID of a staff audience communication template sent when the Event goes live, from `communicationTemplates` |
| 50 | `FALLBACK_LOG_EVENT_ID` | `REPLACE_WITH_FALLBACK_LOG_EVENT_ID` | ID of a draft Event kept only to hold failure logs when no Event is known |
| 40, 50 | `SUMMARY_WORKFLOW_ID` | `REPLACE_WITH_ID_OF_RECOMPUTE_EVENT_SUMMARY_WORKFLOW` | ID of 20 |
| All | `ORIGIN_IDENTIFIER` | `Automator - Instructor Invitations` | Label for log entries. Keep it identical everywhere |
| 20 to 70 | `COORDINATOR_ROLE_NAME` | `Coordinator` | Exact name of your coordinator staff role. It is excluded from staffing counts, receives notifications and can record answers and publish |
| 30, 40, 50 | `FALLBACK_ROLE_NAME` | `Instructor` | Name of the fallback role |
| 20 to 60 | `VERB_BASE` / `VERB_GERUND` | `deliver` / `delivering` | What people do with an Event, as in *available to deliver this event*. Enter in lower case and keep them the same everywhere |
| 40 | `ANSWER_LABEL_YES` / `_NO` / `_MAYBE` / `_UNABLE` | `Yes, available` and so on | Answer wording in emails and the log |
| 40 | `LINK_RESEND_MINUTES` | `10` | Minimum gap between personal link emails to the same person |
| 20, 50, 70 | `CHECKBOX_TRUE_VALUES` | `true,1,yes,on` | Field values that count as Yes. Anything else means No |
| 20, 70 | `PUBLISHED_STATE` | `published` | Lifecycle state used when publishing |
| 20 | `COMPLETE_PREFIX` | `Staffing complete` | Start of a complete summary. Also the marker that the completion email has gone, so changing it makes every draft Event look newly complete once |
| 50 | `CHASEABLE_STATUSES` | `maybe,reminded,invited` | Statuses that still get reminders |
| 50 | `REMINDER_FIRST_DAYS` / `REMINDER_SECOND_DAYS` / `EXPIRY_DAYS` | `3` / `7` / `10` | Reminder and expiry cadence in days |
| 10 | `TRANSIENT_MARKERS`, `EVENT_ID_NODES`, `EVENT_ID_FIELDS` | | How the error handler classifies failures and finds the Event |
| 20 to 70 | `BRAND_NAME` | `Your Organisation` | Masthead text on pages and emails |
| 30, 40, 60, 70 | `BRAND_CSS` | Neutral blue theme | Page styling |
| 20, 40, 50 | `EMAIL_PALETTE`, `EMAIL_SHELL`, `EMAIL_BUTTON` | Neutral blue theme | Email colours, frame and button. Palette entries are `NAME=value` pairs separated by `\|`. Use inline styles only. Never quote a font name |

Useful lookup queries:

```graphql
query { staffRoles { edges { node { id name } } } }
query { sendingEmailAddresses { edges { node { id name address verified } } } }
query { communicationTemplates { edges { node { id name audience } } } }
```

#### 7. Invitation email template

Create communication templates for the invitations you send from the Event. Each needs a link to **30 Response Form** with the Event and Contact encoded IDs:

- **Contacts audience** (Email to Contacts, for candidates):
  `https://YOUR-N8N-HOST/webhook/<30 path>?e={{ event.encoded_id }}&c={{ contact.encoded_id }}`
- **Staff audience** (Email to Staff, for people already on the Event and for the coordinator):
  `https://YOUR-N8N-HOST/webhook/<30 path>?e={{ staff.event.encoded_id }}&c={{ staff.contact.encoded_id }}`

Check the merge field names in your template editor. Tell recipients that the link takes them to a page where they can ask for their personal response link. To record answers on someone's behalf, the coordinator opens the same link from a staff email and asks for their own personal link.

### Testing

1. Fill in every placeholder, activate 10, 30, 40, 50, 60 and 70 (20 runs when called) and set the Error Workflow on each.
2. On a draft test Event, set required personnel (for example 1 Instructor), staff a coordinator and set **Send Reminders** to Yes.
3. Send yourself the invitation from a test Contact that holds the Instructor role. Open the link: you should see **Get your personal response link**, and nothing should be written to the Data Tables.
4. Press **Email me my link**. A row with status `invited` and a token appears, and the personal link arrives by email.
5. Change one character of the token in the URL and open it: you should see **This link is invalid or has expired**. Post the form with a wrong token (for example from your browser's developer tools): you should get the same page and nothing changes.
6. Open the correct link and answer **Maybe**. The coordinator and the responder are emailed, the row becomes `maybe` and the Staffing Summary shows `1 undecided`.
7. Answer **Yes** from the same link. The Contact is assigned as Instructor, the summary reads `Staffing complete - ...` and the coordinator receives a staffing complete email with a publish link.
8. Open the publish link, then publish. Open the link again: it now says the Event is already published. Change its token: you should get the invalid link page.
9. Repeat with a Contact that lacks the role to check the *cannot be recorded yet* path, and answer **No** as someone already staffed to check removal.
10. Run 50 by hand with a row whose `updated_at` is four days old to see a reminder with a personal link.
11. On a second draft Event with no coordinator, answer **Yes** until staffing is complete. The summary ends with a *no coordinator* warning and no email goes. Add the coordinator in Administrate and run 50 by hand: the coordinator gets one staffing complete email with a publish link. Run 50 again: nothing more is sent.
12. Delete test rows from the Data Tables when you have finished (delete rows, not the tables).

## ⚙️ How It Works

1. **Invite**: the coordinator sends the invitation from Administrate. The link opens **30 Response Form** with `e` and `c` only.
2. **Ask for a personal link**: without a valid token, 30 shows no Event details, only an **Email me my link** button. It posts to **40**, which reads the Event and Contact, reuses or creates the token, saves the invitation row (`invited`, or `coordinator` for the Event's coordinator, who is never chased or counted) and emails the personal link.
3. **Open the personal link**: 30 checks the token against the invitation row, reads the Event's staffing and shows one of three views. A candidate sees Yes, No and Maybe. Someone already staffed sees the same with a note that No takes them off. The coordinator sees an on-behalf-of picker. If the Contact cannot hold any role the Event still needs, the page explains why and offers to tell the coordinator.
4. **Submit**: **40** checks the token again, then re-reads staffing from Administrate. It upserts the invitation row, appends a correspondence row, assigns the needed role on Yes or removes the staff record on No, emails the coordinator and responder, logs the outcome and calls **20**.
5. **Summarise**: **20** compares accepted rows with each required role and writes the summary field only when it changed.
6. **Complete**: when staffing has *just* become complete and that summary write succeeded, 20 publishes automatically if **Publish Automatically** is Yes. Unless **Suppress Completion Email** is Yes, it saves a fresh publish token (replacing the Event's old one) and emails the coordinator a link to **60**, or tells them the Event is now live. This is the same whether 40 or 50 called it. The complete summary on the Event is the marker that the email has gone, so later calls send nothing and one completion gives one email. If the summary write fails, nothing is sent and the next call tries again.
7. **Publish**: **60** checks the publish token and shows staffing against requirements. Submitting posts to **70**, which checks the token, the coordinator role, the opt-out field and the draft state, then publishes, optionally emails everyone staffed using `CONFIRMED_TEMPLATE_ID` and logs who published.
8. **Chase**: every morning **50** reads every Event with invitation rows, 100 at a time, following the pages to the end (up to 5,000 Events). It reminds people with `maybe`, `reminded` or `invited` rows on opted-in draft Events, using their personal link, and tells the coordinator when someone reaches `EXPIRY_DAYS`. It also refreshes the summary of every draft Event with invitation rows through 20, so an Event that has become complete since it was last checked, for example because a coordinator was added after the last acceptance, gets its staffing complete email then.
9. **Failures**: **10** recovers the Event ID from the failed execution and writes a plain-language entry to that Event's External Integration Log.

## 🧯 Troubleshooting

**The invitation link shows "Get your personal response link" every time:**
- This is expected for the link in the Administrate email. The personal link is the one emailed after pressing **Email me my link**.

**"Email me my link" shows "Check your email" but nothing arrives:**
- The Contact has no email address, or another request was made within `LINK_RESEND_MINUTES`
- `SENDING_ADDRESS_ID` is not a verified sending address
- `e` or `c` in the invitation link is wrong, for example a merge field was mistyped. Check 40's executions.

**A personal link says "invalid or expired":**
- The row was deleted or its `token` changed. Ask for a new link from the invitation page.
- The link was copied incompletely. The token is 64 characters.

**A publish link says "invalid or expired":**
- A newer staffing complete email has replaced the token. Use the most recent email or publish in Administrate.

**Error "No secure random source available":**
- The Code node cannot reach a secure random generator. On self-hosted n8n, set `NODE_FUNCTION_ALLOW_BUILTIN=crypto` and restart.

**Answers are refused with "Answer cannot be recorded yet":**
- The Contact's record lacks the staff role the Event needs, or is not marked as an instructor or qualified. Add the role in Administrate and ask them to open their personal link again.

**No staffing complete email:**
- The Event has no staff member with `COORDINATOR_ROLE_NAME`. The summary says so. Add one and the email goes on the next 50 sweep.
- **Suppress Completion Email** is Yes, or the Event is not in draft
- The summary already started with `COMPLETE_PREFIX`, which means the email has already gone for this completion
- Writing the summary failed. Nothing is sent, and the next answer or 50 sweep tries again.
- The External Integration Log on the Event says the email to the coordinator failed. Check `SENDING_ADDRESS_ID` in 20, then send the link by hand or publish in Administrate.

**Two staffing complete emails for one Event:**
- Expected only when staffing dropped below complete (someone declined or was removed) and then completed again. Each email carries a new publish token and older links stop working.
- Otherwise check that 40 and 50 both call the same 20 and that nobody has cleared the Staffing Summary field by hand

**Nobody is chased:**
- **Send Reminders** must be Yes on the Event and the Event must be a draft. Rows only exist once someone has asked for their link or answered.

**Some Events are never chased or refreshed:**
- 50 reads at most 50 pages of 100 Events. Above 5,000 Events with invitation rows, raise **Max Pages** in the pagination options of **Fetch Draft Events**, or delete rows for Events that have finished.

**Data Table or sub-workflow errors:**
- Check every `*_TABLE_ID`, that the tables have all their columns and that `SUMMARY_WORKFLOW_ID` is the ID of 20.

**Failures are not logged on the Event:**
- Set the Error Workflow on each workflow. Error workflows fire on production runs only.
- Give **Fetch Failed Execution** an n8n API credential from the same project.

## ⚖️ How It Compares to Instructor Jump Ball

[Instructor Jump Ball](../instructor-jump-ball/) is a single workflow for a quick first-come assignment: the instructor clicks Accept or Decline in an email and is assigned straight away. It is easy to set up, but opening the link is the answer, so mail scanners can trigger it. Anyone who knows the IDs can also build a working link.

This example is the fuller process: personal tokens, a form that only acts on submission, Maybe answers with comments, any staff role from the Event's required personnel, on-behalf-of recording, reminders and expiry, a live staffing summary and a publish step. Choose Jump Ball for simple single-instructor Events; choose this when several roles must be filled and tracked before an Event goes live.
