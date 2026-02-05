# Bulk Resource Removal

**Version:** 1.0.0
**Last Updated:** 2026-02-05

## Problem

Sometimes Users need to remove all Resources from an Event. This requires the User to go into each Session and remove each Resource manually.

## Automator Solution

With 2 clicks, the User can choose to Remove All Resources and it will remove all Resources in bulk at once.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Appropriate permissions to modify event resources

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

1. Configure your Administrate OAuth2 credentials in all HTTP Request nodes
2. Activate the workflow
3. The "Remove All Resources" option will appear in the Automator dropdown on events

## Video Walkthrough

[View the video demonstration](https://fast.wistia.com/embed/medias/80e7ygci36)
