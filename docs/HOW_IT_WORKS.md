# How the personal hourly scraper works

## Processing flow

1. **Schedule Trigger → Init Run:** starts hourly in Asia/Kolkata and records the execution start time. No shared workflow static data is used.
2. **Fetch Job Roles → Filter Active Roles → Prepare Searches:** reads the config table, keeps `active=true`, normalizes legacy plus-separated keywords, and creates the first page of each search. The last-hour window is enforced in the workflow, not configurable through CSV.
3. **Loop Searches → Wait 1s → Guest API Search:** processes one search at a time. Sends URL-encoded plain-space keywords and location, `f_TPR=r3600`, `sortBy=DD`, and the current offset. No unrelated geographic ID is sent.
4. **Parse Search Page:** extracts job cards, counts all raw cards for pagination, normalizes full-stack/fullstack and front-end/frontend title variants, validates the location and age, and removes repeated IDs within the page. All requested title words must appear as whole normalized words; seniority prefixes are allowed.
5. **Has Jobs?:** a page with no accepted jobs proceeds directly to pagination. It does not imply that the raw search results are exhausted.
6. **Expand Jobs → If row does not exist → Has New Jobs?:** checks each candidate's `Job_ID` against the output table. `alwaysOutputData` preserves a control item when all jobs are duplicates; the following IF keeps that empty item out of detail requests and continues pagination.
7. **Guest API Details → Parse Detail Page → Insert to Data Table:** fetches details only for unseen jobs, parses every response with its linked input job, then inserts all new jobs. Description text is limited to 3,000 characters. Hiring-person fields are optional. Names are not guessed from profile URL slugs.
8. **Next Page → More Pages?:** increments the offset by the number of raw cards returned, even if none matched. A nonempty page leads to another request; an empty page advances the outer search loop. Repeated page signatures and the 20-page cap cause visible errors.
9. **Run Complete:** the outer loop has completed all searches.

## Posting-age rules

- Full timestamps with an explicit UTC offset or `Z` are compared to the parser's current time. Accepted ages are `0 <= age < 3,600,000 ms`.
- For date-only timestamps, English seconds/minutes-ago labels or “just now”/“moments ago” provide the age estimate.
- “1 hour ago,” older labels, unknown labels, and future timestamps are rejected.
- Relative labels are approximate. `Posted_Date` preserves the source value; `Posted_Ago` preserves the label. Neither is presented as a fabricated exact timestamp.
- The endpoint's time filter is an additional search constraint, not the sole age check.

## Output columns

All columns are strings. Column names are case-sensitive.

| Column | Meaning |
| --- | --- |
| Job_Title | Listing title |
| Job_ID | LinkedIn numeric ID stored as a string; database duplicate key |
| Job_Link | Canonical `https://www.linkedin.com/jobs/view/<id>` |
| Posted_Date | Source datetime, often date-only |
| Job_Owner | Company name |
| Source | LinkedIn |
| Fetched_At | Search-page processing time in ISO format |
| Snippet | Location and relative posting label |
| Dedup_Hash | Stable `linkedin:<id>` identifier; retained field name for compatibility |
| Job_Description | Plain text, up to 3,000 characters |
| Hiring_Person | Profile anchor text, if available |
| Seniority_Level | Source job criterion |
| Employment_Type | Source job criterion |
| Job_Function | Source job criterion |
| Industries | Source job criterion |
| Search_Keyword | First search that saved this job |
| hiring_person_url | Profile link, if available |
| Job_Location | Location from the card |
| Posted_Ago | Posting-age label from the card |

## Request volume and failures

With 18 active config rows, an execution with no results needs 18 search requests and no detail requests. Each nonempty search adds pages until empty, up to 20 pages total per search. Each unseen accepted job adds one detail request. HTTP failures retry up to three times with two seconds between attempts. Search requests and detail requests are paced independently at one second.

A nonempty unrecognized response, block page, missing detail content, repeated page, pagination cap, or exhausted HTTP retries stops the run as a failure. Already inserted jobs remain saved; re-running skips them. Inspect the failed node in n8n. A missing individual detail can stop later searches, so monitor execution failures. There is no catch-up of jobs that age out during an outage.

The duplicate check uses persisted table rows, not an execution-local cache. It handles overlap between pages, searches, and successive runs. It is not safe against simultaneous executions racing to insert the same ID. Keep one active writer and do not manually execute while a scheduled run is active.

## Configuration and future extensions

Change the config table to enable/disable searches. Chennai is intentionally included alongside the statewide search for coverage, with persisted IDs removing overlap. Bengaluru is the search spelling; both Bengaluru and Bangalore pass the card filter. No remote-only or experience restriction is applied.

For future Telegram alerts, attach notification handling after successful insertion. Those items represent newly saved jobs. A reliable delivery/retry design should track notification status separately so message failures do not lose alerts after the job has already been saved.

Use the setup and validation checklist in [README.md](../README.md). No notification destination or credentials are bundled.
