# Datadog Best Practices for Log Explorer and Dashboard creation (Users)

What this error means “Your current query request volume is exceeding the concurrent query capacity of the selected Flex Compute tier…”

Flex Logs compute has hard limits on:
 No of concurrent queries, and
 max logs scanned per query. 
When the concurrent query limit is reached, Datadog will throttle/slow down new queries and may ask you to retry later; if no “slot” becomes available, the query won’t run and you’ll see the error.

Common causes we see

Many users querying at the same time (especially during incidents).
Dashboards hammering logs repeatedly (auto-refresh + short time windows = frequent refresh). Datadog itself notes dashboards can be a top source of slowdowns. 
Log Explorer “Live” usage and broad searches: Live Tail is intended for near real-time use and should be scoped down when volume is high

Best practices to avoid flex logs throttling (users)

✅ Do's

Always filter by service:<name> and env:<env> (and add status:error etc. when relevant)
Start with Past 15 minutes or Past 1 hour, then expand only if needed
Stop continuous updates
Pause live updates: when not needed (click the Pause(||) button control in the Log Explorer UI) so the page doesn’t keep issuing queries in the background. (UI behavior visible in Log Explorer; see screenshot section below.)  so the page doesn’t keep refreshing and consuming Flex compute in the background.

<img width="987" height="211" alt="image" src="https://github.com/user-attachments/assets/f04d1fbd-2b1d-44d6-ba9b-275aab71d732" />



❌ Avoid

Leaving Log Explorer open in Live/Live Tail when you are not actively debugging
Avoid Live Tail unless needed: Live Tail is for near real-time streaming during active investigations—please don’t keep it open when not required.
Running broad searches across large time windows (increases scanned volume and compute usage)
Keeping multiple tabs/dashboards open that continuously refresh and query Flex Logs (increases concurrent query usage)

<img width="885" height="201" alt="image" src="https://github.com/user-attachments/assets/0799078e-84e2-4e28-8436-76fc9164a796" />

 


Dashboard Best Practices (when dashboards query logs)

Dashboards can be a major source of slowdowns, and Datadog recommends improving dashboards experiencing slowdowns by pausing auto-refresh and optimizing widgets.

1) Pause auto-refresh for dashboards that don’t require live updates
Datadog provides a dashboard setting “Pause Auto-Refresh” that “optimizes compute usage and reduces background activity” and applies to all users viewing that dashboard.

<img width="1598" height="485" alt="image" src="https://github.com/user-attachments/assets/534c17d6-72e2-4f9b-b096-9c22f5b0b4a9" />



2) Use collapsed widget groups for heavy sections
Datadog recommends organizing widgets into groups and keeping them collapsed until needed—collapsed groups do not load when the dashboard is opened.

<img width="1585" height="774" alt="image" src="https://github.com/user-attachments/assets/ae54869b-0a66-4893-b596-285ae30c8520" />


