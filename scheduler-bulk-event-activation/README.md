# Scheduler - Bulk Event Activation

**Version:** 1.0.0
**Last Updated:** 2026-02-02

## Problem

When using Scheduler to schedule hundreds or thousands of events, some users want to be able to make all these draft events Active without having to go through each one manually. This is time-consuming and inefficient when dealing with large batches of scheduled events.

## Solution

This workflow renders a form which the user can use to submit the ID of the Schedule. Once entered, it will go through and make all Events active automatically. The workflow can be extended with additional logic if you need extra information populated in your Events before activation.

## Features

- Web form interface for easy schedule ID input
- Batch processing of draft events
- Automatic activation of all events in a schedule
- Error handling for failed activations
- Extensible design for custom business logic

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- A schedule with draft events in your Administrate instance

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

1. **Set OAuth Credentials**:
   - Open each HTTP Request node
   - Select your Administrate OAuth2 credential from the dropdown
   - Ensure the credential has the necessary permissions to update events

2. **Update Webhook URLs** (if deploying to production):
   - Open the Webhook node
   - Update the path to match your instance naming convention
   - Example: `myinstance-scheduler-activation`

3. **Test the Workflow**:
   - Activate the workflow
   - Access the webhook URL
   - Enter a test schedule ID
   - Verify events are activated correctly

## Usage

1. Activate the workflow in Automator
2. Navigate to the webhook URL (provided after activation)
3. Enter the Schedule ID you want to activate
4. Click Submit
5. All draft events in that schedule will be activated

## Workflow Overview

1. **Webhook Trigger**: Receives the schedule ID via web form
2. **GraphQL Query**: Fetches all draft events for the schedule
3. **Iteration Loop**: Processes each event
4. **Event Update**: Changes event status to Active
5. **Response**: Returns success confirmation

## Video Walkthrough

[View the video demonstration](https://fast.wistia.com/embed/medias/56p9rknadn)

## Troubleshooting

**Events not activating:**
- Verify OAuth credentials have update permissions
- Check that events are in draft status
- Review node execution logs for specific errors

**Webhook not accessible:**
- Ensure workflow is activated
- Check webhook path configuration
- Verify network/firewall settings