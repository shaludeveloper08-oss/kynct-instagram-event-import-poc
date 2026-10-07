# KYNCT Event Import Workflows

## Simple Guide for Client

This document explains the two event-import workflows used for KYNCT.

The workflows automatically collect events from:

1. Uitagenda Rotterdam
2. Ticketmaster

They compare the collected events with existing Google Sheets and process only new or changed events.

---

# 1. What These Workflows Do

## Workflow 1 – Uitagenda Rotterdam

This workflow:

1. Gets events from Uitagenda Rotterdam.
2. Collects event details such as name, date, time, venue, description and image.
3. Opens the actual event page to look for a ticket/booking link.
4. Checks the event against the existing Uitagenda Google Sheet.
5. Saves only new or changed events.
6. Creates a Google Doc containing the events that need to be reviewed/imported.

## Workflow 2 – Ticketmaster

This workflow:

1. Gets events from Ticketmaster for Rotterdam.
2. Collects event details such as name, date, time, venue, description, ticket link and image.
3. Checks the events against the existing Ticketmaster Google Sheet.
4. Saves only new or changed events.
5. Creates a Google Doc containing the events that need to be reviewed/imported.

---

# 2. What the Client Needs to Provide

| Item | What is needed | Used for |
|---|---|---|
| Ticketmaster | **Consumer Key** | Getting events from Ticketmaster |
| Google Account | Access to the required Google Sheets and Google Docs | Reading/writing Sheets and creating Docs |
| Uitagenda | **Nothing** | Public event data does not require an API key |

> **Important:** The Ticketmaster **Consumer Secret is NOT required** for this workflow.

---

# 3. Ticketmaster – Which Key to Use?

Open the Ticketmaster Developer Portal and go to your application's **Credentials** section.

You will see:

- **Consumer Key** → USE THIS
- **Consumer Secret** → DO NOT USE THIS

The **Consumer Key** is the Ticketmaster API key required by this workflow.

### n8n configuration

The credential is configured as:

```text
Name: apikey
Value: <Ticketmaster Consumer Key>
```

Then this credential is selected in the Ticketmaster event-fetching step.

### Important

The **Consumer Secret is not required** for this workflow.

Do not put either credential into the README or any public document. The key should be entered securely in n8n.

---

# 4. Google Account – What Access Is Needed?

The Google account connected to n8n needs access to:

### Google Sheets

- Uitagenda event spreadsheet
- Ticketmaster event spreadsheet

The workflows use these Sheets to check existing events, add new events and update changed events.

### Google Docs

The same Google account needs permission to:

- Create Google Docs
- Write event information into those documents

No separate Google Docs API key is required.

---

# 5. Uitagenda Rotterdam Workflow

### Simple flow

```text
Uitagenda
    ↓
Get events
    ↓
Read event details
    ↓
Open event page
    ↓
Find ticket link
    ↓
Check Google Sheet
    ↓
Find NEW / UPDATED events
    ↓
Save to Google Sheet
    ↓
Create Google Doc
```

### Ticket links

For each Uitagenda event, the workflow opens the actual event page and checks the page for a ticket or booking link.

If a ticket link is found, it is saved with the event.

If no ticket link is available, the event can still be processed.

### API key

Uitagenda does **not** require an API key in the current workflow.

---

# 6. Ticketmaster Workflow

### Simple flow

```text
Ticketmaster
    ↓
Get Rotterdam events
    ↓
Read event details
    ↓
Check Google Sheet
    ↓
Find NEW / UPDATED events
    ↓
Save to Google Sheet
    ↓
Create Google Doc
```

The current configuration is:

```text
City: Rotterdam
Country: NL
```

Ticketmaster provides information such as:

- Event name
- Event ID
- Date
- Time
- End date
- Venue
- Location
- Description
- Ticket URL
- Image
- Timezone


---


---

# 5. Uitagenda Workflow – Node-by-Node Explanation

The Uitagenda workflow contains the following steps:

```text
Schedule Trigger
      ↓
Prepare Uitagenda Config
      ↓
Fetch Uitagenda Events
      ↓
Normalize Uitagenda Events
      ↓
Fetch Uitagenda Event Page
      ↓
Extract Ticket Link
      ↓
Get Existing Uitagenda Events
      ↓
Compare Uitagenda Events
      ↓
Upsert Uitagenda Events
      ↓
Build Uitagenda Import Document
      ↓
Create Uitagenda Google Doc
      ↓
Write Uitagenda Events to Google Doc
```

### 1. Schedule Trigger
Starts the workflow automatically according to the configured schedule.

### 2. Prepare Uitagenda Config
Prepares the settings needed to request events from Uitagenda.

### 3. Fetch Uitagenda Events
Connects to the Uitagenda event source and retrieves the available event information.

### 4. Normalize Uitagenda Events
Converts the received information into a consistent format so that all events have the same structure.

### 5. Fetch Uitagenda Event Page
Opens the actual webpage of each event.

### 6. Extract Ticket Link
Looks at the actual event page and searches for a ticket or booking link.

### 7. Get Existing Uitagenda Events
Reads the events that are already stored in the Uitagenda Google Sheet.

### 8. Compare Uitagenda Events
Checks whether each event is new, updated, or unchanged.

The comparison uses the event's `source_event_id`.

### 9. Upsert Uitagenda Events
Adds new events to the Google Sheet or updates events whose information has changed.

### 10. Build Uitagenda Import Document
Prepares the new and updated events for the review document.

### 11. Create Uitagenda Google Doc
Creates the Google Doc for the event import/review.

### 12. Write Uitagenda Events to Google Doc
Writes the processed event information into the Google Doc.

---

# 6. Ticketmaster Workflow – Node-by-Node Explanation

The Ticketmaster workflow contains the following steps:

```text
Ticketmaster Schedule Trigger
      ↓
Prepare Ticketmaster Config
      ↓
Fetch Ticketmaster Events
      ↓
Normalize Ticketmaster Events
      ↓
Get Existing Ticketmaster Events
      ↓
Compare Ticketmaster Events
      ↓
Upsert Ticketmaster Events
      ↓
Build Ticketmaster Import Document
      ↓
Create Ticketmaster Google Doc
      ↓
Write Ticketmaster Events to Google Doc
```

### 1. Ticketmaster Schedule Trigger
Starts the Ticketmaster workflow automatically according to the configured schedule.

### 2. Prepare Ticketmaster Config
Prepares the settings used to request Ticketmaster events.

The current location is:

```text
City: Rotterdam
Country: NL
```

### 3. Fetch Ticketmaster Events
Connects to the Ticketmaster Discovery API and retrieves event information.

### 4. Normalize Ticketmaster Events
Converts the Ticketmaster response into the common event format used by the workflow.


### 5. Get Existing Ticketmaster Events
Reads the existing Ticketmaster events from the Google Sheet.

### 6. Compare Ticketmaster Events
Checks each Ticketmaster event against the existing events.

The comparison uses:

```text
source_event_id
```

The event is classified as:

- New
- Updated
- Unchanged

### 7. Upsert Ticketmaster Events
Adds new events to the Google Sheet and updates events whose information has changed.

### 8. Build Ticketmaster Import Document
Prepares the new and updated events for the review document.

### 9. Create Ticketmaster Google Doc
Creates the Google Doc for the processed Ticketmaster events.

### 10. Write Ticketmaster Events to Google Doc
Writes the event information into the Google Doc.

---

# 7. How Duplicate Events Are Prevented

Both workflows use a unique event ID:

```text
source_event_id
```

### New event

If the event ID does not exist in the Google Sheet:

```text
NEW
```

The event is added.

### Updated event

If the event ID already exists but some information has changed:

```text
UPDATED
```

The event is updated.

### Unchanged event

If the event already exists and nothing has changed:

```text
UNCHANGED
```

The event is skipped.

This prevents the same event from being repeatedly processed.

---

# 8. What Happens on the First Run?

If the Google Sheet is empty, all fetched events are treated as new events.

```text
Get events
    ↓
Sheet is empty
    ↓
All events = NEW
    ↓
Save events
    ↓
Create Google Doc
```

On later runs, only new or changed events are processed.

---

# 9. What Happens If Nothing Has Changed?

If all events are already present and their information is unchanged:

```text
Get events
    ↓
Compare with Google Sheet
    ↓
No new or changed events
    ↓
Stop
```

No unnecessary updates or documents are generated for unchanged events.

---

# 10. Google Sheets Used

There are two separate Google Sheets:

### Uitagenda Sheet

Stores events collected from Uitagenda Rotterdam.

### Ticketmaster Sheet

Stores events collected from Ticketmaster.

The Google account connected to n8n must have access to both Sheets.

---

# 11. Google Docs

Both workflows create a Google Doc containing only:

- New events
- Updated events

Unchanged events are not included.

The Google Docs are used as review/import documents.

---

# 12. What the Client Needs to Share

### Ticketmaster

- **Consumer Key** from the Ticketmaster Developer Portal
- Confirmation that Ticketmaster API access is active

### Google

- Google account with access to the required event Sheets
- Permission to create/edit Google Docs
- Access to both the Uitagenda and Ticketmaster Sheets

### Uitagenda

- No API key is required

---

# 13. Security

Please do not add credentials to this README.

Do not publicly share:

- Ticketmaster Consumer Key
- Ticketmaster Consumer Secret
- Google OAuth credentials
- Google access/refresh tokens
- n8n credential secrets
- Private Google Sheet or Google Doc credentials

Credentials should be entered securely in n8n.

---

# 14. Quick Summary

| Workflow | Data Source | API Key Required? | Google Sheet | Google Doc |
|---|---|---|---|---|
| Uitagenda | Uitagenda Rotterdam | No | Yes | Yes |
| Ticketmaster | Ticketmaster | **Yes – Consumer Key** | Yes | Yes |

### Most important point

```text
Ticketmaster Consumer Key     → USE THIS
Ticketmaster Consumer Secret  → DO NOT USE
```

```text
Google Account → Must have access to the required Sheets and Docs
```

```text
Uitagenda API Key → Not required
```

Once these access requirements are configured, both workflows can run automatically and keep the event data up to date while avoiding unnecessary duplicate processing.
