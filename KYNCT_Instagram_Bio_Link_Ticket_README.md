# KYNCT Instagram Bio Link Import Workflow

## Overview

This workflow monitors Instagram posts through RSS, extracts post information, checks the Instagram profile bio for ticket links, analyzes the post with AI, filters duplicates, and prepares the event information in Google Docs.

The ticket-link process is dynamic and does not require account-specific hardcoding.

---

## Workflow

```text
Instagram RSS Trigger
        ↓
Extract Instagram Post Data
        ↓
Fetch Instagram Profile
        ↓
Extract Instagram Bio Links
        ↓
IF - Bio Links Found
   ├── YES → Expand Bio Links
   │              ↓
   │       Fetch Bio Link Page
   │              ↓
   │       Find Ticket URL
   │              ↓
   └── NO ─────────────────┐
                           ↓
                AI Event & Image Analysis
                           ↓
                  Parse AI Response
                           ↓
                     IF - Is Event?
                           ↓
                  Check Existing Events
                           ↓
                Filter New Instagram Events
                           ↓
                       Save Event
                           ↓
             Prepare Google Doc Content & Images
                           ↓
                  Create New Google Doc
                           ↓
             Write Event Content to Google Doc
                           ↓
                    Insert Cover Image
                           ↓
                Import Completion Status
```

## 1. Instagram RSS Trigger

The workflow starts with the configured RSS feed and receives Instagram posts periodically.

The current RSS trigger is configured to check the feed every 6 hours.

## 2. Extract Instagram Post Data

This node extracts:

- Caption
- Image URL
- Instagram post permalink
- Timestamp
- Instagram username
- Source

The original post information is retained for later processing.

## 3. Fetch Instagram Profile

Using the Instagram username, the workflow fetches the profile page:

```text
https://www.instagram.com/<instagram_username>/
```

This step is required because the ticket link may be present in the profile bio rather than in the Instagram post itself.

## 4. Extract Instagram Bio Links

The profile HTML is parsed to extract the available `bio_links`.

Each link can contain:

- Title
- URL
- Link type
- Link ID

Example:

```text
Website → https://example.com
Tickets → https://buytickets.at/example/12345
WhatsApp → https://chat.whatsapp.com/...
```

All available bio links are collected dynamically.

No Instagram account is hardcoded.

## 5. IF - Bio Links Found

The workflow checks whether the profile contains bio links.

### Bio links found

The links are sent through:

```text
Expand Bio Links
        ↓
Fetch Bio Link Page
        ↓
Find Ticket URL
```

### No bio links

The workflow skips ticket-link processing and sends the original post directly to the AI node.

In this case:

```text
ticket_url = null
```

## 6. Expand Bio Links

When multiple links are present in the bio, this node temporarily expands the `bio_links` array so each URL can be fetched.

Example:

```text
Website
Tickets
WhatsApp
```

Each URL can then be processed by the HTTP request node.

This is an internal processing step. The final result is restored to one record per original Instagram post.

## 7. Fetch Bio Link Page

Each bio URL is fetched using an HTTP request.

For example:

```text
Instagram Bio
      ↓
Website → https://example.com
      ↓
Fetch Bio Link Page
      ↓
Website HTML
```

The request uses a browser-style User-Agent and is configured so that a failed website request does not stop the complete workflow.

# 8. How Ticket Links Are Found

The `Find Ticket URL` node uses two approaches.

## Method 1 — Direct Ticket Link in Instagram Bio

First, the workflow checks the links directly extracted from the Instagram bio.

It checks both:

- Bio link URL
- Bio link title

The workflow looks for ticket-related terms such as:

```text
ticket
tickets
buyticket
buy-tickets
eventbrite
tickettailor
ticketmaster
weeztix
eventix
buytickets.at
```

Example:

```text
Instagram Bio
      ↓
Tickets
      ↓
https://buytickets.at/example/12345
      ↓
Direct ticket URL detected
```

If a direct ticket URL is found, that URL is stored as `ticket_url`.

The workflow does not need to successfully open the ticket website to keep a URL that was already present directly in the Instagram bio.

## Method 2 — Ticket Link Inside the Bio Website

If no direct ticket URL is found in the bio, the workflow checks the HTML returned from the bio website.

It scans HTML anchor tags such as:

```html
<a href="https://example.com/tickets">
    Buy Tickets
</a>
```

Both the following are checked:

1. The `href` URL
2. The visible link text

Example:

```text
Instagram Bio
      ↓
Website → https://example.com
      ↓
Fetch Bio Link Page
      ↓
Scan HTML
      ↓
"Buy Tickets"
      ↓
https://example.com/tickets
      ↓
Ticket URL detected
```

This means a general website link in the Instagram bio can still lead to a detected ticket URL.

## Ticket Detection Result

If a ticket URL is found:

```json
{
  "ticket_url": "https://example.com/tickets"
}
```

If no ticket URL is found:

```json
{
  "ticket_url": null
}
```

The detected value is then passed to the AI Event & Image Analysis node.

The AI is instructed to preserve a valid ticket URL exactly as supplied and return `null` when no ticket URL is available.

# 9. AI Event & Image Analysis

The AI receives the original Instagram post data:

- Caption
- Image
- Post permalink
- Timestamp
- Detected ticket URL

It determines whether the post represents an event and extracts structured event information.

The AI is instructed to use only information available in the Instagram caption and image and not invent missing details.

## 10. Parse AI Response

The AI response is parsed into JSON.

If the response cannot be parsed, the workflow records the parsing error rather than stopping the complete workflow.

## 11. Event and Duplicate Filtering

The workflow checks whether the AI identified the post as an event.

Non-events are discarded.

For valid events, the workflow checks existing records using the Instagram source URL and source ID to avoid saving the same event post multiple times.

## 12. Save Event

New events are saved to the configured Google Sheet.

## 13. Google Docs Output

The workflow prepares a Google Doc containing the extracted events.

The document includes information such as:

- Event title
- Category
- Date and time
- Location
- Description
- Ticket link
- Source URL
- Source ID
- Confidence flags
- Maximum attendees
- Safety review information
- Cover image

# Ticket Detection Summary

```text
Instagram Post
      ↓
Fetch Instagram Profile
      ↓
Extract Bio Links
      ↓
Are Bio Links Available?
      │
      ├── NO
      │    ↓
      │  ticket_url = null
      │
      └── YES
           ↓
      Check Direct Bio URLs
           ↓
      Ticket URL Found?
           │
           ├── YES → Use direct ticket URL
           │
           └── NO
                ↓
          Fetch Bio Website
                ↓
          Scan HTML Links
                ↓
          Ticket URL Found?
             │
             ├── YES → Use detected URL
             │
             └── NO → ticket_url = null
```

## Important Limitation

Ticket detection depends on the HTML/content returned by the linked website.

Some websites may:

- Render links only through JavaScript
- Block automated HTTP requests
- Use Cloudflare or other bot protection
- Return an error page
- Hide links behind client-side interactions

In these cases, a ticket URL may not be discoverable through a normal HTTP request.

However, if the ticket URL is already directly present in the Instagram bio, the workflow can preserve that URL without needing to successfully load the ticket website.

## Key Benefits

- Dynamically extracts Instagram bio links
- Supports multiple bio links
- No account-specific ticket-link hardcoding
- Detects direct ticket URLs from Instagram bios
- Searches linked website HTML for ticket URLs
- Returns `null` when no ticket URL is available
- Keeps the final result associated with the original Instagram post
- Passes the detected ticket URL to AI as metadata
- Prevents duplicate event records
- Produces a reviewable Google Doc
