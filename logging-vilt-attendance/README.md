# Logging vILT Attendance

**Version:** 1.0.0
**Last Updated:** 2026-02-05

## Problem

When running vILT Events, Users find it cumbersome and time consuming logging into Zoom/Teams etc to find out who attended. They then need to go back into Administrate and log the attendance.

## Automator Solution

Once this workflow is configured and Active, the Automator dropdown on events should have the option "Record Zoom Attendance". Using this option will take the primary email address of the event's learners and compare that with the list of participants from the Zoom meeting. If there is a match, their attendance will be recorded in Administrate.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Zoom API credentials (Account ID, Client ID, Client Secret)

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

1. Configure your Administrate OAuth2 credentials in all HTTP Request nodes
2. Configure your Zoom OAuth2 credentials for accessing meeting participant data
3. Update the webhook URL in the workflow
4. Activate the workflow
5. The "Record Zoom Attendance" option will appear in the Automator dropdown on events

### Usage

1. Navigate to an event in Administrate
2. Select "Record Zoom Attendance" from the Automator dropdown
3. The workflow will fetch Zoom meeting participants and match them with event learners by email
4. Attendance will be automatically recorded in Administrate for matching participants

## Video Walkthrough

Coming soon
