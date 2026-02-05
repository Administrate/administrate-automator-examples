# Managing Low Attendance

**Version:** 1.0.0
**Last Updated:** 2026-02-05

## Problem

If an Event does not reach its fill rate X weeks before the start date, some Organizations may have to cancel the Event or run it at a commercial loss.

## Automator Solution

X weeks (defined by the User) before the Event, if the fill rate is below the target Fill Rate, it will email current participants with a referral link and code in order to drive registrations through friends and family.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

1. Configure your Administrate OAuth2 credentials in all HTTP Request nodes
2. Set the number of weeks before the event start date to trigger the workflow
3. Define your target fill rate percentage
4. Configure your email template with referral link and code
5. Set up the schedule trigger to run regularly (e.g., daily)
6. Activate the workflow

## Video Walkthrough

Coming soon
