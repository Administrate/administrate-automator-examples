# Instructor and Resource Utilization Dashboards

## Problem

Training managers need to know how busy their instructors and resources (rooms, equipment, labs) are. Who is teaching the most hours? Which rooms sit empty? How does this year compare with last year? Administrate holds the data, but there is no single utilization view. Building one by hand means exporting sessions and bookings and working them through in a spreadsheet.

## Automator Solution

Three workflows serve two browser dashboards straight from Automator. Each is a self-contained HTML page with Chart.js charts, protected by HTTP Basic Auth:

- **Instructor Utilization**: teaching hours and teaching days per instructor, built from event sessions. A daily job caches the session data in an n8n Data Table so the dashboard loads quickly. Instructor names are looked up live when the page is opened and are never stored in the cache.
- **Resource Utilization**: days and hours booked per resource, by resource type and location, built from resource bookings each time the page is opened.

Both dashboards open on the last 13 months and offer a Prior year preset.

## Features

- Bookmarkable dashboard URLs protected by Basic Auth
- Instructor teaching hours and distinct teaching days, excluding cancelled events and sessions
- Resource days booked (distinct calendar days) and hours booked (total booking length)
- Filters by resource type and location, and date presets (Last 13 months, Prior year, Full range)
- Optional booking type exclusions and resource type inclusions
- Cursor pagination with a configurable page size, page cap and request interval
- Instructor cache stores contact IDs only. Names are resolved at view time.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read events, sessions, staff and resource bookings
- n8n Data Tables enabled on your Automator instance

### Installation

Import the workflows in this order. In your Automator instance, click the menu (⋮), select "Import from File" and upload each file from the `workflows/` directory:

1. `10-instructor-cache-builder.json` (Instructor Utilization - Builder (cache))
2. `20-instructor-utilization-dashboard.json` (Instructor Utilization - Dashboard)
3. `30-resource-utilization-dashboard.json` (Resource Utilization - Days and Hours Booked)

The resource dashboard (30) is independent and can be installed on its own.

### Configuration

#### 1. Create the cache Data Table

Create an n8n Data Table (for example `Instructor Utilization Cache`) with these columns:

| Column | Type |
| --- | --- |
| `cacheKey` | string |
| `payload` | string |

The builder upserts a single row with `cacheKey` = `instructor`. Select this table in:
- `Write Cache` in the Builder workflow
- `Read Cache` in the Instructor Dashboard workflow

Both nodes ship with the placeholder `REPLACE_WITH_INSTRUCTOR_CACHE_DATA_TABLE_ID`.

#### 2. Set Credentials

- **Administrate OAuth2**: `Fetch Events & Sessions` (Builder), `Fetch Instructor Names` (Instructor Dashboard), `Fetch Resource Bookings` (Resource Dashboard)
- **Basic Auth** (HTTP Basic Auth credential): `Dashboard Link` in both dashboard workflows. Create the credential with a username and a strong password of your choice. This is the login people use to open the dashboards. No password is stored in the workflow files. You can use one credential for both dashboards or a separate one for each.

#### 3. Review the Config nodes (optional)

The defaults work for most instances.

**Builder `Config`**

| Setting | Default | Meaning |
| --- | --- | --- |
| `apiUrl` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |
| `lookbackMonths` | `13` | Months of history, plus the whole previous calendar year |
| `pageSize` | `200` | Events per API page |
| `maxPages` | `100` | Page cap |
| `requestIntervalMs` | `0` | Pause between pages, in ms |

**Instructor Dashboard `Config`**

| Setting | Default | Meaning |
| --- | --- | --- |
| `apiUrl` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |
| `lookbackMonths` | `13` | Default view window in months |
| `namesFirst` | `1000` | How many current instructors' names to look up |

**Resource Dashboard `Config`**

| Setting | Default | Meaning |
| --- | --- | --- |
| `apiUrl` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |
| `lookbackMonths` | `13` | Default view window in months (the previous calendar year is always loaded as well) |
| `pageSize` | `200` | Bookings per API page |
| `maxPages` | `200` | Page cap |
| `excludeBookingTypes` | empty | Comma-separated booking types to ignore, for example `plan` |
| `includeResourceTypes` | empty | Comma-separated resource type names to include. Empty includes all. |
| `requestIntervalMs` | `0` | Pause between pages, in ms |

### Testing

1. In the Builder, click **Run Instructor Report** and check that `Write Cache` wrote one row
2. Activate the Builder so **Daily Refresh** runs every day at 05:00
3. Activate both dashboard workflows
4. Open each `Dashboard Link` production URL in a browser and sign in with your Basic Auth username and password
5. Check the figures for one instructor and one resource against Administrate

## How It Works

**Builder (cache)**
1. **Trigger**: Daily at 05:00, or manually
2. **Build Window**: Works out the load range (last `lookbackMonths` months plus the whole previous calendar year)
3. **Fetch Events & Sessions**: Reads events in the range, page by page, with sessions and staff
4. **Prepare Instructor Records**: Keeps one record per instructor per non-cancelled session (contact ID, course title, dates, hours)
5. **Save Cache / Write Cache**: Compresses the records into one JSON payload and upserts it into the Data Table

**Instructor Dashboard**
1. **Trigger**: A browser opens the Basic Auth webhook
2. **Fetch Instructor Names / Resolve Names**: Looks up current instructors' names live
3. **Read Cache / Load Cache**: Loads the cached records
4. **Build Dashboard / Serve Dashboard**: Returns the HTML dashboard

**Resource Dashboard**
1. **Trigger**: A browser opens the Basic Auth webhook, or run manually
2. **Build Window**: Works out the load range
3. **Fetch Resource Bookings**: Reads resource bookings that start in the range, page by page
4. **Prepare Bookings**: Applies the filters and keeps one compact record per booking
5. **Build Dashboard / Serve Dashboard**: Returns the HTML dashboard

## Troubleshooting

**Browser keeps asking for a password:**
- Check the Basic Auth credential on `Dashboard Link` and the username and password you are typing

**Instructor dashboard is empty:**
- Run the Builder at least once, and check both Data Table nodes point at the same table

**Instructors show as IDs instead of names:**
- The instructor has left or is not a current instructor, or there are more than `namesFirst` instructors. Raise `namesFirst`.

**Resource dashboard is slow or times out:**
- Every booking in the range is embedded in the page on each request. Narrow `includeResourceTypes`, add `excludeBookingTypes`, or lower `lookbackMonths`.

**Totals look too low:**
- `maxPages` x `pageSize` may have been reached. Raise `maxPages`.

**"Administrate GraphQL error":**
- Check the OAuth2 credential and its permissions
