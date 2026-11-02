# 🔄 Learner Substitution

## 🧩 Problem

A learner often cannot attend and their organisation wants to send a colleague in their place. Done by hand, the administrator cancels one registration and creates another. In between, the place is briefly free and can be taken from a waitlist, and in reporting the swap looks like a withdrawal plus a new booking. Nobody can tell afterwards who replaced whom or why.

## 🤖 Automator Solution

The administrator requests the substitution on the learner record itself, by setting a custom field and entering the colleague's email. On its next run this workflow checks the request, **registers the substitute first** and only then releases the original place, so the place is never exposed in between. Both registrations are tagged as a substitution, notes are added to both and a success entry is logged on the event once the release is verified. A request that cannot be applied is marked as refused, with the reason in the learner notes.

## ✨ Features

- Requested from inside Administrate: no form or external page to open
- Substitute registered before the original place is released, so the place cannot be lost to a waitlist
- Both registrations tagged, so a substitution is not reported as a withdrawal
- Linked audit notes on both registrations, plus an External Integration Log entry on the event written only after the release is verified
- Requests are processed one at a time, so two substitutions on the same event cannot compete for one free place
- A run interrupted after the substitute was registered is finished by the next run instead of being refused
- Every Administrate response is checked: unexpected failures stop the run with a clear message for the error workflow
- Checks before any write: missing email, unknown email, several contacts sharing the email, same person, substitute already registered, event full
- Refused requests are marked `Substitution refused` with the reason, so they are not retried
- Scans a look-back window on a schedule, so a missed run is caught by the next one
- Optional confirmation email to the substitute, styled from the Config node
- Dry-run mode that reports decisions without writing anything

## 🚀 Setup Instructions

### 📋 Prerequisites

- Access to Administrate Automator (n8n)
- An Administrate OAuth2 credential with permission to read and update learners, register learners on events and write External Integration Logs
- A verified Administrate sending email address (only if `SEND_EMAIL` is on)
- Two custom fields on **Learner**, created in Administrate:

| Label (suggested) | Type | Options |
|---|---|---|
| Registration Change Type | Choice | `Substitution requested`, `Substitution completed - place released`, `Substitution completed - place taken over`, `Substitution refused` |
| Substitute Email | Email | |

The option values must match the `VALUE_*` keys in the Config node exactly. You can rename them in both places if your organisation prefers other wording.

### 📥 Installation

1. Download `workflow.json` from this folder.
2. In Automator, create a workflow, open the menu (⋮) and select **Import from File**.
3. Follow the configuration steps below.

### ⚙️ Configuration

#### 🔑 1. Set OAuth credentials

Attach your Administrate OAuth2 credential to the eight HTTP Request nodes: `Scan Registrations`, `Read Replacement and Event`, `Check Existing Registration`, `Register Substitute`, `Release Place`, `Log Success`, `Send Email` and `Mark Refused and Log`.

#### 📝 2. Fill in the Config node

| Key | Set to |
|---|---|
| `CHANGE_TYPE_FIELD_KEY` | `REPLACE_WITH_CHANGE_TYPE_FIELD_KEY` -> definition key of the Registration Change Type field |
| `SUBSTITUTE_EMAIL_FIELD_KEY` | `REPLACE_WITH_SUBSTITUTE_EMAIL_FIELD_KEY` -> definition key of the Substitute Email field |
| `VALUE_REQUESTED`, `VALUE_RELEASED`, `VALUE_TAKEN_OVER`, `VALUE_REFUSED` | The four options of the Registration Change Type field, spelled exactly as in Administrate |
| `LOOKBACK_MINUTES` | How far back each run looks for changed registrations (default `180`) |
| `SCAN_PAGE_SIZE` | Registrations read per request while paging through the scan (default `100`) |
| `DRY_RUN` | `true` to report decisions without writing anything (default `false`) |
| `SEND_EMAIL` | `false` to skip the email to the substitute (default `true`) |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` -> ID of a verified sending address |
| `BRAND_NAME`, `BRAND_COLOUR` | Name and header colour shown in the email |
| `ORIGIN_IDENTIFIER` | Name shown against entries in the event's External Integration Log |

To find the custom field keys:

```graphql
query { customFieldTemplate(type: Learner) { customFieldDefinitions { key label type } } }
```

To find the sending address ID:

```graphql
query { sendingEmailAddresses { edges { node { id name address verified } } } }
```

#### 🚨 3. Optional: set an error workflow

If you use a shared error workflow, such as [Log Automation Failures to Administrate](../error-logging-to-administrate/), select it under **Workflow settings > Error workflow**. The workflow stops with a clear error whenever Administrate returns an unexpected response, for example if the substitute was registered but releasing the original place failed. The error names the learners and the event so an administrator can finish by hand.

### 🧪 Testing

1. Set `DRY_RUN` to `true`.
2. Register a test contact on an upcoming event with free places. On that registration, set Registration Change Type to `Substitution requested` and Substitute Email to the email of a second test contact.
3. Run the workflow with **Run Now** and check the `Dry Run Report` node: it should show `substitute` for your test learner.
4. Set `DRY_RUN` to `false` and run again. Check that:
   - the second contact is registered on the event, tagged `Substitution completed - place taken over`
   - the first registration is cancelled and tagged `Substitution completed - place released`
   - both registrations carry the same substitution note
   - the event's External Integration Logs show a success entry
5. Test the refusals: an unknown email, the learner's own email, a colleague already registered and an empty email. Each registration should be set to `Substitution refused` with the reason added to its notes.
6. Run the workflow again and confirm nothing is processed twice.
7. Optional: put two requests on the same event with one free place left, and check that both succeed.
8. Activate the workflow when you are happy with the results.

## 🔍 How It Works

1. **Every 5 Minutes** (or **Run Now**) reads the Config node.
2. **Build Scan Query** and **Scan Registrations** read every active registration on an upcoming event that changed within `LOOKBACK_MINUTES`, paging through the results `SCAN_PAGE_SIZE` at a time (up to 50 pages per run).
3. **Match Requests** keeps those whose Registration Change Type is `VALUE_REQUESTED`, one item per request. If the page limit is reached before the end of the results, it stops the run rather than treat a partial scan as complete.
4. **Loop Over Requests** handles the requests one at a time, end to end.
5. **Read Replacement and Event** looks up the substitute by email and reads the event and its remaining places. **Decide** refuses the request if the email is missing or unknown, matches several contacts or matches the learner giving up the place.
6. **Check Existing Registration** and **Confirm Not Registered** look for an active registration of the substitute on the event. One created by an earlier, interrupted run for this same request is resumed; any other one leads to a refusal. A fresh substitution is refused if the event is full. Nothing is written in steps 5 and 6.
7. **Register Substitute** registers the substitute with the `VALUE_TAKEN_OVER` tag and a note, and **Check Registration** confirms Administrate accepted it. If Administrate refuses, the request is refused with its message.
8. **Release Place** tags and cancels the original registration, and **Check Release** verifies both. **Log Success** then writes the success entry on the event, and **Check Log** verifies it.
9. **Build Email**, **Send Email** and **Check Email** send and verify the confirmation to the substitute when `SEND_EMAIL` is on.
10. **Prepare Refusal**, **Mark Refused and Log** and **Check Refusal** set refused requests to `VALUE_REFUSED`, add the reason to the learner notes, write a failed entry on the event and verify both.

### 📊 Scan volume

The Administrate API cannot yet filter learners by a custom field value, so each run reads every upcoming registration changed within `LOOKBACK_MINUTES` and keeps the substitution requests in the workflow. On busy instances, keep `LOOKBACK_MINUTES` as short as your schedule allows.

### ⏱️ Overlapping runs

The workflow assumes one run at a time. With the default 5 minute schedule a run normally finishes well before the next one starts. If you run it by hand while a scheduled run is in progress, or schedule it more often, set **Workflow settings > Timeout** below the schedule interval so two runs never work on the same request.

## 🛠️ Troubleshooting

**The run stops with "Scan stopped at the page limit":**
- More registrations changed within `LOOKBACK_MINUTES` than 50 pages can hold. Shorten `LOOKBACK_MINUTES` or raise `SCAN_PAGE_SIZE` (100 at most), then run again


**Nothing happens after setting the request:**
- Check the workflow is active, or run it with **Run Now**
- Check `CHANGE_TYPE_FIELD_KEY` and `VALUE_REQUESTED` match the custom field exactly
- The request is only picked up while the registration changed within `LOOKBACK_MINUTES`. Save the learner record again to bring it back into the window

**Refused with "The event is full":**
- The substitute is registered before the original place is released, so the event needs one free place. Increase the capacity by one, run the workflow, then set it back

**Refused with "Several contacts share the email":**
- Merge or correct the duplicate contacts, then set the request back to `Substitution requested`

**The run stops with "is registered ... but releasing the place ... did not complete":**
- The substitute is already registered. Cancel the original registration by hand and add the note, using the details in the error message

**The run stops with "is complete, but the success log could not be written" or "the confirmation email ... was not sent":**
- The substitution itself is done. Add the log entry or contact the substitute by hand

**No email received:**
- Check `SEND_EMAIL` is `true` and `SENDING_ADDRESS_ID` is a verified sending address
