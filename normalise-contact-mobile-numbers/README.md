# Normalise Contact Mobile Numbers

**Version:** 1.0.0
**Last Updated:** 2026-09-29

> **UK-only by default.** Out of the box, this workflow only recognises and rewrites **UK mobile numbers** (country code `44`, trunk prefix `0`, mobiles starting `7`). Numbers in any other format are left unchanged. To use it for another country, change the values in the **Config** node (see [Configuration](#configuration)). It handles one country per workflow.

## Problem

Mobile numbers get into Administrate in many formats: `07xxx xxxxxx`, `+44 (0)7xxx xxxxxx`, `0044 7xxxxxxxxx`, `447xxxxxxxxx`. Mixed formats break SMS integrations, cause duplicate-matching problems and make reports messy. Fixing them by hand does not scale, and a nightly clean-up leaves bad data in place for hours.

## Automator Solution

Whenever a Contact is created or updated, Automator reads that contact's **mobile** number, converts it to international (E.164) format such as `+447xxxxxxxxx`, and writes it back only if it changed. Each change is logged to an n8n Data Table.

The workflow is loop-safe. Its own update fires another "Contact Updated" webhook, but on the second pass the number is already correct, so nothing is written and the run stops.

## Features

- Runs in real time on Contact Created and Contact Updated webhooks
- Only touches the phone number with usage `mobile`. Office, home and fax numbers are left alone.
- Strips spaces, brackets, dashes and dots
- Handles `0…`, `00<cc>…`, `<cc>…` and `+<cc> (0)…` forms
- Writes back only when the number actually changes (no needless updates, no loops)
- Country rules held in one **Config** node
- Audit log of every change (old number, new number, status, errors) in an n8n Data Table

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials
- The Administrate trigger node (`CUSTOM.administrateTrigger`) available in your Automator instance
- n8n Data Tables enabled

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Select your **Administrate OAuth2** credential in:
- `Contact Created` and `Contact Updated` (trigger nodes, which register the webhooks)
- `Get Contact`
- `Update Mobile (Webhook)`

#### 2. Create the log Data Table

Create an n8n Data Table (for example `Phone Normalization Log`) with these columns, all of type **string**:

| Column | Contents |
|---|---|
| `run_at` | ISO timestamp of the run |
| `contact_id` | Administrate Contact ID |
| `contact_name` | First and last name |
| `old_number` | Number before normalisation |
| `new_number` | Number written back |
| `status` | `updated` or `error` |
| `detail` | `webhook`, or the JSON errors returned by `contact.update` |

Copy the table's ID.

#### 3. Fill in the Config node

| Field | Default | Meaning |
|---|---|---|
| `PHONE_LOG_TABLE_ID` | `REPLACE_WITH_PHONE_NORMALIZATION_LOG_TABLE_ID` | ID of the Data Table from step 2 |
| `COUNTRY_CODE` | `44` | Country calling code, without `+` |
| `INTERNATIONAL_DIAL_PREFIX` | `00` | International dialling prefix used in that country (`00`, `011`, ...). Leave blank to ignore this form. |
| `TRUNK_PREFIX` | `0` | National trunk prefix. Leave blank if the country has none. |
| `MOBILE_NATIONAL_PREFIX` | `7` | First digit(s) of a mobile number after the trunk prefix. Only national numbers starting `TRUNK_PREFIX + MOBILE_NATIONAL_PREFIX` are converted, so landlines stored as mobile are left alone. |
| `ACCEPT_BARE_COUNTRY_CODE` | `true` | Treat a number starting with the country code but no `+` (e.g. `447…`) as international. Set to `false` if national numbers in your country can start with the same digits as the country code. |

The defaults reproduce UK behaviour. Test carefully before you change them for another country. The rules are simple prefix rules, not a full numbering-plan library.

### Testing

1. Activate the workflow (this registers the two Administrate webhooks)
2. In Administrate, create or edit a Contact and set the mobile number to a UK mobile in national format, such as `07xxx xxxxxx`
3. Within a few seconds the mobile number should read `+447xxxxxxxxx` (the leading 0 replaced by +44, spaces removed)
4. Check the Data Table for a row with `status = updated`
5. Save the contact again without changes. No new log row should appear.

## How It Works

1. **Trigger**: `Contact Created` or `Contact Updated` webhook fires with the contact's ID
2. **Config**: Adds the country rules and log table ID (input fields are passed through)
3. **Get Contact**: Reads the contact's name and phone numbers via GraphQL `node(id)`
4. **Normalize Contact**: Finds the `mobile` phone number, applies the Config rules and outputs an item only if the result differs from the stored value
5. **Update Mobile (Webhook)**: Calls `contact.update` with the new mobile number. The API merges phone numbers by usage, so other numbers are not changed.
6. **Log (Webhook)**: Writes a row to the Data Table. `status` is `error` and `detail` holds the error JSON if the mutation returned errors.

## Troubleshooting

**Numbers are not changing:**
- Confirm the workflow is active and the two webhooks exist in Administrate
- The number must be stored with usage **mobile**
- The number must match the configured country. By default, non-UK numbers and UK landlines (e.g. `020…`) are deliberately left unchanged.

**Log node fails:**
- Check `PHONE_LOG_TABLE_ID` in Config and that the table has all seven columns

**Log shows `status = error`:**
- Read the `detail` column for the Administrate validation message
- Check that the OAuth2 credential can update Contacts

**Workflow runs twice per edit:**
- This is expected. The second run is caused by the workflow's own update, finds nothing to change and stops.
