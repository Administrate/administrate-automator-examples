# Create Achievement Type from Course Template

## 🧩 Problem

When a new Course Template goes live, someone usually has to create a matching Achievement Type by hand and then award it to every instructor who is qualified to teach the course. It is repetitive admin, the names and validity periods drift from the course they belong to and instructors can be missed.

## 🤖 Automator Solution

This workflow adds a **Create Achievement Type** manual event to Course Templates. Running it creates an Achievement Type named after the course code, with the course title as its description and a configurable validity period. You then get a Slack message with a link to a simple web form that lists your instructors. Tick the instructors who should hold the Achievement, submit the form and the workflow awards it to each of them.

## ✨ Features

- One click from a Course Template creates a matching Achievement Type
- Configurable validity period in years
- Slack notification with a direct link to the instructor selection form
- Creation errors are sent to Slack so nothing fails silently
- Instructor list pages through all instructors (100 per page, up to 5,000)
- Select all and clear all shortcuts on the form
- Confirmation page showing which awards succeeded and which failed
- GraphQL variables are used throughout, so quotes and special characters in course titles are safe

## 🛠️ Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read Course Templates and Contacts, create Achievement Types and award Achievements
- A Slack OAuth2 credential that can send direct messages
- Instructors set up as Contacts in Administrate

### Installation

1. Download the [workflow.json](workflow.json) file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:

- `Course Template Webhook`
- `Get Course Template`
- `Create Achievement Type`
- `Get Instructors`
- `Award Achievement`

Select your Slack credential on:

- `Slack: Achievement Type Created`
- `Slack: Creation Error`

#### 2. Fill in the Config node

| Field | Value |
|-------|-------|
| `adminUrl` | Replace `YOUR-INSTANCE` in `https://YOUR-INSTANCE.administrateapp.com/graphql` with your Administrate instance. |
| `formDisplayUrl` | Replace `https://YOUR-N8N-HOST` with your Automator host. It must match the production URL of the **Form Display** webhook node. |
| `formSubmitUrl` | Replace `https://YOUR-N8N-HOST` with your Automator host. It must match the production URL of the **Form Submission** webhook node. |
| `validityPeriodYears` | How many years the Achievement stays valid. Defaults to `3`. |
| `SLACK_USER_ID` | Replace `REPLACE_WITH_SLACK_USER_ID` with the Slack member ID to notify (for example `U01234ABCDE`). In Slack, open the person's profile, click ⋮ and choose "Copy member ID". |

The easiest way to get the two form URLs is to open each webhook node after importing, switch to **Production URL** and copy it into Config.

### Testing

1. Activate the workflow. The Administrate Trigger registers the *Create Achievement Type* manual event on Course Templates.
2. Open a test Course Template in Administrate and run **Create Achievement Type** from its actions.
3. Check that a new Achievement Type exists with the course code as its name and the course title as its description.
4. Check that Slack sent the message and open the link. The form should list your instructors.
5. Tick one or two test instructors and submit. The confirmation page should show a tick next to each name.
6. Open each instructor's Contact record and confirm the Achievement is listed.
7. Run the manual event again on the same Course Template. If Administrate rejects the duplicate, the error should arrive in Slack.
8. Submit the form without ticking anyone. You should see the "No instructors selected" page.

## ⚙️ How It Works

The workflow has three entry points. Each one runs through **Config** and then **Route by Entry Point**, which sends it down the right path.

**Creating the Achievement Type**

1. **Course Template Webhook**: the *Create Achievement Type* manual event fires on a Course Template.
2. **Get Course Template**: fetches the Course Template's code and title.
3. **Set Course Details**: keeps the code and title for later steps.
4. **Create Achievement Type**: runs the `achievementType.create` mutation with the configured validity period.
5. **If Achievement Created**: if there are no errors, **Slack: Achievement Type Created** sends the form link. Otherwise **Slack: Creation Error** sends the error messages.

**Showing the form**

1. **Form Display**: receives the browser request from the Slack link, with the Achievement Type ID, course code and title in the query string.
2. **Get Instructors**: fetches every Contact marked as an instructor, 100 per page.
3. **Build Instructor Form**: builds an HTML form with a checkbox per instructor.
4. **Respond: Show Form**: returns the form to the browser.

**Awarding the Achievement**

1. **Form Submission**: receives the submitted form.
2. **Parse Selections**: turns the ticked instructors into one item each, valid from today.
3. **If No Selection**: returns a friendly page if nobody was ticked.
4. **Award Achievement**: runs the `contact.awardAchievement` mutation for each instructor.
5. **Format Status / Build Confirmation**: builds a results page listing each award.
6. **Respond: Confirmation**: returns the results page to the browser.

## 🔒 Security

The **Form Display** and **Form Submission** webhooks are unauthenticated. Anyone who has the URLs can open the form, see your instructor names and award Achievements. Keep the Slack message private and consider adding authentication to both webhook nodes (for example Header Auth or Basic Auth) or restricting access to your Automator host.

## 📝 Notes

This workflow creates the Achievement Type but does not link it to the Course Template. If you want learners to earn the Achievement by completing the course, attach the Achievement Type to the Course Template separately in Administrate.

## 🩺 Troubleshooting

**The manual event does not appear on Course Templates:**
- Check the workflow is active. The event is registered when the workflow is activated.
- Check the Administrate credential on `Course Template Webhook` belongs to the right instance.

**No Slack message arrives:**
- Check `SLACK_USER_ID` is a member ID, not a display name.
- Check the Slack credential on both Slack nodes and that the Slack app can message that user.

**The Slack link shows a "webhook not registered" page:**
- Check `formDisplayUrl` matches the **Production URL** of the **Form Display** node and the workflow is active.

**Submitting the form fails:**
- Check `formSubmitUrl` matches the **Production URL** of the **Form Submission** node.

**The form says "No instructors found":**
- Check your instructors are flagged as instructors on their Contact records.
- Check the Administrate credential can read Contacts.

**An award shows a cross on the confirmation page:**
- The message next to the name comes from Administrate. A common cause is the instructor already holding an active Achievement of that type.
