# Instructor Jump Ball

**Version:** 1.0.0
**Last Updated:** 2026-02-02

## Problem

Training organizations often work with both internal and external instructors. While internal instructors' schedules are usually visible, external or contract instructors' availability is typically unknown. Planning training sessions therefore requires manually reaching out via phone, email, or spreadsheets to see who is available.

The decision process for assigning an instructor can also vary: some organizations assign the first instructor who responds (a "jump ball" approach), while others use more complex criteria such as instructor ratings, past performance, pay rates, or preferences. This manual process is time-consuming, error-prone, and difficult to track.

## Solution

The Instructor Jump Ball workflow streamlines this process. Event administrators can send RSVP requests directly to preferred instructors from within the event, view responses in real time, and select the instructor—all in one centralized place. This eliminates back-and-forth communication, reduces scheduling errors, and ensures fair and transparent assignment decisions.

## Features

- Email invitation system for instructors
- Beautiful Bootstrap-styled RSVP interface
- Accept/Decline response tracking
- Automatic internal notes logging
- Automatic instructor assignment upon acceptance
- Real-time response tracking
- Professional "Thank You" confirmation pages

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Email sending capability (via n8n's email node or SMTP)
- Instructor contacts in your Administrate instance

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Configure your Administrate OAuth2 credentials in all HTTP Request nodes:
- `HTTP Request` (Fetch event details)
- `HTTP Request1` (Fetch instructor details)
- `HTTP Request2` (Update event notes - accept)
- `HTTP Request3` (Fetch event for response)
- `HTTP Request4` (Add instructor to event)
- `HTTP Request6` (Update event notes - decline)

#### 2. Update Webhook URLs

There are two webhooks in this workflow:

**Webhook 1: Invitation Trigger**
- Node name: "When this URL is hit"
- Path: Update to `<your-instance>-instructor-invite`
- Example: `mycompany-instructor-invite`

**Webhook 2: Response Handler**
- Node name: "Webhook1"
- Path: Update to `<your-instance>-instructor-response`
- Example: `mycompany-instructor-response`

#### 3. Update RSVP Button URLs

In the "Respond to Webhook" node, update the URLs in both Accept and Decline buttons:

```html
<!-- Find these lines and update the base URL -->
<a href="https://n8n.administratehq.com/webhook/instructor-response?type=accept&eventId=...">
<!-- Change to -->
<a href="https://YOUR-N8N-INSTANCE.com/webhook/YOUR-INSTANCE-instructor-response?type=accept&eventId=...">
```

Replace:
- `n8n.administratehq.com` with your Automator instance URL
- `instructor-response` with your custom path (e.g., `mycompany-instructor-response`)

### Testing

1. Activate the workflow
2. Trigger the invitation webhook with test parameters:
   - `eventId`: ID of a test event
   - `instructor`: ID of a test instructor contact
3. Verify the invitation email/page displays correctly
4. Click Accept or Decline and verify:
   - Response is recorded in event internal notes
   - Instructor is added to event (if accepted)
   - Thank you page displays

## Usage

### Sending an Invitation

Trigger the invitation webhook with these parameters:
```
GET /webhook/your-instance-instructor-invite?eventId=EVENT_ID&instructor=INSTRUCTOR_ID
```

Parameters:
- `eventId`: The Administrate event ID
- `instructor`: The contact ID of the instructor

### How It Works

1. **Invitation Request**: Admin sends the invitation email to instructors
2. **Event Details Fetch**: Workflow retrieves event information via GraphQL
3. **Display RSVP Page**: Instructor sees event details with Accept/Decline buttons
4. **Response Handling**:
   - **Accept**: Instructor added to event, internal note logged
   - **Decline**: Internal note logged with decline timestamp
5. **Confirmation**: Instructor sees "Thank You" page

### Response Flow

**When Instructor Accepts:**
1. Fetch current event details and instructor name
2. Add timestamped acceptance note to event internal notes
3. Add instructor to event as Instructor role
4. Display thank you confirmation

**When Instructor Declines:**
1. Fetch current event details and instructor name
2. Add timestamped decline note to event internal notes
3. Display thank you confirmation

Example internal note:
```html
<strong>John Smith</strong> accepted on 02/02/2026 3:45 PM
```

## Troubleshooting

**RSVP page not loading:**
- Verify webhook is activated
- Check eventId and instructor parameters are valid
- Review HTTP Request node OAuth credentials

**Buttons not working:**
- Ensure button URLs match your Webhook1 path
- Check that workflow is activated
- Verify URL encoding is correct

**Instructor not being added:**
- Confirm OAuth credentials have staff management permissions
- Check that instructor contact ID is valid
- Review HTTP Request4 node execution logs

**Dates displaying incorrectly:**
- Check timezone settings in Code node
- Verify event start/end times are valid ISO dates

## API Permissions Required

Your Administrate OAuth2 credential needs:
- **Events**: Read and update permissions
- **Contacts**: Read permissions
- **Event Staff**: Create permissions