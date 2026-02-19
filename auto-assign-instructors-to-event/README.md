# Auto Assign Instructors to Event

**Version:** 1.0.0
**Last Updated:** 2026-02-19

## Problem

When creating an Event for a Course Template, any Approved Instructors defined on the Course Template must be manually added to the Event's Personnel table after creation. This is repetitive, easy to forget, and doesn't account for instructor availability — leading to scheduling conflicts and wasted effort.

## Solution

The Auto Assign Instructors to Event workflow eliminates this manual step. When an Event is created, the workflow automatically retrieves the Course Template's Approved Instructors, checks each one for availability and workplace location match, and adds the first valid candidate to the Event's Personnel table - all without any manual intervention.

## Features

- Automatic retrieval of Approved Instructors from the Course Template
- Availability checking - only instructors with no booking conflicts are considered
- Workplace location matching - instructors are filtered by whether their workplace location matches the Event location
- Configurable instructor count - set how many instructors to auto-assign (default: 1)
- External integration logging for full audit trail of all assignment decisions

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Approved Instructors assigned to your Course Templates

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Configure your Administrate OAuth2 credentials in all HTTP Request nodes:
- `Get Approved Instructors` (Fetch approved instructors for the course)
- `HTTP Request` (Fetch instructor workplace locations)
- `Add Instructor to Event` (Add selected instructor to the event)
- All `External Logs` nodes (Write audit logs to Administrate)

#### 2. Create an Event Created Webhook

Follow the [Administrate Webhook setup guide](https://developer.getadministrate.com/docs/core/Webhooks/02_setup.md) to create an **Event Created** webhook:

1. Set the **URL** to the n8n webhook URL provided by the Webhook node in the workflow
2. Set the **query** to:
   ```graphql
   query test($objectid: ID!) {
     node(id: $objectid) {
       id
       ... on Event {
         id
         courseTemplate { id }
         location { id }
       }
     }
   }
   ```

#### 3. Set Number of Instructors to Add

In the **"Number of Instructors to Add"** node, change the **Max Items** value to control how many instructors are automatically assigned to each Event. The default is 1.

### Testing

1. Activate the workflow
2. Create a new Event for a Course Template that has Approved Instructors
3. Verify that the instructor(s) appear in the Event's Personnel table
4. Check the External Integration Logs for audit entries

## How It Works

1. **Event Created**: An Event Created webhook fires and sends the Event ID, Course Template ID, and Location ID to the workflow
2. **Fetch Approved Instructors**: The workflow queries for all instructors who are approved for the Course Template and confirmed
3. **Check Availability**: Each instructor is checked for booking conflicts during the Event's date range — unavailable instructors are filtered out and logged
4. **Match Workplace Location**: Available instructors are checked against the Event's location — instructors whose workplace doesn't match are filtered out and logged
5. **Select Instructors**: Remaining candidates are sorted alphabetically and limited to the configured number
6. **Assign to Event**: Selected instructors are added to the Event's Personnel table via the `addStaffBulk` mutation
7. **Audit Logging**: Every decision (assigned, unavailable, location mismatch) is logged to Administrate's External Integration Logs

## Important Notes

- If an Event is created via the **Duplicate** feature, any instructors on the original Event will be carried over to the new Event regardless of availability. This workflow runs in addition to that behaviour.
- The workflow processes instructors in **alphabetical order** by name. If you need a different selection strategy (e.g. round-robin or rating-based), customise the sorting node.

## Troubleshooting

**Instructors not being assigned:**
- Verify the Event Created webhook is configured and pointing to the correct URL
- Check that the Course Template has Approved Instructors
- Confirm OAuth credentials have the required permissions
- Review the External Integration Logs for "not available" or "workplace not matched" entries

**Wrong number of instructors assigned:**
- Check the **Max Items** setting in the "Number of Instructors to Add" node

**Webhook not firing:**
- Ensure the webhook is set up for the `Event Created` event type
- Verify the webhook query includes `courseTemplate { id }` and `location { id }`

## API Permissions Required

Your Administrate OAuth2 credential needs:
- **Events**: Read permissions
- **Contacts**: Read permissions (including contact stats)
- **Event Staff**: Create permissions
- **External Integration Logs**: Create permissions

[View the video demonstration (no audio)](https://fast.wistia.com/embed/medias/mnzlrttlmu)
