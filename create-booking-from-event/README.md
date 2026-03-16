# Create Booking from Event

**Version:** 1.0.0
**Last Updated:** 2026-03-16

## Problem

Users often navigate to an Event to check details like dates, availability, or pricing — and then want to create a Booking for that specific Event. Today, this requires navigating away from the Event screen to create a Booking via the Booking page (or via the Account), and then searching for the Event to add it as an Interest. This back-and-forth is time-consuming and disruptive, especially when you've already found the Event you need.

## Automator Solution

A Booking can be created directly from an Event with a single click. The Event is automatically added as an Interest on the newly created Booking — no need to navigate away or search for the Event again.

Under the 'Automator' menu on an Event, you'll see a "Create Booking with Event as Interest" option.

## Features

- One-click Booking creation directly from an Event
- Automatic Event Interest attachment to the newly created Booking
- Supports both Public and Private Event types
- Automatic currency and financial unit detection
- External integration logging for full audit trail of all Booking creation attempts

## Event Criteria

**Public Events** must meet the following criteria:
- Active
- Start Date in the future
- Default Price Level set

_Please note: Training Tokens cannot be set as a Default Price_

**Private Events** must meet the following criteria:
- Active
- Start Date in the future
- Price set

_Please note: Training Tokens cannot be set as a Price_

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Configure your Administrate OAuth2 credentials in all HTTP Request nodes:
- `Create Booking` (Create booking for public events)
- `Private Create Booking` (Create booking for private events)
- `Create Interest on Booking` (Add event as interest - public)
- `Private Create Interest on Booking` (Add event as interest - private)
- All `External Log` nodes (Write audit logs to Administrate)

#### 2. Create a Manual Event Webhook

Follow the [Administrate Webhook setup guide](https://developer.getadministrate.com/docs/core/Webhooks/02_setup.md) to create a **Manual Event** webhook:

1. Set the **webhookTypeId** to: `V2ViaG9va1R5cGU6bWFudWFsX2V2ZW50`
2. Set the **name** to: `Create Booking with Event as Interest`
3. Set the **url** to the n8n webhook URL provided by the Webhook node in the workflow
4. Set the **query** to:
   ```graphql
   query test($objectid: ID!) {
     node(id: $objectid) {
       id
       ... on Event {
         code
         type
         bookingAccount { id }
         price
         region { id }
         defaultTaxType { id }
         financialUnit { id }
         defaultPrice {
           amount
           financialUnit { id }
         }
       }
     }
   }
   ```

#### 3. Update the Entry Step ID

In both the **"Create Booking"** and **"Private Create Booking"** nodes, update the `entryStepId` to your preferred Booking Step that the Booking should be created in.

### Testing

1. Activate the workflow
2. Navigate to an Event that meets the criteria above
3. Open the **Automator** menu on the Event
4. Click **"Create Booking with Event as Interest"**
5. Verify that a new Booking is created with the Event added as an Interest
6. Check the External Integration Logs for audit entries

## How It Works

1. **Trigger**: User clicks "Create Booking with Event as Interest" from the Automator menu on an Event
2. **Store Event Data**: The workflow stores the Event's region, pricing, and financial unit details
3. **Determine Event Type**: A Switch node routes the workflow based on whether the Event is Public or Private
4. **Decode Financial Unit**: The Base64-encoded financial unit ID is decoded for use in the Booking creation
5. **Create Booking**: A new Booking (Opportunity) is created via the `opportunities.create` mutation, using the Event's region and financial unit
6. **Add Event as Interest**: The Event is added as an Interest on the newly created Booking via the `opportunities.addInterest` mutation, including quantity, price, and tax details
7. **Audit Logging**: The result (success or failure) is logged to Administrate's External Integration Logs

## Troubleshooting

**"Create Booking with Event as Interest" not appearing in the Automator menu:**
- Verify the Manual Event webhook is configured and pointing to the correct URL
- Ensure the webhook name is set correctly

**Booking not being created:**
- Confirm the Event meets the required criteria (active, future start date, pricing set)
- Check that OAuth credentials have the required permissions
- Review the External Integration Logs for error details

**Interest not being added to the Booking:**
- Verify the Event has a valid Default Price (public) or Price (private)
- Ensure Training Tokens are not set as the price
- Check the External Integration Logs for failure entries

**Wrong Booking Step:**
- Update the `entryStepId` in both the "Create Booking" and "Private Create Booking" nodes

## Video Walkthrough

[View the video demonstration](https://fast.wistia.com/embed/medias/q1g1ufow02)
