# Updating Billing & Shipping Addresses

**Version:** 1.0.0
**Last Updated:** 2026-02-05

## Problem

Some Organizations only receive a Default address from Students. This is the same as their Billing & Shipping and if it is missing it can cause problems with taking payments or sending materials.

## Automator Solution

The Automator looks at any Account that is Created or Updated. If it is missing all the primary fields, it will copy the Default Address to the Billing and Shipping fields.

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
2. Set up the webhook trigger to listen for Account Created or Updated events
3. Activate the workflow
4. The workflow will automatically run whenever an Account is created or updated

## Video Walkthrough

[View the video demonstration](https://fast.wistia.com/embed/medias/yseueocgr9)
