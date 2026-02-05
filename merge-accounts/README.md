# Merge Accounts

**Version:** 1.0.0
**Last Updated:** 2026-02-05

## Problem

Some Accounts are duplicated either during a sync from an external HRIS or CRM system, or from Bookings or by human error. This creates a messy CRM and potential for miscommunication.

## Automator Solution

When the Webhook URL is clicked, the User is prompted to enter the Account IDs they wish to merge. One click and the Accounts are merged.

Future enhancements here would be to authenticate the form to the User's credentials and merge more data (such as Training Passes).

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Appropriate permissions to merge and update accounts

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

1. Configure your Administrate OAuth2 credentials in all HTTP Request nodes
2. Update the webhook URL in the workflow
3. Activate the workflow
4. Share the webhook URL with users who need to merge accounts

### Usage

1. Click the webhook URL to open the merge form
2. Enter the Account IDs you wish to merge
3. Submit the form to merge the accounts

## Video Walkthrough

Coming soon
