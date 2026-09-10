# KYNCT Instagram Event Import Workflow

## Client Documentation

## Overview

The **KYNCT Instagram Event Import Workflow** automatically collects Instagram posts through an RSS feed, extracts the post caption, image, permalink, and timestamp, uses Gemini AI to analyze both the caption and image, identifies whether the post represents a real-world event, filters out non-event posts, checks event posts against previously processed Instagram source records in Google Sheets, and prepares new events for Google Docs.

The workflow is designed to reduce manual event-information collection while keeping the extracted information structured and reviewable.

> **Important:** The workflow uses the Instagram RSS feed as the source. The RSS feed is provided through RSS.app. Google Sheets is used as the duplicate/reference store, while Google Docs is used for the event-import output.

---

# Workflow at a Glance

**Instagram Post**  
↓  
**RSS Trigger Node**  
↓  
**Extract Instagram Post Data**  
↓  
**AI Event & Image Analysis**  
↓  
**Parse AI Response Node**  
↓  
**IF - Is Event?**

**False → Not An Event (Discard)**

**True → Check Duplicate**  
↓  
**Filter New Instagram Events**  
↓  
**Save Event**  
↓  
**Prepare Google Doc Content & Images**  
↓  
**Create New Google Doc**  
↓  
**Write Event Content to Google Doc**  
↓  
**Insert Cover Image**  
↓  
**Import Completion Status**

---

# Node-by-Node Explanation

## 1. RSS Trigger Node

**Purpose:** Starts the workflow when new content is available from the configured Instagram RSS feed.

The node uses an RSS.app feed URL and is configured to poll the feed every **6 hours**.

The RSS feed is the starting point of the workflow and passes the available Instagram post data to the extraction stage.

---

## 2. Extract Instagram Post Data

**Purpose:** Converts the RSS item into a consistent structure for the AI stage.

The node extracts:

- **Caption**
- **Image URL**
- **Instagram permalink**
- **Post timestamp**
- **Source**

### Image extraction

The workflow supports images provided through different RSS structures. It can look for:

- An image inside RSS HTML content
- `enclosure.url`
- `media:content`
- `media:thumbnail`

This is important because event information may be present in the image even when the caption contains little or no useful information.

---

## 3. AI Event & Image Analysis

**Purpose:** Analyzes the Instagram caption and image together to determine whether the post represents a real-world event and extract the available event information.

The Gemini image-analysis node receives:

- Instagram caption
- Image URL
- Post permalink
- Post timestamp

The AI is explicitly instructed to use **only information visible in the provided caption or image**.

It must not:

- Invent missing information
- Guess dates or times
- Guess locations
- Create unsupported ticket links
- Estimate attendee limits
- Use outside knowledge to fill missing details

### Event detection

A post is classified as an event only when there is sufficient evidence that it announces, promotes, or provides information about an actual real-world event.

If the post does not provide sufficient evidence of an event, the AI returns:

```json
{
  "is_event": false,
  "event_rejection_reason": "..."
}
```

If an event clearly exists but information is missing, the AI keeps:

```json
"is_event": true
```

and records the missing information in `confidence_flags`.

---

# Event Information Extracted by AI

For event posts, the workflow can extract:

| Field | Description |
|---|---|
| `title` | Event name |
| `category` | KYNCT event category |
| `description` | Concise event description |
| `event_purpose` | Purpose of the event when supported |
| `social_impact` | Positive, Neutral, Potentially negative, or Unknown |
| `start_date` | Event start date |
| `start_time` | Event start time |
| `end_date` | Event end date |
| `end_time` | Event end time |
| `location_name` | Event location |
| `location_address` | Exact address when available |
| `max_attendees` | Capacity when explicitly stated |
| `source_url` | Original Instagram permalink |
| `source_id` | Instagram post/reel/tv identifier |
| `ticket_link` | Ticket URL when explicitly provided |
| `cover_image` | Original event image |
| `confidence_flags` | Missing, unclear, or conflicting information |
| `safety_policy_flags` | Clearly supported safety concerns |
| `safety_risk_level` | None, Low, Medium, or High |
| `safety_review_reason` | Explanation of the safety classification |

---

# 4. Parse AI Response Node

**Purpose:** Converts the Gemini response into structured JSON that can be used by the remaining workflow.

The node:

1. Reads the AI response.
2. Removes Markdown JSON code fences if present.
3. Attempts to parse the response as JSON.
4. Records parsing information if the response cannot be parsed.

This ensures that the following nodes receive structured event data instead of raw AI text.

---

# 5. IF - Is Event?

**Purpose:** Separates event posts from non-event posts.

The node checks:

```text
is_event === true
```

### True branch

Event posts continue to:

**Check Duplicate → Filter New Instagram Events → Save Event → Google Docs processing**

### False branch

Non-event posts go to:

**Not An Event (Discard)**

No Google Sheets record is created and no Google Doc is generated for these posts.

---

# 6. Not An Event (Discard)

**Purpose:** Acts as the dead-end path for posts that are not identified as events.

Examples include:

- Regular photos
- Generic updates
- Congratulations posts
- Birthday posts without an actual event
- Merchandise promotions
- General information
- Recruitment/application announcements without a real-world gathering
- Posts where there is insufficient evidence that an event exists

Nothing is written to Google Sheets or Google Docs from this branch.

---

# 7. Check Duplicate

**Purpose:** Reads the Google Sheets reference data used to identify posts that have already been processed.

Google Sheets acts as the workflow's duplicate/reference store.

The workflow uses:

- `source_url`
- `source_id`

to build a duplicate key:

```text
source_url || source_id
```

The lookup is executed once and returns the existing stored records used by the filtering stage.

---

# 8. Filter New Instagram Events

**Purpose:** Keeps only event posts that have not already been recorded.

For each event post, the node:

1. Gets the Instagram permalink.
2. Extracts the Instagram source ID from URLs such as:
   - `/p/...`
   - `/reel/...`
   - `/tv/...`
3. Builds the same `source_url || source_id` key used for existing records.
4. Removes posts whose key already exists in Google Sheets.
5. Passes only new events to the next stage.

If a post is already present, it is excluded from the new-event path.

---

# 9. Save Event

**Purpose:** Records the new event's Instagram source information in Google Sheets.

The workflow appends:

- `source_url`
- `source_id`

to the configured Google Sheet.

This record becomes the reference used by future workflow runs to prevent the same Instagram post from being imported again.

> **Note:** The Google Sheet is primarily used as a processed-post/reference store. The event's detailed content is prepared for Google Docs.

---

# 10. Prepare Google Doc Content & Images

**Purpose:** Builds the content and image-insertion requests required for the Google Docs output.

The node collects all valid event records and creates:

- Document title
- Import date
- Import time
- Event count
- Structured event sections
- Cover-image insertion requests

The generated document title follows this format:

```text
KYNCT Event Import - DD/MM/YYYY HH:MM
```

### Event content structure

Each event is formatted with sections such as:

- Event number
- Cover image
- Title
- Category
- Date & Time
- Location Name
- Location Address
- Description
- Social Impact
- Ticket Link
- Source URL
- Source ID
- Confidence Flags
- Maximum Attendees
- Safety Policy Flags
- Safety Risk Level
- Safety Review Reason

Missing values are displayed as **Not specified** in the generated document.

### Cover image handling

The workflow finds the position of each event's `TITLE` section and creates Google Docs `insertInlineImage` requests using the event's `cover_image` URL.

Multiple images are sorted from the bottom of the document upward before insertion so that earlier document indexes are not incorrectly shifted by later image insertions.

---

# 11. Create New Google Doc

**Purpose:** Creates the Google Doc that will contain the imported event information.

The Google Docs API is called through an HTTP Request node.

The document title is taken from:

```text
$json.doc_title
```

The Google OAuth credential is used for authentication.

---

# 12. Write Event Content to Google Doc

**Purpose:** Writes the prepared event content into the newly created Google Doc.

The node sends a Google Docs `batchUpdate` request and inserts:

```text
$json.doc_body
```

at the end of the document.

The document therefore receives the structured event information before the cover images are inserted.

---

# 13. Insert Cover Image

**Purpose:** Inserts the event cover images into the Google Doc.

The node sends another Google Docs `batchUpdate` request using the image requests created by:

**Prepare Google Doc Content & Images**

Each image is inserted as an inline image using its source URL.

---

# 14. Import Completion Status

**Purpose:** Provides a final summary of the workflow run.

The node calculates:

- Total event candidates
- Number of new events
- Number of duplicate events skipped
- Whether a document was created
- Overall import message

Example status:

```text
Import completed successfully.
2 new event(s) added and 1 duplicate(s) skipped.
```

The final status is returned as structured JSON.

---

# Event Classification

The AI distinguishes between genuine event posts and ordinary Instagram content.

## Examples of Events

Possible event types include:

- Concerts
- Performances
- Festivals
- Workshops
- Lectures
- Sports activities
- Student meetups
- Networking gatherings
- Parties
- Recruitment gatherings

A recruitment-related post can be treated as an event when it clearly describes a real-world gathering, such as recruitment drinks, a recruitment evening, or a recruitment meetup.

## Examples of Non-Events

Examples include:

- Regular photos
- Generic updates
- Congratulations posts
- Birthday posts without an event
- Merchandise promotions
- General information
- Recruitment/application announcements without a real-world gathering
- Posts where there is insufficient evidence that an actual event exists

---

# Category Structure

The workflow uses exactly six supported categories:

- **social** — student meetups, networking, dinners, BBQs, association gatherings
- **cultural** — cultural celebrations, exhibitions, performances
- **sports** — tournaments, matches, fitness and sporting activities
- **study** — lectures, workshops, seminars, guest talks, academic activities
- **party** — parties, club/pub nights, borrels, DJ nights, nightlife activities
- **other** — genuine events that do not clearly fit another category

No additional category values are created.

---

# Date and Time Handling

The workflow expects:

### Date

```text
DD/MM/YYYY
```

### Time

```text
HH:MM
```

using the 24-hour format.

The AI can obtain date and time information from either:

- The Instagram caption
- The event image

This allows the workflow to handle event posters where most of the useful information is contained in the image.

If information is missing, the workflow does not intentionally invent it.

---

# Image Handling

The event image is an important input to the AI analysis.

Information that may be visible only in the image includes:

- Event title
- Date
- Time
- Location
- Address
- Other event details

The same source image can then be used as the event cover image in Google Docs.

This allows the workflow to process cases where the caption contains little information but the event poster contains the relevant details.

---

# Confidence Flags

Confidence flags identify information that is missing, unclear, or conflicting.

Examples include:

```text
missing_title
missing_start_date
missing_start_time
missing_location
conflicting_date
conflicting_time
conflicting_location
unclear_event_type
unclear_purpose
safety_review_required
```

A genuine event is not automatically rejected because one or more fields are missing.

Instead, missing required information is represented as `null` in the AI output and the relevant confidence flag can be added.

---

# Social Impact and Safety Review

The AI also evaluates the available source content for social impact and clearly supported safety concerns.

## Social Impact

Allowed values:

- **Positive**
- **Neutral**
- **Potentially negative**
- **Unknown**

The classification must be based only on evidence in the caption or image.

Political, religious, cultural, activist, or controversial content is **not automatically treated as negative**.

## Safety Flags

Allowed safety flags include:

- `violence`
- `threats_or_intimidation`
- `hate_or_discrimination`
- `extremism`
- `illegal_activity`
- `dangerous_activity`
- `other_safety_concern`

If no clearly supported concern exists:

```json
"safety_policy_flags": []
```

Safety risk levels are:

- **None**
- **Low**
- **Medium**
- **High**

The workflow is designed to flag only concerns supported by the source material and not infer safety issues from viewpoints alone.

---

# Duplicate Protection

The duplicate-protection process is:

1. An Instagram post enters through the RSS feed.
2. The post permalink is extracted.
3. The Instagram source ID is extracted from the permalink.
4. Existing source records are read from Google Sheets.
5. A duplicate key is created using:
   ```text
   source_url || source_id
   ```
6. If the key already exists, the event is excluded from the new-event path.
7. If the key does not exist, the event is saved to Google Sheets.
8. The new event continues to Google Docs processing.

This prevents repeated imports of the same Instagram source.

---

# Google Sheets Reference Record

Google Sheets is used as the processed-post/reference store.

The workflow records:

| Field | Purpose |
|---|---|
| `source_url` | Original Instagram post URL |
| `source_id` | Instagram post/reel/tv identifier |

These values are used together to identify whether a source has already been processed.

---

# Google Docs Output

For new event posts, the workflow creates a Google Doc containing a structured import report.

The document includes:

- Import date
- Import time
- Total events
- Event title
- Category
- Date and time
- Location
- Location address
- Description
- Social impact
- Ticket link
- Source URL
- Source ID
- Confidence flags
- Maximum attendees
- Safety policy flags
- Safety risk level
- Safety review reason
- Event cover image

The output is intended to make the extracted information easy to review.

---

# End-to-End Example

## Example 1 — Event Post

**Instagram Post:**

> Recruitment Drinks  
> 7 September  
> 5:00 PM  
> Smitse, Campus Woudstein

### Workflow behavior

1. RSS Trigger Node receives the post.
2. Extract Instagram Post Data extracts the caption, image, permalink, timestamp, and source.
3. Gemini analyzes the caption and image.
4. The AI identifies the post as a real-world recruitment gathering.
5. Parse AI Response Node converts the response into JSON.
6. IF - Is Event? sends the post through the event branch.
7. Check Duplicate checks the Google Sheets reference.
8. Filter New Instagram Events determines whether the source is new.
9. Save Event records the source URL and source ID.
10. Prepare Google Doc Content & Images formats the event.
11. Create New Google Doc creates the import document.
12. Write Event Content to Google Doc writes the event details.
13. Insert Cover Image adds the event image.
14. Import Completion Status reports the result.

---

## Example 2 — Non-Event Post

**Instagram Post:**

> Beautiful sunset today.

### Workflow behavior

1. RSS Trigger Node receives the post.
2. Extract Instagram Post Data extracts the available source data.
3. Gemini analyzes the caption and image.
4. The AI determines there is insufficient evidence of a real-world event.
5. Parse AI Response Node parses the non-event response.
6. IF - Is Event? sends the post to the false branch.
7. Not An Event (Discard) ends processing.

The post is not saved to the event reference sheet and is not added to Google Docs.

---

# Important Operational Notes

- The RSS trigger uses the configured RSS.app feed.
- The current RSS trigger is configured to check the feed every **6 hours**.
- The RSS feed is a third-party mechanism for obtaining Instagram content.
- Gemini analyzes both the caption and available image.
- Image information is important when event details are contained in an event poster.
- The AI is instructed not to invent missing information.
- Genuine events can continue even when required event fields are missing.
- Missing or conflicting information is represented through `null` values and confidence flags.
- Non-event posts are discarded before duplicate checking and document generation.
- Google Sheets stores processed source references.
- Google Docs stores the formatted event-import output.
- Cover images are inserted into the Google Doc using the extracted image URL.
- The final status reports the number of event candidates, new events, duplicates skipped, and document creation status.
